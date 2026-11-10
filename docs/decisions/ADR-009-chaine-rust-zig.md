# ADR-009 — Chaîne de compilation Rust et Zig, versions consignées

**Statut** : **proposé — A TRANCHER** (10 novembre 2026). Versions **non consignées** à ce jour.
**Régime** : décision de dépendance et d'interface ; ne tranche pas l'ABI (zone 2 de §14) ni le modèle de concurrence (zone 3).
**Preuves** : aucune version consignée le 10 novembre 2026. NEVER inventer une version.

## Contexte

Le moteur est Rust + Zig + WebGPU (§4.7). Deux faits s'appliquent :

1. L'interface Rust ↔ Zig est une **ABI** : signatures, limites de types, conventions d'erreurs, modes d'appel, transferts de buffers, format des structures partagées. Une ABI non consignée ne se vérifie pas : un décalage de struct est détectable **avant** l'encodage des commandes, jamais par l'observation d'une sortie.
2. La chaîne de compilation (compilateur Rust, compilateur Zig, `zig cc` / liaison, `bindgen`, `build.rs`) détermine le **jeu d'instructions vectorielles** disponible pour le backend CPU (§4.6). NEVER supposer une largeur de vector : MUST la mesurer (précision, largeur de vector, débit mesuré) et MUST la consigner en P0.

Fait constaté sur la machine de développement (`ADR-006`) : **AMD Ryzen 9 7950X3D**, 16 cœurs. La largeur de vector effective est une propriété **mesurée** de l'environnement de compilation, NEVER une spécification marketing du processeur.

Fait constaté sur le **runtime de référence de la modélisation** (`README.md` + `modeling_dqwen3_5.py` de `UT-IFML/dQwen3.5-0.8B-Base`, lus le 10 novembre 2026) : `transformers >= 5.13`, `torch 2.7.1+cu128` (testé par l'auteur), `flash-linear-attention 0.5.1`, GPU CUDA obligatoire, `trust_remote_code=True`, **taille de batch 1 uniquement**. MUST consigner ces versions **avec** les versions Rust / Zig dans chaque rapport de comparaison : une sortie d'Alethe.ia n'est comparable à la référence que si la référence est exercée sur le runtime qu'elle déclare. NEVER traiter un résultat de référence obtenu sur un runtime non déclaré comme un fait de validation. NEVER imiter les versions : MUST les lire.

## Décision proposée

MUST consigner, au démarrage de P0 : version du compilateur Rust, version du compilateur Zig, version de la chaîne de liaison, commit de la source des deux côtés de l'ABI, machine d'exercice (`ADR-006`). MUST les consigner dans chaque rapport ; NEVER une version isolée de sa machine.

MUST écrire l'ABI dans une **définition partagée** (une source de vérité pour les structures, NEVER deux définitions écrites à la main en miroir) ; MUST la vérifier par un exercice de transfert de buffer avant l'encodage des commandes. MUST faire échouer l'encodage si un offset, une taille ou un alignement ne correspond pas, NEVER laisser un décalage produire une sortie.

MUST consigner la **convention d'erreur** de part et d'autre de l'ABI : un code d'erreur typé, NEVER une exception traversant la frontière, NEVER un code d'erreur remplacé par un défaut.

MUST mesurer en P0, pour le backend CPU : la précision effective (largeur de mantisse utilisée en accumulation), la largeur de vector **mesurée**, le débit mesuré. NEVER une capacité du backend CPU déclarée sans mesure (§8.1).

## Options

| Option | Effet | Verdict |
|---|---|---|
| ABI définie deux fois (Rust et Zig), version non consignée | décalage détectable uniquement par la sortie | rejeté |
| **ABI dans une définition partagée, versions consignées avec la machine** | décalage détecté avant encodage, mesures transportables | **retenu** |
|ABI en C, sans structures partagées typées | perte des types du moteur, aucune vérification de bornes | rejeté |

## Conséquences

- Zone 2 (ABI) et zone 3 (modèle de concurrence) de §14 : **restent A TRANCHER** ; cette ADR consigne l'obligation de consignation, NEVER le choix de l'ABI.
- §4.6 : les capacités du backend CPU sont un fait mesuré, consigné en P0 ; NEVER une vectorisation supposée.
- §13.1 : la ligne d'outillage MUST être complétée (versions + commits + machine) ; NEVER une ligne laissée à « consigné » seul.
