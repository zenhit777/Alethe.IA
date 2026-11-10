# Audit de `_Alethe.IA.md`

# 6 / 10

Note globale — aptitude à servir de référence normative à un agent IA

Révisions importantes nécessaires

J'ai examiné la version actuelle de [`_Alethe.IA.md`](https://github.com/zenhit777/Alethe.IA/blob/main/_Alethe.IA.md?utm_source=chatgpt.com), sur la branche `main`, ainsi que les quatre fiches techniques et le fichier `références.txt` auxquels elle renvoie.

Mon avis général : le document est ambitieux, structuré et contient un véritable travail de conception. Il est nettement plus avancé qu'un simple document de vision. Mais je ne le confierais pas encore comme référence normative unique à un agent IA chargé de développer Alethe.ia de manière autonome.

Le problème n'est pas un manque de détails. C'est plutôt l'inverse : certaines hypothèses deviennent des obligations (`MUST`) avant d'avoir été suffisamment démontrées, certaines décisions antérieures ont été modifiées sans clarification explicite et quelques prescriptions techniques sont incorrectes ou irréalisables dans leur formulation actuelle.

Il y a donc un risque important qu'un agent respecte scrupuleusement le document et produise malgré tout une architecture qui ne correspond pas à l'intention du projet.

Périmètre : il s'agit d'un audit documentaire et architectural. Le dépôt consulté contient le document et ses fiches de conception ; je n'ai pas audité une implémentation du moteur.

## 1. Évaluation par dimension

| Dimension                      | Note   | Mon appréciation                                                                                              |
| ------------------------------ | ------ | ------------------------------------------------------------------------------------------------------------- |
| Vision et identité du moteur   | 8,5/10 | La priorité dLLM est clairement affirmée.                                                                     |
| Structure et lisibilité        | 8/10   | Le découpage facilite la navigation et le suivi des exigences.                                                |
| Formalisation des exigences    | 7/10   | Les identifiants R, I et C sont utiles, mais certaines exigences sont trop prescriptives.                     |
| Cohérence architecturale       | 4,5/10 | Plusieurs invariants imposent des unifications discutables.                                                   |
| Fondement scientifique         | 5,5/10 | Les travaux sont bien identifiés, mais certains mécanismes sont généralisés au-delà de leur périmètre.        |
| Précision technique            | 5/10   | Quelques paramètres WebGPU sont mal nommés et certaines capacités ne sont pas portables.                      |
| Traçabilité des sources        | 5,5/10 | Présence d'un index utile, mais plusieurs références sont confondues ou insuffisamment reliées aux exigences. |
| Exploitabilité par un agent IA | 4,5/10 | Le document donne beaucoup d'ordres, mais ne distingue pas toujours les décisions démontrées des hypothèses.  |

La note de 6/10 ne sanctionne donc pas l'ambition du document. Elle évalue sa fiabilité comme contrat technique à partir duquel un agent pourrait directement développer le moteur.

# 2. Ce que je trouve particulièrement réussi

Avant d'aborder les problèmes, plusieurs choix méritent d'être conservés.

### 2.1. La structure normative est une très bonne idée

La séparation entre :

- exigences `R-x` ;
- invariants `I-x` ;
- conventions `C-x` ;
- non-objectifs `N-x` ;
- jalons `P0...P6` ;
- zones non spécifiées ;

est excellente pour un projet développé avec l'assistance d'agents IA.

Elle permet de relier une modification de code à une intention précise. C'est beaucoup plus exploitable qu'une documentation purement descriptive.

L'instruction selon laquelle un agent doit signaler une ambiguïté avant d'écrire le code correspondant est également pertinente.

### 2.2. La séparation entre position logique et position physique

L'invariant `R-8` est particulièrement important.

Il ouvre la voie à des stratégies de réorganisation des tokens, de gestion des fenêtres, de réutilisation du préfixe et d'optimisation mémoire sans confondre ordre de calcul et ordre logique de la séquence.

C'est une abstraction utile, notamment pour les mécanismes de WeDLM.

### 2.3. La gestion des états récurrents

Le document a correctement identifié qu'un état de génération ne se réduit pas toujours à un KV cache.

La partie consacrée au rewind des états récurrents, à la reconstruction du préfixe causal et à la distinction entre états provisoires et états validés est l'un des passages les plus intéressants.

C'est particulièrement pertinent pour les architectures hybrides et les modèles utilisant Gated DeltaNet.

La gestion de ces états doit effectivement être intégrée aux stratégies de validation et de persistance.

### 2.4. La validation comme composante architecturale

Les exigences R-52 à R-57 donnent une place importante à la correction numérique et aux tests de régression.

C'est une excellente orientation.

La comparaison des logits, des états intermédiaires et des décisions de validation permet de distinguer trois choses trop souvent confondues :

- un moteur qui fonctionne techniquement ;
- un moteur qui produit des résultats plausibles ;
- un moteur qui reproduit correctement le comportement attendu du modèle.

Je conserverais cette ambition, tout en clarifiant les mécanismes de validation, comme expliqué plus loin.

### 2.5. L'importance accordée à la mémoire

La hiérarchie VRAM/RAM/NVMe, l'admission des ressources, les transferts masqués par le calcul, les caches et les checkpoints sémantiques constituent une bonne base de travail.

Le document ne réduit pas la performance à la vitesse des kernels. C'est essentiel pour un moteur destiné aux machines grand public.

# 3. Les problèmes architecturaux les plus importants

Ce sont les points qui me préoccupent le plus, car ils peuvent influencer toute l'organisation du moteur.

## 3.1. L'unification AR/dLLM est devenue trop contraignante

Priorité critique

Le document affirme :

> I-1 — Un seul modèle de génération : AR et dLLM sont des configurations du même cycle de blocs.

Il ajoute :

> I-2 — Un seul ordonnanceur.

Et R-15 impose de servir un modèle AR comme le cas dégénéré du cycle dLLM :

```
refinement_steps = 0
block_width = 1
validation = immediate
```

Je comprends parfaitement l'objectif : éviter deux moteurs indépendants et mutualiser les mécanismes autant que possible.

Mais le document transforme une analogie architecturale utile en contrainte d'exécution universelle.

Ce n'est pas ce que nous avions initialement retenu.

Notre discussion avait établi une distinction importante :

- un modèle AR peut être représenté comme une limite particulière d'un modèle de génération par blocs ;
- cette représentation peut permettre de mutualiser des abstractions ;
- cela ne signifie pas que chaque modèle AR doit réellement emprunter la même stratégie d'inférence qu'un dLLM.

Un AR conserve notamment sa logique causale et son KV cache. Un dLLM peut nécessiter une attention bidirectionnelle, des positions masquées et plusieurs étapes de débruitage.

WeDLM illustre une possibilité de décodage diffusion sous attention causale, mais son article explique précisément que cette propriété est au cœur de sa stratégie particulière. Ce n'est pas une démonstration que l'attention bidirectionnelle est universellement inutile.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://arxiv.org\&sz=32)

arxiv.org+1



Ma recommandation : conserver un modèle d'exécution commun, mais ne pas imposer un unique algorithme de génération.

Je distinguerais :

1. un contrat commun d'opérations et d'états ;
2. une stratégie d'inférence dLLM ;
3. une stratégie d'inférence AR ;
4. un système de ressources et d'exécution partagé.

AR peut alors être exposé comme un cas particulier dans certaines interfaces ou abstractions, sans perdre son chemin algorithmique natif.

Ce n'est pas une renonciation à l'unification. C'est une unification qui respecte les différences réelles entre les modèles.

## 3.2. L'unique ordonnanceur fusionne deux responsabilités différentes

Priorité critique

La décision `I-2` impose un seul scheduler qui pilote :

- l'horizon ;
- la fenêtre ;
- le rythme de commit ;
- les ressources matérielles ;
- les réallocations de VRAM.

Or ces décisions ne relèvent pas toutes de la même logique.

L'ordonnanceur algorithmique doit comprendre le modèle : il décide quelles positions résoudre, quelles opérations effectuer et quand valider les tokens.

L'ordonnanceur de ressources doit comprendre la machine : il décide où placer les données, quand transférer les poids, quelles allocations réutiliser et comment répartir le travail entre CPU et GPU.

Ces deux systèmes doivent évidemment coopérer. Ils doivent même partager des états et des métriques. Mais les fusionner dans une seule autorité décisionnelle risque de créer un composant trop complexe, difficile à tester et à faire évoluer.

C'est d'autant plus important que nous avions précédemment retenu le principe de deux ordonnanceurs coordonnés :

- ordonnanceur d'inférence : quoi calculer et quand ;
- ordonnanceur de ressources : où et à quel coût.

Je recommande de revenir à cette distinction.

## 3.3. L'invariant d'une seule validation est excessif

L'invariant I-4 veut qu'un même mécanisme décide de la validité :

- d'un token ;
- d'un préfixe ;
- d'une couche de poids ;
- d'un plan de kernels.

Ces validations ne portent pourtant pas sur les mêmes propriétés.

Un token peut être validé selon une politique probabiliste ou un seuil de confiance. Un checkpoint doit être contrôlé selon son intégrité, ses métadonnées et sa compatibilité. Un plan de kernels doit être validé selon les capacités du device et ses contraintes d'exécution. Une sortie numérique doit être comparée à une référence.

Il faut bien une architecture de validation centralisée au sens de la gouvernance, des outils et des rapports.

Mais il faut plusieurs validateurs spécialisés, avec des critères de correction propres à chaque objet.

Je remplacerais « un seul modèle de validation » par « un système de validation commun, composé de validateurs spécialisés et typés ».

## 3.4. Le document modifie également la place du CPU

Contradiction interne

Trois prescriptions ne sont pas compatibles :

- `I-11` : le CPU est un chemin de référence/debug, jamais de production sauf absence de GPU adapté ;
- `C-9` : le CPU ne doit pas être un chemin de production ;
- `N-7` : ne pas faire du CPU un chemin de production.

Mais `R-37` impose au moteur de calculer les experts manquants sur CPU en parallèle avec le GPU.

Et la vision du produit prévoit explicitement son utilisation sur des machines sans GPU discret.

Cela pose deux questions :

1. Le CPU peut-il participer à l'inférence de production, notamment pour le calcul d'experts MoE ?
2. Le CPU peut-il constituer un chemin complet d'inférence sur une machine sans GPU compatible ?

Il faut répondre séparément à ces deux questions.

Ma recommandation est de distinguer le backend CPU de référence, le CPU offload/hybride en production et le mode CPU-only. Le mode CPU-only peut être hors priorité, mais l'interdire et le permettre simultanément n'est pas acceptable dans une spécification normative.

# 4. La conversion AR vers diffusion n'est pas correctement cadrée

Priorité critique

C'est une contradiction particulièrement importante.

La section des non-objectifs indique :

> N-1 — Ne pas entraîner des modèles.

Mais le document impose ensuite :

- `R-71` : conversion AR vers dLLM par supervision bilingue clean/noisy ;
- `R-72` : supervision de type REPR-ALIGN ;
- `R-73` : masques de blocs et packing au niveau document ;
- `R-74` : logit shift et masquage aléatoire du noisy-stream ;
- `P6` : conversion de checkpoints AR vers dLLM dans le moteur.

Ces fonctions impliquent des mécanismes d'entraînement ou d'adaptation de modèles, pas simplement le chargement d'un checkpoint.

C'est cohérent avec les travaux cités : WeDLM repose sur une adaptation entraînée de modèles AR, et FLARE présente explicitement un cadre de conversion avec supervision clean/noisy et objectifs AR/diffusion.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://arxiv.org\&sz=32)

arxiv.org+2



REPR-ALIGN est également une méthode d'adaptation entraînée par alignement des représentations, et non une simple opération de serving.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://arxiv.org\&sz=32)

arxiv.org



Il faut choisir entre deux périmètres :

Option A — Moteur d'inférence strict

Alethe.ia charge et sert les checkpoints dLLM natifs et adaptés, mais ne les entraîne pas. La conversion est réalisée par des outils externes.

Option B — Écosystème Alethe.ia étendu

Le projet inclut un outil distinct de préparation, d'adaptation et de conversion des checkpoints, avec son propre périmètre, ses dépendances, ses ressources et ses validations.

Je recommande l'option A pour le moteur initial, avec une architecture permettant éventuellement de construire l'option B ultérieurement.

C'est beaucoup plus cohérent avec la priorité absolue à l'inférence dLLM.

# 5. Audit des références scientifiques

Il faut ici distinguer les mécanismes correctement retranscrits des mécanismes dont la portée est devenue trop générale.

## 5.1. WeDLM : bonne retranscription, mais périmètre à limiter

La formule R-25 :

\\[ \widetilde H_i = H_i + \lambda d_i \\]

est bien présente dans l'article. La sélection par seuil favorise les positions proches du début de la fenêtre et améliore la croissance du préfixe contigu réutilisable. La fenêtre glissante est également décrite dans le papier.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://arxiv.org\&sz=32)

arxiv.org+1



C'est donc un point bien documenté.

La difficulté se situe ailleurs : R-25 figure dans les exigences générales du scheduler.

Or cette formule appartient à la stratégie de streaming causale de WeDLM. Elle ne doit pas devenir une obligation générale pour toutes les architectures dLLM.

Pour un modèle qui exploite une attention bidirectionnelle native, la notion de préfixe immédiatement cache-valide n'a pas nécessairement la même portée. Le papier WeDLM explique justement que l'attention bidirectionnelle peut empêcher ce type de réutilisation directe du KV cache.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://arxiv.org\&sz=32)

arxiv.org+1



Correction : déplacer cette règle dans une stratégie `WeDLM-compatible`, ou plus généralement dans un adaptateur de décodage causal qui déclare explicitement ses préconditions.

## 5.2. FLUID : attention aux formules et à la terminologie

La référence FLUID est pertinente. L'article introduit bien les horizons élastiques et une approche d'adaptation strictement causale des modèles AR.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://www.alphaxiv.org\&sz=32)

alphaXiv



En revanche, les exigences R-20 à R-22 deviennent très précises :

- frontière de compétence ;
- seuil de confiance γ ;
- stride `1 + Δ` ;
- confiance moyenne au-dessus du seuil.

Le document ne permet pas immédiatement de savoir si ces formules sont une transcription exacte de l'algorithme FLUID ou une adaptation proposée pour Alethe.ia.

Il manque notamment une définition mathématique complète de Δ, de son domaine de valeurs et de la manière de convertir le stride en nombre entier de positions.

Correction : donner l'équation exacte, le numéro de l'algorithme ou de l'équation source, puis préciser séparément les modifications proposées par Alethe.ia. À défaut, présenter ce contrôleur comme une hypothèse de recherche à valider.

## 5.3. FLARE : très intéressant, mais les modes sont spécifiques aux checkpoints préparés

Le document retranscrit correctement plusieurs idées importantes :

- Diffusion-Trust ;
- AR-Trust ;
- validation spéculative ;
- correction à partir de la loi de proposition ;
- rewind des états GDN ;
- causal replay après finalisation d'un bloc.

L'article distingue effectivement ces deux modes et détaille la nécessité de restaurer l'état récurrent à la frontière d'acceptation.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://arxiv.org\&sz=32)

arxiv.org+3



La difficulté vient de la terminologie :

Le document parle de « checkpoint bilingue ».

C'est une très mauvaise traduction technique. Il ne s'agit pas d'un checkpoint bilingue au sens linguistique, mais d'un modèle entraîné avec deux flux, clean et noisy.

Je recommande :

- `checkpoint dual-stream` ;
- `clean stream` / flux propre ;
- `noisy stream` / flux bruité ;
- `Diffusion-Trust` ;
- `AR-Trust`.

Surtout, ces capacités doivent être déclarées par les modèles qui ont été préparés selon le protocole compatible. Elles ne sont pas exigibles de n'importe quel checkpoint dLLM.

## 5.4. REPR-ALIGN : bonne référence, mauvais emplacement fonctionnel

REPR-ALIGN est correctement identifié dans l'index. Mais son mécanisme doit être séparé du moteur d'inférence.

Je le placerais dans un futur sous-système de préparation de modèles, pas dans le chemin d'exécution de production.

# 6. Audit technique : des problèmes concrets à corriger

C'est ici que je trouve les risques les plus immédiatement dangereux pour un agent de développement.

## 6.1. Des noms de limites WebGPU incorrects

Bloquant pour P0

Dans le jalon P0, le document exige de mesurer notamment :

- `maxBoundStorageBufferSize`
- `maxInvocationsPerInvocation`
- `maxTotalBufferSize`
- `maxBoundUniformBuffers`

Ces noms ne correspondent pas aux propriétés normalisées de WebGPU.

Les noms corrects comprennent notamment :

| Nom utilisé dans le document      | Nom WebGPU correct                  |
| --------------------------------- | ----------------------------------- |
| `maxBoundStorageBufferSize`       | `maxStorageBufferBindingSize`       |
| `maxInvocationsPerInvocation`     | `maxComputeInvocationsPerWorkgroup` |
| `maxTotalBufferSize`              | `maxBufferSize`                     |
| `maxBoundUniformBuffers`          | `maxUniformBuffersPerShaderStage`   |
| `maxStorageBuffersPerShaderStage` | Déjà correct                        |
| `maxComputeWorkgroupStorageSize`  | Déjà correct                        |

Ces propriétés sont définies dans la spécification W3C WebGPU.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://www.w3.org\&sz=32)

w3.org+1



Ce n'est pas une simple question de vocabulaire : un agent qui suit littéralement le cahier des charges risque de rechercher des propriétés inexistantes.

Il faut corriger ces noms avant de commencer P0.

## 6.2. Le document affirme que WGSL ne possède pas de type f16

La section 4.4 indique :

> absence de type f16 dans le langage de shaders

Cette affirmation est incorrecte dans la spécification actuelle.

WGSL définit bien `f16`, mais son utilisation requiert l'activation de l'extension correspondante et le support de la fonctionnalité `shader-f16` par le périphérique.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://gpuweb.github.io\&sz=32)

gpuweb.github.io+1



Il est raisonnable de choisir un chemin portable qui ne dépend pas systématiquement de f16.

Mais il faut remplacer l'affirmation par une règle du type :

> Le chemin de référence doit fonctionner sans f16. Lorsque le périphérique supporte `shader-f16`, une variante spécialisée peut être proposée après validation numérique et benchmark.

C'est beaucoup plus cohérent avec l'agnosticisme matériel.

À noter : la contrainte sur les atomics WGSL en types entiers `i32` et `u32`, sans atomic f32 dans le langage de base, est en revanche correctement identifiée.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://gpuweb.github.io\&sz=32)

gpuweb.github.io



## 6.3. La mémoire « pinned zero-copy » ne peut pas être une garantie WebGPU portable

R-42 impose :

> mémoire pinnée zéro-copy

C'est un problème sérieux.

L'API WebGPU ne promet pas un accès portable à de la mémoire hôte physiquement épinglée et directement exploitable comme mémoire de stockage GPU.

La spécification encadre strictement le mapping des buffers : `MAP_READ` ne peut être combiné qu'avec `COPY_DST`, et `MAP_WRITE` qu'avec `COPY_SRC`. De plus, un buffer mappé n'est pas disponible pour les opérations GPU tant qu'il reste mappé.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://www.w3.org\&sz=32)

w3.org+1



Il faut distinguer :

- réutilisation de buffers ;
- réduction des copies ;
- staging buffers ;
- allocations internes optimisées par le backend ;
- mémoire hôte épinglée, lorsqu'un backend natif le permet ;
- véritable zéro-copy, qui dépend de mécanismes matériels et logiciels spécifiques.

Je déconseille de rendre le zéro-copy obligatoire dans l'architecture portable.

L'exigence devrait être de minimiser les copies et de rendre les transferts explicites. Les optimisations propres à un backend peuvent ensuite être exploitées sans devenir des garanties universelles.

## 6.4. La capture de graphes de calcul

La section 4.4 reconnaît que WebGPU ne fournit pas une capture de graphe équivalente aux graphes CUDA.

C'est bien de l'avoir identifié.

Mais il faut éviter que l'expression « émuler un plan d'exécution » laisse croire qu'un `command encoder`, des pipelines précompilés et des bind groups réutilisés donnent automatiquement les mêmes propriétés qu'un mécanisme natif de graph replay.

Le document devrait distinguer :

- la représentation statique d'un plan ;
- la réutilisation des pipelines et layouts ;
- la réutilisation des buffers ;
- l'encodage des commandes ;
- la soumission GPU ;
- l'éventuel replay natif d'un graphe lorsqu'un backend le permet.

Le chemin WebGPU portable ne doit pas être présenté comme équivalent à CUDA Graphs.

## 6.5. La politique mémoire est trop rigide pour un moteur agnostique

R-30 à R-42 imposent des emplacements assez précis à des catégories de données :

- experts en RAM épinglée ;
- cache d'experts en VRAM ;
- poids quantifiés et checkpoints sur NVMe ;
- tables n-gram ;
- états récurrents et KV ;
- buffers de staging ;
- etc.

Le problème est que ces catégories ne sont pas universelles.

Tous les modèles n'ont pas de GDN, de routeur MoE, de table n-gram, de couche draft ou de KV cache comparable.

En outre, imposer que l'ensemble des experts reste épinglé en RAM peut être incompatible avec la promesse d'exécuter des modèles dont la taille dépasse la mémoire système disponible.

Je recommande une politique de placement fondée sur les rôles des objets, leur fréquence d'accès, leur taille et leur coût de transfert, et non une répartition matérielle fixe pour toute architecture.

Les modèles doivent déclarer leurs besoins ; le scheduler de ressources doit décider où les satisfaire.

# 7. Confusion importante dans les références Strata

Priorité critique — source canonique

Le fichier `références.txt` et le tableau de la section 14 renvoient à :

[`Niko1221/Strata`](https://github.com/Niko1221/Strata?utm_source=chatgpt.com)

Or, dans notre cadrage précédent, le projet Strata que nous avions identifié était :

[`ro99/strata`](https://github.com/ro99/strata?utm_source=chatgpt.com)

Ce ne sont pas les mêmes projets.

Le dépôt `ro99/strata` se présente comme un moteur d'inférence MoE C++20/CUDA conçu pour exécuter des modèles dépassant la VRAM ou la RAM, avec placement VRAM/RAM/NVMe et préservation de la précision et du routage.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub



Le dépôt `Niko1221/Strata` est un autre projet, présenté comme une solution d'inférence locale accessible sur matériel grand public, avec application desktop, CLI et API compatibles OpenAI et Anthropic.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub



Or la fiche technique associée décrit Strata comme un moteur C++/CUDA spécialisé dans le placement mémoire et l'exécution MoE. Cette description correspond au projet `ro99/strata`, pas à l'identité du dépôt actuellement répertorié dans le document.

C'est exactement le type de confusion que tu souhaitais éviter dans la documentation destinée aux agents IA.

Correction impérative : décider du dépôt de référence puis remplacer toutes les occurrences incorrectes, dans :

- `références.txt` ;
- la fiche d'inspirations Strata ;
- le tableau de la section 14 ;
- les renvois éventuels dans les autres fiches.

Il ne faut surtout pas laisser deux projets homonymes sous une seule référence « Strata ».

# 8. D'autres omissions et contradictions avec la vision du projet

## 8.1. Les modèles initiaux ne sont pas nommés

Le document ne nomme pas explicitement DiffusionGemma et Clover comme premières architectures dLLM à prendre en charge.

C'est un manque important.

Pour un document dont la priorité absolue est le dLLM, l'agent doit connaître les premiers modèles cibles, leurs dépôts, leurs configurations et leurs caractéristiques.

Il faut ajouter une section « Matrice de support des modèles » avec pour chaque modèle :

- dépôt et fiche officiels ;
- révision du checkpoint ;
- architecture ;
- mécanismes d'attention ;
- tokenizer ;
- format de poids ;
- stratégie d'inférence ;
- état du support : non commencé, expérimental, partiel, validé.

## 8.2. L'API Anthropic a disparu

Le document ne définit qu'une API OpenAI-compatible, notamment à R-48.

Or nous avions retenu deux interfaces d'inférence :

- OpenAI-compatible ;
- Anthropic-compatible.

Cette décision doit être restaurée ou explicitement révisée.

Il ne suffit pas de dire « compatible Anthropic » : il faudra une matrice de compatibilité comprenant les endpoints, les messages, le streaming, les erreurs, les contenus et les fonctionnalités prises en charge.

## 8.3. WASM a disparu de l'architecture

Le document ne mentionne plus WASM, WASI ni le Component Model.

Dans notre discussion précédente, nous avions retenu WASM comme environnement possible pour des composants portables et isolés, pas comme API de service ni comme passage obligatoire du moteur.

Cette disparition peut être un choix volontaire, mais elle doit être explicitée.

Je recommanderais :

- WASM hors du chemin critique initial ;
- composants WASM uniquement lorsque l'isolation ou la portabilité apporte une valeur réelle ;
- aucune dépendance de l'inférence native à WASM.

## 8.4. Le multi-GPU n'est pas explicitement exclu

La section 15 évoque le multi-device, mais ne formalise pas clairement notre décision : le multi-GPU n'est pas une priorité.

Il faudrait ajouter un non-objectif explicite :

> Le multi-GPU et le calcul distribué sont hors périmètre initial. Leur étude pourra être engagée si l'adoption du produit et les besoins utilisateurs le justifient. ZML pourra alors être étudié comme source d'inspiration potentielle, sans constituer une dépendance retenue.

Cela évitera à un agent d'interpréter la mention du multi-device comme une invitation à construire une infrastructure multi-GPU prématurée.

## 8.5. L'objectif grand public n'est pas entièrement aligné sur les non-objectifs

Le document définit le public comme des intégrateurs, opérateurs et agents IA. Il exclut par ailleurs toute interface graphique.

Ce n'est pas nécessairement mauvais : Alethe.ia peut être d'abord un moteur local, un serveur et un CLI destiné aux utilisateurs techniques.

Mais si l'objectif est une véritable exploitation grand public, il faut préciser le produit livré :

- moteur uniquement ;
- moteur + CLI + API ;
- application desktop avec interface graphique ;
- ou moteur extensible dont l'interface graphique viendra ultérieurement.

La distinction doit être explicite. Sinon, les agents risquent de confondre le public du moteur et celui de l'application finale.

## 8.6. La persistance des sessions demande un volet sécurité

R-43 à R-47 imposent la persistance de caches, KV et états récurrents sur NVMe.

C'est une fonctionnalité pertinente, mais elle implique aussi de stocker des données potentiellement sensibles : contextes, prompts, historiques et états de session.

Le document doit préciser au minimum :

- activation explicite ou automatique ;
- durée de rétention ;
- suppression des sessions ;
- cohérence après crash ;
- permissions des fichiers ;
- chiffrement éventuel ;
- invalidation après changement de modèle ou de version.

La persistance ne doit pas devenir une fonctionnalité opaque pour l'utilisateur.

# 9. Les exigences sont parfois trop ambitieuses pour leur niveau de maturité

Le recours à `MUST` donne au document une grande force normative. Mais toutes les exigences ne sont pas au même niveau de maturité.

Quelques exemples :

| Exigence                                        | Problème                                                                    | Recommandation                                            |
| ----------------------------------------------- | --------------------------------------------------------------------------- | --------------------------------------------------------- |
| R-40 : prédiction des experts pour 1 à 2 tokens | Stratégie MoE spécifique, pas une propriété universelle des dLLM            | Passer en expérimental ou optionnel, avec bénéfice mesuré |
| R-41 : capacité minimale d'un pas complet       | La définition d'un « pas complet » varie selon l'architecture et le routage | Définir une formule dépendante du modèle                  |
| R-42 : mémoire épinglée zéro-copy               | Garantie non portable avec WebGPU                                           | Remplacer par minimisation des copies et staging optimisé |
| R-45 : réutilisation de préfixes entre requêtes | La validité dépend des positions, masques, versions et états                | Définir les clés d'identité et conditions d'invalidation  |
| R-49 : workers et batching                      | La politique de batch peut modifier le coût mémoire et la latence des dLLM  | Définir les critères de regroupement et d'admission       |
| R-60 : nombreux formats numériques              | Le périmètre est très vaste                                                 | Définir les formats P0/P1 et les extensions ultérieures   |
| R-71 à R-74 : conversion AR → dLLM              | Dépasse le périmètre d'un moteur d'inférence strict                         | Séparer l'outillage de conversion                         |

Le problème n'est pas d'avoir des objectifs ambitieux. Il est de leur attribuer le même statut que des propriétés techniques déjà comprises et validées.

Je recommande quatre catégories de maturité supplémentaires :

- `STABLE` : comportement défini et validé ;
- `EXPERIMENTAL` : mécanisme implémentable mais non validé sur l'ensemble des cibles ;
- `RESEARCH` : hypothèse ou mécanisme à étudier ;
- `FUTURE` : capacité hors périmètre courant.

Un agent ne devrait pas interpréter un `MUST` appartenant à une fonction `RESEARCH` comme un ordre de développer immédiatement une implémentation définitive.

# 10. Ce que je modifierais avant de confier le document à un agent

Je ne recommande pas de recommencer la rédaction. La structure générale est bonne. Je recommande une correction ciblée, dans cet ordre.

## P0 — À corriger impérativement avant développement

1. Clarifier la portée de l'unification AR/dLLM : conserver l'infrastructure commune, sans imposer une sémantique unique.
2. Réintroduire les deux ordonnanceurs coordonnés, ou formaliser explicitement la décision de les fusionner.
3. Corriger les contradictions CPU entre I-11, R-37, C-9 et N-7.
4. Résoudre la contradiction entre N-1 et R-71 à R-74.
5. Corriger les identifiants de limites WebGPU et l'affirmation concernant f16.
6. Supprimer la garantie portable de mémoire épinglée zéro-copy.
7. Clarifier la référence canonique Strata : `ro99/strata` ou `Niko1221/Strata`.

## P1 — À corriger avant la première intégration de modèle

1. Définir DiffusionGemma et Clover comme modèles de référence initiaux, avec leurs spécificités architecturales.
2. Restaurer ou réviser explicitement l'API Anthropic.
3. Restaurer ou exclure explicitement WASM du périmètre.
4. Limiter WeDLM, FLUID et FLARE aux checkpoints et stratégies réellement compatibles.
5. Remplacer « checkpoint bilingue » par « checkpoint dual-stream ».
6. Clarifier les conditions de validité du cache, des états récurrents et des checkpoints persistés.

## P2 — À préciser pour industrialiser le développement

1. Définir la matrice des formats de poids réellement prioritaires.
2. Définir les interfaces et ABI exactes entre Rust, Zig et WebGPU.
3. Formaliser les tests de référence et les seuils de tolérance numérique.
4. Ajouter sécurité, rétention et suppression des données persistées.
5. Clarifier la cible grand public et le rôle des interfaces graphiques.
6. Épingler les versions et commits des références externes et dépendances.

## 11. Un point particulièrement important : la traçabilité des exigences

La section 14 donne une bonne base, mais elle doit être plus rigoureuse.

Actuellement, elle associe des groupes entiers d'exigences à des sources externes. Cela ne suffit pas pour un agent de développement.

Je recommande une matrice de ce type :

| ID   | Exigence                            | Source                     | Nature                    | Limite d'application             | Validation                 |
| ---- | ----------------------------------- | -------------------------- | ------------------------- | -------------------------------- | -------------------------- |
| R-25 | Sélection par entropie et distance  | WeDLM, équation 11         | Reprise adaptée           | Décodage causal compatible WeDLM | Test de sélection + cache  |
| R-36 | Masquage du transfert par le calcul | FreeToken                  | Inspiration transposée    | Selon capacités du backend       | Benchmark transfert/calcul |
| R-46 | Cohérence de l'état récurrent       | FLARE                      | Reprise adaptée           | Architectures à état récurrent   | Test de commit/rewind      |
| R-60 | Déquantification FP4                | Source du format + backend | Exigence d'implémentation | Formats explicitement listés     | Référence numérique        |

La colonne Nature est essentielle :

- reprise directe d'un mécanisme ;
- adaptation de mécanisme ;
- décision propre à Alethe.ia ;
- hypothèse à valider.

Cela évite qu'un agent attribue à un article scientifique une décision qui est en réalité une invention ou une extrapolation du projet.

# 12. Ma conclusion

Je considère que `_Alethe.IA.md` est un document de conception sérieux, avec une ambition technique élevée et plusieurs idées architecturales fortes.

Son point fort est de passer de la vision à des exigences identifiables et à des critères de validation.

Son point faible est de confondre parfois trois niveaux de certitude :

1. ce que les modèles et les articles démontrent ;
2. ce qu'Alethe.ia décide d'implémenter ;
3. ce que nous supposons pouvoir obtenir en combinant ces mécanismes.

Cette confusion est particulièrement dangereuse dans un document normatif destiné à un agent IA.

Un agent n'a pas besoin que chaque idée soit déjà résolue. Il a besoin de savoir précisément ce qui est décidé, ce qui est obligatoire, ce qui est expérimental et ce qu'il ne doit pas décider lui-même.

Je garderais donc la structure et la majorité du contenu. Je corrigerais les invariants, les contradictions, les références et les prescriptions techniquement impossibles. Je ne réduirais pas l'ambition d'Alethe.ia ; je rendrais simplement le document plus fiable.

Mon verdict est le suivant :

- Comme document de vision et de recherche architecturale, il est proche d'un bon niveau de maturité.
- Comme cahier des charges technique, il nécessite encore des arbitrages.
- Comme source normative unique pour un agent IA, il n'est pas encore suffisamment sûr.

Je ne donnerais pas encore à un agent l'autorisation d'implémenter librement l'ensemble des exigences R-1 à R-74. Je commencerais par corriger les P0, puis je lui confierais uniquement le jalon P0 avec des critères WebGPU corrigés et vérifiables.

C'est la meilleure manière de protéger la cohérence de l'architecture avant que des choix prématurés ne se transforment en dette technique.

### Références principales utilisées pour l'audit

Le document audité : [\_Alethe.IA.md — dépôt officiel](https://github.com/zenhit777/Alethe.IA/blob/main/_Alethe.IA.md?utm_source=chatgpt.com).

Références techniques vérifiées : [Strata `ro99/strata`](https://github.com/ro99/strata?utm_source=chatgpt.com), [Strata `Niko1221/Strata`](https://github.com/Niko1221/Strata?utm_source=chatgpt.com), [WeDLM — arXiv](https://arxiv.org/abs/2512.22737?utm_source=chatgpt.com), [FLUID — arXiv](https://arxiv.org/abs/2605.27387?utm_source=chatgpt.com), [FLARE — arXiv](https://arxiv.org/abs/2606.01774?utm_source=chatgpt.com), [REPR-ALIGN — arXiv](https://arxiv.org/abs/2605.06885?utm_source=chatgpt.com), [WebGPU — W3C](https://www.w3.org/TR/webgpu/?utm_source=chatgpt.com) et [WGSL — W3C](https://www.w3.org/TR/WGSL/?utm_source=chatgpt.com).