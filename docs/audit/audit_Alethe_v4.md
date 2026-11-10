J'ai pu lire les 1151 lignes cette fois. Je corrige mon audit provisoire sur trois points : P0 est bien une tranche verticale exploitable (§8), la matrice jalons ↔ exigences existe (§16), et la liste des décisions ouvertes est explicite (§14).

Le dépôt contient déjà deux audits (`docs/audit/`, le second noté 7,5/10). Je ne m'en suis pas servi : j'ai relu le document tel qu'il est au dernier commit (`a6d15a8`) et vérifié mécaniquement les renvois et la couverture des exigences.

## Note : 7,5 / 10

| Critère | Note | Pourquoi |
|---|---|---|
| Garde-fous contre les dérives d'un agent | 9 | Deux axes de maturité, R-97, références invérifiables retirées, interdictions explicites |
| Gouvernance et traçabilité | 8,5 | Matrice Nature → exigences, matrice exigences → jalons, critères globaux, zones non spécifiées listées |
| Exactitude technique | 8 | Limites WebGPU, `f16`, atomics, E8M0, mapping de buffers : tout ce que j'ai pu contrôler est correct, sauf une erreur (voir 2) |
| Séquencement des jalons | 7 | P0 bien conçu, mais P1 est démesuré |
| Cohérence interne | 6,5 | Plusieurs contradictions et renvois cassés |
| Testabilité | 6,5 | Peu de seuils chiffrés, aucune machine de référence |
| Exploitabilité par un agent | 6 | 106 Ko (~32 000 tokens), 315 MUST contre 2 SHOULD |

C'est une très bonne spécification d'architecte. Comme document de référence pour un agent, il lui manque encore de la cohérence et de la modularité. Le document fait d'ailleurs ce qu'il demande aux autres : il pose des règles (R-97, la règle des identifiants invérifiables) pour empêcher l'agent d'inventer. Ces règles ne couvrent pas les défauts du document lui-même.

## Ce qui est excellent

- **Les deux axes de maturité** (spécification × implémentation). C'est ce qui évite qu'un agent traite une hypothèse de recherche comme un ordre de livraison.
- **L'honnêteté du statut** : tout est `NOT_STARTED`, l'Annexe B est volontairement vide (« A COMPLETER »), et le document distingue ce qui vient d'une source, d'une adaptation ou d'une hypothèse.
- **Les jalons** : P0 est une vraie tranche verticale (limites WebGPU mesurées, deux kernels, backend CPU sur un checkpoint réduit). Il commence par mesurer avant d'optimiser.
- **Les frontières de couches** (§15) sont vérifiables dans l'arborescence du dépôt : `core/` ne contient pas de représentation mémoire, `mem/` ne contient pas de décisions d'ordonnancement.
- **La sécurité de la persistance** (R-75 à R-79) : rare dans une spécification de moteur d'inférence.

## Défauts, par ordre de gravité

### 1. Des instructions contradictoires pour l'agent
- **Qui décide de l'architecture ?** Le §0.7 réserve les arbitrages (format de poids, périmètre d'API, dépendances) au mainteneur. Le §14 dit pourtant que les 25 zones non spécifiées « MUST être décidées par l'agent de développement ». Or ces zones incluent le format de checkpoint canonique, l'ABI Rust↔Zig et le modèle de concurrence, c'est-à-dire le cœur de l'architecture, et P0 (R-58 à R-64) en dépend. Il faut un processus de décision explicite : l'agent propose (note de décision), le mainteneur tranche, la décision est consignée.
- **`Hybride`** : le §5.1 définit `Mode = AR | Diffusion | Hybride`. Le §7.1 dit que `Hybride` n'est pas une valeur valide et qu'il faut séparer famille, topologie et capacités. Le document contredit lui-même une règle qu'il édicte.
- **Placement mémoire** : R-30 interdit toute « partition matérielle fixe par catégorie d'objet », mais R-32 et R-64 imposent aux tables n-gram de rester sur NVMe.
- **Renvois cassés** : le §6.4 ne contient aucune règle de persistance, mais le §8 (P5), le §11 (règle 17) et le §17 (critère 13) le citent pour cela. R-58 renvoie à « §14.1 », qui n'existe pas (le §14 est une liste numérotée). La numérotation saute du §17 au §19, placé après les annexes. R-13, R-14 et R-70 ne figurent dans aucun jalon de la matrice §16 (R-13 et R-14 sont `FUTURE`, mais R-70 est un oubli).
- **Le §19 est redondant avec le §8** et risque de dériver. L'Annexe C renvoie à des audits : l'historique appartient à un changelog, pas à un document normatif.

### 2. Une erreur factuelle à corriger
La ligne N07 de la matrice §13.2 développe GDN en « Gated Differential Network ». Il s'agit de **Gated DeltaNet** (Gated Delta Networks). Un agent qui chercherait la source sous ce nom se tromperait. C'est d'autant plus gênant que le document interdit de fonder une décision sur une référence invérifiable.

### 3. Trop gros et trop impératif pour un agent
Le §0.1 impose de lire le fichier entier avant toute modification, soit ~32 000 tokens. À cette taille, les règles vitales (« C++ NEVER », « pas de checkpoint fabriqué », « mesurer avant d'optimiser ») se noient dans des règles de détail. Avec 315 MUST et 56 NEVER pour 2 SHOULD, la hiérarchie des règles disparaît.

Recommandation : un noyau de 3 à 5 Ko, toujours chargé (invariants, règles du §0, non-objectifs, processus de décision), et des fiches par domaine chargées selon la tâche. Le statut d'implémentation (Annexe A.2) devrait vivre dans un fichier séparé, qui change à chaque jalon, alors que la spécification doit rester stable.

Redéfinir MUST pour les exigences `RESEARCH` (« MUST être étudié ») détourne la RFC 2119. Mieux vaut SHOULD/MAY, ou laisser la colonne de maturité porter la contrainte.

### 4. P1 est démesuré
P1 réunit en un seul jalon : le sous-système mémoire complet (rôles, admission, réallocation, double buffering), le cycle dLLM complet, l'architecture encodeur-décodeur de DiffusionGemma avec cross-attention et multi-canvas, le MoE et l'arrêt adaptatif, sur un modèle de 26 milliards de paramètres. Le saut depuis P0 (checkpoint réduit, deux kernels) est énorme. Je le scinderais, par exemple : P1a = cycle dLLM sur le checkpoint réduit ; P1b = DiffusionGemma avec mémoire polymorphe.

De plus, P0 dépend d'un checkpoint réduit qui n'est pas encore choisi (§14, point 22). Tant que le mainteneur ne l'a pas nommé, rien ne peut démarrer : cette décision devrait figurer en tête du plan.

### 5. Critères d'acceptation peu mesurables, pas de machine de référence
- « cible de référence » apparaît une douzaine de fois sans définition explicite (matériel, VRAM, RAM, version du modèle).
- Aucun seuil de performance. Je comprends la logique (mesurer d'abord, P0), mais il faut un critère d'entrée pour passer de P0 à P1 (par exemple : débit minimal sur le checkpoint réduit).
- Le déterminisme est exigé partout (R-28, I-11, L2/L3/L6 « exact ») sans dire s'il s'agit du déterminisme dans un backend ou entre backends. L'exactitude bit à bit entre un GPU WebGPU et un CPU n'est pas réaliste avec des réductions en `f32`.
- Dans la matrice §16, 15 exigences figurent dans plusieurs jalons (par exemple R-37 à R-40 en P2 et P3, R-64 en P0 et P5). Sans règle (« première livraison / validation finale »), un jalon peut être déclaré terminé à tort.

### 6. Des sujets absents côté WebGPU
- **Le découpage des poids en plusieurs buffers.** Les limites par défaut (`maxStorageBufferBindingSize` 128 Mo, `maxBufferSize` 256 Mo) forcent à répartir les poids sur de nombreux buffers. Le document demande de lire les limites (R-86) mais ne dit nulle part comment le moteur gère des tenseurs plus gros que la limite. C'est la contrainte n° 1 pour charger un LLM en WebGPU.
- **Les subgroups** (réductions et matmuls performants) ne sont pas mentionnés, alors que la liste des fonctionnalités détaille des compressions de textures sans usage évident en inférence.
- Le périmètre P0 demande déjà BF16, FP8, MXFP4, INT4, FP4 et INT8 (R-60), alors qu'il n'existe pas de type FP8 ou FP4 natif en WGSL : tout passe par du décodage logiciel. Et « P0 » désigne à la fois le jalon et la priorité des formats.

### 7. Ce que je n'ai pas pu vérifier
DiffusionGemma, Clover, Qwen3.6 et les identifiants arXiv 2606.x / 2608.x sont postérieurs à mes connaissances. Je ne peux pas confirmer leur existence ni leurs caractéristiques. Le document lui-même note que plusieurs révisions sont « à confirmer » : ce travail reste à faire avant P1 et P3.

## Priorités

1. Résoudre la contradiction §0.7 / §14 et instaurer un processus de décision écrit. C'est le plus risqué pour un agent.
2. Corriger `Hybride` (§5.1), les renvois cassés, la numérotation, GDN, et couvrir R-70 dans la matrice.
3. Scinder le document (noyau court + fiches), sortir le statut d'implémentation, ajouter un changelog.
4. Découper P1, nommer le checkpoint réduit, définir la machine de référence et les critères d'entrée entre jalons.
5. Ajouter la gestion des buffers au-delà des limites et trancher le périmètre du déterminisme.

Avec ces corrections, 8,5/10 est atteignable. Si vous voulez, je peux mettre cet audit dans un document modifiable (Word ou PDF) pour le versionner dans `docs/audit/`.

---

## Vérification du 10 novembre 2026 (complément à l'audit, audit figé à la date du 9 novembre 2026)

Cette section **ne modifie pas** l'audit ci-dessus (figé à sa date). Elle consigne les faits lus et les décisions prises après l'audit, en réponse à la priorité 4 (« définir la machine de référence ») et à la priorité 5 (« gestion des buffers au-delà des limites et trancher le périmètre du déterminisme »).

### Ce qui est constaté, lu à la source

| Constat | Preuve | Effet sur la spécification |
|---|---|---|
| **`GSAI-ML/LLaDA-8B-Instruct` n'est pas un checkpoint adapté à la démonstration de P1a** : 32 couches d'attention **pleine**, `n_heads = 32`, `n_kv_heads = 32` (pas de GQA), `intermediate_size = 12288`, `max_sequence_length = 4096`, `use_cache = false`, **aucune couche récurrente**, **aucune taille de bloc déclarée** | `config.json` + `configuration_llada.py`, lu le 10 nov. 2026 | P1a exercé sur un chemin **uniquement dense** : l'état GDN, la granularité déclarée, la fenêtre glissante et la cross-attention ne sont pas exercés → repositionné, `ADR-001` |
| `UT-IFML/dQwen3.5-0.8B-Base` **déclare** la topologie GDN, le contexte 262 144, le token de masque natif `<|fim_middle|>` = id `248061` ; `model.safetensors` = **1 504 822 808 octets** pour **752 393 024 paramètres BF16**, licence Apache-2.0, base `Qwen/Qwen3.5-0.8B`, `arxiv:2609.20751` | API Hugging Face + `tokenizer_config.json` + `tokenizer.json` (`added_tokens`), lus le 10 nov. 2026 | checkpoint de référence de P0 / P1a (`ADR-001`) |
| Les distributions de référence sont publiées en **BF16** (mesuré à l'API le 10 nov. 2026 : dQwen `1 504 822 808` octets ; **`Diffuspace/Clover-35B-A3B-Preview` 69 322 795 672 octets** pour **34 661 350 848** paramètres BF16, 26 shards + 2 fichiers `bd_gdn_conv_right_weight.safetensors` (1 478 024 octets) et `bd_gdn_softmax_scalars.safetensors` (15 584 octets) ; la valeur `69 342 784 805` citée ici le 10 nov. **est fausse**, NEVER la reproduire) | API Hugging Face (`safetensors.parameters`, `tree/main`) | BF16 exercé en P0 (`ADR-002`) ; les deux fichiers `bd_gdn_*` **prouvent au niveau des noms** les paramètres GDN bidirectionnels (convolution droite + scalars de softmax) décrits en §7.2 |
| **`google/diffusiongemma-26B-A4B-it`** (id exact, l'API renvoie `Invalid username or password` sur `google/diffusion-gemma-26b`) : `model_type = diffusion_gemma`, `architectures = ["DiffusionGemmaForBlockDiffusion"]`, **licence Apache-2.0**, `canvas_length = 256`, `sliding_window = 1024`, `max_position_embeddings = 262 144`, `vocab_size = 262 144`, `num_hidden_layers = 30`, `num_experts = 128`, `top_k_experts = 8`, `hidden_size = 2816`, `tie_word_embeddings = true`, masque **`<mask>`**, `boi_token_id = 255 999`, `eoi_token_id = 258 882`, sha `f7f5b7f5fa82ffc52addd066915886d497f5517b`, **25 823 781 228 paramètres BF16 = 51 647 701 024 octets en 11 shards** ; pipeline **`image-text-to-text`**, `vision_config` **27 couches** (`gemma4_vision`), `image_token_id = 258 880`, `use_bidirectional_attention = "vision"` | `config.json` + API HF lus le 10 nov. 2026 | P1b (canva 256, fenêtre 1024, 30 couches **confirmées**) ; **la multimodalité est réelle** : NEVER présenter DiffusionGemma comme texte-only, la position `N-4` (multimodal refusé) MUST être relue à cette aune ; NEVER la taille « 25,2 B » |
| **Machine de développement constatée** (observation directe, non un choix normatif) : **NVIDIA GeForce RTX 5090, 32 607 MiB, compute capability 12.0 (Blackwell), pilote 616.64** ; **AMD Ryzen 9 7950X3D (16 cœurs)** ; RAM **47,04 GiB** ; **Ubuntu 24.04.4 LTS sous WSL2**, noyau 6.18.33.1 | `nvidia-smi`, `nproc`, `free`, `wslinfo`, 10 nov. 2026 | zone 26 ; NEVER une capacité admissible déduite de « 32 Go » (`ADR-006`) |
| Limites WebGPU (`maxStorageBufferBindingSize`, `maxBufferSize`, `maxComputeWorkgroupStorageSize`, `maxComputeInvocationsPerWorkgroup`, `maxComputeWorkgroupsPerDimension`, `maxBindGroups`, `minStorageBufferOffsetAlignment`, …) : **non lues**, version de `wgpu` **non consignée** | aucune lecture effectuée | zone 27, `ADR-008` ; MUST être lues au démarrage de P0 |
| Versions de Rust, de Zig, de la chaîne de liaison : **non consignées** | aucune lecture effectuée | zone 27, `ADR-009` |

### Ce qui est tranché, en réponse aux priorités de l'audit

**Priorité 4 (machine de référence, critères d'entrée entre jalons)** :
- la machine de référence est **à trancher** (zone 26) ; la machine de développement est **constatée** et ses limites MUST être lues. NEVER transplanter une mesure d'une machine à l'autre (§9, R-44).
- **empreinte mesurée en P0** : poids, tampons de travail, activations, états récurrents (par position, par couche), séquences de test, copies et conversions, peak par classe, occupation réservée vs peak ; MUST être comparée à la réservation d'admission (R-32) ; MUST consigner l'écart. NEVER un modèle admissible sans empreinte mesurée. C'est la réponse exacte à « 32 Go de VRAM n'est pas une machine » : la capacité admissible est **mesurée**, never devinée.
- **seuil de débit consigné par la machine de référence** en critère d'entrée de chaque jalon (§8), NEVER un seuil implicite.

**Priorité 5 (buffers au-delà des limites, déterminisme)** :
- **contrat de découpage (tiling)** : un contrat minimal **partagé** (dimensions, limites vis-à-vis des lectures de device, offsets en éléments **et** en octets, alignements vérifiés avant l'encodage, bornes `[start, end)` de lecture et d'écriture, résultats partiels **typés et bornés**, recomposition **déclarée et déterministe**, traitement des restes, limites matérielles vérifiées) ; chaque kernel **spécialise** sa stratégie. NEVER un découpage implicite déduit de la taille d'un buffer ; NEVER une algorithmique de découpage universalisée. MUST être exercé en P0 (§8.1, R-86, `ADR-004`).
- **chemin de référence de la réduction par nature** : réductions flottantes → réduction hiérarchique à **ordre d'accumulation fixé et consigné par kernel** ; atomics entiers → chemin de référence ; atomics `f32` → **jamais** chemin de référence, MAY en option à quatre conditions (déclaration, conséquence sur la reproductibilité, tolérances, publication des deux sorties) ; subgroup = premier niveau de la hiérarchie, variante mesurée, jamais base de base. C'est la réponse exacte à « le déterminisme est tranché » : la détermination est un **ordre consigné**, never une propriété de la concurrence (`ADR-007`).
- **P0 clos sur trois rôles de format** (BF16 = format de poids de la distribution de référence, Q8_0 = corpus GGUF LLaDA, FP32 = oracle de comparaison, **jamais format de poids à servir en production**), tolérance chiffrée **par format**, NEVER par format générique ; **FP16 est `F-3` de R-60, exercé à son jalon** (la mention « P0 clos sur quatre formats, dont FP16 » de la version 0.2.1 est corrigée le 10 novembre 2026, `ADR-014`). **MUST séparer deux exercices CPU-only** : (a) BF16 → Q8_0 avec activations **en FP32** sur le checkpoint BF16 de dQwen, (b) Q8_0 **d'origine** (GGUF, sans conversion préalable) avec la **moyenne de la distribution** des poids **à partir des octets stockés** et activations décodées localement. NEVER un seul exercice couvrant les deux.
- **version sémantique de la spécification** + `docs/CHANGELOG.md` obligatoire à chaque modification normative ; version applicative distincte (`ADR-005`).

### Ce qui reste ouvert

Machine de référence (`ADR-006`), version de `wgpu` et limites lues (`ADR-008`), versions Rust / Zig et ABI partagée (`ADR-009`), modèle de concurrence (`ADR-010`), convention de format canonique (`ADR-011`), tolérances chiffrées par format, matrice d'API Annexe B. NEVER une ligne de §14 résolue sans ADR consigné.

### Complément du 10 novembre 2026 (priorités 1, 2, 3)


**Priorité 1 (contradiction §0.7 / §14, processus de décision)** : **clos**. §14 porte un **Processus de décision des zones** écrit : l'agent de développement (1) **signale** la zone, (2) **propose** une note de décision (options, invariants respectés, conséquences sur les exigences, coûts), (3) **soumet la note au mainteneur**, (4) **consigne la décision tranchée** dans la ligne de §14 (décision retenue + date) **avant** d'écrire le code. MUST NOT implémenter une zone non tranchée ; MUST NOT trancher seul un arbitrage d'architecture (§0, règle 8). NEVER une zone résolue par un choix d'implémentation non consigné. Les 11 ADR de `docs/decisions/` sont **des notes de décision** : `ADR-001` (checkpoint) est **consignée**, `ADR-003` / `ADR-004` / `ADR-005` / `ADR-007` sont **acceptées** ; `ADR-006`..`ADR-011` sont **proposées, non tranchées** — NEVER les traiter comme applicables.

**Priorité 2 (corrections ciblées)** : **clos sur les six points relevés**.
- `Mode = AR | Diffusion | Hybride` **supprimé** de §5.1 ; les trois dimensions canoniques de §7.1 sont seules employées.
- **Renvois réparés** : §6.4 **contient** les règles de persistance (R-75..R-79) que §8 (P5), §11 (règle 17) et §17 (critère 13) citent ; le renvoi de R-58 vers « §14.1 » (section inexistante) a été retiré ; la numérotation des sections est **continue de §0 à §17** suivie des annexes A et B — **il n'y a plus de §19 après les annexes**, et l'**Annexe C** (historique des audits) a été **supprimée** : l'historique vit dans `docs/CHANGELOG.md` + `docs/audit/`, NEVER dans le document normatif.
- **R-70 couvert dans la matrice §16** (ligne P3) ; `R-13` / `R-14` **sortent de `FUTURE`** et sont rattachés à la cible déclarative de la zone 24.
- **GDN = Gated DeltaNet** : la ligne `N07` et l'index §13.1 portent `arxiv:2412.06464` + `NVlabs/GatedDeltaNet`, NEVER « Gated Differential Network ».
- **Numérotation des exigences vérifiée mécaniquement le 10 novembre 2026** : **103 exigences définies** (`R-1..R-102` **continue**, aucun numéro manquant, + `R-160`, numéro réservé, jamais renuméroté). Toutes les plages citées (`R-1..R-97`, `R-1..R-102`, `R-7..R-12`, `R-19..R-24`, `R-29..R-31`, `R-29..R-35`, `R-29..R-42`, `R-33..R-35`, `R-36..R-40`, `R-37..R-41`, `R-43..R-47`, `R-48..R-50`, `R-48..R-51`, `R-52..R-57`, `R-58..R-64`, `R-65..R-68`, `R-65..R-70`, `R-67..R-69`, `R-71..R-74`, `R-75..R-79`, `R-80..R-82`, `R-84..R-89`, `R-86..R-88`, `R-90..R-97`, `R-98..R-102`, `R-92..R-96`, `R-98..R-101`) sont **définies intégralement** ; NEVER une plage franchissant une exigence non définie. MUST relire la couverture à chaque ajout.

**Priorité 3 (taille et modularité)** : **partiellement traité, zone ouverte**.
- **Fait mesuré le 10 novembre 2026 (dernière mesure du jour, `wc -l -c`)** : le document normatif fait **1 331 lignes / 184 675 octets** (~38 600 tokens au ratio consigné) ; la trajectoire du jour est **1 294 → 1 302 → 1 308 → 1 331 lignes** (154 660 → 157 687 → 164 251 → 165 366 → 165 378 → 169 967 → 184 424 → 184 675 octets). La règle 1 de §0 impose la lecture **intégrale** avant toute modification. MUST re-mesurer (`wc -l -c`) à chaque modification normative : NEVER citer une taille non mesurée. **Nature des ajouts du jour** : faits lus à la source, **correction des citations fausses du 25 octobre 2026**, **reclassification de dQwen (0.3.0)** et **§7.4 (R-98..R-102)**. **Comptages vérifiés mécaniquement ce jour** : **103 exigences** (`R-1..R-102` continu + `R-160` réservé), **30 zones** de §14 dont **28 zones `A TRANCHER`** (zones closes : 4 et 22), **59 lignes de tableau** à **3 champs** dans `docs/CHANGELOG.md` (les pipes de `<\|fim_middle\|>` et `<\|mask\|>` sont échappés, NEVER laissés bruts dans une ligne de tableau).
- **Faits acquis** : le **statut d'implémentation** et le **changelog** sont **hors** de la spécification (`docs/etat_implementation.md`, `docs/CHANGELOG.md`, `ADR-005`) ; l'**Annexe B** reste volontairement `A COMPLETER` (refus explicite, R-82).
- **Zone 28 ajoutée** : fractionnement **noyau court + fiches par domaine** — options (A) document unique, (B) noyau + fiches, (C) (B) + statut séparé (déjà partiellement fait) — **A TRANCHER avant P1** ; NEVER augmenter la taille du document sans trancher.
- **Zone 29 ajoutée** : emploi des mots RFC 2119 sur les exigences `RESEARCH` (une exigence `RESEARCH` porte des `MUST`, ce qui met une hypothèse au rang d'ordre). Options (A) garder `MUST` + colonne de maturité, (B) `SHOULD` / `MAY` sur `RESEARCH` — **A TRANCHER avant P1**.

**Priorité 4** : réponse consignée ci-dessus (machine **constatée** ≠ machine **de référence**, zone 26 ; empreinte **mesurée** en P0, `ADR-006`). **Priorité 5** : **clos sur le contrat** (`ADR-004` tiling exercé en P0, `ADR-007` chemin de réduction par nature, **portée du déterminisme en trois niveaux** — bit-à-bit **dans un même backend**, tolérance **par dtype** entre CPU et WebGPU, tolérance de sortie entre sessions ; les réductions `f32` **ne sont jamais** bit-à-bit entre les deux chemins, NEVER comparées par `==`) ; les **valeurs chiffrées** restent à mesurer (zones 9 / 19).

**Ce que l'audit relevait sur P1 et qui est traité** : **P1 est scindé en P1a / P1b** — P1a = cycle dLLM complet sur le **checkpoint réduit nommé** (`UT-IFML/dQwen3.5-0.8B-Base`, `ADR-001`), P1b = DiffusionGemma 26B à la taille réelle. Le checkpoint réduit **est nommé** : la zone 22 est **close** sur le choix du modèle ; restent à consigner la **révision épinglée**, le **commit du dépôt de quantization** (LLaDA), le **profil de décodage** (`tau`, `block_length`, `order`) et la machine de référence.

**Reste ouvert au 10 novembre 2026 (après la révision 0.3.0)** : **28 zones `A TRANCHER`** de §14 (1, 2, 3, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 23, 24, 25, 26, 27, 28, 29, **30**) ; zones closes : **4** (`N-9`), **22** (`ADR-001`). **Aucune ligne de code n'existe** ; les prérequis d'entrée de P0 sont listés dans `docs/etat_implementation.md` et **non remplis**.

## Complément du 10 novembre 2026 — audit du mainteneur sur la version 0.2.1 et corrections de la 0.3.0

**Faits de la carte lue le 10 novembre 2026** (`UT-IFML/dQwen3.5-0.8B-Base/raw/main/README.md` + `config.json`) : « A masked diffusion language model **adapted from** Qwen3.5-0.8B », « only its **attention layers** are made bidirectional, and the **Gated DeltaNet layers stay causal** », `architectures = ["DiffuQwen35"]`, `model_type = diffuqwen35`, `num_hidden_layers = 24` (18 `linear_attention` + 6 `full_attention`, `full_attention_interval = 4`), `auto_map = { "AutoConfig": "modeling_dqwen3_5.DiffuQwen35Config", "AutoModel": "modeling_dqwen3_5.DiffuQwen35" }`, `max_position_embeddings = 262144`, `vocab_size = 248320`, `dtype = bfloat16`, frontmatter **`inference: false`** ; carte : « Needs a CUDA GPU and `transformers>=5.13` », exemple `trust_remote_code=True`, `generate` dénoie **le canva entier** (`tau = 0.9`), `block_length = 32` **optionnel**, `tau = None, steps_per_block = k`. Famille dQwen3.5 publiée : 0.75B, 1.88B, 4.21B, 8.95B (+ dQwen3-1.7B, full attention).

**Quatre énoncés de la 0.2.1 étaient faux, corrigés en 0.3.0** : (1) dQwen décrit comme « base à convertir, jamais servi comme dLLM natif » → **reclassé comme dLLM déjà adapté** (méthode `dQwen35` ajoutée à la liste de R-16, §7.1 catégorie 2, §7.2.1, §8, §13.1, `ADR-001`) ; (2) `B = 32` figé comme granularité de référence de dQwen → **R-160 défigé** : `block_length` est une option d'appel, NEVER une propriété du checkpoint ; (3) quatre formats imposés en P0 (BF16 / FP16 / FP32 / Q8_0) → **trois rôles distincts** (BF16 poids de référence, FP32 oracle jamais cible de production, Q8_0 corpus GGUF LLaDA) et **FP16 reporté à son jalon** (`F-3` de R-60) ; (4) un seul exercice CPU-only → **deux exercices distincts** (R-90 : BF16 dQwen, Q8_0 LLaDA).

**Énoncé conforme vérifié** : la chaîne `generation_family = Hybrid` de la carte de `yuchen-zhu-zyc/HybridDiffusion-2B` **n'est pas** une valeur de l'énumération canonique (§7.1). La 0.3.0 ajoute la **règle de mappage** : chaîne source consignée + valeur normalisée `Diffusion` + liste de capacités `[AR, Diffusion]`, NEVER un mode moteur.

**Deux principes ajoutés par la 0.3.0** (§7.4, `ADR-013`, `ADR-014`, zone 30) : **découverte de la configuration** (R-98, R-101, R-102 : lecture des champs canoniques, validation de cohérence, descripteur interne, table d'adaptateurs, `reconnu` ≠ `exécutable`) et **offline-first** (R-99, R-100 : inspection / chargement / inférence sans réseau, téléchargements explicites et consignés, **`auto_map` et `trust_remote_code` jamais exécutés** — la définition de l'architecture de dQwen vit dans `modeling_dqwen3_5.py` du dépôt, Alethe.ia MUST l'implémenter elle-même).

**Point rappelé au mainteneur** : `ADR-006` constate la machine de développement (RTX 5090 **32 607 MiB**, Ryzen 9 7950X3D, 47,04 GiB, WSL2) **sans l'adopter** ; la zone 26 reste `A TRANCHER`, NEVER une cible normative implicite. **Les six relectures arxiv désignées par l'audit v3 ne sont pas exercées.**