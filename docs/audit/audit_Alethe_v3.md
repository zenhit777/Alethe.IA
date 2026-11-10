Le nouveau document montre une évolution importante : plusieurs problèmes structurels du précédent audit sont désormais traités dans le texte normatif, et pas simplement ajoutés sous forme de notes.

J’ai notamment vérifié que la distinction entre maturité de spécification et état d’implémentation est maintenant explicite, que les deux ordonnanceurs restent séparés, et que l’AR et la diffusion partagent une infrastructure sans être forcés dans un algorithme de génération unique.

Le document réintroduit aussi explicitement l’API de contrôle (R-92 à R-96), ce qui rétablit une décision d’architecture importante. Je vérifie maintenant si les corrections numériques, les profils de modèles et les références sont cohérents jusque dans les sections détaillées.
Verdict : 8,3/10
8,3/10
Progression nette

Je le situe désormais au niveau d'une spécification d'architecture sérieuse, exploitable pour commencer le développement par lots bornés. Il n'est pas encore totalement cohérent au point de pouvoir confier à un agent une implémentation autonome et transversale.
1. Les corrections importantes sont bien intégrées
Point du précédent audit	Verdict
Encodage E8M0	Corrigé
Détection des fonctionnalités WebGPU	Corrigé
Deux axes de maturité	Corrigé
Clover natif, pas adapté	Corrigé
Checkpoints dual-stream sans cible réelle	Encadré en FUTURE
Profils d'attention par modèle	Corrigé
p_cache et fenêtre WeDLM	Mieux circonscrits
Tables n-gram conditionnelles	Corrigé
Offload CPU conditionné par le coût	Corrigé
API de contrôle	Réintroduite
Conversion externe	Clarifiée

La correction E8M0 est particulièrement importante : 0x00 est bien conservé comme \(2^{-127}\), et 0xFF est traité comme NaN avec refus explicite du checkpoint. C'est une correction de fond, pas une simple amélioration rédactionnelle.
2. Les incohérences qui restent
Je relève quatre points à traiter.

A. §5.1 réintroduit l'ancien problème du type Mode

Le document établit correctement en §2.5 et §7.1 trois dimensions indépendantes :
- generation_family
- architecture_topology
- generation_capabilities
Mais §5.1 contient encore :

Mode = AR | Diffusion | Hybride

C'est contradictoire avec la règle de §7.1 qui précise que Hybride n'est pas une valeur de generation_family.

Correction : supprimer cette ancienne définition et employer partout les trois dimensions canoniques.

B. Le jalon P1 impose encore p_cache à DiffusionGemma

C'est plus subtil, mais important.
- R-26 réserve explicitement p_cache au profil WeDLM.
- R-91 distingue le profil WeDLM du profil multi-canvas de DiffusionGemma.
- Pourtant, le critère d'acceptation P1 exige encore que p_cache croisse de façon contiguë, alors que P1 vise DiffusionGemma.
Le critère P1 doit tester la progression propre au profil multi-canvas : dénoisage, validation du canvas, ajout au contexte et cohérence des états. Il ne doit pas imposer une métrique propre à WeDLM.

C. Le checkpoint réduit de P0 n'est toujours pas nommé

Le document exige désormais un checkpoint réduit, compatible CPU, tenant en RAM, avec une topologie et un format de poids déclarés.

C'est une bonne contrainte, mais le choix reste à faire. Sans checkpoint concret, le premier jalon risque de devenir un ensemble de tests artificiels qui ne valide pas réellement le chargement et l'exécution d'un modèle.

Il faut fixer un modèle, une révision, un tokenizer, une topologie et un format de poids.

D. La matrice API reste vide

L'Annexe B contient toujours des A COMPLETER.

Ce n'est pas bloquant pour P0 à P3 puisque le document impose explicitement de la compléter avant P4. Mais ce sera un véritable verrou avant de développer les API OpenAI et Anthropic : il faudra spécifier les champs, les événements SSE, les erreurs, les tool calls et les comportements non supportés.
3. Un point de vigilance supplémentaire : les références techniques
Le registre des sources est mieux encadré, mais je ne considérerais pas encore toutes les fiches techniques comme auditées simplement parce que leurs noms de dépôts sont corrigés.

Il reste à vérifier les mécanismes attribués à chaque projet contre des fichiers et fonctions précis de leurs dépôts. Le document a d'ailleurs maintenant une règle qui l'exige : c'est une bonne décision.

4. Mon appréciation globale
Dimension	Note
Vision et priorité dLLM	9,5
Séparation AR / diffusion	9
Architecture des ordonnanceurs	9
Gestion mémoire et CPU	8,5
Fidélité aux modèles de référence	8,5
WebGPU et numérique	8,5
Traçabilité et maturité	8,5
Précision des exigences	7,5
Sources et références	7,5
Préparation à l'implémentation autonome	8

Ma conclusion : le document a franchi un seuil qualitatif. Il ne ressemble plus à une accumulation d'idées techniques et de résultats de recherche ; il commence à former un véritable contrat d'architecture.

Je ne recommande pas une nouvelle refonte. Je recommande une dernière passe de cohérence ciblée, puis de commencer P0.
Ordre de priorité que je retiendrais
1. Corriger Mode = AR | Diffusion | Hybride en §5.1.
2. Corriger le critère P1 qui impose p_cache à DiffusionGemma.
3. Choisir et épingler le checkpoint réduit de P0.
4. Vérifier les références Strata et DwarfStar4 au niveau des fichiers sources.
5. Compléter l'Annexe B avant P4, sans retarder P0.
Le document actuel est consultable ici : Alethe.IA — document normatif.

Mon conseil : ne cherchez plus à enrichir massivement la spécification. À ce stade, le risque principal n'est plus le manque d'idées, mais la dérive entre les exigences, les profils de modèles et leur implémentation. Il faut maintenant faire travailler le document contre du code et des mesures réelles.