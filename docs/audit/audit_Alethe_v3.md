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

---

## Vérification du 10 novembre 2026 (corps figé ci-dessus)

Le corps de cet audit n'est **pas** modifié ; cette section consigne l'état de chacun des points qu'il relevait, vérifié à la source le 10 novembre 2026.

| Point de l'audit | État au 10 nov. 2026 | Preuve |
|---|---|---|
| **A. `Mode = AR \| Diffusion \| Hybride` en §5.1** | **clos** | la définition a été supprimée ; `grep` sur le document ne trouve plus la ligne. Les trois dimensions canoniques de §7.1 (`generation_family`, `architecture_topology`, `generation_capabilities`) sont seules employées. `Hybride` n'est **pas** une valeur de `generation_family` |
| **B. `p_cache` imposé à DiffusionGemma au jalon P1** | **clos** | P1a : « `p_cache` n'est **pas** un critère de complétion », progression mesurée par **positions dénoisées par pas** et **nombre de pas jusqu'à complétion du canva**. P1b : « `p_cache` mesuré comme métrique, **jamais** comme critère de complétion ». R-26 réserve `p_cache` au **seul profil WeDLM** |
| **C. Checkpoint réduit de P0 non nommé** | **clos** | nommé : **`UT-IFML/dQwen3.5-0.8B-Base`**, `sha` de la révision épinglable (`main` `1b6ae942…`, `step25000-swa` `006d0b07…`, `step50000-swa` `9a69f0e2…`, lus le 10 nov. 2026), tokenizer `Qwen2Tokenizer` (vocabulaire 248 320, mask `<\|fim_middle\|>` = `248061`), topologie **18 `linear_attention` + 6 `full_attention`** (`config.json` lu), format **BF16** (`model.safetensors` = 1 504 822 808 octets). `ADR-001` |
| **D. Matrice API de l'Annexe B vide** | **ouvert, volontairement** | la matrice porte **12 lignes `A COMPLETER`**. Les **endpoints** sont consignés (`POST https://api.anthropic.com/v1/messages`, `POST https://api.openai.com/v1/chat/completions`). **Zones 6 et 23 de §14 = A TRANCHER avant P4.** NEVER interpréter une ligne `A COMPLETER` comme une fonctionnalité implémentée : c'est un **refus explicite** (R-82) |
| **Vigilance sur les références techniques** | **clos sur les références lues, ouvert sur les relectures non exercées** | vérifié à la source le 10 nov. 2026 : les **8 cartes de modèles** (API `200`, `id` / `sha` / `safetensors.parameters` / `license` / `base_model` lus), les fichiers `config.json`, `tokenizer_config.json`, `tokenizer.json`, `README.md`, `LICENSE`, `modeling_dqwen3_5.py`, la structure ggml de Q8_0 ; **les trois dépôts d'influence lus par tarball** (`Niko1221/Strata` 35 148 702 octets, `FlashML-org/FreeToken` 3 158 952 octets, `antirez/ds4` 16 400 610 octets, `ro99/strata` pour la homonymie, `DiffuSpace/Clover-35B-A3B-Preview` 5 670 277 octets). **Le rattachement des mécanismes empruntés aux fichiers précis de leurs dépôts est exercé** (`ADR-012`, zone 25). **Fait nouveau** : les `curl` du 25 octobre 2026 qui renvoyaient `000` renvoient **`200`** ce jour (`github.com`, `raw.githubusercontent.com`, `codeload.github.com`, `api.github.com`, `arxiv.org`) ; NEVER citer `000` comme état actuel. **Reste non exercé** : la relecture des articles arxiv (`2512.22737`, `2606.01774`, `2608.16157`, `2412.06464`, `2502.09992`, `2508.19982`) — MUST les relire avant toute modification normative qui s'y appuie |

**Ce que l'audit ne prévoyait pas et qui est constaté depuis** : (1) les comptages de paramètres et d'octets utilisés antérieurement pour dQwen (0.8B et 2B) étaient **faux** et sont corrigés ; (2) le token de masque des deux checkpoints dQwen est **`248061` (`<\|fim_middle\|>`)** — la valeur `248068` est `<think>`, `248077` n'existe pas dans `added_tokens` ; (3) `Vals-ai/HybridDiffusion-2B` **n'existe pas** ; (4) un **contrôle iso-famille** est disponible (`UT-IFML/dQwen3-1.7B-Base`, 28 couches toutes `full_attention`, **même méthode d'adaptation**) et est MUST exercé comme contre-épreuve de la contribution GDN ; (5) la **structure réelle de Q8_0** (34 octets / 32 éléments, échelle `float16`, **pas d'E8M0**) contredit la description antérieure ; (6) le runtime de référence de la modélisation exige **GPU CUDA + `transformers >= 5.13` + batch 1**, ce qui limite la comparaison de référence en CPU-only.

(7) **les citations de la fiche d'influences du 25 octobre 2026 étaient fausses** : `Tier {Vram, Ram, Peer, Nvme}`, `ResidencyManager`, `LruOrder`, `ReplacementPolicy` appartiennent à `ro99/strata` (homonyme), **pas à `Niko1221/Strata`** ; `CacheBudget` n'existe pas dans FreeToken ; `PeerExperts` est à `peer_experts.hpp:36` (et non `:29`), `RemoteExpertOpt` à `remote_expert_opt.hpp:14` (et non `expert_source.hpp:855`), `ExpertCache` à `expert_cache.hpp:86` (et non `:45`). **Conséquence normative** : Strata **ne chasse pas** ses experts (`expert_cache.hpp:111`, `:125`, `:158`) et son tier froid est la **page cache du fichier mmappé en zero-copy** (`docs/DETAILS.md:290`, `STRATA_COLD_OVERLAP=4`) ; **IQ2_XS n'est pas déquantisé** par la source (`dequant_bf16.cu:4`, `iq_kernels.cu:973`). NEVER reproduire les citations du 25 octobre 2026.

**Recommandation de l'audit (ne plus enrichir massivement, commencer P0)** : ** maintenue**. Les prérequis d'entrée de P0 sont listés dans `docs/etat_implementation.md` ; les zones 2, 3, 9, 14, 19, 23, 26, 27 restent **A TRANCHER** (`ADR-006`..`ADR-011` sont **proposées**, NEVER applicables avant décision du mainteneur).