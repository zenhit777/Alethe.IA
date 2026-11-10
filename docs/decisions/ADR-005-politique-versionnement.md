# ADR-005 — Politique de versionnement de la spécification

**Statut** : accepté (10 novembre 2026). Résout la zone 15 de §14 **pour la spécification**.
**Régime** : décision documentaire ; ne fixe pas la version applicative ni la version de dépôt.

## Contexte

Le document normatif `_Alethe.IA.md` portait un numéro de version non défini, sans règle de changement. L'application, la spécification et le dépôt de code ont trois cycles de changement distincts : la spécification change par décision consignée, l'application change par validation mesurée, le dépôt change par commit. Un numéro unique ne décrit pas ces trois cycles ; une politique unique les mélange.

La règle de non-spéculation est déjà normative (§0 règle 1 : lecture complète avant modification ; §14 zone A TRANCHER interdite de codage). Une politique de versionnement MUST rendre lisible, à la lecture de l'en-tête, ce qui a changé et ce qui est exigeable.

## Décision

Version sémantique `M.m.n` appliquée à la **spécification** :
- `M` — modification qui casse une exigence existante (un consommateur qui respecte la version précédente MUST être informé d'un changement de contrat) ;
- `m` — ajout d'exigences, de jalons, de cibles, de formats ; aucune exigence existante n'est cassée ;
- `n` — correction d'une erreur factique, d'un renvoi, d'une formulation, sans changement de contrat.

La version est portée dans l'en-tête de `_Alethe.IA.md` avec la date et la nature des changements ; **toute** modification normative MUST créer une ligne dans `docs/CHANGELOG.md` (date, décision, emplacement). NEVER une modification normative sans ligne de changelog. NEVER une version de spécification portée sans changelog.

L'application porte **sa propre** version (`ADR-008` pour la version de `wgpu`, `ADR-009` pour la chaîne Rust / Zig) ; MUST NOT confondre la version de la spécification avec la version de l'artefact exécuté. MUST consigner, dans chaque rapport de jalon, la version de la spécification **et** la version de l'artefact exercé. NEVER une validation consignée sans la version de la spécification lue.

## Conséquences

- La première révision sous cette politique est la version **0.1.0** (ajout de R-160 et R-97, re-nommage du checkpoint de référence, ajout de BF16, ajout du contrat de tiling, ajout du chemin de réduction, ajout de l'empreinte mesurée). **Aucune exigence existante n'est cassée** : les exigences dont le contenu factuel change (R-60, R-28, R-33, R-34, R-86) sont **complétées**, NEVER retirées.
La version **0.2.0** (10 novembre 2026) est un `m` : ajout de la table de tolérances par format (§6.3), des zones 28 et 29 (§14), des entrées `docs/` de la structure de dépôt (§15), des endpoints d'API consignés, des lignes de changelog de `ADR-005` / `ADR-010` / `ADR-011`, des sections de vérification datées des quatre audits. NEVER porter une version sans la nature des changements consignée dans l'en-tête **et** dans le changelog.
La version **0.2.1** (10 novembre 2026) est un `n` : corrections factiques **sans** contrat ni critère d'acceptation modifié — comptage de LLaDA-8B-Base corrigé à **8 015 581 184** paramètres BF16, identifiant exact `google/diffusiongemma-26B-A4B-it` (l'id `google/diffusion-gemma-26b` **ne résiste pas à la relecture de l'API**), sha vérifiés (`f7f5b7f5fa…`, `4b0402b65e523d0183a14c431955438714cb16ff`), tailles **mesurées** sur `tree/main` (51 647 701 024 octets / 11 shards ; 69 322 795 672 octets / 26 shards + 2 fichiers `bd_gdn_*`, la valeur `69 342 784 805` est **fausse**). **La seule conséquence normative** est la relecture de `N-4` : la multimodalité de DiffusionGemma est **réelle** (`image-text-to-text`, `vision_config` 27 couches) — NEVER qualifier un checkpoint de texte-only sans lire sa carte ; la position `N-4` (multimodal refusé) est **maintenue**, NEVER contredite par un fait lu. NEVER qualifier de `patch` une modification qui change un critère d'acceptation : celle-ci n'en change aucun.
- La zone 15 de §14 est **résolue pour la spécification** ; reste ouverte pour la version applicative (`ADR-008`, `ADR-009`).
- `docs/decisions/` devient le registre des ADR ; la ligne de §14 consigne la référence à l'ADR. NEVER une décision sans ADR et sans ligne de §14.
