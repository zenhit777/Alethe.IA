# ADR-008 — Version de wgpu et limites lues du device

**Statut** : **proposé — A TRANCHER** (10 novembre 2026). Version **non consignée** à ce jour.
**Régime** : décision de dépendance ; la lecture des limites est une exigence, la version est un fait à consigner.
**Preuves** : aucune version consignée le 10 novembre 2026. NEVER inventer une version.

## Contexte

Les noms de limites (`maxStorageBufferBindingSize`, `maxBufferSize`, `maxComputeWorkgroupStorageSize`, `maxComputeInvocationsPerWorkgroup`, `maxComputeWorkgroupSize`, `maxComputeWorkgroupsPerDimension`, `maxBindGroups`, `maxBindingsPerBindGroup`, `maxDynamicBindingCounts`, `minStorageBufferOffsetAlignment`, `maxTextureDimension*`), les noms de fonctionnalités (`shader-f16`, `timestamp-query`, `subgroups`, `texture-compression-bc`, `texture-compression-etc2`, `texture-compression-astc`, `float32-filterable`) et la sémantique du mapping des tampons (`map` asynchrone, `MAP_READ` / `MAP_WRITE`) proviennent de la **spécification W3C WebGPU**, exposée par le crate `wgpu`. La version de `wgpu` détermine quels noms sont exposés et lesquelles des limites sont **accessibles** depuis l'API Rust.

Fait : une limite lue sur un device n'est pas une limite de la spécification ; NEVER coder une valeur de limite en dur. NEVER supposer une fonctionnalité disponible : MUST la lire.

## Décision proposée

MUST consigner, au démarrage de P0 : **la version du crate `wgpu` exercée + le commit de la source** (§13.1, zone 15), MUST la consigner dans chaque rapport, MUST la relire à chaque changement de version. MUST lire les limites **au démarrage** (§4.5) et les consigner dans le rapport de capacités par device. NEVER une valeur de limite en dur dans un kernel.

MUST déclarer, par device : fonctionnalités **exposées** et fonctionnalités **absentes** ; NEVER une fonctionnalité supposée disponible. MUST refuser un plan d'exécution qui franchit une limite lue, NEVER exécuter partiellement (R-86, `I-14`).

MUST NOT choisir la version de `wgpu` sans consigner la **machine** sur laquelle elle est exercée (`ADR-006`) : une version est un couple (artefact + machine), NEVER un numéro isolé. NEVER une validation consignée sans les deux.

## Options

| Option | Effet | Verdict |
|---|---|---|
| Version épinglée par commit, consignée au démarrage de P0 | limites et noms stables, reproductibilité mesurable | **retenu** |
| Version mobile (`latest`) | noms et limites changeants, mesures non transportables | rejeté |
| Version de `wgpu` consignée sans la machine | mesure sans lieu d'exercice | rejeté |

## Conséquences

- §13.1 : la ligne `wgpu` est **complétée** (version + commit + machine), NEVER laissée à « commit consigné » seul.
- Zone 15 de §14 (versioning applicatif) : **reste A TRANCHER** jusqu'à la consignation.
- P0 MUST produire le rapport de capacités par device (limites lues, fonctionnalités exposées / absentes, taille de subgroup) ; NEVER une clôture de P0 sans ce rapport.
