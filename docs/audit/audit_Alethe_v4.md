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