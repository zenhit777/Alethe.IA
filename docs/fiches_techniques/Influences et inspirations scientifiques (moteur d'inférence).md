# Fiche technique — Influences et inspirations scientifiques du moteur d'inférence

**Périmètre :** 
Exploitation des travaux FLARE, WeDLM, FLUID et REPR-ALIGN dans la conception du moteur d'inférence.

**Orientation prioritaire :** 
Modèles de langage à diffusion (dLLM).

**Objectif :** 
Identifier les mécanismes scientifiques et techniques susceptibles de produire des synergies au sein d'un moteur unique, capable de servir des architectures de diffusion natives, hybrides et issues de modèles autorégressifs.


**Précautions de lecture (contraintes vérifiées)** :
- **Précondition commune** : WeDLM, FLARE et REPR-ALIGN présupposent un checkpoint préparé en **dual-stream** (flux `clean` / `noisy`), un modèle de bruit, un prédicteur d'inputs masqués, une fonction de loss et un checkpoint entraîné sur les deux flux. MUST déclarer cette précondition et refuser les checkpoints qui ne la remplissent pas ; NEVER parler de « checkpoint bilingue ».
- **WeDLM** est le seul nom (jamais « WeDLLM ») ; son adaptateur causal strict suppose décodage causal, préfixe contigu et logits disponibles pour les positions masquées.
- **GDN = Gated DeltaNet** (`arxiv:2412.06464`, dépôt `NVlabs/GatedDeltaNet`), jamais « Gated Differential Network » : l'état récurrent `S ⁽ ᴿ ⁾` / `S ⁽ ᴿ⁺¹ ⁾` et le rewind à `S ⁽ ʳ ⁾` de la fiche §6 reposent sur cette formulation.
- **Checkpoint réduit de P0 / P1a** : `GSAI-ML/LLaDA-8B-Instruct` (`arxiv:2502.09992`, MIT, dépôt `ML-GSAI/LLaDA`), exercé en GGUF Q8_0. dLLM natif à génération par blocs, token de masque dédié ; **le checkpoint ne déclare pas la taille de bloc** → paramètre de configuration consigné. NEVER le confondre avec la cible de référence de P1b (DiffusionGemma 26B).
- **Δ du stride (FLUID)** : `stride = 1 + Δ` est cité par l'article sans définition de Δ, de son domaine ni de sa conversion en nombre de positions. MUST l'écrire avec la référence d'équation source avant de l'implémenter ; à défaut, la politique est une **hypothèse de recherche**.
- **τ (FLUID)** est un seuil de confiance par position ; sa définition exacte et son domaine MUST être cités depuis l'article source.
- **Nature des reprises** : chaque mécanisme repris est une `Source` (équation, protocole, format), une `Adaptation` (translation aux structures d'Alethe.ia) ou une `Hypothèse de recherche` (politique proposée par Alethe.ia). La matrice de traçabilité est §13.2 de `_Alethe.IA.md`.

---

# 1\. Vision directrice

Le moteur s'inspire de quatre travaux complémentaires qui abordent des problèmes différents de la génération par diffusion :

- **WeDLM :** organisation des tokens, réordonnancement topologique, engagement progressif et réutilisation du préfixe.
- **FLUID :** adaptation dynamique de la progression du débruitage en fonction de l'incertitude.
- **REPR-ALIGN :** préservation des représentations linguistiques acquises par un modèle autorégressif lors de son adaptation à la diffusion.
- **FLARE :** intégration des architectures hybrides, coexistence des objectifs AR et diffusion, et optimisation conjointe des mécanismes d'entraînement et d'inférence.

Ces travaux ne doivent pas être reproduits comme quatre implémentations indépendantes.

Leur intérêt réside dans l'extraction de mécanismes complémentaires pouvant être réunis dans une architecture commune.

Le moteur doit notamment chercher à combiner :

1. La liberté d'organisation des tokens.
2. L'adaptation dynamique du débruitage.
3. La réutilisation des calculs et des états de mémoire.
4. La prise en charge des architectures hybrides.
5. La possibilité d'exploiter des modèles issus de l'adaptation AR vers diffusion.
6. La réduction des étapes séquentielles et du travail de calcul inutile.

# 2\. Influence de WeDLM — Organisation topologique et génération continue

**Nature de l'inspiration :** Organisation des séquences, gestion des blocs, cache et ordonnancement.

## 2.1. Mécanismes étudiés

WeDLM démontre qu'une génération de type diffusion peut être réalisée avec une attention strictement causale.

Son mécanisme central, le *Topological Reordering*, consiste à dissocier l'ordre logique des tokens de leur disposition physique dans la séquence traitée.

Les tokens déjà observés sont déplacés vers le préfixe physique, tandis que les positions encore masquées sont placées après eux. Les positions logiques originales sont conservées.

Cette organisation permet aux positions masquées d'accéder aux tokens observés au moyen d'une attention causale standard.

Le second mécanisme majeur est le *Streaming Parallel Decoding* :

- les tokens suffisamment confiants sont engagés progressivement ;
- le préfixe validé s'agrandit ;
- les nouvelles positions masquées alimentent une fenêtre glissante ;
- le moteur évite d'attendre systématiquement la stabilisation complète d'un bloc.

## 2.2. Inspirations à intégrer

**A. Séparation entre position logique et position physique**

Le moteur ne doit pas confondre :

- la position d'un token dans la séquence finale ;
- son emplacement physique dans un bloc ;
- son emplacement dans les structures de mémoire ;
- son état de validation.

Cette séparation permet d'envisager différentes organisations physiques sans modifier arbitrairement la sémantique de la séquence.

**B. Réorganisation des blocs et des tokens**

Le moteur doit pouvoir organiser les tokens selon les contraintes du calcul, sous réserve de respecter les dépendances d'attention et les règles de positionnement propres au modèle.

**C. Engagement progressif**

La validation d'un token ne doit pas nécessairement attendre celle de toutes les positions du bloc.

Les tokens stabilisés peuvent être engagés individuellement lorsque le modèle et son algorithme de génération le permettent.

**D. Fenêtre glissante**

Les positions libérées par les tokens validés peuvent être réutilisées pour alimenter la génération.

Cette approche permet de réduire les périodes d'inactivité entre les blocs.

**E. Compatibilité avec le cache de préfixe**

Le moteur doit rechercher les conditions permettant de réutiliser immédiatement les calculs associés aux tokens validés.

Cette réutilisation dépend cependant du régime d'attention et des dépendances du modèle. Elle ne peut pas être généralisée sans vérification aux modèles bidirectionnels.

## 2.3. Synergie recherchée

Combiner le réordonnancement topologique avec le débruitage adaptatif de FLUID.

L'objectif est de permettre au moteur de choisir simultanément :

- quelles positions traiter ;
- dans quel ordre physique les organiser ;
- quelles positions raffiner ;
- quels tokens engager ;
- quand alimenter de nouvelles positions.

**Apport attendu :** réduire les attentes entre les passes, améliorer la réutilisation des calculs et maintenir une charge de travail plus régulière.

---

# 3\. Influence de FLUID — Débruitage à horizon adaptatif

**Nature de l'inspiration :** Stratégie de débruitage et ordonnancement adaptatif.

## 3.1. Mécanismes étudiés

FLUID introduit les *Elastic Horizons*, une méthode qui adapte dynamiquement les pas de débruitage selon la densité d'information locale, estimée notamment par l'entropie.

Plutôt que d'imposer une progression uniforme, le mécanisme adapte l'horizon de débruitage aux caractéristiques des prédictions.

FLUID s'appuie également sur une organisation strictement causale pour faciliter l'adaptation de modèles AR pré-entraînés à la génération par diffusion.

## 3.2. Inspirations à intégrer

**A. Budget de raffinement variable**

Le nombre de passes de débruitage ne doit pas nécessairement être uniforme pour toutes les régions d'une séquence.

Le moteur doit pouvoir exploiter des stratégies dans lesquelles le raffinement dépend de l'incertitude et de la stabilité des prédictions.

**B. Horizon de débruitage dynamique**

Le scheduler doit pouvoir adapter la progression du débruitage en fonction des caractéristiques locales du bloc.

**C. Décisions fondées sur l'information**

L'incertitude prédictive, la confiance des tokens et la stabilité des blocs doivent pouvoir alimenter les décisions du scheduler.

Ces indicateurs doivent rester distincts : ils ne mesurent pas exactement la même propriété.

**D. Adaptation de la largeur de travail**

Le moteur doit pouvoir étudier la variation de la largeur des fenêtres actives et du nombre de positions traitées à chaque passe.

Cette adaptation doit être compatible avec les contraintes architecturales du modèle.

## 3.3. Synergie recherchée

Combiner les horizons élastiques de FLUID avec l'engagement progressif et la fenêtre glissante de WeDLM.

Le scheduler pourrait :

- identifier les positions les plus stables ;
- engager celles qui satisfont les critères de validation ;
- adapter le budget de raffinement des positions restantes ;
- réutiliser les emplacements libérés ;
- maintenir une activité continue du moteur.

**Apport attendu :** éviter de dépenser le même effort de calcul sur des tokens présentant des niveaux d'incertitude très différents.

L'objectif n'est pas uniquement de diminuer le nombre de passes, mais de maximiser la quantité de tokens utiles obtenus pour chaque unité de calcul.

---

# 4\. Influence de REPR-ALIGN — Préservation des représentations linguistiques

**Nature de l'inspiration :** Adaptation des modèles et constitution du catalogue de modèles.

## 4.1. Mécanismes étudiés

REPR-ALIGN propose de préserver les représentations internes d'un modèle AR pré-entraîné pendant son adaptation à la diffusion.

Le modèle AR d'origine est conservé comme référence gelée. Le modèle de diffusion est entraîné avec :

- un objectif standard de débruitage masqué ;
- une fonction d'alignement des représentations cachées avec celles du modèle AR, couche par couche.

L'alignement repose sur la similarité cosinus des états cachés.

Les auteurs montrent que cette approche permet de réutiliser une partie des représentations linguistiques déjà apprises, et rapportent une accélération de l'entraînement pouvant atteindre 4× dans leurs expériences.

## 4.2. Inspirations à intégrer

REPR-ALIGN ne fournit pas directement un mécanisme d'inférence à intégrer dans le moteur.

Son intérêt concerne principalement la manière dont les modèles peuvent être préparés avant leur déploiement.

**A. Préserver les connaissances acquises**

L'adaptation d'un modèle AR vers la diffusion peut chercher à préserver les représentations linguistiques existantes plutôt que de les réapprendre intégralement.

**B. Favoriser la conversion de modèles existants**

Le moteur doit pouvoir accueillir des modèles dLLM issus de modèles AR pré-entraînés, sans supposer qu'ils ont été développés nativement pour la diffusion.

**C. Séparer les capacités du modèle de son mode d'exécution**

Le moteur doit distinguer :

- un checkpoint AR natif ;
- un checkpoint dLLM natif ;
- un checkpoint AR adapté à la diffusion ;
- un checkpoint entraîné pour plusieurs modes de génération.

Ces modèles peuvent partager une architecture ou une origine commune sans posséder les mêmes capacités d'inférence.

**D. Préserver la compatibilité des architectures**

Les mécanismes d'adaptation doivent tenir compte des architectures, des dimensions et des représentations attendues par le modèle.

## 4.3. Synergie recherchée

Combiner l'approche de préservation des représentations de REPR-ALIGN avec l'approche multi-objectifs de FLARE.

REPR-ALIGN apporte une piste pour préserver le savoir linguistique d'origine.

FLARE apporte une piste pour apprendre conjointement les comportements AR et diffusion.

Ces deux approches peuvent inspirer une stratégie de préparation de modèles qui cherche à conserver les capacités du modèle source tout en lui donnant accès à plusieurs modes de génération.

**Apport attendu :** élargir les possibilités de constitution du catalogue de modèles dLLM, en facilitant l'exploitation de bases AR existantes.

Les objectifs d'alignement et les opérations d'entraînement restent hors du chemin d'inférence en production.

---

# 5\. Influence de FLARE — Architectures hybrides et coexistence des modes de génération

**Nature de l'inspiration :** Architecture du modèle, gestion de la mémoire, exécution hybride et mutualisation des modes de génération.

## 5.1. Mécanismes étudiés

FLARE propose une méthode d'adaptation des modèles AR à attention hybride vers la diffusion.

Ces architectures peuvent associer plusieurs mécanismes, notamment :

- attention softmax ;
- attention linéaire ou récurrente, telle que Gated DeltaNet.

FLARE utilise deux vues complémentaires des blocs pendant l'entraînement :

- une vue propre, avec un objectif AR causal ;
- une vue bruitée, avec un objectif de diffusion et une attention bidirectionnelle au niveau du bloc.

Le modèle apprend ainsi à prendre en charge deux modes de génération avec un même checkpoint :

- génération AR avec vérification spéculative ;
- génération par débruitage parallèle.

Le travail présente également des optimisations spécifiques à la gestion des états récurrents et aux calculs des différentes vues.

## 5.2. Inspirations à intégrer

**A. Prise en charge des architectures hybrides**

Le moteur ne doit pas supposer que toutes les couches d'un modèle utilisent le même mécanisme d'attention.

Il doit pouvoir exécuter des architectures associant plusieurs mécanismes de calcul et de mémoire.

**B. Gestion de plusieurs types d'états**

La mémoire ne doit pas être réduite à un KV cache uniforme.

Le moteur doit pouvoir prendre en charge :

- les états de l'attention softmax ;
- les états récurrents ;
- les états associés aux blocs de diffusion ;
- les états intermédiaires nécessaires aux différentes passes.

**C. Gestion des dépendances entre les flux**

Les architectures hybrides peuvent nécessiter des ordonnancements différents selon les couches et les mécanismes employés.

Le moteur doit pouvoir respecter les dépendances des états récurrents, les limites des blocs et les règles de visibilité des tokens.

**D. Coexistence des modes de génération**

Un modèle adapté peut exposer plusieurs modes d'inférence.

Le moteur doit pouvoir sélectionner le mode approprié sans dupliquer intégralement son infrastructure d'exécution.

**E. Optimisation des calculs intermédiaires**

Les mécanismes de FLARE concernant les états récurrents et les calculs fusionnés constituent une source d'inspiration pour réduire les transferts mémoire et les recalculs.

## 5.3. Synergie recherchée

Combiner les mécanismes de mémoire hybride de FLARE avec le réordonnancement et l'engagement progressif de WeDLM.

Le moteur pourrait gérer un ensemble de blocs dont les états dépendent simultanément :

- des tokens validés ;
- des tokens provisoires ;
- des états attentionnels ;
- des états récurrents.

Les opérations de validation et de réorganisation devraient déclencher uniquement les mises à jour nécessaires.

**Apport attendu :** rendre possible l'exécution efficace de dLLM construits sur des architectures hybrides, sans imposer une représentation mémoire uniforme à tous les modèles.

---

# 6\. Synergies transversales

Les quatre travaux ne constituent pas quatre axes indépendants. Leur combinaison permet d'envisager plusieurs mécanismes nouveaux.

## 6.1. Génération continue et débruitage adaptatif

**WeDLM + FLUID**

Association de l'engagement progressif, de la fenêtre glissante et des horizons élastiques.

Le moteur pourrait adapter le travail de débruitage au niveau d'incertitude tout en alimentant continuellement les blocs actifs.

L'objectif est de limiter les périodes d'attente et d'éviter les passes inutiles.

## 6.2. Mémoire hybride et organisation topologique

**FLARE + WeDLM**

Association de la gestion des états récurrents et attentionnels avec la séparation entre ordre logique et ordre physique des tokens.

Cette synergie pourrait permettre de gérer des séquences réorganisées tout en préservant les dépendances propres aux architectures hybrides.

Elle nécessitera une gestion explicite des états et de leurs frontières de validité.

## 6.3. Adaptation des modèles et double mode d'inférence

**REPR-ALIGN + FLARE**

Association de la préservation des représentations linguistiques et de l'apprentissage conjoint des objectifs AR et diffusion.

Cette combinaison constitue une piste pour préparer des modèles qui conservent davantage de capacités de leur checkpoint d'origine tout en acquérant une génération parallèle par diffusion.

Elle concerne le cycle de préparation des modèles, pas le traitement de chaque requête.

## 6.4. Scheduler sensible à l'incertitude et à la mémoire

**FLUID + FLARE + WeDLM**

Le scheduler pourrait prendre ses décisions en considérant simultanément :

- l'incertitude des positions ;
- leur stabilité ;
- les possibilités d'engagement ;
- le coût des recalculs ;
- les dépendances des états récurrents ;
- les possibilités de réutilisation du cache.

Il ne piloterait donc pas uniquement le nombre de passes de débruitage, mais l'ensemble du travail utile à effectuer.

## 6.5. Une abstraction commune sans uniformisation des modèles

**Enseignement transversal des quatre travaux**

Les modèles peuvent partager une infrastructure d'exécution sans partager :

- les mêmes masques d'attention ;
- les mêmes mécanismes de mémoire ;
- les mêmes stratégies de débruitage ;
- les mêmes règles de validation ;
- les mêmes objectifs d'entraînement.

Le moteur doit mutualiser les mécanismes compatibles, et non imposer une architecture artificiellement uniforme.

---

# 7\. Traduction des inspirations en capacités du moteur

| Capacité recherchée | Inspiration | Nature de l'intégration |
| --- | --- | --- |
| Réordonnancement logique et physique | WeDLM | Exécution |
| Engagement progressif des tokens | WeDLM | Exécution |
| Fenêtre glissante | WeDLM | Ordonnancement |
| Réutilisation du préfixe | WeDLM | Mémoire |
| Horizon de débruitage adaptatif | FLUID | Ordonnancement |
| Mesure de l'incertitude locale | FLUID | Décision |
| Préservation des représentations AR | REPR-ALIGN | Préparation des modèles |
| Alignement des états cachés | REPR-ALIGN | Entraînement |
| Attention hybride | FLARE | Exécution |
| Mémoire récurrente et attentionnelle | FLARE | Mémoire |
| Objectifs AR et diffusion conjoints | FLARE | Entraînement |
| Modes AR et diffusion sur un checkpoint | FLARE | Contrat du modèle et exécution |
| Optimisation des états et des transferts | FLARE | Exécution et kernels |

# 8\. Principes de conception qui en découlent

Les influences scientifiques retenues conduisent aux principes suivants :

1. **L'ordre des tokens doit être dissocié de leur disposition physique.** Le moteur doit pouvoir organiser les données selon les besoins de l'exécution sans altérer leur sémantique.
2. **Le débruitage doit pouvoir être adaptatif.** Le nombre de passes et la progression des blocs ne doivent pas nécessairement être uniformes.
3. **La validation doit être progressive lorsque le modèle le permet.** Les tokens stabilisés doivent pouvoir être engagés et leurs calculs réutilisés sans attendre inutilement la fin du bloc.
4. **La mémoire doit être polymorphe.** Le moteur doit prendre en charge les mécanismes attentionnels et récurrents, ainsi que les états intermédiaires des modèles.
5. **Les architectures hybrides doivent être natives dans la conception.** Elles ne doivent pas être considérées comme des cas particuliers à traiter ultérieurement.
6. **La préparation des modèles doit être distincte de leur exécution.** Les méthodes de conversion et d'alignement inspirent notre écosystème de modèles sans alourdir le chemin d'inférence.
7. **La mutualisation doit être fonctionnelle, pas artificielle.** Le moteur partage les ressources et les mécanismes d'exécution compatibles tout en respectant les particularités de chaque architecture.

---

# 9\. Périmètre d'intégration et priorités

Les inspirations ne doivent pas toutes être intégrées au même niveau ni au même moment.

### Priorité 1 — Fondations du moteur

- Gestion indépendante des positions logiques et physiques.
- Gestion des tokens provisoires et validés.
- Prise en charge de plusieurs régimes d'attention.
- Gestion de plusieurs types d'états mémoire.
- Scheduler capable de piloter les étapes de débruitage.

### Priorité 2 — Optimisation du débit

- Engagement progressif des tokens.
- Fenêtres glissantes.
- Adaptation des horizons de débruitage.
- Réutilisation des états et des calculs.
- Réduction des transferts mémoire et des recalculs.

### Priorité 3 — Écosystème de modèles

- Prise en charge des modèles AR adaptés à la diffusion.
- Prise en charge des modèles hybrides.
- Exploitation de checkpoints possédant plusieurs modes de génération.
- Intégration éventuelle de pipelines de conversion inspirés de REPR-ALIGN et FLARE.

---

# 10\. Conclusion

FLARE, WeDLM, FLUID et REPR-ALIGN constituent quatre sources d'inspiration complémentaires.

WeDLM apporte une nouvelle manière d'organiser les tokens et de faire progresser leur validation.

FLUID apporte une manière d'adapter le débruitage aux caractéristiques de l'information traitée.

REPR-ALIGN apporte une méthode pour préserver les connaissances linguistiques des modèles AR lors de leur adaptation à la diffusion.

FLARE apporte une vision plus large de l'intégration entre architectures hybrides, objectifs de génération et modes d'inférence.

Leur combinaison suggère un moteur capable de dépasser le simple enchaînement de passes de débruitage.

**L'ambition est de construire un moteur qui organise intelligemment les tokens, adapte le travail de débruitage, exploite les états mémoire disponibles et maximise la réutilisation des calculs, tout en respectant les propriétés propres à chaque modèle.**

La synergie recherchée n'est donc pas la juxtaposition de quatre algorithmes, mais la création d'une architecture d'inférence dont les mécanismes sont mutuellement optimisés.