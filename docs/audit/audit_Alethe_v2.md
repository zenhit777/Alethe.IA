# Deuxième audit de `_Alethe.IA.md`

# 7,5 / 10

Nouvelle note — aptitude à servir de référence normative à un agent IA

Nettement amélioré — quelques blocages subsistent

J'ai examiné la nouvelle version de [`_Alethe.IA.md`](https://github.com/zenhit777/Alethe.IA/blob/main/_Alethe.IA.md?utm_source=chatgpt.com), ainsi que ses fiches techniques, son index de références et le dernier historique de commits.

Version auditée :

- Branche : `main`
- Dernier commit observé : `86938fcc5d60a28eaad8346a04eacc55b8f4381d`
- Document : blob `b11c8cc97982d3caae999962a19f4036fa447c85`
- Commit du 10 novembre 2026 : [Clarifie le cadre normatif d’Alethe.ia](https://github.com/zenhit777/Alethe.IA/commit/86938fcc5d60a28eaad8346a04eacc55b8f4381d?utm_source=chatgpt.com)

Je considère cette nouvelle version comme une véritable amélioration architecturale, et non comme une simple correction rédactionnelle.

Le document est désormais beaucoup plus proche de ce que nous recherchions : une spécification normative permettant à un agent IA de comprendre les responsabilités, les limites et les priorités du moteur sans inventer sa propre architecture.

Je ne lui attribue cependant pas encore une note supérieure à 8, car j'ai identifié plusieurs problèmes qui pourraient conduire un agent à implémenter des comportements incorrects, notamment dans les représentations numériques, la prise en charge des modèles et les références techniques.

## 1. Les corrections qui me paraissent particulièrement réussies

### 1.1. L'unification AR/dLLM est désormais correctement formulée

Correction réussie

C'est probablement la correction architecturale la plus importante.

La nouvelle version abandonne l'idée d'un unique algorithme de génération et retient :

> Infrastructure commune, pas algorithme unique.

La section 2.5 précise ce qui est mutualisé et ce qui reste propre à chaque famille.

C'est exactement la distinction que je souhaitais voir apparaître.

Le moteur peut désormais partager les contrats d'opérations, les ressources, les états, la mémoire, les APIs et les mécanismes de validation, tout en conservant deux stratégies de génération distinctes.

Cette formulation protège la priorité dLLM sans sacrifier la compatibilité AR.

### 1.2. Les deux ordonnanceurs sont réintroduits

Correction réussie

L'invariant I-2 introduit :

- un ordonnanceur d'inférence, responsable de ce qui doit être calculé et de son ordre ;
- un ordonnanceur de ressources, responsable du placement, des transferts et des budgets.

La communication par interface typée est également une bonne décision.

Cela réduit le risque de construire un scheduler monolithique qui mélange des décisions algorithmiques et matérielles.

Je conserverais cette séparation comme un principe structurant d'Alethe.ia.

### 1.3. Le CPU est désormais pleinement reconnu comme une ressource de production

Correction réussie

L'invariant I-11 et R-90 sont conformes à tes précisions.

Le document reconnaît maintenant trois usages du CPU :

1. Backend de référence et de débogage.
2. Calcul complémentaire dans une exécution hybride CPU/GPU, notamment pour les experts MoE.
3. Backend complet d'inférence sur une machine sans GPU compatible.

C'est cohérent avec une application qui doit s'adapter à des configurations matérielles hétérogènes.

Il reste une nuance à corriger dans R-37, mais la décision architecturale elle-même est maintenant correctement représentée.

### 1.4. La conversion des checkpoints est correctement externalisée

Correction réussie

Le moteur est maintenant explicitement un moteur d'inférence strict.

La distinction entre checkpoints natifs, checkpoints adaptés et checkpoints AR est conservée. L'`AdaptationFile` apporte une bonne idée : faire accompagner un checkpoint adapté des informations nécessaires pour comprendre son origine, ses mappings et ses préconditions.

La conversion devient un projet externe, et les exigences R-71 à R-74 sont clairement situées hors du moteur.

C'est beaucoup plus cohérent avec le produit que nous avions défini.

### 1.5. Les niveaux de maturité apportent une vraie valeur

La distinction `STABLE`, `EXPERIMENTAL`, `RESEARCH` et `FUTURE` est excellente.

Elle permet enfin de séparer :

- les capacités destinées à être implémentées ;
- les optimisations qui doivent être mesurées ;
- les hypothèses scientifiques ;
- les ambitions futures.

L'agent reçoit également une consigne claire : ne pas transformer une hypothèse de recherche en chemin de production.

C'est particulièrement pertinent pour WeDLM, FLUID et les mécanismes issus de FLARE.

### 1.6. La gestion de la mémoire devient plus agnostique

R-30 et R-31 remplacent une répartition rigide des objets par une classification selon leur rôle, leur taille, leur fréquence d'accès et leur coût de transfert.

Le modèle peut déclarer ses besoins, et le scheduler de ressources décide où les satisfaire.

C'est une meilleure traduction de notre philosophie d'adaptation fluide : les objets ne doivent pas être enfermés dans une catégorie matérielle prédéterminée.

### 1.7. Les contraintes WebGPU sont beaucoup mieux documentées

Les corrections concernant :

- les noms exacts des limites ;
- le caractère optionnel de `shader-f16` ;
- l'absence d'atomic f32 dans le langage de base ;
- les règles de mapping des buffers ;
- la distinction entre plans statiques et replay natif de graphes ;

sont pertinentes.

La nouvelle formulation ne présente plus WebGPU comme une abstraction capable de reproduire automatiquement toutes les fonctionnalités CUDA.

C'est une amélioration importante pour un projet qui veut rester portable.

### 1.8. La sécurité et la persistance deviennent des exigences explicites

Les exigences R-75 à R-79 sont bienvenues.

Le fait d'imposer une activation explicite de la persistance, des règles de rétention, des permissions, des mécanismes de suppression et une information sur le chiffrement évite que la gestion des états d'inférence devienne une fonctionnalité opaque.

# 2. Les problèmes qui subsistent

C'est maintenant le plus important. Je les classe selon le risque qu'ils représentent pour l'implémentation.

## 2.1. E8M0 : une erreur numérique qui doit être corrigée avant P0

Bloquant — priorité absolue

J'ai retrouvé dans le document une prescription concernant les échelles E8M0 :

> Les codes E8M0 0 et 255 MUST être traités comme des valeurs extrêmes (`+0`, `+inf`).

Cette règle est incorrecte.

Selon la spécification officielle des formats microscaling de l'OCP, E8M0 est un format d'exposant non signé, avec biais 127. Il ne possède pas de représentation du zéro ni de l'infini.

Les deux encodages concernés sont :

| Encodage | Valeur réelle                                     |
| -------- | ------------------------------------------------- |
| `0x00`   | \\(2^{-127}\\), plus petite valeur positive finie |
| `0xFF`   | NaN, valeur réservée                              |

Sources : [OCP Microscaling Formats Specification](https://www.opencompute.org/documents/ocp-microscaling-formats-mx-v1-0-spec-final-pdf?utm_source=chatgpt.com) et [spécification ONNX des types numériques](https://onnx.ai/onnx/technical/float8.html?utm_source=chatgpt.com).&#x20;

[image](https://www.google.com/s2/favicons?domain=https://www.opencompute.org\&sz=32)

OCP Microscaling Formats (MX) Specification+2



Ce n'est pas une subtilité théorique.

Une mauvaise interprétation de ces échelles peut produire des valeurs de poids incorrectes, altérer les logits et rendre toute comparaison numérique trompeuse.

Correction demandée :

- préserver `0x00` comme valeur finie \\(2^{-127}\\) ;
- traiter `0xFF` comme NaN selon les règles du format et du checkpoint ;
- définir explicitement le comportement d'admission si une échelle NaN est rencontrée ;
- corriger toutes les occurrences dans R-59, R-60, C-9 et les sections de validation concernées.

Je considère cette erreur comme un bloqueur de P0, parce que le décodage et la fidélité numérique des poids font partie des fondations du moteur.

## 2.2. Les fonctionnalités WebGPU comportent encore deux noms incorrects

À corriger avant P0

La section 4.5 indique :

> Fonctionnalités à déclarer : `shader-f16`, `float64`, `timestamp-query`, `texture-compression`.

La liste n'est pas exacte pour WebGPU.

- `shader-f16` existe.
- `timestamp-query` existe.
- `float64` n'est pas une fonctionnalité WebGPU normalisée.
- `texture-compression` n'est pas un nom de fonctionnalité générique.

Les fonctionnalités de compression sont notamment spécifiées par format : `texture-compression-bc`, `texture-compression-etc2`, `texture-compression-astc` et autres variantes.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://www.w3.org\&sz=32)

w3.org+1



Je recommande de remplacer cette liste par les noms exacts des fonctionnalités que le moteur envisage réellement d'utiliser.

Pour `float64`, il faut simplement préciser que les calculs CPU peuvent éventuellement utiliser des types double précision, mais qu'il ne s'agit pas d'une capacité GPU WebGPU à détecter.

## 2.3. Le niveau `STABLE` ne distingue pas suffisamment la maturité de la spécification et celle de l'implémentation

Ambiguïté normative importante

La section 2.4 définit `STABLE` comme :

> comportement défini et validé sur au moins une cible de référence.

Mais les exigences R-1 à R-90 sont très majoritairement marquées `STABLE`, alors que :

- les modèles de référence sont encore déclarés non commencés ;
- P0 prévoit encore la découverte et la validation des premières opérations ;
- les ordonnanceurs complets sont à construire ;
- le backend CPU-only reste à implémenter ;
- les APIs restent à réaliser.

Il y a donc une ambiguïté sur la signification du terme.

Je soupçonne que `STABLE` désigne ici la maturité de la définition de l'exigence, plutôt que la maturité d'implémentation. Mais le document ne le dit pas.

Je recommande de séparer deux dimensions :

| Dimension                    | Valeurs proposées                                        |
| ---------------------------- | -------------------------------------------------------- |
| Maturité de la spécification | `DEFINED`, `EXPERIMENTAL_SPEC`, `RESEARCH`, `FUTURE`     |
| État d'implémentation        | `NOT_STARTED`, `IN_PROGRESS`, `IMPLEMENTED`, `VALIDATED` |

Ainsi, R-12 peut avoir une spécification définie, tout en ayant un état d'implémentation `NOT_STARTED`.

Cette distinction évitera à un agent de confondre « exigence stable » et « fonctionnalité déjà validée ».

# 3. Les modèles dLLM : le document doit encore être plus précis

C'est le cœur d'Alethe.ia. Je suis donc particulièrement exigeant sur ce point.

## 3.1. Clover est mal classé dans R-16

Contradiction modèle

R-16 indique :

> checkpoint adapté dLLM (WeDLM, FLUID, FLARE, Clover)

Mais la section 7 classe Clover comme un checkpoint dLLM natif.

Et c'est bien ce que décrit la documentation officielle de Clover : un modèle de diffusion natif construit sur une architecture MoE hybride GDN et attention complète, avec attention bidirectionnelle intra-bloc.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub+1



Clover ne doit donc pas être placé dans une liste de méthodes ou de modèles AR adaptés.

Je recommande de distinguer :

- modèles dLLM natifs : DiffusionGemma, Clover ;
- checkpoints AR adaptés : produits externes par les méthodes WeDLM, FLUID, FLARE ou REPR-ALIGN ;
- modèles AR natifs.

D'ailleurs, WeDLM, FLUID et FLARE sont d'abord des travaux ou protocoles d'adaptation, pas des identifiants de checkpoints. Le nom du modèle adapté et la méthode utilisée doivent être des champs distincts de l'`AdaptationFile`.

Enfin, R-16 impose un « streaming causal » aux checkpoints adaptés, puis inclut Clover, alors que son architecture ne doit pas être ramenée à ce régime causal. Il faut corriger cette exigence.

## 3.2. Les exigences dual-stream n'ont pas encore de cible de validation

R-13 et R-14 exigent les modes Diffusion-Trust et AR-Trust pour un checkpoint dual-stream.

Mais la matrice des modèles indique :

| Modèle         | Dual-stream |
| -------------- | ----------- |
| DiffusionGemma | Non         |
| Clover         | Non         |

Le document précise lui-même que ces modes restent indisponibles tant qu'un checkpoint dual-stream n'est pas fourni.

Or le critère d'acceptation globale numéro 1 exige déjà un checkpoint dual-stream.

Il manque donc une cible concrète pour valider ces fonctionnalités.

Ce n'est pas nécessairement un problème si ces modes doivent être développés plus tard. Mais dans ce cas :

- R-13 et R-14 ne doivent pas bloquer les premiers jalons ;
- leur maturité doit être adaptée ;
- le critère global doit être reporté ;
- une cible externe de validation doit être identifiée avant leur implémentation.

En l'état, l'agent pourrait être tenté de fabriquer un faux checkpoint ou de simuler un comportement dual-stream pour satisfaire l'exigence.

## 3.3. `Mode = AR | Diffusion | Hybride` mélange deux dimensions

Le document utilise deux définitions :

- §2.5 : `Mode = AR | Diffusion`
- §5.1 : `Mode = AR | Diffusion | Hybride`

Et dans `Capabilities`, `mode_natif` possède également les trois valeurs.

Pourtant DiffusionGemma est un dLLM natif, mais possède une architecture encodeur-décodeur hybride entre traitement autoregressif du contexte et débruitage du canvas.

Clover, lui, possède une architecture hybride GDN + attention complète, tout en étant également un dLLM natif.

Le terme `Hybride` ne permet donc pas de savoir si l'on parle :

- d'une architecture hybride ;
- d'une stratégie de génération multi-mode ;
- d'une combinaison AR/diffusion dans le même checkpoint.

Je recommande de séparer clairement :

```
generation_family:
  AR
  Diffusion

architecture_topology:
  DecoderOnly
  EncoderDecoder
  HybridAttention
  HybridRecurrent

generation_capabilities:
  AR
  Diffusion
  Speculative
  DualStream
```

Ce serait beaucoup plus extensible.

## 3.4. Les règles d'attention doivent être définies par architecture

La matrice générale §2.5 présente la politique d'attention dLLM comme :

> bidirectionnelle intra-bloc, croissante, horizon

Cela correspond notamment au modèle Clover.

Mais DiffusionGemma possède un chemin distinct : son encodeur autoregressif construit le contexte et son décodeur traite le canvas avec une attention bidirectionnelle, associée à une cross-attention sur le contexte en cache. Le canvas débruité est ensuite ajouté au contexte.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://ai.google.dev\&sz=32)

ai.google.dev+1



Ce fonctionnement ne doit pas être ramené à une simple règle générique d'attention causale sur préfixe et bidirectionnelle intra-bloc.

Je recommande d'ajouter à chaque fiche de modèle un profil d'attention explicite, décrivant les différents sous-modules et leurs relations.

## 3.5. `p_cache` et fenêtre glissante restent trop génériques

R-25 est correctement limité à l'adaptateur causal compatible WeDLM.

Mais R-26 et R-27 imposent toujours globalement :

- de maximiser `p_cache` ;
- de conserver une fenêtre glissante fixe ;
- de remplacer les tokens validés par un nombre égal de masques.

Ces mécanismes sont propres à certaines stratégies de décodage causal et à leurs dépendances.

Ils ne doivent pas nécessairement être des objectifs de chaque dLLM natif.

Je déplacerais ces exigences dans le profil de stratégie WeDLM, puis j'ajouterais une interface commune permettant à chaque stratégie de déclarer ses propres métriques de progression, ses conditions de cache et ses règles de fenêtre.

## 3.6. Les tables n-gram restent obligatoires alors que les modèles de référence ne les utilisent pas

R-32 et R-64 imposent une gestion des tables n-gram sur NVMe.

Or `table_ngram = false` pour DiffusionGemma et Clover.

Il faut que la règle soit conditionnelle :

> Lorsque le checkpoint déclare la présence d'une table n-gram, le moteur doit prendre en charge son stockage et son chargement selon le format correspondant.

Cela évite qu'une caractéristique d'une famille ou d'un système tiers soit imposée à toutes les architectures.

# 4. CPU : la décision est bonne, mais R-37 est trop prescriptif

Tu as confirmé deux choses :

- le CPU peut faire partie de l'inférence en production ;
- le CPU peut assurer une inférence complète sans GPU compatible.

Le document respecte ces deux décisions.

En revanche, R-37 impose :

> Le moteur MUST calculer les experts manquants sur CPU en place.

Ce choix est plus fort que ta décision.

Tu as validé l'utilisation du CPU, pas l'obligation de calculer systématiquement chaque expert manquant sur CPU.

Il existe des situations dans lesquelles le transfert d'un expert depuis la RAM vers le GPU sera préférable au calcul CPU. D'autres dans lesquelles la co-exécution sera avantageuse. D'autres enfin où le coût des transferts ou la saturation du CPU rendra cette approche contre-productive.

La bonne règle serait :

Le CPU peut participer à l'exécution des opérations et des experts lorsque l'ordonnanceur de ressources estime, sur la base des coûts mesurés, que cette exécution est pertinente.

Le mode CPU-only complet reste une exigence distincte et doit être maintenu.

Je rendrais donc R-37 conditionnel à une décision mesurée d'offload, tout en conservant une exigence forte de compatibilité et de correction du backend CPU.

# 5. Audit des sources : le problème Strata est mieux traité, mais le registre reste fragile

C'est un point sur lequel le document a progressé.

La section 13 précise désormais explicitement que le projet retenu est `Niko1221/Strata` et que `ro99/strata` est un projet homonyme distinct.

C'est très bien.

Cependant, j'ai vérifié le contenu du dépôt Niko1221/Strata, et la fiche technique semble encore porter des caractéristiques héritées de l'autre Strata.

Le dépôt sélectionné se présente principalement comme un moteur/app local destiné à exécuter des modèles sur des machines personnelles, avec interface desktop, CLI, installation locale, API compatibles OpenAI et Anthropic, ainsi qu'un support matériel assez large. Son README documente notamment des configurations NVIDIA, AMD, Intel et Apple Silicon, et des usages depuis le SSD lorsque le modèle dépasse la mémoire disponible.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub



La fiche conserve pourtant cette formulation :

> Strata est un moteur C++/CUDA conçu pour exécuter des modèles qui ne tiennent pas nécessairement dans la VRAM, voire dans la RAM disponible.

Elle décrit ensuite des mécanismes techniques très spécifiques : LRU d'experts, placement VRAM/RAM/NVMe, routage, etc.

Je ne dis pas que le dépôt Niko1221/Strata ne contient pas de tels mécanismes. Je dis que la fiche doit les rattacher à des fichiers précis de ce dépôt, et non conserver une description générique provenant de l'autre projet Strata.

Il faut distinguer :

- ce que le README du projet annonce ;
- ce que son code implémente ;
- ce que l'équipe d'Alethe.ia en extrait comme inspiration ;
- ce qu'Alethe.ia décide d'adapter.

### 5.1. Une autre incohérence : DwarfStar

La nouvelle fiche technique identifie DwarfStar comme :

`BenoitMalo/DwarfStar4`

En revanche, le fichier `références.txt` conserve :

`antirez/ds4`

Et la source officielle que nous avions précédemment identifiée est bien le dépôt [`antirez/ds4`](https://github.com/antirez/ds4?utm_source=chatgpt.com).

Il faut unifier ce nom dans les fiches et le registre principal, ou documenter explicitement une décision de changement de référence.

### 5.2. Plusieurs identifiants de dépôts ne sont pas vérifiables

Dans la table de sources de §13.1, j'ai identifié plusieurs références qui ne sont pas récupérables sous les identifiants indiqués :

| Entrée actuelle         | Problème                                                 | Référence à vérifier                     |
| ----------------------- | -------------------------------------------------------- | ---------------------------------------- |
| `BenoitMalo/xLLaMA.cpp` | Dépôt non retrouvé publiquement sous ce nom              | Identifier l'origine exacte ou supprimer |
| `ggermet/ggml`          | Identifiant non retrouvé                                 | `ggml-org/ggml` et sa spécification GGUF |
| `rustwasm/wgpu`         | Identifiant incorrect                                    | `gfx-rs/wgpu`                            |
| `OpenAI/openai`         | Ne constitue pas une référence de dépôt d'API vérifiable | Documentation officielle OpenAI          |
| `Anthropic/anthropic`   | Même problème                                            | Documentation officielle Anthropic       |
| `google-gemma`          | Nom d'organisation, pas identifiant complet de dépôt     | Model card exacte et révision des poids  |

Références vérifiables : [wgpu](https://github.com/gfx-rs/wgpu?utm_source=chatgpt.com), [ggml](https://github.com/ggml-org/ggml?utm_source=chatgpt.com), [spécification GGUF](https://github.com/ggml-org/ggml/blob/master/docs/gguf.md?utm_source=chatgpt.com).&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub+1



Il faut également éviter les formulations telles que « version courante » dans une table annoncée comme contenant des révisions épinglées.

Une référence destinée à un agent doit être immuable ou explicitement datée.

# 6. Les autres sujets à ne pas perdre de vue

### 6.1. L'API de contrôle a disparu

Dans notre architecture initiale, nous avions retenu deux interfaces distinctes :

1. API d'inférence compatible OpenAI et Anthropic.
2. API de contrôle pour l'administration du moteur : chargement des modèles, déchargement, état des sessions, ressources, configuration et métriques.

La nouvelle version documente bien les deux APIs d'inférence, mais je ne retrouve pas la définition complète d'une API de contrôle distincte.

Il ne faut pas confondre l'absence de console graphique d'administration et l'absence d'API de contrôle.

À moins que cette décision ait été volontairement abandonnée, je recommande de la réintroduire.

### 6.2. WASM : il faut préciser ce qui est réellement exclu

N-8 exclut WASM et le navigateur comme cible de production.

Cela clarifie l'absence d'un moteur d'inférence navigateur, mais ne dit pas explicitement si les composants WASM isolés restent envisageables.

Je rappelle notre distinction précédente :

- WASM pour l'inférence navigateur : hors priorité ;
- WASM comme environnement de composants portables et isolés : possibilité future ;
- API de service : indépendante de WASM.

Si tu souhaites supprimer complètement WASM du projet, c'est parfaitement cohérent ; sinon, il faut garder cette distinction dans le document.

### 6.3. La matrice d'API reste vide

L'annexe B est encore constituée d'emplacements vides, alors que R-48 et R-80 sont `STABLE`.

Ce n'est pas bloquant pour P0 à P3, mais ce sera indispensable avant P4.

Il faudra préciser :

- les endpoints ;
- les schémas de messages ;
- les headers ;
- les événements SSE ;
- les tool calls ;
- les erreurs et codes HTTP ;
- les champs acceptés, ignorés ou refusés.

La matrice devrait être considérée comme une spécification à compléter, et non comme une simple annexe informative.

### 6.4. P0 risque d'être trop chargé

P0 demande notamment une session CPU-only complète, plusieurs formats numériques, deux kernels WebGPU, la détection du device, une base de plans à forme fixe et les premières validations numériques.

C'est ambitieux pour un premier jalon.

Surtout, le mode CPU-only n'a pas encore de modèle de référence et de profil de validation explicitement définis.

Il faudrait identifier un modèle dLLM adapté à la validation CPU-only initiale, ou préciser que le jalon P0 valide d'abord l'infrastructure CPU sur un modèle réduit, puis que P1 étend cette couverture au modèle dLLM de référence.

Cela évitera de confondre preuve de fonctionnement du backend CPU et validation de toutes les architectures.

# 7. Nouvelle notation

Voici comment je réévalue le document.

| Dimension                           | Ancienne note | Nouvelle note |
| ----------------------------------- | ------------- | ------------- |
| Vision et priorité dLLM             | 8,5           | 9             |
| Cohérence AR/dLLM                   | 4,5           | 8,5           |
| Architecture des ordonnanceurs      | 4             | 8,5           |
| Rôles Rust/Zig/WebGPU/CPU           | 7             | 8             |
| Exigences et critères d'acceptation | 7             | 8             |
| Maturité des exigences              | 4             | 6,5           |
| Fondement scientifique              | 5,5           | 7             |
| Précision technique                 | 5             | 6,5           |
| Traçabilité des sources             | 5,5           | 6             |
| Exploitabilité par un agent IA      | 4,5           | 8             |

# 7,5 / 10

Progression : +1,5 point

Le document est passé d'une spécification ambitieuse mais contradictoire à une base d'architecture et d'implémentation nettement plus cohérente.

La progression est réelle. La note reste limitée principalement par les erreurs numériques, les incohérences de classification des modèles et la fiabilité du registre de sources.

## 8. Mon verdict pour le développement

Je considère que cette version peut désormais servir de base de travail à l'agent IA, mais je distingue deux niveaux d'autorisation.

Ce que je considère désormais suffisamment solide :

- concevoir les contrats généraux entre Rust, Zig et WebGPU ;
- définir la distinction entre stratégies AR et dLLM ;
- concevoir les deux ordonnanceurs coordonnés ;
- structurer le modèle de mémoire par rôle ;
- établir les interfaces de validation typées ;
- préparer les mécanismes d'admission et d'observabilité ;
- travailler sur les premiers prototypes d'exécution CPU et GPU.

Ce que je ne laisserais pas encore l'agent implémenter sans arbitrage ou correction :

- le décodage des formats de poids avec échelles E8M0 ;
- les politiques de génération WeDLM/FLUID généralisées aux modèles natifs ;
- les modes dual-stream sans cible réelle ;
- les chemins Clover et DiffusionGemma avec une classification et un contrat d'attention génériques ;
- les optimisations MoE obligatoires qui n'ont pas encore démontré leur rentabilité ;
- les dépendances dont les références ne sont pas fiables.

### Mes priorités finales

P0 — Bloquants techniques

À corriger avant exécution

Corriger E8M0, les noms de fonctionnalités WebGPU, la définition des niveaux de maturité et les références canoniques erronées.

P1 — Cohérence du moteur dLLM

Avant intégration des modèles

Corriger R-16, clarifier dual-stream, séparer famille de génération et topologie d'architecture, définir les profils d'attention propres à chaque modèle, conditionner p_cache et n-gram.

P2 — Complétude du produit

Avant industrialisation

Clarifier R-37, réintroduire l'API de contrôle si elle est toujours souhaitée, stabiliser le périmètre WASM, remplir la matrice d'API et définir les références CPU-only.

## 9. Conclusion

Je suis nettement plus favorable à cette nouvelle version.

L'agent a bien intégré les corrections les plus structurantes. Il a notamment corrigé les deux problèmes qui menaçaient le plus la cohérence du projet : l'unification excessive AR/dLLM et la fusion des deux ordonnanceurs.

Il a aussi correctement intégré tes décisions sur le CPU, la conversion externe des modèles et le choix de `Niko1221/Strata`.

Mais je souhaite attirer ton attention sur un point : un document normatif n'est pas fiable simplement parce qu'il est très détaillé et utilise beaucoup de `MUST`. Il est fiable lorsque chaque obligation est correcte, applicable à un périmètre précis et vérifiable.

La prochaine amélioration ne doit donc pas consister à ajouter encore des exigences. Elle doit consister à éliminer les dernières ambiguïtés et à renforcer la justesse des prescriptions existantes.

Je recommande de corriger les points P0 avant de lancer un développement substantiel du moteur. Après ces corrections, je pense que le document pourrait atteindre un niveau de référence de l'ordre de 8,5/10, suffisamment solide pour guider un agent IA de développement sur des jalons bien délimités.

Mon appréciation finale : la nouvelle spécification est un progrès sérieux. L'ambition d'Alethe.ia est mieux protégée qu'auparavant, et je ne recommande pas de refaire le document — seulement de le fiabiliser sur les points encore identifiés.