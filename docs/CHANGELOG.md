# Historique du document normatif

Ce fichier porte l'historique. `_Alethe.IA.md` est le **document normatif** : il ne raconte pas son passé, il énonce ce qui est exigé.

## Décisions consignées

| Date | Décision | Emplacement |
|---|---|---|
| 2026-11-10 | Deux axes de maturité séparés : axe 1 (spécification `DEFINED` / `EXPERIMENTAL_SPEC` / `RESEARCH` / `FUTURE`), axe 2 (implémentation `NOT_STARTED` / `IN_PROGRESS` / `IMPLEMENTED` / `VALIDATED`). L'interprétation « un MUST d'une exigence RESEARCH se lit MUST être étudié » est **supprimée** : une exigence RESEARCH s'exprime en SHOULD, les gates de production en MUST / MUST NOT. | §2.4, Annexe A, §11 règle 10 |
| 2026-11-10 | L'état d'implémentation sort du document normatif → `docs/etat_implementation.md`. | Annexe A.2 |
| 2026-11-10 | `Mode = AR \| Diffusion \| Hybride` supprimé : le contrat commun est typé par `generation_family` + `architecture_topology` + `generation_capabilities`. `Hybride` n'est pas une valeur. | §5.1, §7.1 |
| 2026-11-10 | P1 scindé en **P1a** (cycle dLLM sur le checkpoint réduit) et **P1b** (cible de référence 26B). `p_cache` retiré des critères de complétion (devient une métrique). | §8, §16, §9 |
| 2026-11-10 | **Checkpoint réduit nommé** : `GSAI-ML/LLaDA-8B-Instruct` (MIT, `arxiv:2502.09992`), GGUF **Q8_0** ↔ INT8 (F-1). Tailles, tokenizer, topologie consignés ; la taille de bloc reste un paramètre de configuration (le checkpoint ne la déclare pas). | §8, zone 22 (résolue), §13.1 |
| 2026-11-10 | **Machine de référence** définie ; « cible de référence » = couple (checkpoint + machine). | §9 |
| 2026-11-10 | **Critères d'entrée entre jalons** : rapport consigné, tolérances chiffrées, seuil de débit consigné par le mainteneur, état d'implémentation. | §8 |
| 2026-11-10 | Processus de décision des zones non spécifiées : l'agent **propose** une note, le mainteneur **tranche**, la décision est consignée avant le code. | §0.7, §11 règle 2, §14 |
| 2026-11-10 | `subgroups` ajouté aux fonctionnalités à tester (`enable subgroups;`, `subgroup_invocation_id`, `subgroup_size`, fonctions collectives, `subgroupMinSize` / `subgroupMaxSize` — disponible Chrome 134) ; **découpe des tenseurs en tuiles** au-delà de `maxStorageBufferBindingSize`. | §4.5, §8.1 |
| 2026-11-10 | Portée du déterminisme définie : bit-à-bit **dans un backend**, tolérance par dtype **entre backends**, tolérance de sortie. Interdiction de l'identité bit-à-bit CPU ↔ WebGPU pour les réductions `f32`. | §6.3, R-28 |
| 2026-11-10 | R-30 réconcilié avec R-32 / R-64 : une contrainte de placement déclarée (n-gram sur NVMe) est une **contrainte d'entrée de l'ordonnanceur de ressources**, pas une partition du matériel. | R-30 |
| 2026-11-10 | GDN corrigé : **Gated DeltaNet** (`arxiv:2412.06464`), pas « Gated Differential Network ». | §13.2 N07, §13.1 |
| 2026-11-10 | Niveaux de priorité de format renommés **F-1 / F-2 / F-3** (jamais confondus avec les jalons P0..P5). | R-60 |
| 2026-11-10 | Matrice exigences → jalons : lecture **première livraison / validation finale** ; R-13, R-14, R-70 ajoutés à la matrice. | §16 |
| 2026-11-10 | §19 supprimé (redondant avec §8) ; Annexe C (correspondance des sections) déplacée dans ce fichier. | §19, Annexe C |
| 2026-11-10 | Règles de QA étendues aux suites de persistance (R-75..R-79) ; §6.4 ne renvoie plus à lui-même. | §6.4 |
| 2026-11-10 | API de contrôle `R-92..R-96` réintroduite ; Annexe B = spécification à compléter avant P4 ; N-8 WASM en décision à trois positions (A/B/C). | §3.7, Annexe B, §12 |
| 2026-11-10 | Audit v2 : sémantique E8M0 (`0x00` = `2^-127`, `0xFF` = NaN réservé), noms de fonctionnalités WebGPU, registre de références daté, familles / topologies / capacités, profils d'attention par modèle, rattachement des mécanismes de Strata aux fichiers du dépôt `Niko1221/Strata`. | §3.9, §4.5, §7.1-7.2.1, §13.1, fiche Influences |

## Audits lus

- `docs/audit/audit_Alethe.md` (v1) — baseline antérieure, score initial.
- `docs/audit/audit_Alethe_v2.md` (v2) — 7.5/10, 4 blocages (E8M0, fonctionnalités WebGPU, maturité, références).
- `docs/audit/audit_Alethe_v3.md` (v3) — cohérence interne : `Mode` à trois valeurs, critère P1 imposant `p_cache`, P0 sans modèle concret, mécanismes de Strata non rattachés.
- `docs/audit/audit_Alethe_v4.md` (v4) — 7 défauts structurants : structure du document, décisions non spécifiées, renvois cassés, processus, redondance, WebGPU, déterminisme.

## Correspondance des sections (baseline v1 → document actuel)

formats de poids (§6) → §3.9 ; validation & QA (§7) → §6 ; écosystème (§8) → §7 ; jalons (§9) → §8 ; métriques (§10) → §9 ; conventions (§11) → §10 ; interface agent (§12) → §11 ; non-objectifs (§13) → §12 ; index des références (§14) → §13.1 ; zones non spécifiées (§15) → §14 ; structure de dépôt (§16) → §15 ; matrice jalons (§17) → §16 ; critères globaux (§18) → §17.

**Les IDs d'exigences `R-1..R-74` sont conservés.** Ajouts : `R-75..R-79` (sécurité de persistance), `R-80..R-82` (API Anthropic), `R-83..R-89` (formats, ordonnanceur de ressources), `R-90` (mode CPU complet), `R-91` (interface des profils de stratégie), `R-92..R-96` (API de contrôle), `R-97` (cible de validation dual-stream).
