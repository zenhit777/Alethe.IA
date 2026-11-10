# ADR-015 — Validation du mode CPU-only par famille de kernel (zone 20)

**Statut** : proposé (10 novembre 2026) — **non tranché**, à soumettre au mainteneur (§14, processus de décision).
**Régime** : décision de validation ; NE MODIFIE PAS les contrats du cycle d'exécution, AJOUTE l'exigence R-105, précise la zone 20.
**Preuves lues à la source** : `config.json` de `UT-IFML/dQwen3.5-0.8B-Base` (10 novembre 2026 : 18 `linear_attention` + 6 `full_attention`, `linear_conv_kernel_dim = 4`, `head_dim = 256`, `num_key_value_heads = 2`) ; `config.json` de `GSAI-ML/LLaDA-8B-Base` (32 couches d'attention pleine, `n_kv_heads = 32` — pas de GQA, **aucune couche récurrente**, contexte 4096) ; dépôt de quantization `diffuse-cpp/LLaDA-8B-Instruct-GGUF` (`llada-8b-q8_0.gguf`, révision `f80c9ca008bc4de577abe02790cb323e431702fe`) ; machine de développement constatée (RTX 5090 **32 607 MiB**, Ryzen 9 7950X3D 16 cœurs, **47,04 GiB** de RAM, WSL2).

## Contexte

La spécification exige un mode CPU-only complet (R-90, I-11, N-7) exercé sur **deux corpus distincts** (BF16 dQwen, Q8_0 LLaDA) et la zone 20 demande la granularité des kernels CPU. **Une mesure globale ne valide pas le mode CPU** : elle ne dit pas si les kernels GDN, la convolution courte, la réduction, l'attention pleine ni les états récurrents tiennent en RAM, ni où se situe la déviation numérique. **La capacité de la carte ne dit rien de la performance CPU** : 32 607 MiB de VRAM n'impliquent rien sur les 47,04 GiB de RAM système, la largeur de vector du Ryzen 9 7950X3D, ni la latence WSL2.

Faits structurants lus à la source : les deux exercices n'exercent pas les mêmes familles. dQwen exerce **GDN + convolution courte + attention pleine + état récurrent** ; LLaDA exerce **attention pleine dense sans état récurrent**, **contexte 4096**, masque `126336`, poids **GGUF Q8_0** (déquantification avec échelle `float16`, §8). Un débit global mélangé ne peut pas attribuer une dégradation à une famille.

## Options

| Option | Coverage | Coût | Risque | Verdict |
|---|---|---|---|---|
| Mesure CPU globale (débit moyen d'une session) | une ligne par exercice | faible | la dégradation n'est attribuable à aucune famille ; la granularité de la zone 20 est tranchée sans donnée | rejetée |
| **Six familles mesurées séparément × deux exercices** | déquantification, GDN, convolution courte, réductions, attention pleine, états récurrents | six lignes par exercice, un protocole de mesure par famille | la RAM peak par famille doit être instrumentée ; le protocole est à écrire | **retenue** |
| Périmètre réduit (familles « élémentaires » seules) | kernels de base uniquement | faible | les familles GDN et états récurrents, qui portent la nouveauté du checkpoint, ne sont pas validées ; contredit R-90 | rejetée |

## Décision

1. **Six familles MUST être mesurées séparément et nommées dans le rapport** (R-105) : (1) déquantification / conversion des poids vers la représentation de calcul (BF16 → FP32, Q8_0 → FP32), (2) opérations GDN (état récurrent, gating, règle Δ), (3) convolutions courtes (noyau déclaré par le checkpoint), (4) réductions (somme, softmax, argmax, top-k — ordre d'accumulation R-28), (5) attention pleine (QK^T, softmax, AV, GQA si déclarée), (6) mémoire des états récurrents (occupation par couche, par canva, par session, coût du rewind R-46).
2. **Par famille et par exercice** MUST être publié : débit mesuré (opérations/s ou octets/s), empreinte RAM peak, volume transféré, écart numérique à l'oracle FP32 avec la tolérance chiffrée (§6.3, R-66).
3. **MUST que la granularité des kernels CPU (zone 20) soit tranchée par famille**, NEVER une granularité unique pour tous les kernels ; MUST que la décision s'appuie sur la ligne mesurée de chaque famille.
4. **MUST refuser la clôture du mode CPU si une famille n'a pas de ligne mesurée**, NEVER présenter une famille comme couverte par une autre.
5. **MUST NOT conclure l'admissibilité d'un modèle en mode CPU depuis la VRAM**, NEVER extrapoler un débit CPU depuis un débit device (§9).

## Conséquences

- §4.6 (R-105), §8.1 (artefact de production), §9 (trois métriques par famille), zone 20, §16 P0.
- Le protocole de mesure par famille est un livrable de P0 ; NEVER un protocole unique partagé par les familles.
- NEVER une clôture de P0 sans les 12 lignes (6 familles × 2 exercices).
- MUST NOT écrire la granularité des kernels CPU **avant** l'acceptation de la présente note et le tranchage de la zone 20 (§0 règle 8).
