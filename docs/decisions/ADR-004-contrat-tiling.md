# ADR-004 — Contrat générique de découpage (tiling)

**Statut** : accepté (10 novembre 2026).
**Régime** : décision d'architecture de kernel ; définit un **contrat**, NEVER un algorithme.
**Preuves lues à la source** : spécification W3C WebGPU (limites de device), 10 novembre 2026.

## Contexte

WebGPU impose `maxStorageBufferBindingSize` par liaison, `maxBufferSize`, `maxComputeWorkgroupStorageSize`, `maxComputeInvocationsPerWorkgroup`, `maxComputeWorkgroupSize`, `maxComputeWorkgroupsPerDimension`, `maxBindGroups`, `maxBindingsPerBindGroup`, `maxDynamicBindingCounts`, `minStorageBufferOffsetAlignment`. Les tenseurs d'un modèle (poids, activations, KV, état récurrent) franchissent la limite de liaison : un découpage en tuiles est obligatoire.

Un découpage **implicite** — déduit de la taille d'un buffer, réglé par la taille d'un workgroup, sans contrat écrit — produit quatre familles de bugs, toutes observables : lecture au-delà de `buffer_len`, écriture de deux tuiles sur la même région, reste (`n % tile_dim`) tronqué silencieusement, accumulation dont l'ordre dépend de la planification des threads (donc non reproductible, R-28).

Un découpage **universalisé** (une seule algorithmique imposée à tous les kernels) est également faux : un kernel de produit matriciel, un kernel d'attention bidirectionnelle sur canvas, un kernel d'état GDN et un kernel de déquantification n'ont pas la même forme de tuile, le même ordre de parcours, ni la même politique d'accumulation.

## Décision

Le découpage d'un tenseur en tuiles est un **contrat minimal partagé** par tous les kernels. MUST être respecté par chaque kernel ; chaque kernel **spécialise** sa stratégie (forme de la tuile, ordre de parcours, facteur de recouvrement, politique d'accumulation) selon sa structure de données. NEVER universaliser l'algorithmique du découpage.

Le contrat MUST préciser, pour chaque exécution, ces champs **explicites et consignés** (liste exacte et obligations : §4.5 de `_Alethe.IA.md`) : dimensions de la tuile, limites vis-à-vis des lectures de device, offsets (en éléments **et** en octets), alignements vérifiés avant l'encodage, bornes `[start, end)` de lecture, bornes `[start, end)` d'écriture, résultats partiels **typés et bornés**, recomposition / accumulation **déclarée et déterministe**, politique de traitement des restes, limites matérielles vérifiées.

MUST refuser un plan qui franchit une limite lue du device, NEVER exécuter partiellement (R-86, `I-14`). MUST séparer la phase de **tuiles** et la phase de **recomposition** dans le plan d'exécution (R-2, §4.3). MUST appliquer à la recomposition l'ordre d'accumulation hiérarchique de la réduction (§4.5, ADR-007, R-28).

MUST consigner le contrat de chaque kernel dans la fiche du kernel. NEVER un découpage implicite déduit de la taille d'un buffer.

## Options envisagées

| Option | Effet | Verdict |
|---|---|---|
| Découpage implicite par taille de workgroup | bornes non déclarées, reste non traité, accumulation non déterministe | rejeté |
| Une algorithmique de tiling unique pour tous les kernels | tuiles inadaptées aux structures (GDN, attention sur canvas, déquantification) | rejeté |
| **Contrat minimal partagé + spécialisation par kernel** | bornes, alignement, restes, accumulation contractés ; liberté de performance par kernel | **retenu** |

## Conséquences

- §4.5, §8.1 : P0 MUST exercer le contrat (nombre de tuiles, taille effective, volume de restes, coût de recomposition, fraction de `maxStorageBufferBindingSize` consommée) et le consigner. NEVER une clôture de P0 sans le contrat exercé.
- §4.3 : la phase de recomposition est un nœud du plan d'exécution, visible dans le profil.
- R-28 : la détermination de l'ordre d'accumulation est une obligation du contrat, pas une propriété de l'implémentation.
- Le contrat est la base de la vérification d'alignement avant encodage ; une violation est une erreur explicite, jamais un comportement non défini.
