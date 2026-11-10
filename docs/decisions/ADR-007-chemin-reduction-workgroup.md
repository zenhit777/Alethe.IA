# ADR-007 — Chemin de référence de la réduction dans la workgroup

**Statut** : accepté (10 novembre 2026).
**Régime** : décision d'architecture de kernel ; définit le chemin de référence **par nature de la réduction**, NEVER un chemin unique pour toutes les réductions.
**Preuves lues à la source** : spécification W3C WebGPU / WGSL (atomics `i32` / `u32` en workgroup memory, extension d'atomics `f32`, `subgroups`, `subgroupMinSize` / `subgroupMaxSize`), 10 novembre 2026.

## Contexte

Une réduction (somme de produits scalaires, accumulation de logits, agrégation de contributions d'attention, réduction d'entropie) est le point où la **reproductibilité** se joue (§6.3, R-28). L'ordre d'accumulation flottante dépend de l'ordre d'exécution des threads ; un ordre dépendant de la planification des threads n'est **pas** bit-identique, NEVER une tolérance suffisante pour la reproductibilité d'un jalon.

Trois mécanismes existent, de natures différentes :

1. **Atomics entiers** (`i32` / `u32`) en workgroup memory : l'arithmétique entière est exacte pour les bornes de tampon, le comptage, les drapeaux, les index de file, les séquences de commandes. La détermination est garantie.
2. **Atomics `f32`** (`atomicAdd`, `atomicMin`, `atomicMax` sur `f32`) : l'ordre d'accumulation dépend de la concurrence. La sortie **n'est pas** bit-identique d'une exécution à l'autre.
3. **Réduction par subgroup** (fonctions collectives, `enable subgroups;`) : dépend de la disponibilité de la fonctionnalité sur le device, expose `subgroupMinSize` / `subgroupMaxSize` lus (§4.5).

Confondre ces trois mécanismes sous un seul terme « atomics » est l'erreur d'origine : elle rend un chemin flottant non déterministe **base de référence**, et elle rend une fonctionnalité optionnelle (subgroup) chemin de base.

## Décision

Le chemin de référence est fixé **par nature de la réduction** (§4.5 de `_Alethe.IA.md`) :

- **Réductions flottantes** → chemin de référence = **réduction hiérarchique déterministe** : agrégation par sous-groupes, puis accumulation en workgroup memory selon un **ordre d'accumulation fixe et consigné par kernel**. MUST consigner, par kernel, l'ordre d'accumulation, la largeur de chaque niveau de la hiérarchie, la précision d'accumulation (`f32` au minimum). MUST les rendre dans le plan d'exécution (R-2, §4.3) et les figer pour la reproductibilité (R-28). NEVER un ordre dépendant de la planification des threads.
- **Atomics entiers** (`i32` / `u32`) → **chemin de référence** pour les compteurs, drapeaux, admissions, occupations, index de file, bornes de tampon, comptage de positions dénoisées, séquence de commandes, horodatage de pas.
- **Atomics `f32`** → **jamais le chemin de référence déterministe.** MAY être employée en option, à quatre conditions cumulatives : (1) déclaration explicite de l'emploi dans le profil du kernel, (2) explicitation de la conséquence sur la reproductibilité (la sortie n'est pas bit-identique), (3) explicitation des tolérances numériques (§6.3, R-61, R-66), (4) publication des **deux** sorties (chemin de référence + variante) dans le rapport de mesure. NEVER présenter une sortie issue d'atomics `f32` comme reproductible au sens de R-28.
- **Réduction par subgroup** → variante mesurée ; MAY être activée après comparaison mesurée (débit, stabilité numérique) sur un device exposant `subgroups`. NEVER chemin de base. MUST être le **premier niveau** de la hiérarchie, NEVER une accumulation globale concurrente.

## Conséquences

- P0 MUST comparer **par mesure** les deux chemins (workgroup memory hiérarchique et subgroup sur le device qui l'expose) ; NEVER une clôture de P0 sans la comparaison.
- §6.3 : l'identité bit-à-bit CPU ↔ WebGPU est interdite pour les réductions `f32` ; la tolérance par dtype entre backends est chiffrée.
- Le champ « ordre d'accumulation » devient un champ du contrat de tiling (ADR-004) pour la phase de recomposition.
- Un kernel qui emploie l'extension d'atomics `f32` sans les quatre conditions est **refusé** ; NEVER une option implicite.
