# ADR-010 — Modèle de concurrence

**Statut** : **proposé — A TRANCHER** (10 novembre 2026). Zone 3 de §14.
**Régime** : décision d'architecture d'exécution ; ne modifie aucune exigence, fixe le lieu de leur exercice.

## Contexte

Le moteur est Rust + Zig (§4.7), pilote des file(s) de commandes WebGPU, sert des sessions dont la charge évolue d'un pas à l'autre. Deux faits rendent la zone bloquante :

1. **Propriété des buffers** : les tampons d'état récurrent, de KV, de pool d'experts et de la mémoire polymorphe (R-20 à R-24) sont **mutables** pendant un pas ; deux threads qui en possèdent un simultanément produisent une sortie non déterministe, NEVER détectable par la tolérance numérique (§6.3).
2. **Synchronisation avec les file(s) de commandes** : les commandes sont encodées, envoyées, puis mappées en lecture (§4.5, §4.6). Un thread qui lit un tampon avant la fin de la commande **ne lit pas la sortie** : MUST être bloqué, NEVER toléré (`I-14`).

La surface de non-bloquage est déjà normative : les appels qui bloquent, la propriété des tampons, le point de synchronisation de chaque étape de pas. NEVER un choix implicite.

## Options

| Option | Effet | Risque |
|---|---|---|
| Threads Rust, verrous explicites | un seul modèle de propriété, vérifiable dans le compilateur | contention sur la phase de mappage, latence ajoutée |
| Thread de command encoder dédié, file de commandes unique | un seul propriétaire des tampons, le point de bloquage est unique et mesurable | Rust ↔ Zig MUST traverser l'ABI sur la file entière |
| **Modèle à propriétaire unique par étape de pas** (une étape = un propriétaire ; la transmission de propriété est un événement consigné) | la propriété est déclarée par étape, NEVER par thread, l'état d'avancement est la source de vérité | impose la phase d'ordonnancement (R-2, §4.3) comme point de transmission |

## Décision proposée

Option **3** (retenu comme proposition) : la **propriété d'un tampon est attachée à une étape de pas**, NEVER à un thread. MUST déclarer, pour chaque étape de pas, le propriétaire et le point de transmission. MUST bloquer tout consommateur non prêt, NEVER lire un tampon non prêt. MUST faire échouer l'encodage si une commande est envoyée sur un tampon non prêt, NEVER laisser un comportement non défini (`I-14`).

MUST consigner, avant P0 : threading Rust, threads Zig, **propriété des buffers par étape**, synchronisation de la file de commandes, **liste des points de bloquage** (mappage, lecture de tampons de sortie, transmission de propriété, admission). NEVER un modèle de concurrence non consigné et une exécution mixte en cours.

MUST mesurer la latence ajoutée par la synchronisation (débit de tokens, coût de la phase d'ordonnancement) et la consigner dans le profil d'exécution (R-2, §4.3), NEVER la déclarer sans mesure.

## Conséquences

- Zone 3 de §14 : **reste A TRANCHER** jusqu'à la décision du mainteneur ; cette ADR consigne l'obligation de consigner les points de bloquage avant P0.
- §4.3 : le profil d'exécution MUST rendre la phase d'ordonnancement comme point de transmission de propriété.
- §8.1 : P0 MUST produire la liste des points de bloquage observés (mappages, lectures, transmissions), NEVER une clôture sans cette liste.
- L'ABI (zone 2, `ADR-009`) MUST transporter la transmission de propriété, NEVER un tampon partagé sans propriétaire.
