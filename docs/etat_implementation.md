# État d'implémentation d'Alethe.ia

Fichier d'état **hors du document normatif** `_Alethe.IA.md` : la spécification change lentement, l'état d'implémentation change à chaque validation. MUST être mis à jour à chaque validation consignée (règle 21 de §11 de `_Alethe.IA.md`) et MUST être reporté dans le rapport de jalon.

Valeurs : `NOT_STARTED` → `IN_PROGRESS` → `IMPLEMENTED` → `VALIDATED`.

- `IMPLEMENTED` : chemin complet exercé, jamais validé sur la cible de référence.
- `VALIDATED` : chemin exercé sur la **couple (checkpoint déclaré + machine de référence)** consigné (§9 de `_Alethe.IA.md`), mesure consignée. NEVER `VALIDATED` sans mesure. NEVER une mesure transplantée d'une machine à une autre.

| Groupe d'exigences | Jalon | État | Date de validation | Artefact de mesure |
|---|---|---|---|---|
| R-1..R-6, R-7..R-11 (cycle, états de jetons) | P0-P1a | NOT_STARTED | — | — |
| R-12..R-17 (modes) | P1a-P3 | NOT_STARTED | — | — |
| R-18..R-28, R-91 (ordonnanceur d'inférence, profils de stratégie) | P1a-P2 | NOT_STARTED | — | — |
| R-29..R-42, R-84..R-89 (mémoire, ordonnanceur de ressources) | P0-P2 | NOT_STARTED | — | — |
| R-43..R-47, R-75..R-79 (persistance, sécurité) | P5 | NOT_STARTED | — | — |
| R-48..R-51, R-80..R-82, R-92..R-96 (APIs de serving, API de contrôle) | P4 | NOT_STARTED | — | — |
| R-52..R-57, R-65..R-70 (validation, QA) | P0-P3 | NOT_STARTED | — | — |
| R-58..R-64, R-83 (formats, poids, n-gram) | P0 | NOT_STARTED | — | — |
| R-90 (mode CPU-only) | P0-P1b | NOT_STARTED | — | — |
| R-71..R-74 (outillage de conversion) | FUTURE | NOT_STARTED | — | — |
| R-97 (cible dual-stream) | FUTURE | NOT_STARTED | — | — |

**Règle de consignation** : l'agent MUST reporter l'état de chaque exigence touchée dans le rapport de jalon ; MUST lier chaque `IMPLEMENTED` / `VALIDATED` à un artefact (chemin du rapport de mesure, nom de la machine de référence, revision du checkpoint exercé). NEVER déclarer un état sans l'artefact.
