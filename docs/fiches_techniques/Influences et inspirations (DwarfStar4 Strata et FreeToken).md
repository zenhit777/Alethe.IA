# Fiche technique — Influences et inspirations de DwarfStar4, Strata et FreeToken

**Périmètre :** Inspirations architecturales et techniques issues de DwarfStar4, Strata et FreeToken pour la conception de notre moteur d'inférence.

**Orientation prioritaire :** Exécution efficace des modèles de diffusion (dLLM), avec prise en charge des modèles autorégressifs et des architectures hybrides.

**Objectif :** Extraire les mécanismes de gestion des ressources, d'exécution, de mémoire, de chargement et de validation susceptibles d'améliorer les performances, la fiabilité et la portabilité de notre moteur.

**Identification des sources** : les références de cette fiche sont **`Niko1221/Strata`** (https://github.com/niko1221/strata), **`antirez/ds4`** (nom d'enregistrement de DwarfStar4) et **`FlashML-org/FreeToken`**. `ro99/strata` est un **projet homonyme distinct** et n'est **pas** une référence d'Alethe.ia. **Preuve lue le 10 novembre 2026** (tarball `codeload.github.com/ro99/strata/tar.gz/refs/heads/main`, `curl` `200`) : le dépôt `ro99/strata` est **header-only** (`include/strata/engine/placement.hpp:260 solve_placement(inventory, hardware, request)`, `:286 plan_model_placement`, `:291 verify_placement_plan(plan, hardware)`, `:294 render_placement_report(plan)` ; `include/strata/engine/residency.hpp:47 class ResidencyManager`, `:16 cold_read_budget_bytes`, `:21 strict_cold_read_budget`, `:39 cold_budget_violations` ; `include/strata/engine/lru_residency.hpp:37 class LruOrder`, `:40 touch(key, position)` ; `include/strata/platform/types.hpp:16 enum class Tier {Vram, Ram, Peer, Nvme}`, `:23 enum class ReplacementPolicy {Lru, Lfu, Lease}`), avec ses propres applications `apps/strata_server.cpp` et `apps/strata_inspect.cpp` — **aucun de ces chemins n'existe dans `Niko1221/Strata`** (dont les sources sont `src/core`, `src/prefill`, `include/strata/core`). MUST nommer le dépôt et l'organisation dans chaque mention de Strata ; NEVER citer un chemin de `ro99/strata` comme preuve d'un mécanisme de Strata.

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

**Source** : `antirez/ds4` — dépôt **lu le 10 novembre 2026** (tarball `codeload.github.com/antirez/ds4/tar.gz/refs/heads/main`, `curl` `200`, 16 400 610 octets). Faits lus : moteur natif centré sur un ensemble restreint de modèles (`README.md:77`), chargement **mmap** (`ds4.c:11`, `ds4.c:2977`), serveur sur `http://127.0.0.1:8000` par défaut (`README.md:124`), parallélisme tensoriel et pipeline entre machines (`README.md:39-40`).

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

**Rattachement aux fichiers de `antirez/ds4`** (dépôt **lu le 10 novembre 2026** : tarball `codeload.github.com/antirez/ds4/tar.gz/refs/heads/main`, `curl` `200`, 16 400 610 octets ; chaque `chemin:ligne` + identificateur est **lu**, NEVER un chemin non lu) :

| Mécanisme emprunté | Fichier + identificateur lus | Nature de la preuve |
|---|---|---|
| Spécialisation par famille de modèles | `ds4.c`, `ds4.h`, `ds4_qwen4_cuda.cuh`, `ds4_deepseek41_cuda.cuh` (un fichier par famille) ; `README.md:77` (« without trying to cover every possible setup ») | Noms de fichiers + déclaration du dépôt |
| KV cache étendu au disque | `ds4_kvstore.h:113 ds4_kvstore_open(kc, dir, budget_mb, …)` ; `:134 ds4_kvstore_store_len` ; `:135 ds4_kvstore_chat_anchor_pos` ; `:471 ds4.h` (« state must not be written to the text-keyed disk KV cache ») | Signatures lues + règle de validité écrite dans l'en-tête |
| Streaming depuis le SSD | `ds4.h:147-150 ssd_streaming_cache_experts`, `ssd_streaming_cache_bytes`, `ssd_streaming_full_layers`, `ssd_streaming_preload_experts` ; `:161-163 ssd_streaming`, `ssd_streaming_cold`, `ssd_streaming_full_layers_set` ; `ds4_ssd.h:26 ds4_ssd_auto_cache_plan(...)` ; `:24 ds4_ssd_cache_experts_for_byte_budget(bytes, per_expert_bytes)` ; `ds4.c:2977 mmap(NULL, st.st_size, PROT_READ, mmap_flags, fd, 0)` ; `ds4.c:11` (« Loading is mmap based ») ; `ds4_layer_pack.h:45 ds4_compute_layer_placement(entry_bytes, …)` | Champs d'état + planificateur de cache + appel d'os mesuré |
| Validation à partir des logits de référence | `tests/test_qwen4_logit_dump.py` ; `docs/QWEN38_FLASH_NEXT.md:433` (« checks complete logits, cache contents »), `:494` (« prefill checkpoints include matching logits at each completed chunk ») ; `EVAL_DATA.md`, `QA_BEFORE_RELEASES.md` | Test nommé + phrase du dépôt |
| Multiples ressources et machines | `ds4_gpu_tp.h:28 ds4_gpu_tp_init(rank, slab, …)` (rang) ; `ds4_gpu_mgpu.h` ; `ds4_distributed.h:68 ds4_dist_session_create(...)` ; `README.md:39` (2 machines 128 GB reliées par RDMA, parallélisme tensoriel) ; `README.md:40` (parallélisme de pipeline pour sommer les RAM) ; `docs/CUDA_MULTI_GPU.md:41 --cuda-tensor-parallel` | Signatures + drapeau + déclaration du dépôt |
| Service local | `README.md:124` : le serveur écoute par défaut `http://127.0.0.1:8000` | Déclaration du dépôt |

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

**Source** : `Niko1221/Strata` — dépôt **lu le 10 novembre 2026** (tarball `codeload.github.com/Niko1221/Strata/tar.gz/refs/heads/main`, `curl` `200`, 35 148 702 octets). Les mécanismes repris (budget de transfert, cache d'experts, prédicteur d'ordre) sont des **adaptations** aux structures d'Alethe.ia, pas des reproductions du code de Strata. La mémoire polymorphe d'Alethe.ia classe les objets **par rôle** (chaud, tiède, froid, persisté, table, scratch), pas par classe d'objet imposée, et l'ordonnanceur de ressources décide le placement. **Le « placement multinode » n'existe pas dans le dépôt lu** : il n'y a pas de placement multi-machine, seulement un tier de GPU pair (`peer_experts.hpp:36`) et une scission par nœud NUMA (`docs/DETAILS.md:1205-1216`) — il est donc **hypothèse de recherche**, pas mécanisme repris.

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

**Faits relus dans le dépôt `Niko1221/Strata` le 10 novembre 2026** (faits lus, pas inférence) : moteur de serving **local** — `docs/DETAILS.md:442` (`base_url="http://127.0.0.1:8080/v1"`) — exposant les API **OpenAI Chat Completions** (`:427`, `POST /v1/chat/completions`), **Anthropic Messages** (`:428`, `POST /v1/messages`) et **OpenAI Responses** (`:429`, `POST /v1/responses`) ; licence MIT (`README.md:19` shield, `:206`) ; **stream les poids depuis le stockage** (`src/core/expert_source.cpp:821` `mmap(nullptr, size, PROT_READ, MAP_SHARED, fd_b, 0)`, `:1024 pread_full`, `:847 preadn` en `O_DIRECT`) ; **hiérarchie publiée** : `docs/DETAILS.md:284` (`### Linux file tier`), `:290` (zero-copy : un expert dont les pages des trois tenseurs sont dans la page cache est calculé **directement depuis le mapping**, sans copie de staging), `:291` (`STRATA_IO_PREFETCH` : les fils d'I/O lisent les experts non cachés **et ceux prédits par le routeur** dans la page cache), `:292` (le gate d'admission de la lecture d'avance se juge toutes les 16 fenêtres de décodage), `:313` (`--stats` imprime les tiers du décodage), `:1469` ; **conserve la quantification d'origine** (`docs/DETAILS.md:1398-1399`, les formats i-quant de llama.cpp sont **transcrits** dans `src/kernels/cuda/iq_kernels.cu` ; `:1055` les K-quant sont lus par MMQ **sans** déquantisation ; `:1121-1122` `STRATA_DEQUANT_WIDE` — déquantisation large de Q4_K, Q5_K, Q6_K et IQ4_XS, **bit-exact**, par défaut sur sm_86 et sm_61) ; **gère la mémoire comme un cache qui ne chasse pas** (`include/strata/core/expert_cache.hpp:86 class ExpertCache`, `:111` « the cache does not evict », `:125` « the CALLER evicts them first », `:158` « **it never evicts**, because eviction policy is a measured question (`R4.1`'s LFU-decay vs LRU) ») ; **publie la stratégie** (`docs/DETAILS.md:133` : le *setup plan* et le log de démarrage nomment ce qui est réellement choisi sur la machine). **Non retrouvé dans le dépôt lu** : « ne convertit pas les checkpoints » et « ne forme pas de modèles » sont des affirmations antérieures (**25 octobre 2026**) que la lecture du 10 novembre **n'a pas retrouvées sous cette forme** ; NEVER les citer comme faits vérifiés ce jour. **Aucun `Tier`, aucun `ResidencyManager`, aucun `LruOrder` dans `Niko1221/Strata`** : ces identificateurs appartiennent au dépôt **homonyme `ro99/strata`** (voir ligne 9). La section « Rattachement » ci-dessous remplace la liste de faits non rattachés.

**Rattachement des mécanismes** — MUST lier chaque mécanisme repris à des fichiers précis du dépôt source (chemin + fonction) **avant** toute réimplémentation. MUST NOT présenter un mécanisme comme une reproduction de Strata sans ce rattachement ; à défaut, MUST le déclarer **adaptation d'Alethe.ia** ou **hypothèse de recherche**.

| Mécanisme | Source dans `Niko1221/Strata` — chemin + identificateur **lus le 10 novembre 2026** | Correspondance Alethe.ia | Statut |
|---|---|---|---|
| Streaming des poids depuis le stockage quand le modèle ne tient pas en RAM | `src/core/expert_source.cpp:821` (`mmap(nullptr, size, PROT_READ, MAP_SHARED, fd_b, 0)`), `:1024 pread_full(fd, dst, n, off)`, `:847 preadn` (O_DIRECT) ; `include/strata/core/expert_source.hpp:198 blob()`, `:226 prefetch(layer, expert)`, `:222 begin_layer(layer, ids, k)` ; `docs/DETAILS.md:291` | `R-30` : streaming du fichier de poids à la granularité des blocs du format, admission décidée par l'ordonnanceur de ressources | **Source** |
| Conservation de la quantification d'origine | `docs/DETAILS.md:1398-1399` (formats i-quant de llama.cpp, « transcribed in `src/kernels/cuda/iq_kernels.cu` » ; la source transcrite est nommée à `iq_kernels.cu:3-4` : `vecdotq.cuh`, `dequantize.cuh`, `quantize.cu`) ; `:1055` (les experts K-quant sont lus par MMQ **sans** déquantisation) ; `:1121-1122` (`STRATA_DEQUANT_WIDE`, bit-exact sur sm_86 et sm_61) ; **liste des formats déquantisés mesurée** : `src/kernels/cuda/dequant_bf16.cu:4` = `q2_0/q4_0/q5_0/q8_0/q3_K/q4_K/q5_K/q6_K/iq4_nl/iq4_xs` — **IQ2_XS n'y est pas** : il est consommé par produit scalaire sur codebook int8 (`iq_kernels.cu:973`, `Split<17>` `:998`, `Split<23>` = IQ4_XS `:1234`) | `R-59`, `R-60`, `I-8`, `C-7` | **Source** — **la liste des formats réellement déquantisés est mesurée dans le fichier** ; la validation de `R-60` (chaque format `F-1` a une référence numérique et un test de déquantification) MUST traiter à part les formats **consommés par produit scalaire sur codebook** (`IQ2_XS`), NEVER exiger une déquantisation pour un format qui n'en a pas. **Aucun code de conversion de checkpoint trouvé dans le dépôt lu.** |
| Mémoire gérée comme un système de cache | `include/strata/core/expert_cache.hpp:86 class ExpertCache` (`:97 open(n_slots, n_layers, n_expert, blob_bytes, err)`, `:144 shrink(keep_bytes, err)`, `:148 grow(want_bytes, err)`, `:211 fill_slot_queued_parts(slot, parts, n, err)`, `:217 verify_slot(slot, host_blob, err, bytes)`, `:227 seal()`, `:126 set_segment_bytes`, `:131 set_slot_guard`) ; `include/strata/core/auto_cache.hpp:33 auto_cache_slots(free_b, reserve_b, blob, min_slots, ...)` | `R-29`, `R-32`, `R-33` : hiérarchie **par rôle** (chaud, tiède, froid, persisté, table, scratch), placement décidé par l'ordonnanceur de ressources, pas par classe d'objet imposée | **Adaptation** — **la hiérarchie par rôle d'Alethe.ia n'existe pas dans le dépôt lu** : Strata nomme un tier de VRAM, un tier de GPU pair (`peer_experts.hpp:1` « a second expert tier on another GPU »), une copie RAM et un *file tier* (`docs/DETAILS.md:284`, `:290` zero-copy depuis la page cache, `:291` io prefetch, `:313` `--stats` imprime les tiers) |
| LRU d'experts | `include/strata/core/expert_cache.hpp:163 replace(layer, old_expert, new_expert)` ; `:185 set_per_layer_admission` / `:186 per_layer_admission` ; **le cache ne chasse pas** : `:111` « the cache does not evict », `:125` « the CALLER evicts them first (they become CPU misses) », `:158` « **it never evicts**, because eviction policy is a measured question (`R4.1`'s LFU-decay vs LRU) » — **aucun `LruOrder`, aucun `ResidencyManager`, aucun `Tier` dans le dépôt lu** : ces identificateurs appartiennent à `ro99/strata` (`engine/lru_residency.hpp:37 LruOrder`, `:40 touch`, `engine/residency.hpp:47 ResidencyManager`, `platform/types.hpp:16 Tier {Vram, Ram, Peer, Nvme}`, `:23 ReplacementPolicy {Lru, Lfu, Lease}`) | `R-41` (pool d'experts), `R-37` (CPU pour experts sélectionnés), `R-39` : la politique est une **spécialisation du profil de stratégie** (`R-91`), pas une politique globale (`R-28`) | **Adaptation** — Strata ne **fige** pas le LRU comme politique globale et **délègue l'éviction à l'appelant** ; MUST NOT présenter une politique d'éviction LRU comme une politique de Strata. **Le mélange des deux homonymes est l'origine des citations fausses du 25 octobre 2026** |
| Budget de transfert explicite | `include/strata/core/expert_source.hpp:110 clamp_resident_budget(requested, physical, commit, headroom)` ; `:104 wait_for_ram(need, max_wait_s, step_s, note_s, ...)` ; `:79 cgroup_available_bytes(limit, stat, bytes)` ; `docs/DETAILS.md:124` (« RAM budget: your RAM less 24 GB, the rest read from the SSD »), `:251` (`--resident-budget-gib N`), `:397` (« The expert cache size is a budget »), `:1076` (part PCIe `--pcie-frac`, `STRATA_ADAPT_FETCH`, `STRATA_ADAPT_OVERLAP`) | `R-85` (budget par device), `R-88` (transfert d'une page dans les limites du budget), `R-84` | **Source** |
| Admission des modèles et signalement de la dépendance aux transferts | `include/strata/core/expert_cache.hpp:185 set_per_layer_admission` / `:186 per_layer_admission`, `:131 set_slot_guard`, `:126 set_segment_bytes` ; `docs/DETAILS.md:1076` (`STRATA_ADAPT_FETCH=1\|2`, `STRATA_ADAPT_FETCH_MIN` = 1, `STRATA_ADAPT_FETCH_MARGIN` = 0.5 — un expert fetché est copié s'il est routé au moins ce margin de plus que le résident le moins routé) ; `:600` (`spec_admission_test` énumère la règle) | `R-34`, `R-84`, `R-94` (état des ressources exposé) | **Adaptation** — ce que le dépôt nomme est l'**admission d'un expert fetché** dans un slot résident, pas l'admission d'un **modèle** ; l'admission de modèle d'Alethe.ia est une **conception d'Alethe.ia** |
| Publication de la stratégie d'ordonnancement | `docs/DETAILS.md:133` (« le *setup plan* et le log de démarrage nomment ce qui est réellement choisi sur votre machine ») ; `:297` (le log nomme ce qui est activé), `:304-305` (un planificateur choisit le chunk et le ring avec le moins d'octets prédits lus par prompt), `:308` (`STRATA_PREFILL_HOST_PLAN=0`, `STRATA_SSD_GBPS`), `:313` (`--stats` imprime les tiers du décodage) | `R-95` (configuration effective), `R-18`, `R-91` (profils typés par topologie) | **Source** |
| Compatibilité API sur l'hôte local | `docs/DETAILS.md:427` (`POST /v1/chat/completions`), `:428` (`POST /v1/messages`, Anthropic), `:429` (`POST /v1/responses`), `:442` (`base_url="http://127.0.0.1:8080/v1"`) ; `README.md:169` (agents clients) | `R-48..R-51`, `R-80..R-82`, `R-96` (API de contrôle sur `127.0.0.1`) | **Source** |
| Prédicteur qui modifie l'ordre sans modifier les calculs | `include/strata/core/foresight_swap.hpp:36 struct ForesightSwap`, `:104 foresight_swap_from_env(layers, experts, card_of_layer, ...)` ; `include/strata/core/expert_source.hpp:226 prefetch(layer, expert)`, `:222 begin_layer(layer, ids, k)` ; `docs/DETAILS.md:291` (les `pread` touchent les experts non cachés **et ceux prédits par le routeur** pour les couches suivantes) | `R-22` : la prédiction de routage ne modifie jamais la sélection d'experts | **Source** |
| Placement multinode | `include/strata/core/peer_experts.hpp:36 class PeerExperts` (file d'en-tête `:1` : « a second expert tier on another GPU » ; `:6` : le primaire calcifie son tier de VRAM et la part PCIe, tout expert restant est une « CPU row »), `include/strata/core/remote_expert_opt.hpp:14 class RemoteExpertOpt` ; `docs/DETAILS.md:1205-1216` (avec 2 nœuds NUMA ou plus le pool se scinde **par nœud**, une tranche par nœud avec son propre compteur de tâches ; `:1215` si un nœud n'a pas assez de mémoire libre pour sa moitié ; `:1216` le chargement affiche `FAILED` quand un nœud tient moins de 95 % de sa tranche) — **aucun placement multi-machine trouvé dans le dépôt lu** | Alethe.ia est **mono-device** (`N-9`) : MUST NOT l'implémenter sans décision écrite | **Hypothèse de recherche** |

**Position d'Alethe.ia sur les fichiers de Strata** : la fiche est une **source d'inspiration**. MUST lire les fichiers cités dans le dépôt source avant toute réimplémentation ; NEVER citer la fiche comme preuve de fonctionnement.

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

**Source** : `FlashML-org/FreeToken` — dépôt **lu le 10 novembre 2026** (tarball `codeload.github.com/FlashML-org/FreeToken/tar.gz/refs/heads/main`, `curl` `200`, 3 158 952 octets). Faits lus : serving MoE « edge-native » traitant GPU, CPU, mémoire hôte et interconnexions comme une plate-forme élastique unifiée (`README.md:18`) ; API **Anthropic/OpenAI compatibles** (`README.md:23`) ; le serveur écoute sur l'hôte local par défaut (`python/freetoken/server/args.py:39 server_host: str = "127.0.0.1"`, `:111 tcp://127.0.0.1:{server_port + 1}`) ; supports matériels NVIDIA RTX 30/40/50 (`README.md:24`). Les mécanismes listés ci-dessous sont **rattachés chacun à un chemin et à un identificateur lus** ; un mécanisme sans rattachement est **adaptation d'Alethe.ia** ou **hypothèse de recherche**, NEVER une reproduction.

Ses mécanismes les plus intéressants comprennent :

- Une politique de co-exécution CPU/GPU adaptée à la bande passante disponible, appelée **q\*** — lu : `README.md:20` (« bandwidth-adaptive CPU–GPU co-execution ($q^\star$ policy) ») ; `python/freetoken/moe/cpu_executor.py:1` (« Python wrapper around the `_cpu_moe` C++ executor (`--moe-strategy cpu`) », séquence D2H → doorbell → attente du pool CPU → H2D, « the whole CUDA-graph-capturable sequence »).
- Le streaming de prefill par couches avec double buffering — lu : `README.md:20` (« full-layer double-buffered prefill streaming ») ; `python/freetoken/moe/offload_cache.py:886 prefetch_prefill_layer(layer_id)`, `:964 _prefetch_split(layer_id, buffer_id)` (deux buffers par couche).
- Un cache global d'experts utilisant une politique LRU — lu : `README.md:20` (« global LRU expert caching ») ; `python/freetoken/moe/offload_cache.py:130 cache_policy: str = "lru"`, `:171` (une couche appartient à un pool, chaque pool a **son propre scope LRU**), `:282 self.lru_stats`, `:1089` (« Capped-fetch LRU for the hybrid backend »), `:1143` (l'accumulation des statistiques LRU est **fondue dans le launch de `ensure_experts`**) ; `python/freetoken/moe/pool_policy.py:83 _check_budget(total, pools)`.
- Une exécution compatible avec les graphes de calcul — lu : `README.md:20` (« graph-compatible execution ») ; `python/freetoken/engine/graph.py:26 class GraphCaptureBuffer`, `:102 _upload_graph(graph, stream)`, `:116 class GraphRunner`, `:66 _determine_cuda_graph_bs`.
- Un format de poids optimisé, **FTW** — lu : `python/freetoken/checkpoint/ftw.py:110 is_ftw_checkpoint(path)`, `:122 ftw_quant_format(path)`, `:131 class FTWWriter` ; `python/freetoken/checkpoint/convert.py` ; `docs/ftw-hotfix.md` (« Repairing old FTW checkpoints »).
- Des checkpoints sémantiques pour les états récurrents et les caches KV — lu : `README.md:21` (« semantic anchor checkpoints for recurrent state and KV caches ») ; `python/freetoken/scheduler/cache.py:157 snapshot_toolcall_anchor(reqs)` (« Freeze each decoding request's **GDN state** at its tool-call anchor, into the **ping-pong** … »), `:202` (la fenêtre se terminant à l'anchor reste reprenable) ; `python/freetoken/kvcache/linear_state_pool.py`.
- Une réallocation dynamique de la VRAM entre cache d'experts et mémoire KV — lu : `README.md:22` (« dynamic, runtime VRAM re-allocation between expert caches and KV memory **without engine restarts or weight reloading** ») ; `python/freetoken/engine/cache_budget.py:38 net_cache_budget_bytes(...)`, `:55 plan_cache_budget(...)` ; `python/freetoken/engine/engine.py:113 _startup_kv_budget(memory_ratio, init_free_memory, new_free_memory)`, `:1750 _check_pin_budget(config, reserved, method, layer_methods)` ; `python/freetoken/kvcache/cache_status.py:77 _pool_budget_free_vram_bytes(engine)`.

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