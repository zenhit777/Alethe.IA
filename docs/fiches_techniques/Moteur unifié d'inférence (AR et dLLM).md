# Fiche technique — Moteur unifié d'inférence AR et dLLM

**Objet :** Définition du traitement des modèles autorégressifs (AR) et des modèles de langage à diffusion (dLLM).

**Principe directeur :** Un moteur d'inférence unique capable d'exécuter différents paradigmes de génération, en partageant autant que possible leurs mécanismes d'exécution, de gestion des blocs et d'ordonnancement.

---

## 1. Vision générale

L'application repose sur un moteur de génération unifié, conçu prioritairement pour les modèles de diffusion (dLLM), tout en prenant en charge les modèles autorégressifs (AR).

L'objectif n'est pas de développer deux moteurs indépendants, mais de représenter les deux paradigmes au moyen d'un même processus général de génération.

**Périmètre exact de l'unification (règle normative)** : le processus partagé est **l'infrastructure d'exécution** — contrats d'opérations, représentation des positions et des blocs, hiérarchie de placement mémoire, deux ordonnanceurs (inférence / ressources), protocole de validation (commit / rewind / rejet), APIs, formats de poids, instrumentation. **Ce n'est pas un algorithme de génération unique.** L'AR est servi par son chemin causal natif (`ValidatorPrefixe`), la diffusion par son chemin de dénoisage (`ValidatorBloc`), la spéculation par `ValidatorSpéculatif` ; NEVER traiter l'AR comme une spécialisation booléenne du dénoisage, NEVER une seconde boucle AR non typée. La conversion AR → dLLM est un **outillage externe** : le moteur charge les checkpoints adaptés, il ne convertit pas et n'entraîne pas.

La différence fondamentale réside dans le traitement des tokens :

- **AR :** les tokens sont prédits séquentiellement, sans étape de débruitage, puis immédiatement validés.
- **dLLM :** plusieurs tokens peuvent être prédits simultanément, puis affinés au cours de plusieurs étapes de débruitage avant leur validation.

Le traitement AR peut ainsi être considéré comme un cas limite du processus général de génération par blocs : aucun raffinement, validation immédiate.

Cette unification concerne le moteur d'exécution. Elle ne signifie pas que tous les modèles possèdent les mêmes capacités ou peuvent employer indifféremment les mêmes stratégies de génération.

## 2. Modèle conceptuel commun

Le moteur organise la génération autour de blocs de tokens.

Chaque bloc suit un cycle de vie général :

1. Initialisation du bloc.
2. Exécution du modèle.
3. Éventuelles étapes de raffinement.
4. Évaluation de l'état des tokens.
5. Validation des tokens répondant aux critères d'engagement.
6. Intégration des tokens validés au contexte de génération.
7. Préparation du bloc suivant.

Le nombre d'étapes de raffinement, la largeur du bloc et les conditions de validation sont configurables selon les capacités du modèle et la stratégie de génération.

### Paramètres conceptuels

| Paramètre | Description |
|---|---|
| Largeur du bloc | Nombre de positions traitées dans le bloc actif. |
| Étapes de raffinement | Nombre de passes successives de débruitage. |
| Politique d'attention | Causale, bidirectionnelle ou hybride, selon le modèle. |
| Politique de validation | Immédiate, progressive ou par bloc. |
| Révision | Possibilité de modifier les tokens provisoires avant validation. |
| Gestion du contexte | Conservation et réutilisation des informations déjà calculées. |

## 3. Traitement des modèles AR

### 3.1 Principe

Un modèle autorégressif prédit les tokens successivement, en conditionnant chaque nouvelle prédiction sur les tokens précédemment générés.

Son traitement correspond au mode sans raffinement du moteur commun.

### 3.2 Cycle d'inférence

1. Le moteur prépare le contexte d'entrée.
2. Le modèle effectue une passe d'inférence pour prédire le token suivant.
3. Le token prédit est immédiatement validé.
4. Le token rejoint le contexte de génération.
5. Le moteur poursuit avec la prédiction suivante.

### 3.3 Caractéristiques

- Génération séquentielle.
- Attention généralement causale.
- Une passe de génération par token produit.
- Aucune étape de débruitage.
- Aucune révision des tokens déjà validés.
- Réutilisation du contexte et du KV cache lorsque l'architecture du modèle le permet.

### 3.4 Représentation dans le moteur commun

| Paramètre | Valeur conceptuelle |
|---|---|
| Raffinement | 0 étape |
| Largeur de génération | 1 token |
| Validation | Immédiate |
| Révision | Désactivée |
| Ordonnancement | Séquentiel |
| Attention | Selon l'architecture du modèle, généralement causale |

Le mode AR constitue ainsi le cas particulier où chaque bloc contient un seul token et où celui-ci est validé sans raffinement.

**Important :** cette représentation est une abstraction d'exécution. 
Elle ne transforme pas un modèle AR conventionnel en modèle de diffusion et ne lui confère pas la capacité de prédire correctement plusieurs tokens masqués en parallèle.

## 4. Traitement des modèles dLLM

### 4.1 Principe

Un modèle de diffusion linguistique génère une séquence en travaillant sur plusieurs positions simultanément.

Au lieu de produire systématiquement un seul token après l'autre, il traite des blocs contenant des positions masquées ou provisoires, puis améliore progressivement leurs prédictions.

Le débruitage peut nécessiter plusieurs passes successives. Les tokens suffisamment stabilisés sont ensuite validés et intégrés au contexte de génération.

### 4.2 Cycle d'inférence

1. Le moteur prépare le contexte et initialise un bloc de génération.
2. Les positions du bloc sont initialisées selon la stratégie de masquage du modèle.
3. Le modèle prédit simultanément les tokens des positions actives.
4. Le moteur évalue les prédictions et détermine les positions à conserver ou à réviser.
5. Le débruitage se poursuit pendant le nombre d'étapes nécessaire.
6. Les tokens répondant aux critères de validation sont engagés.
7. Les tokens validés sont intégrés au contexte ou au préfixe de génération.
8. Le moteur poursuit avec les positions restantes ou le bloc suivant.

### 4.3 Caractéristiques

- Génération parallèle de plusieurs positions.
- Débruitage potentiellement multi-étapes.
- Possibilité de réviser les tokens provisoires.
- Validation progressive ou par bloc.
- Ordonnancement fondé notamment sur la confiance des prédictions.
- Réutilisation des calculs lorsque la structure du modèle et la stratégie d'inférence le permettent.

### 4.4 Représentation dans le moteur commun

| Paramètre | Valeur conceptuelle |
|---|---|
| Raffinement | Une ou plusieurs étapes |
| Largeur de génération | Plusieurs tokens |
| Validation | Progressive ou par bloc |
| Révision | Autorisée selon le modèle |
| Ordonnancement | Parallèle et itératif |
| Attention | Bidirectionnelle, causale ou hybride |

Le moteur doit notamment pouvoir gérer les dépendances entre les positions actives, les états provisoires et les tokens déjà validés.

## 5. Attention : une capacité indépendante du paradigme de génération

Le moteur ne doit pas imposer un régime d'attention unique.

L'attention est une propriété de l'architecture du modèle et doit être prise en charge indépendamment du nombre d'étapes de raffinement.

Les principaux régimes à considérer sont :

- **Causal :** chaque position ne peut accéder qu'aux positions autorisées par l'ordre causal.
- **Bidirectionnel :** les positions d'un bloc peuvent exploiter le contexte disponible des deux côtés.
- **Hybride :** certaines parties du modèle utilisent une attention causale, tandis que d'autres utilisent une attention bidirectionnelle ou un mécanisme distinct.

Cette flexibilité est essentielle pour prendre en charge des architectures telles que DiffusionGemma et Clover.

Elle permet également d'étudier les approches de diffusion causale illustrées par WeDLM et FLUID, sans faire de l'attention causale une contrainte générale du moteur.

## 6. Gestion des tokens et des blocs

Le moteur doit distinguer trois états fondamentaux :

| État | Signification |
|---|---|
| Non initialisé / masqué | Position dont le contenu reste à prédire. |
| Provisoire | Token prédit, mais encore susceptible d'être révisé. |
| Validé | Token définitivement engagé dans la séquence. |

La distinction entre tokens provisoires et validés est centrale.

Elle permet de mutualiser le cycle de vie des tokens entre AR et dLLM, tout en conservant des politiques de validation différentes.

Dans le mode AR, la transition de prédiction à validation est immédiate.

Dans le mode dLLM, un token peut traverser plusieurs étapes de prédiction et de révision avant d'être validé.

Une fois validé, le token peut rejoindre le contexte historique. Les calculs associés peuvent être conservés ou réutilisés lorsque cela est compatible avec le mécanisme d'attention et de mémoire du modèle.

## 7. Ordonnancement et exécution

Le moteur doit permettre différents niveaux de parallélisme :

- Parallélisme entre les positions d'un même bloc.
- Parallélisme entre plusieurs blocs lorsque leurs dépendances le permettent.
- Parallélisme entre plusieurs requêtes.
- Exécution de requêtes ayant des états de génération et des budgets de raffinement différents.

Le scheduler doit tenir compte de la nature des opérations à effectuer, de l'état des blocs et des ressources disponibles.

Pour les modèles AR, il privilégie l'enchaînement rapide des prédictions et la réutilisation du contexte.

Pour les dLLM, il cherche à maximiser le travail utile réalisé pendant chaque passe de débruitage, tout en limitant les recalculs et les étapes de synchronisation inutiles.

## 8. Objectif architectural

L'unification doit permettre de mutualiser :

- Le cycle de vie des blocs.
- La gestion des tokens et de leurs états.
- L'ordonnancement des opérations.
- La gestion des requêtes concurrentes.
- Les mécanismes de mémoire et de cache compatibles avec les modèles.
- L'exécution des graphes de calcul.
- La gestion des ressources matérielles.

En revanche, les éléments suivants doivent rester configurables ou spécifiques aux architectures :

- Les masques d'attention.
- Les mécanismes de débruitage.
- Les critères de confiance et de validation.
- Les stratégies de révision.
- Les mécanismes de mémoire récurrente ou attentionnelle.
- Les règles de positionnement des tokens.

## 9. Synthèse

Le moteur est conçu comme un système général de génération par blocs, dont le comportement dépend des capacités du modèle et de sa stratégie d'inférence.

| Propriété | Modèle AR | Modèle dLLM |
|---|---|---|
| Paradigme | Autorégressif | Diffusion |
| Traitement | Séquentiel | Parallèle et itératif |
| Raffinement | 0 étape | Plusieurs étapes possibles |
| Bloc actif | Généralement un token | Plusieurs tokens |
| Révision | Non | Oui, selon le modèle |
| Validation | Immédiate | Progressive ou par bloc |
| Attention | Généralement causale | Causale, bidirectionnelle ou hybride |
| Moteur d'exécution | Commun | Commun |

**Principe fondateur :**

Le moteur ne doit pas être conçu comme un moteur AR auquel on ajoute la diffusion. 
Il doit être conçu prioritairement pour la diffusion, avec une abstraction de génération suffisamment générale pour traiter également l'autorégressif comme un cas particulier : 
	zéro étape de raffinement, génération unitaire et validation immédiate.

Cette approche vise à maximiser la mutualisation des mécanismes d'exécution sans sacrifier les spécificités architecturales des modèles servis.