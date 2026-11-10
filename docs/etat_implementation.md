# État d'implémentation d'Alethe.ia

Fichier d'état **hors du document normatif** `_Alethe.IA.md` : la spécification change lentement, l'état d'implémentation change à chaque validation. MUST être mis à jour à chaque validation consignée (règle 21 de §11 de `_Alethe.IA.md`) et MUST être reporté dans le rapport de jalon.

Valeurs : `NOT_STARTED` → `IN_PROGRESS` → `IMPLEMENTED` → `VALIDATED`.

- `IMPLEMENTED` : chemin complet exercé, jamais validé sur la cible de référence.
- `VALIDATED` : chemin exercé sur le **couple (checkpoint déclaré + machine de référence)** consigné (§9 de `_Alethe.IA.md`), mesure consignée. NEVER `VALIDATED` sans mesure. NEVER une mesure transplantée d'une machine à une autre.

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
| R-98..R-102 (découverte de configuration, offline-first, adaptateurs, mappage canonique) | P0 | NOT_STARTED | — | — |
| R-103 (découverte **GGUF** comme import distinct : clés de métadonnées, table des tenseurs, mapping des types sur F-1) | P0 | NOT_STARTED | — | — |
| R-104 (contrat de découverte : frontière des origines de valeur, **manifeste interne** écrit hors du dépôt du modèle, versionné) | P0 | NOT_STARTED | — | — |
| R-105 (validation CPU **par famille de kernel** : 6 familles × 2 exercices) | P0 | NOT_STARTED | — | — |
| R-90 (mode CPU-only, **deux exercices distincts** : BF16 dQwen / Q8_0 LLaDA) | P0-P1b | NOT_STARTED | — | — |
| R-71..R-74 (outillage de conversion) | FUTURE | NOT_STARTED | — | — |
| R-97 (cible dual-stream) | FUTURE | NOT_STARTED | — | — |

**Règle de consignation** : l'agent MUST reporter l'état de chaque exigence touchée dans le rapport de jalon ; MUST lier chaque `IMPLEMENTED` / `VALIDATED` à un artefact (chemin du rapport de mesure, nom de la machine de référence, revision du checkpoint exercé). NEVER déclarer un état sans l'artefact.

## Prérequis d'entrée de P0 (état `NOT_STARTED`, non tranchés à ce jour)

MUST être **consignés avant** P0 (règle 21 de §11) ; NEVER une exécution de P0 sur un prérequis non consigné.

| Prérequis | Source normative | État | Ce qui bloque |
|---|---|---|---|
| Revision épinglée du checkpoint de référence (dQwen `main` / `step25000-swa`), du dépôt de quantization GGUF (LLaDA), du commit de `diffuse-cpp` | §8, zone 22, zone 23, `ADR-001` | **non consignée** | chaîne de poids de la déquantification non vérifiable |
| Version de `wgpu` + limites **lues** du device | §4.5, zone 27, `ADR-008` | **non consignée** | aucune limite, aucune fonctionnalité exposée consignée |
| Modèle de concurrence + liste des points de bloquage | zone 3, `ADR-010` | **non tranché** | propriété des tampons non déclarée |
| Convention de format canonique (clé de topologie explicite) | zone 1, `ADR-011` | **non tranché** | topologie non lisible, format canonique non exercé (R-58) |
| Tolérances chiffrées par format **exercé** (BF16 = poids de référence, Q8_0 = corpus GGUF, FP32 = oracle) | §6.3, zones 9 et 19, `ADR-002`, `ADR-003` | **non chiffrées** | P0 sans seuil de comparaison numérique ; FP16 (`F-3`) MUST être chiffré à son propre jalon, NEVER exigé en P0 |
| Ordre d'accumulation consigné par kernel (réduction) | §4.5, R-28, `ADR-007` | **non consigné** | reproductibilité de la réduction non assurable |
| Empreinte mémoire mesurée et comparée à la réservation d'admission | §8.1, R-32, R-38 | **non mesurée** | admissibilité non décidable, OOM possible |
| Schéma du descripteur interne + table des adaptateurs **par import** (`reconnu` ≠ `exécutable`) | §7.4, zone 30, `ADR-013` | **non tranché** | journal de découverte non productible, adaptateur non sélectionnable |
| **Frontière des origines de valeur, version du schéma du manifeste, format du journal à cinq colonnes, `sha` des fichiers épinglés** | §7.4, R-104, zone 15, zone 30, `ADR-013` | **non tranché** | un champ généré peut être présenté comme natif ; divergence non détectée au chargement |
| **Corpus GGUF exercé épinglé** (`llada-8b-q8_0.gguf`, `diffuse-cpp/LLaDA-8B-Instruct-GGUF`, révision `f80c9ca008bc4de577abe02790cb323e431702fe`), table des tenseurs validée contre la taille du fichier, mapping des types GGUF sur F-1 | §7.4, R-103, zone 1, zone 23 | **non consigné** | la découverte GGUF n'a pas de source épinglée, le type quantifié n'est pas mappé |
| **Rapport CPU par famille de kernel (6 familles × 2 exercices), granularité tranchée par famille** | §4.6, R-105, zone 20, `ADR-015` | **non tranché** | le mode CPU n'est pas mesurable famille par famille, la granularité est sans base |
| Acceptation de `ADR-014` (offline-first, `auto_map` / `trust_remote_code` jamais exécutés) | §7.4, R-99, R-100, `ADR-014` | **proposé, non accepté** | inspection et chargement sans réseau non garantis |
| Machine de référence nommée et ses limites **lues** | zone 26, `ADR-006` | **non tranchée** | la machine de développement (RTX 5090 32 607 MiB, WSL2) n'est **pas** une cible normative |

Le projet **ne peut pas entrer en P0** sans ces décisions consignées : format canonique du checkpoint (`ADR-011`, zone 1), tolérances numériques chiffrées (zone 9, §6.3), ABI Rust ↔ Zig (`ADR-009`), modèle de concurrence et de cancellation (`ADR-010`), version et limites de `wgpu` (`ADR-008`), machine de référence nommée (zone 26, `ADR-006`), **schéma du descripteur interne, table des adaptateurs par import, version du schéma du manifeste et format du journal de découverte (zone 30, `ADR-013`)**, **acceptation du fonctionnement offline-first (`ADR-014`)**, **granularité des kernels CPU tranchée par famille de kernel (zone 20, `ADR-015`)**. Les zones 1, 2, 3, 8, 9, 13, 14, 19, 20, 22, 26, 27, 30 sont ouvertes ; aucune n'est résolue. **MUST NOT écrire le code de découverte, de normalisation, de manifeste, d'adaptateur ni de mappage avant l'acceptation de `ADR-013` et le tranchage de la zone 30 ; MUST NOT trancher la granularité CPU avant l'acceptation de `ADR-015` et de la zone 20 (§0 règle 8).**

**Avant P1a** : les **paramètres de génération par checkpoint** (zone 8) — `tau` effectif, granularité retenue (`block_length` lu ou consigné), ordre de dénoisation, `steps_per_block`, politique d'arrêt, température de validation (`0` = argmax), `stop_strings` — MUST être tranchés et consignés avec la décision ; NEVER un paramètre inventé sans lecture du checkpoint. NEVER la surface d'API (zone 6), l'échantillonnage (zone 7), K/B/S/W (zone 8) et le format de persistance (zone 13) comme prérequis d'entrée de P0 : leurs échéances sont P1a, P2, P3, P4, P5.

| Jalon | Zones ouvertes |
|---|---|
| P0 | 1, 2, 3, 8, 9, 13, 14, 19, 20, 22, 26, 27, 30 |
| P1a | 8, 14, 22 |
| P1b | 26 |
| P2 | 10, 12 |
| P3 | 7 |
| P4 | 6, 18, 21, 25, 28, 29, 30 |
| P5 | 13 |

MUST NOT déclarer P0 clos sans les prérequis ci-dessus, NEVER une exception. MUST NOT coder une zone non tranchée (§0 règle 8).
