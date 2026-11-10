# ADR-006 — Machine de référence

**Statut** : **proposé — A TRANCHER par le mainteneur** (10 novembre 2026). Aucun code de jalon de performance ne MUST être écrit sur une machine de référence non désignée.
**Régime** : décision d'architecture de validation ; ne modifie aucune exigence, désigne le lieu de leur exercice.
**Preuves** : observation directe le 10 novembre 2026 (`nvidia-smi`, `nproc`, `free`, `wslinfo`).

## Constat de la machine de développement (observation directe, non un choix normatif)

| Composant | Valeur observée |
|---|---|
| GPU | **NVIDIA GeForce RTX 5090**, **32 607 MiB** de VRAM, compute capability **12.0** (architecture Blackwell), NVENC **non vérifié** |
| Pilote | **616.64** (lu par `nvidia-smi`) |
| CPU | **AMD Ryzen 9 7950X3D**, 16 cœurs |
| RAM système | **47,04 GiB** |
| OS | **Ubuntu 24.04.4 LTS** sous **WSL2** (`wslinfo --version` 2.7.8.0), noyau **6.18.33.1-microsoft-standard-WSL2** |
| Limites du device | MUST être **lues** au démarrage (§4.5), NEVER déduites d'une notice commerciale |
| Runtimes / outils | versions de `wgpu`, de Rust, de Zig : **non consignées** (`ADR-008`, `ADR-009`) |

**Le « 32 Go de VRAM » n'est pas une description suffisante.** Ne nomme ni le model, ni la compute capability, ni le pilote, ni la RAM, ni la couche de virtualisation. NEVER en déduire une capacité admissible pour un modèle : l'empreinte mesurée (§8.1) est la seule base d'une décision d'admission (R-32). NEVER traiter WSL2 comme équivalent d'une installation native : la couche de virtualisation modifie la latence de transfert et la disponibilité de certaines capacités (codage matériel, accès direct aux tampons) ; MUST être mesurée et consignée en P0.

## Options

| Option | Effet sur les jalons de performance | Coût | Risque |
|---|---|---|---|
| **A. Machine de développement comme unique référence** (RTX 5090 / WSL2) | un seul lieu de mesure, mesurable immédiatement | aucun | WSL2 masque des capacités natives ; les chiffres ne sont pas transportables à une installation native |
| **B. Couple (native + WSL2)** | deux machines consignées, mesures comparées, écart chiffré en P0 | deux jeux de mesures | coût de maintenance des deux environnements |
| **C. Machine native nommée, WSL2 comme environnement de développement non normatif** | les jalons de performance se lisent sur la machine native ; WSL2 sert P0 (fonctions, non débit) | nécessite une machine native | P0 ne fournit pas le seuil de débit sur la machine normative |

## Décision proposée

Option **B** : MUST consigner **deux** machines — la machine de développement (constatée ci-dessus, complétée par les limites lues et les versions de runtimes) et la machine native de référence. MUST mesurer l'écart WSL2 ↔ native (latence de transfert RAM → device, débit de tokens, disponibilité de `shader-f16`, `timestamp-query`, `subgroups`, accès direct aux tampons) et le consigner en P0. NEVER transplanter une mesure d'une machine à l'autre (§9). NEVER valider sur une machine non consignée.

MUST nommer la machine de référence **avant P0**, MUST la consigner dans chaque rapport, MUST consigner les limites **lues** (§4.5) et les versions des checkpoints exercés. NEVER une clôture de P0 sans la machine consignée.

## Conséquences

- Zone 14 de §14 : **reste A TRANCHER** jusqu'à la décision du mainteneur.
- §9 : « cible de référence » se lit comme le couple (checkpoint déclaré + machine de référence) ; NEVER une validation déclarée sur une machine non consignée.
- §8, critères d'entrée : le seuil de débit est consigné sur la machine de référence ; NEVER un seuil implicite, NEVER un seuil mesuré hors de la machine consignée.
