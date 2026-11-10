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

## Prérequis d'entrée de P0 (état `NOT_STARTED`, non tranchés à ce jour)

MUST être **consignés avant** P0 (règle 21 de §11) ; NEVER une exécution de P0 sur un prérequis non consigné.

| Prérequis | Source normative | État | Ce qui bloque |
|---|---|---|---|
| Machine de référence désignée | §9, zone 26, `ADR-006` | **non tranché** | seuil de débit non consigné, mesures non transportables |
| Version de `wgpu` + limites **lues** du device | §4.5, zone 27, `ADR-008` | **non consignée** | aucune limite, aucune fonctionnalité exposée consignée |
| Versions de Rust / Zig + commits + ABI partagée | §4.7, zones 2 et 27, `ADR-009` | **non consignées** | décalage d'ABI détectable uniquement par la sortie |
| Modèle de concurrence + liste des points de bloquage | zone 3, `ADR-010` | **non tranché** | propriété des tampons non déclarée |
| Convention de format canonique (clé de topologie explicite) | zone 1, `ADR-011` | **non tranché** | topologie non lisible, R-106 non exercé |
| Tolérances chiffrées par format (BF16, FP16, FP32, Q8_0) | §6.3, zones 9 et 19, `ADR-002`, `ADR-003` | **non chiffrées** | P0 sans seuil de comparaison numérique |
| Contrat de découpage (tiling) écrit par kernel | §4.5, R-86, `ADR-004` | **non écrit** | bornes, restes, accumulation non contractés |
| Ordre d'accumulation consigné par kernel (réduction) | §4.5, R-28, `ADR-007` | **non consigné** | reproductibilité de la réduction non assurable |
| Empreinte mémoire mesurée et comparée à la réservation d'admission | §8.1, R-32, R-38 | **non mesurée** | admissibilité non décidable, OOM possible |
| Checkpoint de référence épinglé | zone 22, `ADR-001` | **consigné** : `UT-IFML/dQwen3.5-0.8B-Base` — `main` = `1b6ae9426071d5e372d58e309bd4ed85de39c762` (défaut), `step25000-swa` = `006d0b07fba4fffba7af33bfa875697d545a7a2f`, `step50000-swa` = `9a69f0e2a8ef14b7793c43c80c3b8ca899b58acf` (lus le 10 nov. 2026) | la revision **exercée** MUST être consignée dans chaque rapport ; NEVER une validation consignée sans la revision |

MUST NOT déclarer P0 clos sans les prérequis ci-dessus, NEVER une exception. MUST NOT coder une zone non tranchée (§0 règle 8).
