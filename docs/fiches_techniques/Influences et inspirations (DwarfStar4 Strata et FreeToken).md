# Fiche technique — Influences et inspirations de DwarfStar4, Strata et FreeToken

**Périmètre :** Inspirations architecturales et techniques issues de DwarfStar4, Strata et FreeToken pour la conception de notre moteur d'inférence.

**Orientation prioritaire :** Exécution efficace des modèles de diffusion (dLLM), avec prise en charge des modèles autorégressifs et des architectures hybrides.

**Objectif :** Extraire les mécanismes de gestion des ressources, d'exécution, de mémoire, de chargement et de validation susceptibles d'améliorer les performances, la fiabilité et la portabilité de notre moteur.

---

# 1. Vision directrice

Les trois projets apportent des inspirations complémentaires.

- **DwarfStar4 :** spécialisation du moteur, intégration verticale, maîtrise du cycle de vie du modèle, persistance des états d'inférence et validation rigoureuse.
- **Strata :** gestion des modèles dépassant la mémoire GPU ou RAM disponible, placement des poids sur plusieurs niveaux de mémoire, respect strict des caractéristiques du checkpoint et ordonnancement conscient des coûts de transfert.
- **FreeToken :** exécution adaptative sur ressources hétérogènes, co-exécution CPU/GPU, streaming, double buffering, cache d'experts et réallocation dynamique de la mémoire GPU.

Ces inspirations doivent être combinées avec les mécanismes de génération par diffusion étudiés dans FLARE, WeDLM, FLUID et REPR-ALIGN.

Leur combinaison doit permettre de construire un moteur qui ne se contente pas d'optimiser les opérations de calcul : il doit également optimiser les conditions dans lesquelles ces opérations sont exécutées.

Les objectifs principaux sont :

1. Exploiter efficacement les ressources matérielles disponibles.
2. Réduire les transferts et les attentes entre CPU, GPU et mémoire système.
3. Adapter le placement des poids, des états et des caches aux besoins réels de l'inférence.
4. Prendre en charge les modèles volumineux et les architectures MoE.
5. Préserver la fidélité du modèle et de ses paramètres d'exécution.
6. Maintenir une exécution continue et efficace pendant les différentes étapes de débruitage.
7. Valider les performances et la correction à l'échelle du système complet.

# 2. Influence de DwarfStar4 — Spécialisation, intégration verticale et maîtrise de l'inférence

**Nature de l'inspiration :** Architecture du moteur, gestion du cycle de vie des modèles, persistance et validation.

## 2.1. Mécanismes étudiés

DwarfStar4 est un moteur d'inférence natif développé autour d'un ensemble volontairement restreint de modèles. Il privilégie une intégration étroite entre le chargement des modèles, leur exécution, la gestion du contexte, le cache KV, le serveur et les outils de validation.

Son approche repose notamment sur plusieurs principes :

- Une implémentation spécialisée plutôt qu'un exécuteur universel cherchant à prendre en charge toutes les architectures.
- Une connaissance approfondie du format et de l'organisation des modèles ciblés.
- Une gestion du cache KV qui peut s'étendre au stockage sur disque.
- Le streaming depuis le SSD lorsque la mémoire disponible ne permet pas de conserver tous les poids.
- Une validation des résultats à partir des logits de référence.
- Une intégration cohérente entre chargement, exécution, persistance et service des requêtes.

DwarfStar4 montre également qu'un moteur peut exploiter plusieurs ressources matérielles et plusieurs machines lorsque cela est nécessaire, notamment par parallélisme tensoriel et pipeline.

## 2.2. Inspirations à intégrer

**A. Spécialisation sans fragmentation du moteur**

Le moteur doit disposer d'un cœur commun, mais ne doit pas chercher à traiter toutes les architectures de manière artificiellement uniforme.

Chaque famille de modèles doit pouvoir bénéficier d'un chemin d'exécution optimisé en fonction de ses caractéristiques.

Cette spécialisation doit notamment concerner :
- les opérations de calcul ;
- les formats de poids ;
- les mécanismes d'attention ;
- les structures de mémoire ;
- les stratégies de débruitage ;
- les opérations de chargement et de transfert.

L'objectif est de conserver une architecture commune sans renoncer aux optimisations propres aux modèles.

**B. Intégration verticale du cycle d'inférence**

Le chargement du modèle, la préparation du contexte, l'exécution, la gestion de la mémoire, la validation des résultats et le service des requêtes doivent être conçus comme les parties cohérentes d'un même système.

Une optimisation ne doit pas être évaluée uniquement au niveau d'un kernel isolé.

Il faut considérer son effet sur le chemin complet de l'inférence.

**C. Persistance des états d'inférence**

DwarfStar4 montre l'intérêt de considérer les états d'inférence comme des données persistantes, et pas uniquement comme des structures temporaires en mémoire vive.

Pour notre moteur, cette inspiration peut s'étendre :
- aux états du contexte ;
- aux caches attentionnels ;
- aux états récurrents ;
- aux blocs de diffusion en cours ;
- aux points de reprise d'une génération.

La persistance doit cependant respecter les dépendances et les règles de validité propres à chaque architecture.

**D. Mémoire secondaire comme prolongement de la mémoire d'inférence**

Le stockage SSD peut devenir un niveau de mémoire exploitable pour les poids et certains états persistants.

Il ne s'agit pas de considérer le SSD comme une extension équivalente de la VRAM : sa latence et sa bande passante imposent une stratégie de placement et de préchargement adaptée.

**E. Validation intégrée**

La correction des résultats doit être contrôlée à plusieurs niveaux :
- chargement et interprétation des poids ;
- calculs intermédiaires ;
- logits ;
- comportement du décodeur ;
- résultats complets des requêtes.

Le moteur doit pouvoir comparer ses résultats à des références indépendantes et reproduire les conditions de test.

## 2.3. Synergies recherchées

**DwarfStar4 + WeDLM**

Associer la persistance des états d'inférence à l'engagement progressif des tokens.

Les tokens stabilisés et les états de génération associés pourraient devenir des points de reprise réutilisables.

**DwarfStar4 + FLARE**

Associer la gestion intégrée des états d'inférence à la prise en charge des architectures hybrides.

Le moteur pourrait persister et restaurer les états attentionnels et récurrents sans supposer qu'ils sont interchangeables.

**DwarfStar4 + FLUID**

Associer une validation intégrée à un scheduler adaptatif.

Les performances du débruitage pourraient être évaluées en fonction de la qualité réellement obtenue, du nombre de passes et du coût de calcul.

**Apport attendu :** un moteur spécialisé dans l'exécution efficace des modèles ciblés, capable de maîtriser l'ensemble du cycle d'inférence et de vérifier ses résultats.

---

# 3. Influence de Strata — Placement intelligent des ressources et fidélité du modèle

**Nature de l'inspiration :** Gestion de la mémoire, placement des poids, exécution MoE et ordonnancement conscient des transferts.

## 3.1. Mécanismes étudiés

Strata est un moteur C++/CUDA conçu pour exécuter des modèles qui ne tiennent pas nécessairement dans la VRAM, voire dans la RAM disponible.

Il répartit les composants d'un checkpoint entre :
- la mémoire GPU ;
- la mémoire système ;
- le stockage NVMe.

Son approche cherche à préserver les caractéristiques déclarées par le modèle, notamment :
- les formats de quantification ;
- les paramètres de routage ;
- le nombre d'experts ;
- le top-k ;
- les caractéristiques des caches et des prédicteurs.

Un principe particulièrement intéressant est la distinction entre l'ordonnancement du travail et la modification du travail lui-même : un prédicteur peut modifier l'ordre d'exécution, mais ne doit pas modifier silencieusement les calculs requis.

Strata présente également une approche explicite de l'admission des modèles : déterminer si un modèle peut être exécuté dans les conditions demandées, et signaler clairement les situations où l'exécution devient dépendante des transferts depuis le stockage.

## 3.2. Inspirations à intégrer

**A. Gestion hiérarchique de la mémoire**

Le moteur doit considérer les différentes mémoires comme des ressources complémentaires, avec des coûts et des caractéristiques distincts.

Le placement des données doit pouvoir être adapté aux besoins de l'inférence.

Les catégories concernées comprennent :
- les poids du modèle ;
- les experts MoE ;
- les états attentionnels ;
- les caches KV ;
- les états récurrents ;
- les blocs de diffusion actifs ;
- les buffers temporaires.

**B. Placement conscient du coût de transfert**

Le moteur doit distinguer le coût d'un calcul de celui du déplacement des données nécessaires à ce calcul.

Une opération GPU plus rapide n'est pas nécessairement avantageuse si elle impose des transferts supplémentaires ou une préparation coûteuse.

Le placement doit chercher à rapprocher les données du lieu où elles seront consommées.

**C. Admission et planification avant exécution**

Avant de charger un modèle ou de lancer une requête, le moteur doit pouvoir évaluer :
- la mémoire nécessaire ;
- les ressources disponibles ;
- les contraintes du modèle ;
- les transferts prévisibles ;
- les risques de saturation ;
- les conditions d'exécution acceptables.

Il doit pouvoir distinguer une exécution entièrement résidente d'une exécution dépendante du streaming.

**D. Fidélité des paramètres du checkpoint**

Les optimisations ne doivent pas modifier silencieusement la sémantique du modèle.

Les formats numériques, les paramètres de routage, le nombre d'experts et les mécanismes de calcul doivent rester conformes au checkpoint et au contrat du modèle.

Une optimisation de performance doit être évaluée sans confondre accélération et dégradation de la qualité.

**E. Ordonnancement sans altération du travail demandé**

Les prédicteurs et heuristiques peuvent aider à choisir l'ordre des opérations, à anticiper les transferts ou à préparer les données.

Ils ne doivent pas modifier le résultat attendu du modèle, sauf si une approximation explicite est autorisée par le mode d'exécution.

**F. Spécialisation des kernels en fonction du véritable goulot d'étranglement**

Strata montre que l'optimisation d'un kernel dépend du régime matériel.

Un modèle dense résident en VRAM peut être limité par le calcul ou la bande passante GPU.

Un modèle MoE dont les experts sont transférés depuis la RAM peut être limité par le staging, le PCIe ou le calcul CPU.

Il ne faut donc pas appliquer systématiquement la même optimisation à toutes les architectures.

## 3.3. Synergies recherchées

**Strata + FreeToken**

Associer le placement hiérarchique des données à une allocation adaptative des ressources.

Le moteur pourrait déterminer quels composants conserver en VRAM, lesquels placer en RAM et lesquels streamer depuis le stockage, puis ajuster ces décisions selon la charge réelle.

**Strata + FLARE**

Associer le placement conscient des transferts à la gestion de plusieurs types d'états mémoire.

Les états récurrents et attentionnels pourraient bénéficier de stratégies de placement différentes, selon leur fréquence d'accès et leur coût de reconstruction.

**Strata + FLUID**

Associer le scheduler de débruitage à une estimation du coût matériel des opérations.

Le choix d'une stratégie de raffinement pourrait tenir compte non seulement de l'incertitude des tokens, mais aussi du coût de déplacement des données et de la disponibilité des ressources.

**Apport attendu :** exécuter des modèles volumineux avec une utilisation rigoureuse des ressources, en préservant autant que possible la fidélité du checkpoint et en évitant les optimisations qui déplacent simplement le goulot d'étranglement.

---

# 4. Influence de FreeToken — Exécution adaptative et exploitation des ressources hétérogènes

**Nature de l'inspiration :** Co-exécution CPU/GPU, streaming, cache d'experts, double buffering et mémoire dynamique.

## 4.1. Mécanismes étudiés

FreeToken est un moteur de serving MoE conçu pour exploiter des ressources hétérogènes sur des machines personnelles et des stations de travail.

Il traite les GPU, CPU, mémoire système et interconnexions comme un ensemble de ressources complémentaires.

Ses mécanismes les plus intéressants comprennent :

- Une politique de co-exécution CPU/GPU adaptée à la bande passante disponible, appelée q*.
- Le streaming de prefill par couches avec double buffering.
- Un cache global d'experts utilisant une politique LRU.
- Une exécution compatible avec les graphes de calcul.
- Un format de poids optimisé, FTW.
- Des checkpoints sémantiques pour les états récurrents et les caches KV.
- Une réallocation dynamique de la VRAM entre cache d'experts et mémoire KV.

## 4.2. Inspirations à intégrer

**A. Co-exécution CPU/GPU adaptative**

Le moteur doit pouvoir exploiter simultanément plusieurs unités de calcul lorsque cela présente un avantage.

La répartition du travail doit tenir compte :
- de la bande passante disponible ;
- du coût des transferts ;
- de la capacité de calcul ;
- de la taille des opérations ;
- de la latence d'exécution.

Une politique adaptative peut choisir de privilégier le GPU, le CPU ou leur combinaison en fonction du régime d'exécution.

Cette capacité doit être conçue de manière compatible avec notre objectif de portabilité GPU via WebGPU.

**B. Double buffering et streaming**

Le moteur doit chercher à superposer le transfert des données et leur consommation.

Pendant qu'un buffer est utilisé pour les calculs, le suivant peut être préparé ou transféré lorsque les dépendances le permettent.

Cette technique peut concerner :
- les poids ;
- les experts ;
- les blocs de données ;
- les entrées et sorties des kernels ;
- les buffers temporaires du débruitage.

L'objectif est de limiter les périodes durant lesquelles les unités de calcul attendent les données.

**C. Cache global des experts**

Les modèles MoE nécessitent une gestion particulière des experts.

Le moteur doit pouvoir conserver en mémoire les experts les plus susceptibles d'être réutilisés et évincer ceux qui présentent une utilité moindre.

Une politique LRU constitue une base, mais elle pourrait être complétée par des informations sur :
- la fréquence d'activation ;
- les séquences de routage ;
- les requêtes concurrentes ;
- le coût de rechargement ;
- la taille des experts.

**D. Réallocation dynamique de la VRAM**

La mémoire GPU ne doit pas être répartie de manière immuable entre les différentes catégories de données.

Le moteur doit pouvoir ajuster l'allocation entre :
- les poids et experts résidents ;
- les caches KV ;
- les états récurrents ;
- les buffers de débruitage ;
- les buffers de transfert.

Cette réallocation doit être réalisée sans redémarrage du moteur ni rechargement complet du modèle lorsque cela est techniquement possible.

**E. Checkpoints sémantiques**

FreeToken montre l'intérêt de conserver des points de référence associés à des états cohérents du contexte.

Dans notre moteur, cette idée pourrait être étendue aux blocs de diffusion et aux états de génération.

Un checkpoint doit identifier précisément :
- la séquence logique ;
- les tokens validés ;
- les positions provisoires ;
- les états de mémoire ;
- les paramètres de génération ;
- les dépendances nécessaires à la reprise.

Il ne suffit pas de sauvegarder des tenseurs : il faut pouvoir garantir qu'ils correspondent au même état logique de génération.

**F. Exécution compatible avec les graphes de calcul**

Le moteur doit chercher à limiter les coûts de lancement et de préparation des opérations répétitives.

Les graphes de calcul peuvent être particulièrement intéressants pour les étapes récurrentes du débruitage, à condition que les formes, les allocations et les dépendances soient suffisamment prévisibles.

**G. Formats de poids adaptés à l'exécution**

Le format des poids peut influencer directement les coûts de chargement, de stockage et de calcul.

L'approche FTW de FreeToken constitue une inspiration pour concevoir ou adopter des représentations optimisées pour les chemins d'exécution ciblés.

Il faut néanmoins préserver une séparation entre format de stockage, format de transfert et représentation consommée par les kernels.

## 4.3. Synergies recherchées

**FreeToken + Strata**

Associer l'allocation dynamique de la mémoire GPU à un placement hiérarchique des données.

Le moteur pourrait déplacer ou streamer les composants selon les besoins de la charge, plutôt que de figer leur emplacement au démarrage.

**FreeToken + WeDLM**

Associer les checkpoints sémantiques et la réallocation mémoire à l'engagement progressif des tokens.

Les blocs validés pourraient devenir des points de cohérence permettant de libérer ou de réaffecter des ressources sans perdre la possibilité de poursuivre la génération.

**FreeToken + FLUID**

Associer le débruitage adaptatif à la co-exécution CPU/GPU.

Le scheduler pourrait tenir compte du coût matériel des étapes de raffinement et adapter le travail à la bande passante disponible.

**FreeToken + FLARE**

Associer le cache d'experts et la gestion dynamique de la VRAM à la coexistence de plusieurs mécanismes de mémoire.

Les états récurrents, les caches attentionnels et les experts pourraient être gérés dans un même système d'allocation, tout en conservant des politiques adaptées à leurs propriétés.

**Apport attendu :** maintenir une exécution efficace malgré des ressources limitées ou hétérogènes, en superposant autant que possible calcul, transfert et préparation des données.

---

# 5. Synergies transversales entre les trois projets

Les mécanismes des trois moteurs deviennent particulièrement intéressants lorsqu'ils sont associés.

## 5.1. Placement intelligent et allocation dynamique

**Strata + FreeToken + DwarfStar4**

Combiner :
- la hiérarchie de mémoire de Strata ;
- la réallocation dynamique de FreeToken ;
- la persistance et le streaming de DwarfStar4.

Le moteur pourrait gérer les ressources comme un ensemble de niveaux complémentaires :

- VRAM pour les données fréquemment utilisées et les opérations critiques ;
- RAM pour les poids et états qui doivent rester rapidement accessibles ;
- stockage secondaire pour les données qui ne nécessitent pas une résidence permanente.

L'allocation serait ajustée en fonction de la charge réelle.

## 5.2. Streaming et exécution continue du débruitage

**FreeToken + DwarfStar4 + WeDLM + FLUID**

Combiner :
- le double buffering ;
- le streaming des données ;
- l'engagement progressif ;
- les fenêtres glissantes ;
- le débruitage adaptatif.

Le moteur pourrait préparer les prochaines données pendant que les opérations courantes sont exécutées, puis exploiter les tokens stabilisés pour alimenter les blocs suivants.

L'objectif est de réduire les périodes d'attente et de maintenir une activité régulière des ressources de calcul.

## 5.3. Mémoire adaptative pour architectures hybrides

**FreeToken + Strata + FLARE**

Associer :
- la gestion des ressources hétérogènes ;
- le placement hiérarchique ;
- la mémoire attentionnelle et récurrente.

La mémoire devient un système dynamique capable de gérer plusieurs catégories d'états avec des besoins différents.

Cette approche est particulièrement intéressante pour les modèles hybrides associant attention softmax, attention linéaire et mécanismes récurrents.

## 5.4. Prédiction des coûts et ordonnancement intelligent

**Strata + FreeToken + FLUID**

Le scheduler ne doit pas prendre ses décisions uniquement en fonction de l'algorithme de génération.

Il pourrait également considérer :
- le coût des transferts ;
- la disponibilité de la VRAM ;
- les cache misses ;
- la bande passante CPU/GPU ;
- le coût des étapes de raffinement ;
- les possibilités de superposition des opérations.

Le choix du prochain travail à effectuer serait donc guidé à la fois par les besoins du modèle et par les conditions matérielles.

## 5.5. Persistance, cohérence et validation

**DwarfStar4 + FreeToken + Strata**

Combiner :
- la persistance des états ;
- les checkpoints sémantiques ;
- la validation stricte des caractéristiques du modèle ;
- la capacité à restaurer une exécution.

L'objectif est de permettre la reprise d'une génération sans ambiguïté sur la cohérence entre poids, contexte, cache et états intermédiaires.

---

# 6. Traduction des inspirations en capacités du moteur

| Capacité recherchée | Inspiration principale | Apport complémentaire |
|---|---|---|
| Spécialisation par architecture | DwarfStar4 | Strata |
| Intégration du cycle complet d'inférence | DwarfStar4 | FreeToken |
| Persistance des états | DwarfStar4 | FreeToken |
| Streaming depuis le stockage | DwarfStar4 | Strata |
| Validation des logits et résultats | DwarfStar4 | Strata |
| Placement hiérarchique des poids | Strata | FreeToken |
| Admission selon les ressources disponibles | Strata | DwarfStar4 |
| Respect des paramètres du checkpoint | Strata | DwarfStar4 |
| Ordonnancement conscient des transferts | Strata | FreeToken |
| Co-exécution CPU/GPU | FreeToken | Strata |
| Double buffering | FreeToken | DwarfStar4 |
| Cache global d'experts | FreeToken | Strata |
| Réallocation dynamique de VRAM | FreeToken | Strata |
| Checkpoints sémantiques | FreeToken | DwarfStar4 |
| Exécution compatible avec les graphes | FreeToken | DwarfStar4 |
| Gestion de formats de poids spécialisés | FreeToken | Strata |

---

# 7. Principes de conception qui en découlent

Les trois projets conduisent aux principes suivants :

1. **Spécialisation contrôlée :** disposer d'un moteur commun sans renoncer aux chemins d'exécution adaptés aux architectures ciblées.

2. **Intégration verticale :** traiter le chargement, le calcul, la mémoire, la persistance et la validation comme les composantes d'un même système.

3. **Mémoire hiérarchique :** considérer VRAM, RAM et stockage secondaire comme des ressources complémentaires, avec des politiques de placement et de transfert adaptées.

4. **Allocation dynamique :** permettre aux différentes catégories de données de partager les ressources selon les besoins réels de l'inférence.

5. **Exécution adaptative :** choisir les ressources et l'ordre des opérations en fonction de la bande passante, de la latence et des dépendances.

6. **Superposition des opérations :** utiliser le streaming et le double buffering pour réduire les attentes entre transferts et calculs.

7. **Fidélité du modèle :** ne pas modifier silencieusement les formats numériques, le routage, le nombre d'experts ou les paramètres du checkpoint.

8. **Persistance cohérente :** pouvoir sauvegarder et restaurer des états d'inférence complets, avec leurs dépendances.

9. **Validation intégrée :** mesurer la correction et les performances au niveau du système complet, et non uniquement au niveau des kernels.

10. **Optimisation fondée sur les mesures :** identifier le véritable goulot d'étranglement avant de développer une optimisation matérielle ou algorithmique.

---

# 8. Périmètre d'intégration et priorités

### Priorité 1 — Fondations de gestion des ressources

- Inventaire des ressources matérielles disponibles.
- Planification du placement des poids et des états.
- Gestion hiérarchique VRAM/RAM/stockage.
- Admission des modèles selon les contraintes matérielles.
- Respect des caractéristiques du checkpoint.

### Priorité 2 — Optimisation de l'exécution

- Streaming et double buffering.
- Co-exécution CPU/GPU lorsque pertinente.
- Cache d'experts.
- Réallocation dynamique de la mémoire.
- Ordonnancement conscient des transferts.
- Exécution compatible avec les graphes de calcul.

### Priorité 3 — Fiabilité et persistance

- Checkpoints cohérents des états d'inférence.
- Sauvegarde et restauration des contextes.
- Validation des logits et des résultats.
- Tests d'intégration du cycle complet d'inférence.
- Mesures reproductibles des performances.

---

# 9. Conclusion

DwarfStar4, Strata et FreeToken apportent trois dimensions essentielles à notre projet.

DwarfStar4 nous inspire une approche intégrée, spécialisée et rigoureuse de l'inférence, dans laquelle le modèle, ses états et sa validation sont maîtrisés de bout en bout.

Strata nous inspire une gestion hiérarchique des ressources, une attention particulière aux coûts de transfert et un respect strict des caractéristiques des modèles.

FreeToken nous inspire une exécution adaptative sur ressources hétérogènes, avec streaming, double buffering, cache d'experts et allocation dynamique de la mémoire.

Leur combinaison permet d'envisager un moteur qui ne cherche pas uniquement à accélérer les calculs, mais à optimiser l'ensemble du chemin d'inférence.

**L'ambition est de construire un moteur capable d'exploiter intelligemment chaque niveau de mémoire, chaque unité de calcul et chaque possibilité de parallélisation, tout en préservant la fidélité du modèle et en maintenant un flux de génération efficace.**

Dans le contexte de notre priorité donnée aux dLLM, ces mécanismes doivent être adaptés au cycle de débruitage : les blocs actifs, les états provisoires, les tokens validés et les mécanismes de mémoire doivent participer à la même stratégie d'allocation et d'ordonnancement.

La synergie recherchée est donc la suivante : **combiner la maîtrise intégrée de DwarfStar4, la rigueur de placement de Strata et l'élasticité d'exécution de FreeToken pour construire une infrastructure matérielle au service d'un moteur de génération prioritairement conçu pour la diffusion.**