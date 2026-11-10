# Alethe.IA — Document de référence unique

**Statut** : référence normative du projet. Ce fichier est **la source unique** pour l'agent IA en charge du développement. En cas de contradiction avec un autre document du dépôt, **ce fichier prévaut** ; les fiches de `docs/fiches_techniques/` restent la source des justifications.

**Mots normalisés (RFC 2119)** : MUST, REQUIRED, SHOULD, RECOMMENDED, MAY, OPTIONAL. `NEVER` = `MUST NOT` ; `AVOID` = `SHOULD NOT`.

**Deux axes de maturité** (définis en §2.4) : **maturité de la spécification** (`DEFINED`, `EXPERIMENTAL_SPEC`, `RESEARCH`, `FUTURE`) et **état d'implémentation** (`NOT_STARTED`, `IN_PROGRESS`, `IMPLEMENTED`, `VALIDATED`). **Chaque exigence porte les deux.** Un `MUST` accompagné de `RESEARCH` n'est jamais un ordre d'implémentation définitive ; un `DEFINED` n'affirme jamais que le code existe.

**Traçabilité** : la matrice §13.2 rattache chaque exigence à sa source et à sa **nature** (`Source`, `Adaptation`, `Hypothèse de recherche` — Annexe A). Un article scientifique ne peut pas être cité comme source d'une décision qui est une extrapolation d'Alethe.ia.

## 0. Usage de ce document

L'agent IA MUST :

1. Lire ce fichier **entièrement** avant toute modification de code.
2. Respecter les exigences (`R-x`), les invariants (`I-x`), les contraintes (`C-x`) et les non-objectifs (`N-x`).
3. Vérifier **les deux axes de maturité** de chaque exigence avant d'écrire du code (axe 1 : `DEFINED` → implémentation autorisée, `EXPERIMENTAL_SPEC` → implémentation mesurable et désactivable, `RESEARCH` → prototype + mesure, jamais chemin de production, `FUTURE` → ne pas implémenter ; axe 2 : `NOT_STARTED` / `IN_PROGRESS` / `IMPLEMENTED` / `VALIDATED` — matrice d'état dans `docs/etat_implementation.md`, MUST être mise à jour à chaque validation).
4. NE PAS inventer de concepts absents de ce document ; AVOID les extensions non décrites.
5. Signaler les contradictions, ambiguïtés et zones non couvertes **avant** d'écrire le code correspondant.
6. Traiter chaque livrable comme adossé à une exigence : un changement sans exigence est une dérive.
7. NE PAS trancher seul un arbitrage d'architecture (fusion des ordonnanceurs, choix d'un format de poids prioritaire, périmètre d'une API, ajout d'une dépendance). Ces décisions appartiennent au mainteneur et MUST être consignées dans ce document avant implementation.
8. **Zones à trancher (§14)** : une zone marquée **A TRANCHER** MUST NOT être codée. L'agent MUST proposer une **note de décision** (options, gains, coûts, effets sur les exigences), MUST la soumettre au mainteneur, MUST consigner la décision retenue dans la ligne de §14 **avant** d'écrire le code.
**Carte de chargement** — le document est lu entièrement avant toute modification de code (règle 1) ; la table ci-dessous indique les sections minimales à relire selon la tâche, sans remplacer la lecture complète :

| Tâche | Sections minimales |
|---|---|
| cycle, jetons, modes | §3.1, §3.2, §3.3, §5 |
| ordonnanceur d'inférence | §3.4, §5.4, §5.5, §6.2, R-18..R-28, R-91 |
| mémoire, placement, offload | §3.5, §3.6, §4.2, §4.5, §4.7, R-29..R-42, R-84..R-89 |
| kernels, WGSL, limites du device | §4.1, §4.4, §4.5, §4.6, R-33..R-35, R-86..R-88 |
| formats de poids, n-gram | §3.9, §4.3, §5.8, R-58..R-64, R-83 |
| serving, API externe, API de contrôle | §3.7, Annexe B, R-48..R-51, R-80..R-82, R-92..R-96 |
| persistance | §3.8, §6.4, R-43..R-47, R-75..R-79 |
| validation numérique | §6.1, §6.3, §9, R-52..R-57, R-65..R-70 |
| jalons, livraisons, état | §8, §16, §17, `docs/etat_implementation.md` |
| zone à trancher | §14, §0 règle 8, §11 règle 2 |

**Sources consultées pour l'écriture de ce document** : les 4 fiches de `docs/fiches_techniques/`, `docs/fiches_techniques/références.txt`, les sources primaires des références (repos Strata `Niko1221/Strata`, FreeToken, DwarfStar4 ; articles WeDLM, FLARE, FLUID, REPR-ALIGN ; spécifications W3C WebGPU et WGSL ; cartes de modèle DiffusionGemma et Clover).

---

## 1. Vision produit

**Alethe.ia** est un moteur d'inférence **natif pour les modèles génératifs à diffusion (dLLM)**, écrit en coopération **Rust + Zig** et accéléré par **WebGPU** de manière portable.

Le moteur sert **deux familles de génération** — la diffusion par blocs et l'autorégression — **sur une infrastructure commune** (contrats d'opérations, modèle de mémoire, ordonnanceurs, système de validation, API). Il ne partage **pas** un algorithme de génération unique : l'AR conserve son chemin causal natif, la diffusion conserve son chemin de dénoisage natif. Voir §2.5 pour le périmètre exact de l'unification.

**Produit livré** (par phases) :

| Phase | Livrable | Interface utilisateur |
|---|---|---|
| P0-P2 | moteur + CLI de validation | technique uniquement |
| P3-P4 | moteur + serveur OpenAI-compatible et Anthropic-compatible + CLI d'exploitation | opérateurs, intégrateurs, agents |
| P5 | moteur + APIs + persistance + formats étendus | opérateurs |
| ultérieure | application avec interface graphique | grand public |

Le **grand public** est une cible d'adoption, pas le périmètre des interfaces livrées aux jalons P0-P5. Un agent ne MUST pas concevoir une interface graphique, un packaging grand public ou une application desktop dans les jalons initiaux (`N-4`).

**Utilisateurs** : (a) l'agent IA de développement, (b) les intégrateurs d'applications locales (CLI, serveurs API, agents de code), (c) les opérateurs de machines hétérogènes (VRAM limitée, RAM faible, NVMe rapide, WSL, iGPU), (d) les agents IA consommateurs des APIs.

**Fonctionnalités primaires** :

- génération dLLM native (dénoisage par bloc, parallèle) ;
- génération AR sur le même moteur, par son chemin natif ;
- préfill/décodage en continu, réutilisation de préfixe, reprise de session ;
- serving **OpenAI-compatible** et **Anthropic-compatible** (streaming, tool calls) ;
- persistance d'états d'inférence (checkpoints de KV, états récurrents, sessions) ;
- gestion polymorphe de la mémoire (KV, états récurrents, états de diffusion) sur VRAM / RAM / NVMe, **par rôle des objets**, pas par partition matérielle fixe ;
- exécution sur GPU grand public, iGPU, **et sur machines sans GPU discret (mode CPU-only)** ;
- exécution d'opérations sur CPU en production dans les sessions servies sur accélérateurs (offload, notamment experts MoE).

**Ce que le moteur ne fait pas** : il n'entraîne pas, n'affine pas, ne convertit pas les checkpoints. La conversion AR → dLLM est un **outillage externe** (§3.11) ; le moteur **charge** les artifacts produits par cet outillage. Voir `N-1`.

**Non-objectifs** (exclusions explicites, `N-x`) : voir §12.

---

## 2. Principes de conception

### 2.1 Architecture des couches

| Couche | Rôle principal |
|---|---|
| **Rust** | cohérence d'état, concurrence, **ordonnanceur d'inférence**, **ordonnanceur de ressources**, contrats d'interface, métadonnées canoniques de tenseurs, formats de fichiers, validation, APIs |
| **Zig** | représentation mémoire des données, layouts, indexation, structures compactes, chemins critiques, décodage de poids, préparation des buffers |
| **WebGPU** | accélération portable des calculs, kernels de dLLM, allocation de buffers, files de commandes, plans d'exécution, synchronisation |
| **CPU** | backend de référence, backend de production (offload d'opérations, mode CPU-only complet) |

**Aucune hiérarchie artificielle** : Rust n'est pas supérieur à Zig, WebGPU n'est pas subordonné aux deux. Chaque couche possède une zone de responsabilité délimitée.

**Règle de non-duplication** : chaque concept (position logique, position physique, bloc, masque, horizon, fenêtre glissante, état récurrent, état de validation, préfixe autorisé, état de session, format de poids) MUST avoir **une seule définition** dans le moteur.

**Règle de séparation stricte** (adaptée de DwarfStar4) : le CLI/server MUST NOT connaître les internals de tenseurs ; l'ordonnanceur d'inférence MUST NOT connaître les formats de poids ; l'ordonnanceur de ressources MUST NOT connaître la sémantique de dénoisage.

### 2.2 Principes d'architecture

| Principe | Formulation normative |
|---|---|
| Primauté du dLLM | les structures principales MUST être conçues pour les processus de dénoisage et de validation |
| Coopération des couches | les responsabilités MUST être clairement définies, sans subordination |
| Portabilité WebGPU | le moteur MUST s'adapter aux capacités des appareils, sans dépendre d'une extension propriétaire |
| Préservation de la sémantique | les représentations, formats et interfaces MUST conserver la sémantique des checkpoints |
| Optimisation mesurée | les optimisations MUST être justifiées par des mesures de mémoire, latence, débit, bandes passantes |
| Limites explicites | les interfaces MUST définir ce que le moteur fait et ne fait pas |
| Dégradation contrôlée | les capacités manquantes (device limitations, format unsupported, kernel fallback, cache thrashing, VRAM fragmentation, absence de `shader-f16`) MUST être explicites, jamais silencieuses |
| Déclaratif | les modèles déclarent leurs besoins (états, formats, capacités de décodage) ; les ordonnanceurs décident où les satisfaire |

### 2.3 Invariants

| # | Invariant |
|---|---|
| I-1 | **Infrastructure commune, pas algorithme unique** : AR et dLLM partagent les contrats d'opérations, la mémoire, les ordonnanceurs, le système de validation et les APIs. La sémantique de génération reste propre à chaque famille. |
| I-2 | **Deux ordonnanceurs coordonnés** : un **ordonnanceur d'inférence** (quoi calculer, dans quel ordre : blocs, pas, sélection, spéculation, admission des requêtes) et un **ordonnanceur de ressources** (où résident les données, quand elles sont déplacées, à quel coût). Ils communiquent par une interface typée, jamais par effets de bord. |
| I-3 | **Un seul modèle de mémoire** : KV, états récurrents, états de blocs, états intermédiaires, poids, tables n-gram, checkpoints persistés sont des entrées d'une **même hiérarchie de placement**, classées par rôle et coût d'accès, pas par adresse matérielle fixe. |
| I-4 | **Un seul système de validation** : interface commune (protocole typé commit / rewind / masque / rejet), instances spécialisées par classe de mécanisme (dénoisage, spéculatif, état récurrent, poids). Un AR strict n'est **pas** un cas particulier du validateur de dénoisage. |
| I-5 | **Sémantique des checkpoints préservée** : formats de poids, structures d'attention, schémas de routage, top-k, nombres d'experts MUST rester fidèles aux checkpoints. |
| I-6 | **Aucune hiérarchie artificielle entre couches** : la frontière se définit par des contrats, pas par la profondeur d'appel. |
| I-7 | **Pas de duplication fonctionnelle** : un concept a une définition, un champ, une sémantique. |
| I-8 | **Pas de lock-in GPU** : les extensions propriétaires MAY être des accélérateurs optionnels, jamais la sémantique principale. |
| I-9 | **Pas de variantes sémantiques permanentes derrière des flags** (règle DwarfStar4) : les switchs de diagnostic valident le chemin de release. |
| I-10 | **Correctivité avant rapidité** : NEVER garder un chemin plus rapide avec attention, KV ou logits drift inexpliqués. |
| I-11 | **Le CPU est un chemin de production** : (a) backend de référence et de debug ; (b) CPU offload d'opérations dans les sessions servies sur accélérateurs, y compris le calcul d'experts MoE ; (c) mode CPU-only complet sur machine sans GPU compatible. Le chemin CPU MUST satisfaire les mêmes exigences numériques, de déterminisme, de persistance, d'API et d'ordonnancement que tout autre backend. |
| I-12 | **C++ NEVER** dans le codebase. |
| I-13 | **Un concept, un champ, un flux** : la mémoire d'un état n'est pas un registre ; les états sont typés et sérialisés. |
| I-14 | **Préconditions déclarées** : tout composant qui suppose une propriété du modèle (causalité stricte, contiguïté du préfixe, checkpoint dual-stream, état récurrent, table n-gram) MUST déclarer cette précondition et refuser les modèles qui ne la remplissent pas. |

### 2.4 Deux axes : maturité de la spécification et état d'implémentation

**Un document normatif ne déclare pas que le code existe.** Chaque exigence porte **deux étiquettes distinctes**, jamais confondues.

**Axe 1 — maturité de la spécification** (colonne `Spéc.` des tables §3) :

| Valeur | Signification | Droit de l'agent |
|---|---|---|
| `DEFINED` | comportement défini, périmètre délimité, préconditions écrites | implémentation autorisée |
| `EXPERIMENTAL_SPEC` | mécanisme défini et implémentable, non validé sur l'ensemble des cibles | implémentation **mesurable et désactivable** |
| `RESEARCH` | hypothèse à étudier, définition incomplète | prototype + protocole de mesure ; NEVER chemin de production, NEVER défaut activé |
| `FUTURE` | hors du périmètre des jalons en cours | NEVER implémentée dans les jalons en cours |

**Axe 2 — état d'implémentation** (matrice dans `docs/etat_implementation.md`) :

| Valeur | Signification |
|---|---|
| `NOT_STARTED` | aucune ligne de code du chemin |
| `IN_PROGRESS` | chemin en cours, non complet |
| `IMPLEMENTED` | chemin complet, non validé sur la cible de référence |
| `VALIDATED` | chemin exercé sur la cible de référence, mesures consignées (§9) |

**État initial du projet : les exigences `R-1..R-97` sont toutes `NOT_STARTED`.** Un `DEFINED` ne signifie pas « implémenté » ; un `VALIDATED` n'est jamais déclaré sans mesure consignée.

Règles :

- Une exigence `RESEARCH` MUST être livrée avec son protocole de mesure et son critère d'abandon.
- Une exigence `EXPERIMENTAL_SPEC` MUST être derrière une option de configuration **explicitement documentée**, pas derrière un flag sémantique permanent (`I-9`).
- L'agent MUST déclarer les deux axes avant d'implémenter (§11, règle 10) et MUST mettre à jour la matrice d'état (`docs/etat_implementation.md`) à chaque validation.
- La promotion d'un axe se décide sur la base des mesures : `RESEARCH` → `EXPERIMENTAL_SPEC` → `DEFINED` (axe 1) par le mainteneur ; `IN_PROGRESS` → `IMPLEMENTED` → `VALIDATED` (axe 2) par l'exécution d'un test ou d'une mesure consignée.

### 2.5 Périmètre de l'unification AR / dLLM

**Ce qui est unifié** (une seule implémentation pour les deux familles) :

- représentation des positions logiques et physiques, masques, blocs, fenêtres ;
- hiérarchie de placement mémoire et ordonnanceur de ressources ;
- protocole de validation (commit / rewind / rejet), ses journalisations et ses invariants de cohérence ;
- cycle de vie des états (session, préfixe, état récurrent, état de diffusion) et leur persistance ;
- admission des requêtes, concurrency, workers ;
- APIs, formats de poids, métadonnées canoniques ;
- instrumentation et critères de validation.

**Ce qui reste propre à chaque famille** (deux implémentations, sous le même contrat) :

| Domaine | AR | dLLM |
|---|---|---|
| ordre de génération | séquentiel causal | parallèle par bloc, itératif |
| granularité d'avancement | 1 jeton par pas | k jetons par pas, k variable |
| granularité de validation | préfixe strictement croissant | fenêtre, ensemble de positions, révisions locales |
| granularité de restauration | point de KV | point de KV **et** point d'état récurrent |
| politique d'attention | profil d'attention déclaré : sous-modules **causal** et **local causal à fenêtre**, pas de positions masquées, préfixe visible = tout le contexte (§7.2.1) | profil d'attention déclaré : sous-modules **bidirectionnel intra-bloc**, **cross-attention** vers le préfixe causal, **récurrent** (GDN), fenêtre glissante, positions masquées, préfixe causal préservé (§7.2.1) |
| sémantique de l'horizon | fenêtre contextuelle | horizon de dénoisage + fenêtre d'attention |

**Conséquence normative** : le contrat commun MUST être typé par **famille de génération** (`generation_family = AR | Diffusion`), **topologie d'architecture** (`architecture_topology`) et **capacités servies** (`generation_capabilities`) — jamais un champ unique. Un agent MUST NOT « unifier » en écrivant un algorithme unique de génération, ni en traitant l'AR comme une spécialisation booléenne du dénoisage, ni en appliquant une politique de fenêtre et de cache globale à toutes les stratégies.



---

## 3. Exigences fonctionnelles

Chaque exigence porte un **critère d'acceptation** et un **niveau de maturité** (§2.4). Les IDs sont celles de la baseline auditée ; les ajouts sont numérotés à la suite.

### 3.1 Cycle de génération

| # | Exigence | Critère d'acceptation | Spéc. |
|---|---|---|---|
| R-1 | Le moteur MUST charger un checkpoint (natif dLLM, AR, adapté par outillage externe §8.1) en structures d'inférence : kernels, buffers, formats de poids, plans d'exécution. | le chargement échoue explicitement si une capacité déclarée n'est pas supportée (`I-14`) | DEFINED |
| R-2 | Le cycle MUST traiter le préfill, la construction des préfixes autorisés, le dénoisage de bloc, la validation de jetons, la mise à jour de l'état d'attention, la décision d'horizon, le commit. | un pas complet observable de bout en bout sur la cible de référence | DEFINED |
| R-3 | Un cycle MUST exposer des points de contrôle observables pour l'ordonnanceur : position du prochain bloc, taille de l'horizon, politique d'attention, politique de validation, révision, fenêtre active. | chaque champ est journalisé et rejouable | DEFINED |
| R-4 | Le cycle MUST être compatible avec la représentation statique d'un plan à formes fixes et avec les appels de kernels (§4.4). | le plan est reproductible sans ré-encodage de la structure de contrôle | DEFINED |
| R-5 | Le cycle MUST permettre des révisions locales : un jeton non validé, une fenêtre annulée, un état intermédiaire rejeté. | le rewind journalise l'état restauré (positions, KV, état récurrent) | DEFINED |
| R-6 | Le cycle MUST être réutilisable pour le préfill, la continuation, la reprise de session, l'interpolation, la reconstruction. | 5 modes sur la même interface de session | DEFINED |

Cycle canonique (7 étapes) :

```
1. Préfixe autorisé       — le préfixe valide est présent dans la mémoire de l'état
2. Horizon                — l'ordonnanceur d'inférence définit l'horizon de génération
3. Fenêtre de bloc        — une fenêtre de blocs est placée pour la dénoisation
4. Forward + attention    — le modèle évalue le contenu masqué
5. Candidats + validation — le modèle produit des jetons candidats, la politique de validation marque les validés
6. Commit                 — les jetons validés deviennent préfixe autorisé
7. Décision d'horizon     — l'ordonnanceur ajuste l'horizon pour le cycle suivant
```

**Périmètre** : les 7 étapes sont le **contrat d'exécution commun** (`I-1`). Elles décrivent un cycle d'obligations, pas un algorithme de génération unique. L'AR les remplit avec `horizon = 1 position`, `fenêtre = 1`, `validation = immédiate`, **par son chemin causal natif** ; il n'est pas une spécialisation du dénoisage.

### 3.2 États de jetons

| # | Exigence | Critère d'acceptation | Spéc. |
|---|---|---|---|
| R-7 | Le moteur MUST distinguer jeton masqué, jeton candidat provisoire, jeton validé. | trois états typés, jamais un booléen | DEFINED |
| R-8 | Le moteur MUST distinguer **position logique** (ordre dans la séquence d'inférence) et **position physique** (ordre dans les buffers/cache). | mapping journalisé par pas | DEFINED |
| R-9 | Le moteur MUST gérer le réordonnement topologique (WeDLM), le réordonnement de blocs (windowing), le réordonnement pour réutilisation de préfixe. | trois réordonnements typés, testés séparément | DEFINED |
| R-10 | Un jeton validé MUST devenir un préfixe autorisé et cache-valide immédiatement dans le régime causal. | préfixe contigu croissant vérifié | DEFINED |
| R-11 | Un jeton révisé MUST rester réversible : l'état d'attention et l'état de bloc MUST pouvoir être restaurés (rollback). | rollback exact sur KV **et** état récurrent | DEFINED |

### 3.3 Modes de génération

| # | Exigence | Critère d'acceptation | Spéc. |
|---|---|---|---|
| R-12 | Le moteur MUST servir un dLLM natif en dénoisage parallèle par bloc. | cycle complet sur la cible de référence | DEFINED |
| R-13 | Le moteur MUST servir un checkpoint **dual-stream** (flux `clean` / `noisy`) en mode `Diffusion-Trust` **uniquement sur un checkpoint réel identifié par une model card publique** (R-97). NEVER l'implémenter, le valider ou le démontrer sur un checkpoint fabriqué, simulé ou synthétisé par l'agent. | checkpoint réel chargé ; refus si le checkpoint n'a qu'un flux | FUTURE |
| R-14 | Le moteur MUST servir un checkpoint **dual-stream** en mode `AR-Trust` (vérification spéculative sur le flux `clean`) **uniquement sur un checkpoint réel identifié (R-97)**. NEVER l'implémenter sur un checkpoint fabriqué. | équivalence de distribution vérifiée (§5.5) sur le checkpoint réel | FUTURE |
| R-15 | Le moteur MUST servir un checkpoint AR **par son chemin causal natif**, sous le contrat commun d'exécution. NEVER l'exprimer comme une spécialisation du cycle de dénoisage, NEVER une seconde boucle AR non typée. | test de causalité stricte : aucune fuite des positions futures | DEFINED |
| R-16 | Le moteur MUST servir un checkpoint **adapté** — AR converti en dLLM par un outillage externe (méthode `WeDLM`, `FLUID`, `FLARE` ou `REPR-ALIGN`) — selon la stratégie et le profil d'attention **déclarés dans son `AdaptationFile`**. MUST traiter `adaptation_method` (méthode) et `adapted_model_name` (nom du modèle adapté) comme des champs distincts. NEVER classer un dLLM natif (DiffusionGemma, Clover) comme checkpoint adapté ; NEVER imposer un régime de streaming causal strict à un checkpoint dont la topologie d'attention n'est pas causale. | `AdaptationFile` lu et vérifié au chargement ; refus si le profil d'attention déclaré n'est pas supporté | DEFINED |
| R-17 | La sélection du mode MUST être déclarative (déduite du checkpoint + configuration), pas un fork de code. | un mode par session, déclaré, sans branche de code | DEFINED |
| R-97 | **Cible de validation des modes dual-stream** : avant toute implémentation de `R-13` / `R-14`, MUST identifier un checkpoint dual-stream réel (model card publique, flux `clean` et `noisy` entraînés) et MUST consigner sa référence. MUST NOT fabriquer, synthétiser ou simuler un checkpoint dual-stream pour faire passer un test. Tant que la cible n'est pas identifiée, les modes restent `FUTURE` et ne figurent dans aucun jalon. | référence du checkpoint réel consignée ; absence de checkpoint synthétique dans les suites de tests | FUTURE |

### 3.4 Ordonnanceur d'inférence

| # | Exigence | Critère d'acceptation | Spéc. |
|---|---|---|---|
| R-18 | L'ordonnanceur d'inférence MUST décider à chaque pas le nombre de positions résolues, la longueur de la fenêtre et la profondeur de vérification, **dans les capacités déclarées du modèle** (longueur de canvas, taille de bloc, fenêtre glissante). | la décision est fonction des paramètres du checkpoint, pas d'une constante globale | DEFINED |
| R-19 | L'horizon MUST être adaptatif, borné par un plafond d'horizon (paramètre K). | K configuré, plafond vérifié | DEFINED |
| R-20 | L'horizon SHOULD être borné par la **frontière de compétence** (au-delà de la confiance de validation, il ne s'étend pas). MUST NOT étendre l'horizon au-delà d'une frontière **non mesurée**. | frontière mesurée et journalisée | RESEARCH |
| R-21 | La porte de confiance SHOULD être un seuil de confiance par position (γ) ; les positions sous le seuil ne sont pas résolues. MUST écrire la définition exacte de γ, de son domaine et de sa source (équation de l'article FLUID) **avant** toute implémentation ; AVOID activer la porte sans γ écrit. | prototype + mesure du débit gagné sur la cible AR | RESEARCH |
| R-22 | Le pas de stride SHOULD être adaptatif : `stride = 1 + Δ`. MUST définir Δ (définition, domaine de valeurs, conversion en nombre entier de positions) et citer l'équation source ; à défaut, la politique reste une **hypothèse de recherche**, NEVER une règle de production. | équation écrite + prototype mesuré | RESEARCH |
| R-23 | L'ordonnanceur MUST s'informer de : incertitude de prédictions, stabilité de préfixe, coût de transfert, disponibilité de VRAM, cache misses, bandes passantes, capacité de kernels. | les informations proviennent de l'ordonnanceur de ressources (R-84..R-89), pas d'une lecture directe du device | DEFINED |
| R-24 | La réallocation dynamique des pools (KV, récurrent, cache d'experts) et l'adaptation de la fenêtre active MUST être décidées par l'**ordonnanceur de ressources** sur requête de l'ordonnanceur d'inférence. | requête / accord / rejet journalisés | DEFINED |
| R-25 | Un adaptateur de décodage **causal compatible WeDLM** MUST déclarer ses préconditions (décodage causal strict, préfixe contigu, logits disponibles pour les positions masquées). Dans ce cadre, la sélection de remplissage MUST utiliser l'entropie pénalisée par la distance `H̃ᵢ = Hᵢ + λ·dᵢ` (dᵢ = distance au masque le plus à gauche) et remplir les positions dont `H̃ᵢ < τ`. | test de sélection + refus de précondition sur un modèle non compatible | DEFINED |
| R-26 | **Dans le profil de stratégie `WeDLM` uniquement** : la métrique de progression MUST être la croissance du préfixe contigu cache-valide (`p_cache`), pas la couverture de masque. MUST être une règle déclarée par le profil, pas une constante de l'ordonnanceur. | `p_cache` croissant mesuré sur un modèle compatible WeDLM | DEFINED |
| R-27 | **Dans le profil `WeDLM` uniquement** : le remplissage MUST se faire à la volée dans la fenêtre glissante, à travail par forward quasi constant ; NEVER les pauses de fin de bloc. MUST être la règle de fenêtre du profil. | travail par forward mesuré quasi constant | DEFINED |
| R-91 | **Interface des profils de stratégie** : chaque stratégie MUST déclarer ses métriques de progression, ses conditions de cache, ses règles de fenêtre, ses préconditions et ses granularités. L'ordonnanceur d'inférence MUST consommer cette interface et MUST NOT utiliser une métrique, une règle de fenêtre ou une précondition globale commune à toutes les stratégies. MUST inclure la stratégie d'ajout du canvas dénoisé au contexte (DiffusionGemma) comme profil distinct de la fenêtre glissante. | interface typée par stratégie, testée sur deux profils distincts (WeDLM et multi-canvas) | DEFINED |
| R-28 | La politique de commit MUST être déterministe pour une graine donnée **à l'intérieur d'un même backend** (mêmes kernels, mêmes formes de workgroup, mêmes limites). MUST définir la tolérance **entre backends** (§6.3) : NEVER exiger une identité bit-à-bit entre le chemin CPU et le chemin WebGPU pour les réductions `f32`. | deux exécutions même graine sur le même backend → même séquence de commits ; parité entre backends dans la tolérance consignée | DEFINED |

### 3.5 Mémoire polymorphe

| # | Exigence | Critère d'acceptation | Spéc. |
|---|---|---|---|
| R-29 | Le moteur MUST gérer une hiérarchie de mémoire unique sur VRAM, RAM, NVMe. | une seule hiérarchie, une seule interface d'allocation | DEFINED |
| R-30 | Le moteur MUST classer les objets par **rôle** (chaud, tiède, froid, persisté, table, scratch) et MUST placer chaque objet selon son rôle, sa taille, sa fréquence d'accès et son coût de transfert. MUST NOT imposer une partition matérielle fixe par catégorie d'objet. **Une contrainte de placement déclarée par le checkpoint ou par un profil de stratégie (ex. `table_ngram` sur NVMe, R-32 / R-64) est une contrainte d'entrée de l'ordonnanceur de ressources, pas une partition du matériel : elle fixe la classe de placement, pas l'adresse.** | le placement est reproductible à partir des besoins déclarés et des limites du device ; la contrainte déclarée est journalisée comme contrainte, pas comme adresse | DEFINED |
| R-31 | Le moteur MUST garder en RAM un ensemble d'experts **selon le budget accordé** ; la politique MUST être compatible avec des modèles dont la taille dépasse la mémoire système disponible. | admission explicite (R-34, R-88) ; le placement est décidé, pas imposé | DEFINED |
| R-32 | **Si le checkpoint déclare `table_ngram = true`** : le moteur MUST garder les tables n-gram complètes sur NVMe et MUST les lire par lignes. MUST refuser un checkpoint déclarant une table n-gram sans support de lecture par lignes. **Si `table_ngram = false` : ne rien stocker, ne rien lire** (DiffusionGemma et Clover la déclarent fausse). | chemins exercés dans les deux cas | DEFINED |
| R-33 | Les décisions de placement MUST prendre en compte le coût de transfert, la taille du bloc, la réutilisation de préfixe, les fenêtres glissantes, la structure des plans d'exécution. | décision journalisée avec le coût de transfert et le coût de calcul | DEFINED |
| R-34 | Le moteur MUST effectuer des contrôles d'admission : une requête, un état de cache, un pool d'experts, une fenêtre de génération est admise ou refusée en fonction des limites mesurées du device. | admission / refus par limites mesurées | DEFINED |
| R-35 | Le moteur MUST utiliser des tampons double pour le streaming de poids et la persistance. | double buffer vérifié sans copie redondante | DEFINED |
| R-36 | Le moteur MUST masquer les chargements (FreeToken) : charger un expert manquant pendant l'inférence de l'expert partagé et des experts déjà résidents ; charger la couche suivante pendant l'inférence de la couche courante. | benchmark transferts / calcul sur une cible à experts en RAM | EXPERIMENTAL_SPEC |
| R-37 | Le moteur MUST garantir la **compatibilité et la correction** du backend CPU pour les experts manquants (calcul en place, même sémantique que le chemin device). La **participation du CPU à l'exécution d'une session sur accélérateurs** est une **décision de l'ordonnanceur de ressources** : elle est activée lorsque le coût de transfert est mesuré supérieur au coût de calcul (R-89), pas par une règle imposée. MUST calibrer le ratio par mesure (`I-11`, R-89). | parité numérique device/CPU vérifiée sur la cible MoE ; activation de l'offload justifiée par les mesures de coût | DEFINED |
| R-38 | Le cache d'experts MUST être un pool LRU à capacité adaptative (taux d'occupation d'une fraction de VRAM). | LRU mesuré, capacité adaptative vérifiée | DEFINED |
| R-39 | Le moteur MUST adapter le cache d'experts aux métriques de routeur (fréquence d'apparition, entropie, couverture de top-k). | adaptation mesurée par métriques de routeur | DEFINED |
| R-40 | Le moteur MAY anticiper la sélection d'experts (prediction routing) : prédire les experts des 1 à 2 prochains pas afin de déclencher le transfert en avance. MUST être mesuré (justesse, gain de débit, coût mémoire) avant activation. | protocole de mesure + seuil d'abandon | RESEARCH |
| R-41 | Le cache MUST avoir un **plancher de capacité** garantissant que le pool tient au moins les experts d'un pas complet, **calculé par la formule du modèle** (experts activés, top-k, taille de bloc, taille de fenêtre), pas par une constante globale. | formule dérivée par cible ; pool d'un pas sans thrashing mesuré | DEFINED |
| R-42 | Le moteur MUST minimiser les copies, rendre les transferts explicites et utiliser des buffers de staging. MUST utiliser les plans d'exécution et la réallocation dynamique de VRAM. **La mémoire hôte épinglée zéro-copy n'est pas une exigence portable** : elle MAY être exploitée par un backend dont le runtime natif l'offre, comme optimisation déclarée. | nombre de copies par pas mesuré ; optimisation de backend déclarée comme optionnelle | DEFINED |

### 3.6 Persistance et reprise

| # | Exigence | Critère d'acceptation | Spéc. |
|---|---|---|---|
| R-43 | Le moteur MUST persister des checkpoints d'états d'inférence (KV, états récurrents, états de blocs) sur NVMe. | reprise à partir du fichier | DEFINED |
| R-44 | Le moteur MUST reprendre une session depuis un checkpoint persisté (sessions locales longues). | reprise après arrêt du processus | DEFINED |
| R-45 | Le moteur MUST réutiliser un préfixe persisté entre requêtes (radix cache). MUST définir les clés d'identité (hachage du préfixe tokenisé, positions, politique d'attention, politique de validation) et les conditions d'invalidation. | deux requêtes à préfixe identique partagent l'état ; un changement de version l'invalidé | DEFINED |
| R-46 | La persistance MUST respecter la sémantique de la politique d'attention : un état récurrent MUST être cohérent avec la position d'acceptation, pas seulement coupé comme un tail de KV. | état récurrent restauré à la position d'acceptation | DEFINED |
| R-47 | Le moteur MUST valider la cohérence d'un checkpoint persisté avant de le charger. | validation au chargement, rejet explicite | DEFINED |

**Sécurité et rétention des données persistées** (la persistance stocke des contextes, prompts, historiques et états de session sensibles) :

| # | Exigence | Critère d'acceptation | Spéc. |
|---|---|---|---|
| R-75 | La persistance des sessions MUST être **explicitement activée** par l'opérateur, jamais par défaut silencieux. L'emplacement écrit MUST être journalisé au démarrage. | configuration explicite, journal de démarrage | DEFINED |
| R-76 | MUST définir la durée de rétention, la suppression d'une session, la suppression de toutes les sessions, et exposer une interface de suppression. | suppression vérifiable, sans résidu | DEFINED |
| R-77 | MUST définir les permissions des fichiers persistés (propriétaire, mode, racine de configuration de l'opérateur) et journaliser l'emplacement écrit. Aucun écrit hors de la racine déclarée. | permissions vérifiées par test | DEFINED |
| R-78 | MUST invalider toute persistance lors d'un changement de modèle, de révision du checkpoint, de tokenizer, de format de poids ou de politique de validation. | invalidation déclenchée par la détection de version | DEFINED |
| R-79 | MUST documenter la sensibilité des données persistées ; MUST proposer un chiffrement au repos ou documenter explicitement son absence, sans la masquer. | avertissement au démarrage si persistance active et non chiffrée | DEFINED |

### 3.7 Serving

| # | Exigence | Critère d'acceptation | Spéc. |
|---|---|---|---|
| R-48 | Le moteur MUST exposer une API **OpenAI-compatible** (chat completions, streaming SSE, tool calls). | matrice Annexe B remplie et testée par endpoint | DEFINED |
| R-49 | Le moteur MUST servir plusieurs sessions avec une file de workers et un ordonnanceur de batch, **selon des critères d'admission explicites** : budget mémoire des états par requête, longueur de préfixe, politique d'attention compatible, mode de génération identique. | critères de regroupement et d'admission configurés ; latence et mémoire mesurées | DEFINED |
| R-50 | Le moteur MUST servir un service local (127.0.0.1) par défaut ; une clé d'API MUST être disponible pour toute exposition au-delà de la boucle locale. | refus d'exposition sans clé | DEFINED |
| R-51 | Le CLI MUST être la porte d'entrée des agents (invocable, non-interactive, sortie JSON). | sortie JSON stable | DEFINED |
| R-80 | Le moteur MUST exposer une API **Anthropic-compatible** (messages, streaming, tool use, `system`, `stop_reason`, format d'erreurs). | matrice Annexe B remplie et testée par endpoint | DEFINED |
| R-81 | MUST documenter la matrice de compatibilité par API : endpoints, champs acceptés, streaming, contenus, erreurs, fonctionnalités prises en charge et **non prises en charge**. | Annexe B remplie, divergence documentée, pas masquée | DEFINED |
| R-82 | MUST refuser explicitement une requête demandant une fonctionnalité non prise en charge ; NEVER la dégrader silencieusement. | réponse d'erreur normalisée par API | DEFINED |

**API de contrôle** (produit local, pilotable par un agent) :

| # | Exigence | Critère d'acceptation | Spéc. |
|---|---|---|---|
| R-92 | MUST exposer une API de contrôle pour **charger** et **décharger** un modèle et pour lister les modèles chargés avec leurs capacités déclarées (R-85) et leur mode courant. | chargement / déchargement observables, refus d'admission explicite | DEFINED |
| R-93 | MUST exposer l'**état des sessions** (liste, état, positions dénoisées, longueur de préfixe, politique active) et le **contrôle** (pause, reprise, arrêt, purge de session). | état lisible, contrôle effectif, purge sans résidu | DEFINED |
| R-94 | MUST exposer l'**état des ressources** : budget accordé par device, objets résidents par rôle, transferts en cours, décisions d'admission, occupation par rôle. | état lisible et corrélé aux décisions journalisées (R-84..R-89) | DEFINED |
| R-95 | MUST exposer la **configuration effective** (racine de persistance, rétention, formats acceptés, limites du device, fonctionnalités absentes) et les **métriques** (§9) en sortie JSON stable. | sortie JSON stable, versionnée | DEFINED |
| R-96 | L'API de contrôle MUST être servie par le **CLI non-interactive** **et** par un endpoint sur la boucle locale (`127.0.0.1`). MUST refuser toute exposition au-delà de la boucle locale sans clé d'API (`R-50`). | CLI et endpoint exercés, refus d'exposition sans clé | DEFINED |

### 3.8 Validation

| # | Exigence | Critère d'acceptation | Spéc. |
|---|---|---|---|
| R-52 | Le moteur MUST valider la sortie GPU contre des logits de référence. | tolérance mesurée | DEFINED |
| R-53 | Le moteur MUST valider les représentations de couches de poids contre des représentations de référence. | reconstruction vérifiée | DEFINED |
| R-54 | Le moteur MUST valider les décisions de validation de jetons (quand la validation est sensible à des seuils de confiance). | la décision est mesurée elle-même, pas seulement la sortie | DEFINED |
| R-55 | Le moteur MUST exécuter des tests de régression de capacités contre un checkpoint réel (suites de cas, pas des scores de leaderboard). | suites exécutées sur la cible de référence | DEFINED |
| R-56 | Le moteur MUST produire des mesures reproductibles (débit, temps kernel, temps de transfert, occupation mémoire) associées à chaque optimisation. | mesure jointe à l'optimisation | DEFINED |
| R-57 | Un changement affectant l'attention, le KV cache, la sémantique de validation, la sélection d'experts, le placement mémoire MUST être testé sur les chemins affectés **avant** d'être retenu. | tests des chemins affectés avant retention | DEFINED |

### 3.9 Format de checkpoint et représentation de poids

| # | Exigence | Critère d'acceptation | Spéc. |
|---|---|---|---|
| R-58 | Le moteur MUST lire des checkpoints au format canonique du dépôt (format interne à spécifier : **zone 1 de §14**), MUST lire GGUF et safetensors comme sources d'import. | import vérifié sur les cibles de référence | DEFINED |
| R-59 | Le format MUST conserver la sémantique du checkpoint : formats de poids, structures d'attention, schémas de routage, top-k, nombres d'experts, échelles globales. | fidèle aux déclarations du checkpoint | DEFINED |
| R-60 | Les formats de poids sont ordonnés par **niveau de priorité de format** : **F-1** (prioritaire) BF16, FP8 E4M3, MXFP4, INT4 group-quantized, FP4 E2M1, INT8 ; **F-2** (secondaire) ggml q<N> (Q3, Q4, Q5, Q6), NVFP4, i-quants ; **F-3** (extensions ultérieures) FP6, FP16 scaled, FP4 scaled, INT3, INT2. MUST nommer les niveaux `F-1` / `F-2` / `F-3` ; NEVER les confondre avec les jalons `P0..P5`. | chaque format F-1 a une référence numérique et un test de déquantification | DEFINED |
| R-61 | Les formats MUST supporter la conversion de formats de poids vers des représentations de stockage, de transfert et de kernel **sans changer la sémantique**. | trois représentations coordonnées par la métadonnée canonique | DEFINED |
| R-62 | Un format MUST stocker les formats de poids, les structures d'attention, les schémas de routage, la métadonnée de validation. | lecture complète au chargement | DEFINED |
| R-63 | L'import MUST refuser un checkpoint dont la sémantique ne peut être reproduite, avec un diagnostic explicite (NEVER un chargement silencieux). | refus au chargement | DEFINED |
| R-64 | **Si `table_ngram = true`** : les tables n-gram MUST rester sur disque et MUST être lues par lignes par token, avec leurs échelles résidentes en RAM. **Si `table_ngram = false`** : aucune table, aucune lecture. MUST refuser un checkpoint déclarant la table sans le chemin de lecture. | chemins exercés dans les deux cas | DEFINED |
| R-83 | MUST refuser un checkpoint portant un format de poids non listé en indiquant le format rencontré ; NEVER requantifier silencieusement. | refus au chargement | DEFINED |

**Trois représentations MUST être définies et coordonnées par la métadonnée canonique** :

```mermaid
flowchart LR
  STORE["Représentation de stockage (q&lt;N&gt;, compacte, bit-packed)"] --> TRANS["Représentation de transfert (alignée 256o, DMA-ready)"]
  TRANS --> KERNEL["Représentation de kernel (layout calculé, décodée)"]
  STORE -.-> KERNEL
```

- Le décodage de poids MUST se faire pendant le transfert, caché par le calcul de la couche courante.
- Les couches de poids MUST rester stockées compactement, MUST être déplacées par le mouvement mémoire, MUST être calculées par les kernels.
- **Échelles E8M0 (spécification OCP Microscaling)** : E8M0 est un exposant **non signé à biais 127** ; il n'a **pas** de représentation du zéro ni de l'infini. `0x00` = `2^-127` (plus petite valeur positive finie), `0xFF` = **NaN réservé**. MUST préserver `0x00` comme valeur finie, NEVER comme `+0` ; MUST traiter `0xFF` comme NaN. MUST définir explicitement le comportement d'admission face à une échelle NaN : **refuser le checkpoint** avec le diagnostic de la couche concernée, NEVER produire une valeur silencieusement fausse.
- MUST appliquer la même règle aux échelles des formats groupés (`FP8` par groupe, `MXFP4`, `NVFP4`, `INT4` group-quantized, `ggml q<N>`), MUST consigner les tolérances de déquantification par format dans la matrice L4 (§6.3).

### 3.10 QA

| # | Exigence | Critère d'acceptation | Spéc. |
|---|---|---|---|
| R-65 | La validation MUST utiliser des logits de référence (sortie d'une implémentation de référence du même checkpoint, même graine, même politique d'échantillonnage). | référence déclarée par cible | DEFINED |
| R-66 | Les tolérances MUST être exprimées par format de poids et par politique d'attention, et MUST être mesurées, pas estimées. | valeurs chiffrées mesurées | DEFINED |
| R-67 | Les tests de régression de capacités MUST tourner contre un vrai checkpoint (suites de cas : noyau, difficile), MUST être re-exécutables sans génération pour la notation. | suites re-exécutables | DEFINED |
| R-68 | Un chemin plus rapide avec drift inexpliqué de l'attention, du KV cache ou des logits MUST être rejeté. | rejet documenté | DEFINED |
| R-69 | Les tests MUST couvrir : chemin GPU normal, streaming NVMe, chemins d'offload, chemin CPU-only, chemins de décodage quantifié, chemins de reprise de session. | chaque chemin exercé | DEFINED |
| R-70 | Les tests live de l'API MUST être exécutés uniquement quand la surface d'API est intentionnellement testée. | tests par endpoint | DEFINED |

### 3.11 Conversion AR → dLLM (outillage externe, hors moteur)

`N-1` fixe la décision : **le moteur ne convertit pas et n'entraîne pas.** R-71..R-74 décrivent l'outillage externe qui produit les checkpoints adaptés ; elles ne sont pas implémentées dans le moteur et ne MUST PAS être intégrées au chemin d'exécution. Le moteur MUST uniquement **charger** leurs artifacts (R-1, R-16, `AdaptationFile` §8.1).

| # | Exigence (outillage externe) | Spéc. |
|---|---|---|
| R-71 | MUST concevoir la conversion AR → dLLM comme un pipeline externe de génération de checkpoints (supervision dual-stream clean/noisy, capacité AR du checkpoint source conservée). | FUTURE |
| R-72 | MUST supporter une supervision de type REPR-ALIGN (alignement cosinus par couche vers un enseignant AR congelé, sans modules d'adaptation, loss pondéré, gel de sous-ensembles de couches). | FUTURE |
| R-73 | MUST supporter les masques de blocs avec packing au niveau document (échantillons de longueurs variables packés sans padding). | FUTURE |
| R-74 | MUST supporter le logit shift et le masquage aléatoire du noisy-stream, sans les traiter comme pilotes primaires de la qualité. | FUTURE |

**Conséquence** : la « conversion de checkpoints dans le moteur » est retirée des jalons. L'outillage de conversion est un projet séparé, avec ses propres critères de validation, son propre périmètre (préparation de modèles, pas exécution de production) et ses propres dépendances.

### 3.12 Ordonnanceur de ressources

| # | Exigence | Critère d'acceptation | Spéc. |
|---|---|---|---|
| R-84 | MUST disposer d'un **ordonnanceur de ressources** distinct de l'ordonnanceur d'inférence : il décide où réside chaque objet (device, RAM, NVMe, CPU), quand il est déplacé, sous quel budget de transfert. | les deux ordonnanceurs communiquent par une interface typée (requêtes de placement, budgets accordés, rejets) | DEFINED |
| R-85 | Les modèles MUST déclarer leurs besoins d'exécution (états requis, formats, capacités de décodage, préconditions) ; l'ordonnanceur de ressources MUST décider où les satisfaire. Un modèle sans déclaration n'est pas chargé. | refus de chargement sans déclaration | DEFINED |
| R-86 | MUST valider les limites du device (`maxBufferSize`, `maxStorageBufferBindingSize`, `maxUniformBufferBindingSize`, `maxComputeInvocationsPerWorkgroup`, `maxComputeWorkgroupStorageSize`, `maxBindGroups`, `minStorageBufferOffsetAlignment`) **avant** l'encodage des commandes, et échouer explicitement, jamais silencieusement. | le plan est validé contre les limites mesurées en P0 | DEFINED |
| R-87 | MUST garantir que la somme des objets résidents tienne dans le budget accordé par le device. | budget champ du plan, mesuré | DEFINED |
| R-88 | MUST refuser explicitement un modèle dont les besoins dépassent la mémoire disponible, en indiquant si l'exécution est **I/O-dépendante** ; NEVER masquer la contrainte derrière un cache. | message d'admission avant chargement | DEFINED |
| R-89 | MUST comparer le coût de transfert et le coût de calcul sur le lieu de résidence pour décider le placement et l'offload (R-37, R-36). | décision journalisée avec les deux coûts | DEFINED |

### 3.13 Mode CPU

| # | Exigence | Critère d'acceptation | Spéc. |
|---|---|---|---|
| R-90 | MUST supporter le **mode CPU-only complet** sur une machine sans GPU compatible : exécution d'une session entière (chargement, calcul, APIs, persistance) avec les mêmes exigences numériques, de déterminisme et de reprise que tout autre backend (`I-11`). | session complète exécutée sans GPU, débit mesuré et informé à l'opérateur | DEFINED |

---

## 4. Architecture technique

### 4.1 Couches et noyau

```mermaid
flowchart TD
  subgraph SERVE["Surface servie (Rust)"]
    CLI["CLI (agent, JSON)"]
    API["API OpenAI + API Anthropic"]
    PERSIST["Persistance & reprise de session"]
  end
  subgraph CORE["Noyau (Rust)"]
    STATE["Machine d'états de session & de jetons"]
    SCHED["Ordonnanceur d'inférence (horizon, fenêtre, sélection)"]
    RES["Ordonnanceur de ressources (placement, budget de transfert)"]
    PLAN["Plans d'exécution & fichiers de kernels"]
    ADMIT["Contrôles d'admission"]
    VALID["Système de validation & mesures"]
  end
  subgraph MEM["Représentation mémoire (Zig)"]
    LAYOUT["Layouts & indexation compactes"]
    POOLS["Pools polymorphes & LRU d'experts"]
    DECODE["Décodage de poids & staging"]
    ABI["ABI canonique (structs C, pas de C++)"]
  end
  subgraph GPU["Accélération portable (WebGPU)"]
    KERNELS["Kernels WGSL"]
    BUFFERS["Buffers & files de commandes"]
    LIMITS["Découverte des limites du device"]
  end
  subgraph CPUP["Backend CPU (Zig)"]
    CPUK["Kernels CPU, offload d'opérations, mode CPU-only"]
  end
  SERVE --> CORE
  CORE --> RES
  RES --> MEM
  CORE --> MEM
  MEM --> GPU
  GPU --> MEM
  CORE --> GPU
  MEM --> CPUP
  CPUP --> MEM
```

Frontière normative : **Rust** possède les décisions, l'état et les contrats ; **Zig** possède la représentation et le mouvement des données ; **WebGPU** et le **CPU** possèdent l'exécution. Un contrat ABI partagé (structs C, pas de C++) est le seul moyen de passer des données entre couches.

### 4.2 Métadonnées canoniques

- Un seul jeu de métadonnées MUST décrire les shapes de tenseurs, les types de kernels, les layouts de buffers, les formats de poids, les exigences d'alignement.
- Le contrat d'interface entre couches MUST être basé sur cette source unique de métadonnées.
- Les couches MUST partager la sémantique par des formats communs, des buffers communs et des contrats explicites — NEVER par des suppositions sur la profondeur d'appel.

### 4.3 Catalogue de kernels dLLM

```mermaid
flowchart LR
  MASK["Mask positions"] --> ATT["Attention (causal / bi- intra-bloc)"]
  ATT --> LOG["Logits"]
  LOG --> CONF["Confidence"]
  CONF --> SEED["Seed tokens"]
  CONF --> COMMIT["Commit & prefix extension"]
  LOG --> SHIFT["Logit shift"]
  COMMIT --> STATE["State update (KV / récurrent)"]
  STATE --> MASK
```

| Kernel | Rôle contractuel |
|---|---|
| mask positions | pose des `[MASK]` sur la fenêtre active |
| attention | causal sur préfixe autorisé ; bidirectionnel intra-bloc si la politique le déclare |
| logits | sortie par position, propre au stream |
| logit shift | aligne la prédiction d'un bloc sur ses positions (FLARE) |
| confidence | score de confiance par position (entropie, marge, seuil) |
| seed tokens | sélection des positions à remplir (entropie + pénalité de distance, R-25) |
| commit & prefix extension | validés → préfixe, préfixe → cache-valide |
| state update | KV softmax et état récurrent linéaire |
| expert gather / scatter | LRU d'experts, H2D, scatter des poids manquants |
| quant decode | décodage q<N> / fp4 / fp8 / ggml vers le layout de kernel |

Chaque kernel MUST être compact, parallèle, compatible avec les plans d'exécution, et MUST éviter les allocations inutiles, les dépendances excessives, les synchronisations superflues. MUST exister en version device **et** en version CPU pour les chemins d'offload (R-37, R-90).

### 4.4 Plans d'exécution et WebGPU

WebGPU n'offre pas de capture de graphe équivalente aux graphes CUDA. MUST distinguer six niveaux, sans jamais les présenter comme équivalents :

1. **représentation statique d'un plan** ;
2. **réutilisation des pipelines et des layouts** ;
3. **réutilisation des buffers** ;
4. **encodage des commandes** ;
5. **soumission GPU** ;
6. **replay native d'un graphe**, uniquement lorsque le backend l'offre.

Le moteur MUST émuler un plan d'exécution par :

1. compilation WGSL au chargement, pipeline cache keyed par identifiant canonique de kernel ;
2. plans à forme fixe : bind group layouts stables, shapes pré-établies, un command encoder regroupe les kernels d'un pas ;
3. offsets dynamiques (workgroup count, dynamic buffer offsets) pour les variantes de taille ;
4. réutilisation de buffers par pool, NEVER un buffer par requête.

**Le chemin WebGPU portable MUST NOT être présenté comme équivalent à CUDA Graphs.**

### 4.5 Limites et fonctionnalités du device

Le moteur MUST lire les limites réelles du device au démarrage et adapter les kernels aux limites, jamais les supposer. Les identifiants MUST être exacts (`GPUSupportedLimits` / wgpu) :

| Limite WebGPU | wgpu (Rust) |
|---|---|
| `maxBufferSize` | `max_buffer_size` |
| `maxStorageBufferBindingSize` | `max_storage_buffer_binding_size` |
| `maxUniformBufferBindingSize` | `max_uniform_buffer_binding_size` |
| `maxComputeInvocationsPerWorkgroup` | `max_compute_invocations_per_workgroup` |
| `maxComputeWorkgroupStorageSize` | `max_compute_workgroup_storage_size` |
| `maxComputeWorkgroupSizeX/Y/Z` | `max_compute_workgroup_size_x/y/z` |
| `maxComputeWorkgroupsPerDimension` | `max_compute_workgroups_per_dimension` |
| `maxStorageBuffersPerShaderStage` | `max_storage_buffers_per_shader_stage` |
| `maxUniformBuffersPerShaderStage` | `max_uniform_buffers_per_shader_stage` |
| `maxSampledTexturesPerShaderStage` | `max_sampled_textures_per_shader_stage` |
| `maxSamplersPerShaderStage` | `max_samplers_per_shader_stage` |
| `maxBindGroups` | `max_bind_groups` |
| `minStorageBufferOffsetAlignment` | `min_storage_buffer_offset_alignment` |
| `minUniformBufferOffsetAlignment` | `min_uniform_buffer_offset_alignment` |

Les noms `maxBoundStorageBufferSize`, `maxInvocationsPerInvocation`, `maxTotalBufferSize`, `maxBoundUniformBuffers` **n'existent pas** dans la spécification WebGPU ; un agent MUST NOT les rechercher.

**Fonctionnalités à déclarer** — MUST utiliser les noms exacts (WebGPU / wgpu), MUST les tester sur le device au démarrage :
- `shader-f16` (`SHADER_F16`) — active le type `f16` dans les shaders.
- `timestamp-query` (`TIMESTAMP_QUERY`) — timing GPU-side.
- `texture-compression-bc`, `texture-compression-etc2`, `texture-compression-astc` (`TEXTURE_COMPRESSION_BC` / `_ETC2` / `_ASTC`) — la compression est spécifiée **par format**, il n'existe pas de feature générique `texture-compression`.
- `float32-filterable` (`FLOAT32_FILTERABLE`) — uniquement si des textures float32 sont échantillonnées avec filtrage.
- **`float64` n'est pas une fonctionnalité WebGPU** : WGSL n'expose pas de type double précision dans les shaders. Les calculs en double précision sont un choix **CPU-side** (backend Zig), déclaré comme capacité du backend CPU, jamais comme capacité GPU à détecter.
- **`subgroups`** (`SUBGROUP`) — parallélisme SIMD dans la workgroup, disponible dans Chrome depuis la version 134 (26 février 2025). MUST être testée sur le device, NEVER activée sans qu'elle soit exposée. WGSL : `enable subgroups;`, built-ins `subgroup_invocation_id` et `subgroup_size`, fonctions collectives (`subgroupAdd`, `subgroupMul`, `subgroupMin`, `subgroupMax`, `subgroupBallot`, `subgroupBroadcast`, `subgroupShuffle`, scans exclusifs / inclusifs, fonctions quad). Le device publie `subgroupMinSize` / `subgroupMaxSize` dans les informations d'adaptateur : MUST lire ces valeurs, NEVER supposer la taille d'un subgroup. `f16` et subgroups se combinent en demandant **les deux** fonctions (`shader-f16` **et** `subgroups`). Extensions WGSL à vérifier sur la cible : `subgroup_id`, `subgroup_uniformity`, contrôle de la taille de subgroup.

**Précision numérique** : la sémantique numérique de la production est **f32 dans les shaders** ; les poids stockés MUST être décodés vers le layout de kernel ; les conversions MUST être explicites. **WGSL définit le type `f16`**, mais son usage requiert `enable f16;` dans le module **et** la fonctionnalité `shader-f16` sur le device. La règle normative : **le chemin de référence MUST fonctionner sans `f16`** ; lorsque le device expose `shader-f16`, une variante spécialisée MAY être proposée après validation numérique et benchmark. NEVER compter sur `f16` comme chemin de base.

**Atomics** : les atomics de base WGSL portent sur `i32` et `u32` ; il n'existe pas d'atomic `f32` dans le langage de base. Les réductions MUST être réalisées par des passes hiérarchiques dans la workgroup memory.

**Mapping des buffers** : `MAP_READ` se combine uniquement avec `COPY_DST`, `MAP_WRITE` uniquement avec `COPY_SRC` ; un buffer mappé n'est pas disponible pour les opérations GPU tant qu'il reste mappé. Conséquence : la mémoire hôte épinglée zéro-copy n'est pas une garantie portable (`R-42`).

**Requêtes de timing** : les requêtes de timestamps GPU MAY ne pas être disponibles. MUST fournir des mesures de timing CPU-side (timer sur submit + map) comme chemin de base, et la précision GPU-side uniquement quand le device l'expose.


**Réduction dans la workgroup** : la réduction par atomics `i32` / `u32` dans la workgroup memory est le **chemin de référence** ; la réduction par subgroup est une **variante mesurée**, MAY activée après comparaison mesurée (débit, stabilité numérique) sur le device qui expose `subgroups`, NEVER le chemin de base.

### 4.6 Backend CPU

- backend de référence et de debug ;
- **backend de production** : mode CPU-only complet (R-90) et offload d'opérations dans les sessions servies sur accélérateurs (R-37) ;
- sémantique partagée avec les kernels device, les formats de poids, les APIs, la persistance ;
- instrumentation du coût de calcul et du coût de transfert évité, afin que la décision d'offload soit mesurée (R-89) ;
- NEVER comme chemin de performance de référence pour les modèles à forte densité de calcul (`N-7`).

### 4.7 Interfaces et ABI

- MUST définir les ABI de bas niveau entre la stack Rust et les modules de performance en Zig **avant P0** : signatures, limites de types, conventions d'erreurs, modes d'appels, transferts de buffers, formats de sérialisation, règles d'alignement (`min_storage_buffer_offset_alignment`).
- MUST partager les formats de tenseurs et les formats de scale.
- MUST conserver la granularité native des poids.
- NEVER dupliquer les formats canoniques ; NEVER convertir silencieusement.

---

## 5. Stratégies de génération

### 5.1 Mapping AR / dLLM sur le contrat commun

| Étape du contrat | dLLM natif | AR |
|---|---|---|
| préfixe autorisé | préfixe + positions masquées | préfixe complet |
| horizon | bloc, fenêtre, horizon élastique | 1 position |
| politique d'attention | causale sur le préfixe + bidirectionnelle intra-bloc | causale stricte |
| validation | seuil de confiance, entropie, marge | validation immédiate |
| mise à jour d'état | KV, état récurrent, état de bloc, état intermédiaire | KV |
| décision d'horizon | taille de bloc, fenêtre glissante, extension | extension de préfixe |
| granularité de restauration | ensemble de positions + masque, état récurrent à la frontière | point de KV |

**Règle normative** : le contrat commun est typé par les **trois dimensions canoniques** de §7.1 — `generation_family` (`AR` | `Diffusion`), `architecture_topology`, `generation_capabilities` — **jamais par un champ unique** ; `Hybride` n'est pas une valeur (`I-1`). MUST partager les objets (bloc, masque, fenêtre, horizon, préfixe, état, validation) et le protocole (commit, rewind, journal). NEVER unifier en écrivant une boucle unique de génération, NEVER traiter l'AR comme une spécialisation booléenne du dénoisage (`I-1`, R-15, `N-11`).

### 5.2 Streaming parallèle (WeDLM)

- Préconditions de l'adaptateur : décodage causale strict, préfixe contigu, logits disponibles pour les positions masquées. MUST refuser un modèle sans ces préconditions (`I-14`).
- La sélection de remplissage MUST utiliser l'entropie pénalisée par la distance : `H̃ᵢ = Hᵢ + λ·dᵢ`, remplissage des positions sous le seuil τ.
- Le préfixe contigu rempli MUST être immédiatement cache-valide sous attention causale stricte.
- La fenêtre glissante MUST conserver une taille W fixe ; à chaque pas les jetons commités sortent et un nombre égal de `[MASK]` entre.
- Le streaming causal MUST rester nativement supportable par les kernels d'attention causale (prefill en continu), sans kernel modifié.
- τ ∈ [0.3, 0.6] est la plage recommandée ; λ > 0 contrôle la préférence gauche→droite. Ces valeurs MUST être mesurées sur la cible de référence avant d'être retenues comme défaut.

### 5.3 Horizon élastique (FLUID)

- Le décodage MUST être strictement causal sur le préfixe autorisé : aucune position masquée n'est visible.
- L'horizon de vérification MUST contenir un nombre borné de `[MASK]` (plafond d'horizon K).
- La porte de confiance MUST être paramétrée par un seuil par position γ.
- La fenêtre MUST avancer avec `stride = 1 + Δ` ; les positions sous le seuil sont décalées vers la droite.
- Le pas MUST être borné par la frontière de compétence : l'horizon ne s'étend pas au-delà de la confiance mesurée.

**Exigence de précision (R-21, R-22)** : MUST écrire la définition exacte de Δ, son domaine de valeurs, la conversion du stride en nombre entier de positions, et citer le numéro d'équation ou d'algorithme de l'article source. MUST séparer ensuite, explicitement, les modifications proposées par Alethe.ia. À défaut, cette politique est une **hypothèse de recherche** (`RESEARCH`), pas une règle de production.

### 5.4 Diffusion-Trust (bloc)

- Bloc de taille B derrière le préfixe causal, dénoisé sur S étapes.
- À chaque étape s : positions non résolues `Rₛ` ; score `cᵢ` par position ; `Aₛ = {i ∈ Rₛ : cᵢ ≥ γₛ}` ; à `s = S`, toutes les positions restantes sont commitées.
- Les forwards intermédiaires contiennent des contenus provisoires : leurs états récurrents MUST NOT être écrits dans l'état causal vivant.
- Après finalisation du bloc, le préfixe autorisé est reconstruit par **causal state replay** : les jetons du bloc sont appliqués à l'état causal.
- Précondition : checkpoint préparé en **dual-stream** (flux `clean` et `noisy`), déclaré par le checkpoint (R-13). NEVER le nommer « bilingue ».

### 5.5 AR-Trust (spéculatif sur clean-stream)

- Fenêtre `[y<t ; d₁..d_K ; [MASK]^(N-1)]` ; vérification clean-stream par jeton draft.
- Acceptation : `aᵢ ~ Bernoulli(min(1, pᵢ(dᵢ)/qᵢ(dᵢ)))`, `qᵢ` étant **exactement la loi de proposition qui a échantillonné dᵢ**. Recompter `qᵢ` pendant la vérification MUST NOT casser l'équivalence de distribution.
- Rejet : échantillonner la correction normalisée `[pᵢ − qᵢ]₊`.
- **Rewind de l'état récurrent à la frontière d'acceptation** : accepter r de K jetons MUST rembobiner l'état récurrent à `S⁽ʳ⁾`, pas seulement couper un tail de KV. Deux chemins : replay (r mises à jour par couche GDN) ou cache-and-scatter (états enregistrés par position de vérification, commit par un kernel fused gather-scatter).
- Précondition : checkpoint dual-stream (R-14). NEVER proposer ce mode sur un checkpoint à flux unique.

### 5.6 Vérification de couches récurrentes

- Pour les architectures hybrides (softmax + linéaire), le moteur MUST préserver la cohérence de l'état récurrent à la frontière de commit. NEVER traiter l'état récurrent comme un KV tail trim.
- Cible de référence : Clover (paramètres GDN bidirectionnels additionnels : right-convolution, contrôles softmax) et les modèles à GDN / Mamba / SSM déclarés.

### 5.7 Politiques de validation

- La politique de validation MUST être déclarée par le mode de génération : seuil d'entropie, marge, confiance de score, acceptation spéculative, arrêt adaptatif.
- MUST être un **validateur typé** (§6.3), pas un champ global.
- MUST être testable : un test MUST mesurer la décision de validation elle-même (pas seulement la sortie finale).
- MUST refuser un checkpoint qui ne remplit pas ses préconditions.

### 5.8 Modes de décodage dLLM et arrêt adaptatif

- MUST supporter `LowConfidence` (remplir les positions dont la confiance dépasse un seuil, sinon la plus confiante pour garantir la progression ; seuil 0.95 et bloc 32 sur Clover), `EntropyBound` (sélection sous une borne d'information mutuelle, ré-noisyation complète des positions non sélectionnées ; borne 0.1, max 48 pas, température 0.8 → 0.4 sur DiffusionGemma), `TopK` (sélection des k positions les plus confiantes).
- MUST supporter l'**arrêt adaptatif** : terminaison anticipée d'un canvas lorsque la confiance est stable, sur des conditions explicites (entropie moyenne sous un seuil **et** prédictions identiques sur deux pas consécutifs). MUST les exposer comme des paramètres mesurés, pas comme un champ opaque.
- MUST respecter la granularité native : canvas 256 (DiffusionGemma), bloc 32 (Clover), valeurs déclarées par les checkpoints adaptés.

---

## 6. Validation et QA

### 6.1 Hiérarchie de validation

```mermaid
flowchart TD
  L1["Logits de référence"] --> V1["Sortie GPU / CPU"]
  L2["Représentations de couches de poids"] --> V2["Reconstruction de couches"]
  L3["Décisions de validation (seuils de confiance)"] --> V3["Politique de validation"]
  L4["Tests de régression de capacités"] --> V4["Qualité des sorties"]
  L5["Mesures reproductibles"] --> V5["Débit, temps kernel, occupation mémoire"]
  V1 & V2 & V3 & V4 & V5 --> ADMIT["Contrôle d'admission de la structure de calcul"]
```

### 6.2 Système de validation

**Une interface, plusieurs validateurs typés.** Le protocole est partagé (journal, commit, rewind, rejet), la logique de décision est spécialisée. NEVER la fusionner en une logique unique (`I-4`, `N-11`).

```
interface Protocol :
  validate(session, step)   -> Commit | Reject(ValidationError)
  commit(session, positions)
  rewind(session, target)   -> Rewind { positions, kv_points, recurrent_points }
  journal(event)
```

| Validateurs | Préconditions | Granularité de restauration |
|---|---|---|
| `ValidatorPrefixe` | décodage causal strict, préfixe contigu | point de KV |
| `ValidatorBloc` | bloc, masque de positions, granularité de sélection | ensemble de positions + masque |
| `ValidatorSpéculatif` | couche de draft, parité avec le mode non-spéculatif, loi de proposition `qᵢ` exacte | point complet + état récurrent à `S⁽ʳ⁾` |
| `ValidatorÉtatRécurrent` | GDN / Mamba / SSM avec points de restauration | point d'état récurrent |
| `ValidatorPoids` | format listé, échelles, tolérance numérique | couche de poids |

**Un AR strict est servi par `ValidatorPrefixe`** ; il n'est pas un cas particulier de `ValidatorBloc`.

### 6.3 Matrice d'équivalence numérique

| Niveau | Comparaison | Tolérance |
|---|---|---|
| L1 | logits de la cible de référence vs implémentation de référence | tolérance par format de poids (mesurée, R-66) |
| L2 | état récurrent (GDN) après restauration à la frontière d'acceptation | exact |
| L3 | état de diffusion après commit vs référence | exact |
| L4 | déquantification des formats prioritaires vs référence FP32 | tolérance par format |
| L5 | sélection de positions sur les mêmes logits entre implémentations | tolérance mesurée |
| L6 | sortie texte d'une session persistée vs session en mémoire | exact |

**Portée du déterminisme** : MUST séparer trois niveaux — (1) **bit-à-bit** à l'intérieur d'un même backend, pour une graine et une forme de workgroup fixées (R-28) ; (2) **tolérance par dtype** entre le backend CPU et le backend WebGPU, consignée dans la matrice ci-dessus (les réductions `f32` ne sont jamais bit-à-bit entre les deux chemins) ; (3) **tolérance de sortie** entre sessions (textes identiques à L6). MUST consigner les valeurs chiffrées à la validation (R-66) ; NEVER déclarer un niveau de déterminisme sans la portée qui le définit.

### 6.4 Règles de QA et de persistance

- R-65..R-70 (R-52..R-57 pour la validation de sortie, de couches, de décisions, de régressions, de mesures, de tests des chemins affectés).
- MUST couvrir le chemin CPU-only (R-90) et les chemins d'offload (R-37) dans les suites de régression (R-69).
- **Persistance et reprise (R-75..R-79)** : suites d'acceptation — écriture explicite d'une session, lecture d'une session sauvegardée, bornes de rétention exercées, suppression effective (fichier supprimé, pas marqué), permissions de fichiers vérifiées, invalidation des clés après redémarrage, chiffrement au repos documenté (mécanisme + avertissement, pas une promesse). MUST NOT déclarer la persistance valide sans ces suites exercées sur la machine de référence.
- MUST consigner les tolérances chiffrées de §6.3 à chaque validation (R-66).

---

## 7. Écosystème de modèles

### 7.1 Trois familles de checkpoints

1. **Checkpoints dLLM natifs** — générés directement comme dLLM par leurs éditeurs : **DiffusionGemma**, **Clover**. Servis par `R-12` selon le profil d'attention déclaré.
2. **Checkpoints AR adaptés** — modèles AR convertis en dLLM par un outillage externe (méthode WeDLM, FLUID, FLARE ou REPR-ALIGN) et livrés avec un `AdaptationFile` : la **méthode** et le **nom du modèle adapté** sont deux champs distincts (R-16).
3. **Checkpoints AR natifs** — servis par le chemin causal natif (R-15).

**NEVER classer un dLLM natif comme checkpoint adapté** ; NEVER confondre une méthode d'adaptation (WeDLM, FLUID, FLARE, REPR-ALIGN) avec un identifiant de modèle.

`AdaptationFile { adaptation_method, adapted_model_name, strategy, attention_profile, mapping_poids, couches_converties, preconditions, limites_validité, références_numériques, version_checkpoint, version_outillage }`. MUST lire l'artifact d'adaptation et refuser un checkpoint adapté sans artifact. NEVER produire l'artifact dans le moteur.

**Capacités déclarées par le checkpoint** (préconditions de R-13, R-14, R-16, R-25, R-85, R-91) :

```
Capabilities {
  generation_family       : AR | Diffusion            # stratégie de génération native
  architecture_topology   : DecoderOnly | EncoderDecoder | HybridAttention | HybridRecurrent
  generation_capabilities : [AR, Diffusion, Speculative, DualStream]   # capacités réellement servies
  profil_attention        : AttentionProfile          # sous-modules et leurs relations (§7.2.1)
  taille_bloc             : u32                       # 256 (DiffusionGemma), 32 (Clover)
  fenêtre_glissante       : u32                       # 1024 (DiffusionGemma)
  contexte_max            : u32                       # 256K / 128K
  experts                 : { total, actifs, partagés, top_k }
  formats_poids           : [Format]
  états_recurrents        : [GDN | Mamba | SSM | None]
  dual_stream             : bool                      # précondition R-13, R-14, R-97
  table_ngram             : bool                      # précondition R-32, R-64
  weDLM_compatible        : bool                      # précondition R-25
  adaptation              : Option<AdaptationFile>
}

AttentionProfile {
  sous_modules      : [ { nom, type_attention, positions_masquées, cross_attention, fenêtre, préfixe_visible } ]
  règles_progression : [ProgressionRule]              # métriques, conditions de cache, règles de fenêtre (R-91)
}
```

**`Hybride` n'est pas une valeur de `generation_family`** : le terme est trop large (architecture hybride, stratégie multi-mode, combinaison AR/diffusion dans un même checkpoint). MUST séparer la famille de génération, la topologie d'architecture et les capacités servies. NEVER utiliser un champ unique pour coder les trois dimensions.

### 7.2 Matrice de support des modèles de référence

| Modèle | DiffusionGemma 26B A4B (it) | Clover-35B-A3B-Preview |
|---|---|---|
| Éditeur | Google DeepMind | DiffuSpace |
| Dépôt / carte | model card `google/diffusiongemma-26B-A4B-it` (Hugging Face) + documentation Google | `Diffuspace/Clover-35B-A3B-Preview` (poids, HF), `DiffuSpace/Clover-35B-A3B-Preview` (code + backends) |
| Licence | Apache 2.0 | Apache 2.0 (base Qwen3.6-35B-A3B, Alibaba Cloud) |
| Revision épinglée | à fixer par commit HF | `4b0402b65e` (cité par la carte), à confirmer |
| Architecture | encoder-décodeur, MoE 25.2B / 3.8B actifs, 30 couches | MoE 35B / ~3B actifs, 40 couches |
| Génération | block-autoregressive multi-canvas, dénoisage parallèle | block diffusion + `LowConfidence` |
| Attention | encoder AR (prefill + KV), décodeur bidirectionnel sur canvas, cross-attention au préfixe, sliding window 1024, contexte 256K | GDN hybride + full attention, bidirectionnel intra-bloc par paramètres GDN additionnels (right-convolution, contrôles softmax), préfixe causal préservé, contexte 128K |
| Granularité | canvas 256, 15-20 jetons par passe, max 48 pas de dénoisage | bloc 32, seuil 0.95 |
| MoE | 8 actifs / 128 total + 1 partagé | top-8 / 256 routés + 1 partagé ; `moe_block_coreset` (top-p 0.9) restreint les experts par bloc |
| Modalité | texte, image, vidéo | texte uniquement |
| Tokenizer | Gemma 4 (262K) | Qwen3.6 |
| Formats de poids | à épingler depuis les shards du dépôt | BF16 (~69 GB) dans la distribution HF de référence |
| Échantillonnage | Entropy-Bounded Denoising, arrêt adaptatif (entropie < 0.005, stabilité sur 2 pas), température 0.8 → 0.4 | `LowConfidence`, contrôles de répétition et de longueur (`dllm_length_penalty`, `dllm_think_end_bias`), self-speculative (SGLang / llama.cpp) |
| Famille de génération | `Diffusion` | `Diffusion` |
| Topologie d'architecture | `EncoderDecoder` | `HybridRecurrent` |
| Capacités servies | `Diffusion` | `Diffusion` |
| Préconditions moteur | `generation_family = Diffusion`, `architecture_topology = EncoderDecoder`, `taille_bloc = 256`, `fenêtre_glissante = 1024`, `dual_stream = false`, `table_ngram = false` | `generation_family = Diffusion`, `architecture_topology = HybridRecurrent`, `taille_bloc = 32`, `états_recurrents = GDN`, `dual_stream = false`, `table_ngram = false` |
| État du support | **non commencé** (cible P1b) | **non commencé** (cible P3) |

**Règles** :

- MUST nommer les cibles dans la configuration, avec la revision épinglée du checkpoint.
- MUST mettre à jour la matrice à chaque progression : `non commencé` → `expérimental` → `partiel` → `validé`.
- MUST traiter la matrice comme le contrat de support : un modèle absent n'est pas pris en charge ; un agent MUST NOT l'ajouter de sa propre initiative.
- MUST respecter les valeurs déclarées (granularité, fenêtre, MoE, arrêt adaptatif) : la longueur de canvas de DiffusionGemma (256) et la taille de bloc de Clover (32) sont des propriétés distinctes ; MUST NOT les confondre.
- MUST NOT supposer un checkpoint dual-stream : aucun des deux modèles de référence ne le déclare. Les modes R-13 / R-14 restent indisponibles tant qu'un checkpoint dual-stream n'est pas chargé.

### 7.2.1 Profils d'attention par modèle

Chaque profil est **contractuel** : le moteur MUST servir le modèle selon son profil, NEVER le ramener à la règle générique « causal sur préfixe + bidirectionnel intra-bloc ».

**DiffusionGemma 26B A4B (it)** — `EncoderDecoder` :

- **encodeur autoregressif** : construit le contexte (prefill causal, KV du contexte) ;
- **décodeur** : dénoisage **bidirectionnel sur le canvas** de 256 positions masquées, **cross-attention vers le KV du contexte**, sliding window 1024 ;
- **progression** : le canvas dénoisé est **ajouté au contexte** ; la génération est multi-canvas, 15-20 jetons par passe, max 48 pas de dénoisage ;
- **conséquence** : la cross-attention encodeur/décodeur et l'ajout du canvas au contexte ne sont **pas** réductibles à une politique d'attention causale sur préfixe. MUST être des sous-modules typés du profil, avec leurs propres conditions de cache.

**Clover-35B-A3B-Preview** — `HybridRecurrent` :

- **GDN hybride + full attention** : couches linéaires (état récurrent GDN) et couches softmax ;
- **bidirectionnel intra-bloc** par paramètres GDN additionnels (right-convolution, contrôles softmax), préfixe causal préservé, bloc 32 ;
- **conséquence** : la restauration à la frontière de commit porte sur l'**état récurrent**, pas sur un tail de KV (§5.6, R-46).

MUST ajouter un profil d'attention à chaque fiche de modèle (§7.2) avant d'implémenter son chemin d'attention. MUST refuser un checkpoint sans profil déclaré.

### 7.3 Rôle de la mémoire vis-à-vis de la conversion

La conversion n'est pas un cas d'usage du moteur. Le moteur MUST uniquement charger les artifacts produits, avec leurs préconditions vérifiées (R-16, R-85), leurs mappings de poids et leurs références numériques (R-53, R-61).

---

## 8. Jalons de développement

| Jalon | Livrables | Critères d'acceptance |
|---|---|---|
| **P0** | device WebGPU découvert, limites lues et appliquées, 2 kernels exécutés, plan d'exécution à forme fixe compilé, **backend CPU exercé sur un checkpoint réduit** | les limites §4.5 mesurées et appliquées aux plans ; `f16` traité comme capacité optionnelle (chemin de référence sans `f16`), absence d'atomic `f32` gérée ; déquantification des formats prioritaires validée contre référence (L4), **y compris la sémantique E8M0 `0x00` / `0xFF`** ; préconditions des politiques refusées (R-57, `I-14`) ; session complète exécutée sur le backend CPU **avec le checkpoint réduit**, débit mesuré |
| **P1a** | cycle dLLM complet sur le **checkpoint réduit** : mémoire polymorphe par rôle, contrôle d'admission, réallocation, tampons double, préfill AR + canvas, arrêt adaptatif | requête admise/refusée par limites mesurées ; plan d'un pas sans thrashing (R-41) ; progression de la dénoisation mesurée par **positions dénoisées par pas** et par nombre de pas jusqu'à la fin du canvas (R-9, L3) ; `p_cache` n'est **pas** un critère de complétion ; parité L1/L3 sur le checkpoint réduit ; débit mesuré |
| **P1b** | DiffusionGemma 26B : mémoire polymorphe par rôle à la taille réelle, canvas 256, arrêt adaptatif, validation CPU-only de la **cible de référence** | parité L1/L3 sur DiffusionGemma ; débit mesuré sur la machine de référence ; seuil de débit consigné en P1a respecté ; `p_cache` mesuré comme métrique, jamais comme critère de complétion |
| **P2** | ordonnanceur d'inférence complet (horizon, fenêtre, sélection, admission), ordonnanceur de ressources (placement par rôle, budget de transfert, offload), instrumentation | décisions journalisées des deux ordonnanceurs ; transferts mesurés ; R-89 prouvé sur une cible MoE (R-37 mesuré en production) |
| **P3** | AR sur son chemin natif + Clover : GDN, MoE top-k, couche de draft, validation par logits de référence | causalité stricte vérifiée (R-15) ; tolérances mesurées (R-66) ; restauration de l'état récurrent à la frontière d'acceptation (R-46, L2) |
| **P4** | APIs OpenAI et Anthropic, **API de contrôle**, streaming, tool calls, matrice Annexe B complétée | conformance par endpoint ; rejets explicites (R-82) ; streaming et tool calls sur les deux APIs ; charge / décharge / état des sessions et des ressources pilotables (R-92..R-96) |
| **P5** | persistance NVMe, reprise de session, sécurité et rétention, formats étendus, tests de régression de capacités | session longue reprise depuis un checkpoint NVMe ; validation §6.4 complète (rétention, suppression, permissions, invalidation, chiffrement documenté) |

**Hors jalons du moteur** : conversion de checkpoints AR → dLLM (R-71..R-74, outillage externe FUTURE), interface graphique (`N-4`), multi-GPU (`N-9`), WASM (`N-8`).

Ordre imposé : P0 → P1a → P1b → P2 → P3 → P4 → P5. NEVER inverser P0 et P3. NEVER implémenter un mode de confiance (`R-13`, `R-14`) avant la validation du cycle de P1a sur un checkpoint **réel** déclarant `dual_stream` (R-97).

**Critères d'entrée entre jalons** — un jalon MUST NOT démarrer sans : (1) le rapport de capacités du jalon précédent consigné (§4.5, §9), (2) les tolérances chiffrées de §6.3 consignées pour les formats exercés, (3) le **seuil de débit** consigné par le mainteneur sur le checkpoint réduit (tokens/s mesurés sur la machine de référence, valeur inscrite dans le rapport du jalon précédent), (4) l'état d'implémentation des exigences de la ligne du jalon. MUST consigner le seuil retenu **avant** de planter P1b ; NEVER démarrer un jalon sur un seuil implicite.

**Checkpoint réduit (P0 / P1a)** — identifié, MUST être déclaré dans le rapport :

| Champ | Valeur |
|---|---|
| Source | `GSAI-ML/LLaDA-8B-Instruct` (license MIT, `arxiv:2502.09992`, dépôt `ML-GSAI/LLaDA`) |
| Format exercé | GGUF **Q8_0** (quantification par bloc, échelles `f32`) — correspondance Alethe.ia : format **INT8** de F-1 (R-60) |
| Topologie | `generation_family = Diffusion`, `architecture_topology = Dense` ; dLLM natif à génération par blocs, token de masque dédié ; **le checkpoint ne déclare pas la taille de bloc** : la taille de bloc est un paramètre de configuration, MUST être consignée |
| Tokenizer | celui publié avec le checkpoint, MUST être lu depuis les métadonnées du fichier, NEVER re-déduit |
| Taille | 8B : environ 8,5 Go en Q8_0, tenant en RAM système |
| Rôles de validation | déquantification F-1 (L4) et sémantique E8M0 ; kernels de base ; formes de workgroups ; session CPU-only complète (R-90) ; cycle dLLM complet (P1a) |

NEVER exiger un checkpoint de 26B dans P0 / P1a : la validation CPU-only complète de la **cible de référence** est reportée à P1b. MUST nommer le checkpoint réduit dans le rapport de P0.

### 8.1 P0 — Périmètre initial et mesures

Livrables : structures canoniques (bloc, masque, fenêtre, horizon, préfixe, état récurrent, état de diffusion, session, mode, politique), hiérarchie de placement par rôle, ordonnanceur d'inférence minimal, ordonnanceur de ressources minimal, kernels de base, session, CLI de validation, **chargement et exécution du checkpoint réduit**.

**P0 MUST mesurer** : mémoire par classe (device, RAM, NVMe), débit par classe de kernel, bandes passantes HBM / RAM / PCIe, latence d'appels, coût de compilation d'un pipeline, coût d'encodage d'un plan, **limites du device lues (§4.5), dont `maxStorageBufferBindingSize` et la découpe effective des tenseurs en tuiles**, formes de workgroups effectives, atomics `i32` / `u32` disponibles, fonctionnalités **exposées et absentes** (`shader-f16`, `timestamp-query`, `subgroups` avec `subgroupMinSize` / `subgroupMaxSize`, `texture-compression-bc`, `texture-compression-etc2`, `texture-compression-astc`, `float32-filterable`), formats de poids réellement décodables (F-1), capacités du backend CPU (précision, largeur de vector, débit mesuré), **machine de référence (§9) consignée**.

**P0 MUST valider** : la déquantification de chaque format F-1 contre une référence de précision (FP32, et l'implémentation de référence du checkpoint si disponible) avec tolérances définies, **y compris la sémantique E8M0 `0x00` / `0xFF`** ; le refus des formats non listés (R-83) ; le refus des préconditions manquantes ; la session complète en mode CPU-only sur le checkpoint réduit ; **deux kernels de réduction comparés par mesure — workgroup memory (chemin de base) et subgroup sur le device qui l'expose**.

**P0 MUST produire** : rapport de capacités par device (limites lues, fonctionnalités exposées / absentes, taille de subgroup), rapport de références numériques (tolérances chiffrées par format F-1), mesures de bandes passantes, formes de workgroups, débit CPU mesuré, **seuil de débit retenu pour l'entrée en P1a**, machine de référence consignée. NEVER une clôture de P0 sans ces artefacts.

---

## 9. Métriques d'acceptation

Le moteur MUST mesurer et exposer :

| Métrique | Définition contractuelle |
|---|---|
| `p_cache` | fraction de préfixe cache-valide à chaque pas |
| tokens/s générés | débit effectif, mesuré sur submit + timer |
| tokens/forward | jetons produits par forward |
| bytes transférés | total H2D / D2H par pas |
| temps de transfert | temps de la copie de poids |
| temps kernel | temps cumulé des kernels |
| occupation par rôle | chaud, tiède, froid, persisté, table, scratch (R-30) |
| taux de cache miss d'experts | misses / top-k par pas |
| décodage de poids | temps de décodage par couche |
| tolérance de logits | écart max contre la référence (L1) |
| taux d'acceptation spéculative | α, horizon K |
| taille de bloc B / pas S | paramètres effectifs (déclarés par le checkpoint) |
| coût de calcul vs coût de transfert | décision d'offload (R-89), par opération |
| coût de réallocation | coût d'un changement de placement (R-24) |
| positions dénoisées par pas | progression de la dénoisation |

Règle de mesure : une optimisation MUST être justifiée par des **mesures explicites** sur des machines représentatives (VRAM de différentes capacités, RAM, bandes passantes de transfert, occupation de buffers, temps de kernels, débit de tokens). NEVER une revendication sans mesure ; NEVER des mesures non produites par l'agent.

**Machine de référence** — MUST être nommée avant P0 et consignée dans chaque rapport : processeur (nom, jeu d'instructions vectorielles), accélérateur (nom, VRAM), RAM système, stockage NVMe, système d'exploitation et version de pilote, **limites du device lues** (§4.5, rapport de P0), versions des checkpoints exercés. « **Cible de référence** » se lit comme le couple (checkpoint déclaré + machine de référence) ; NEVER une validation déclarée sur une machine non consignée, NEVER une mesure transplantée d'une machine à une autre.

**Règles de terminaison** : un cycle MUST se terminer par un commit, une révision explicite, une limite de pas atteinte, une limite de mémoire atteinte, un rejet de précondition ou une dégradation mesurée. NEVER une terminaison implicite.

**Lisibilité de la sortie** : tokens, métriques de progression (positions dénoisées, pas par canvas), métriques de latence (temps par pas), métriques de qualité (entropie, confiance, stabilités de prédiction).

---

## 10. Conventions d'implémentation

Le code base est compact, les interfaces sont étroites, les commentaires sont instructifs.

| # | Convention |
|---|---|
| C-1 | Le code de production MUST être compact et direct ; la duplication de logique MUST être supprimée. |
| C-2 | MUST éviter les allocations inutiles et les dépendances excessives. |
| C-3 | MUST partager des formats communs, des buffers communs et des contrats explicites. |
| C-4 | MUST garder les interfaces étroites et les noms évocateurs. |
| C-5 | MUST implémenter des commentaires instructifs **à côté du code**, pas en périphérique. |
| C-6 | Les kernels MUST être compact, parallèle, compatible avec les plans d'exécution. |
| C-7 | MUST utiliser des tampons double et la réallocation dynamique de VRAM. |
| C-8 | MUST utiliser les limites mesurées du device. |
| C-9 | Le code de production MUST respecter la granularité native des tenseurs et MUST partager les formats de tenseurs et de scale. MUST conserver les valeurs des scales exactes : E8M0 est un exposant non signé à biais 127, `0x00` = `2^-127` (finie), `0xFF` = NaN réservé (`R-61`). MUST éviter les copies redondantes, les synchronisations superflues et les conversions implicites de scales. MUST minimiser les conversions. MUST respecter les formats de poids des modèles convertis. |
| C-10 | MUST éviter la redondance logique, les couches superflues, les dépendances de modules, l'accumulation de conversions, les couches d'adaptation sans valeur sémantique. |
| C-11 | MUST respecter la sémantique des formats de poids lors des transformations de representations. |
| C-12 | MUST minimiser les conversions et les allocations de buffers. |
| C-13 | MUST éviter les passes inutiles et les synchronisations superflues. |
| C-14 | MUST respecter la sémantique de la persistance. |
| C-15 | MUST valider les limites du device. |

Règle de structure : un module de production MUST avoir une responsabilité unique, une interface étroite, une taille bornée, sans redondance, sans conversion implicite et sans adaptation de convenance.

---

## 11. Interface de l'agent de développement

L'agent de développement MUST respecter les règles suivantes :

1. MUST lire les exigences, les invariants et les jalons avant d'implémenter.
2. MUST proposer une décision pour les zones non spécifiées (§14), MUST la faire valider par le mainteneur, MUST consigner la décision retenue dans la ligne de §14 **avant** d'écrire le code.
3. MUST joindre une mesure à chaque optimisation.
4. MUST déclarer les préconditions non remplies avant d'implémenter.
5. MUST reporter les dépassements de limites du device, les refus d'admission et les dégradations.
6. MUST respecter la granularité native des tenseurs.
7. MUST garder les kernels compacts.
8. MUST journaliser les décisions d'horizon, de placement, d'admission et de commit.
9. MUST respecter les zones de responsabilité des couches (R-4, §15).
10. **MUST déclarer les deux axes de maturité de chaque exigence touchée avant d'implémenter** (axe 1 : `DEFINED` / `EXPERIMENTAL_SPEC` / `RESEARCH` / `FUTURE` ; axe 2 : `NOT_STARTED` / `IN_PROGRESS` / `IMPLEMENTED` / `VALIDATED`).
11. **MUST reporter un conflit de design au lieu de résoudre localement par une adaptation de convenance.**
12. MUST refuser un modèle sans préconditions et l'indiquer explicitement.
13. MUST fournir un protocole de mesure pour les exigences `RESEARCH` et respecter le seuil d'abandon.
14. MUST déclarer les limites du device.
15. MUST déclarer les formats de poids refusés.
16. MUST valider la sémantique des formats de poids et des échelles.
17. MUST respecter les exigences de persistance et de sécurité (§6.4, R-75..R-79).
18. MUST valider la compatibilité du checkpoint et de la fonctionnalité demandée avant l'exécution.
19. MUST refuser une exécution sans préconditions.
20. MUST refuser une conversion silencieuse.
21. MUST mettre à jour la matrice d'état d'implémentation (`docs/etat_implementation.md`) à chaque validation consignée, et MUST la reporter dans le rapport de jalon.

---

## 12. Non-objectifs

| # | Non-objectif |
|---|---|
| N-1 | Le moteur ne convertit pas des checkpoints et n'entraîne pas des modèles. L'outillage de conversion AR → dLLM est un projet séparé avec ses propres critères de validation, ses dépendances et ses interfaces d'export (R-71..R-74 FUTURE). NEVER intégrer la conversion dans le chemin d'exécution du moteur. |
| N-2 | Le moteur ne vise pas les scores de leaderboard. |
| N-3 | Le moteur ne vise pas le training de pleine ampleur. |
| N-4 | Pas d'interface graphique, de gestion de fichiers, de console d'administration, de modélisation 3D, de support d'entrée multimodale (images), de conversion de checkpoints dans le moteur. |
| N-5 | Pas de mode de dégradation silencieux. |
| N-6 | Pas de conversion implicite de sémantique. |
| N-7 | Pas de mode CPU **dégradé** : le mode CPU complet est un mode normal (R-90) et le calcul CPU d'experts est un mode de production (R-37). NEVER le mode CPU comme chemin de performance de référence pour les modèles à forte densité de calcul. |
| N-8 | **WASM / navigateur** : le périmètre exact MUST être décidé avant toute implémentation. Trois positions : (A) **hors périmètre** — recommandé pour P0-P5, le moteur ne vise pas le navigateur ; (B) **cible de portage** — le moteur est compilable en WASM, mais les sessions servies en WASM ne sont pas un objectif de production ; (C) **produit complet** — sessions servies dans le navigateur, cible de production, avec ses propres contraintes de mémoire et de device. MUST trancher par décision écrite ; NEVER laisser l'ambiguïté entre (A), (B) et (C) dans les exigences. |
| N-9 | Pas de support multi-GPU (la mémoire polymorphe est mono-device). |
| N-10 | Pas de conversion de format de poids non listée, pas de requantification silencieuse. |
| N-11 | Pas de fusion des logiques de validation en une logique unique. |

---

## 13. Traçabilité et index des références

### 13.1 Index des sources (revisions épinglées)

| Source | Identifiant vérifiable | Revision / date épinglée | Usage |
|---|---|---|---|
| WebGPU | spécification W3C `webgpu` + `WGSL` | **version datée consignée au démarrage de P0** | limites, atomics, `f16`, mapping, features |
| wgpu | `gfx-rs/wgpu` (crate `wgpu`) | version + commit consigné au démarrage de P0 | noms de limites, features, mapping |
| llama.cpp | `ggml-org/llama.cpp` | commit consigné | formats de poids, décodage, serving MoE |
| GGUF / ggml | `ggml-org/ggml`, spec `docs/gguf.md` | commit consigné | formats de poids, quantifications, tables n-gram |
| SGLang | `sgl-project/sglang` | commit consigné | serving, MoE, self-speculative |
| Strata | `Niko1221/Strata` — README, `docs/DETAILS.md`, `docs/AI_REFERENCE.md` | app & Terminal 0.2.5, Control & engines 0.1.50 (révision reviewée le 25 octobre 2026) | exécution sur machines personnelles, API OpenAI/Anthropic sur localhost, usage depuis le SSD quand le modèle dépasse la mémoire |
| FreeToken | `FlashML-org/FreeToken` + article `arxiv:2608.16157` | article / commit consigné | chargement d'experts pendant la phase de calcul |
| DwarfStar4 | `antirez/ds4` | commit consigné | conventions de code compact, frontières de responsabilités |
| WeDLM | `arxiv:2512.22737` | article daté | sélection d'entropie pénalisée, fenêtre glissante |
| FLARE | `arxiv:2606.01774` | article daté | logit shift |
| FLUID | `arxiv:2605.27387` | article daté | horizon élastique, seuil par position, stride |
| REPR-ALIGN | `arxiv:2605.06885` | article daté | supervision d'alignement cosinus par couche |
| DiffusionGemma | model card `google/diffusiongemma-26B-A4B-it` (Hugging Face) + documentation Google | **revision des poids épinglée par commit HF au téléchargement** | cible de référence dLLM |
| Clover | `DiffuSpace/Clover-35B-A3B-Preview` (code + backends), `Diffuspace/Clover-35B-A3B-Preview` (poids) | `4b0402b65e` — **à confirmer avant P3** | cible de référence dLLM |
| Checkpoint réduit | `GSAI-ML/LLaDA-8B-Instruct` (MIT, `arxiv:2502.09992`, dépôt `ML-GSAI/LLaDA`) | **quantization GGUF exercée à épingler** (commit du dépôt de quantization) | checkpoint de P0 / P1a |
| GDN | `arxiv:2412.06464` — **Gated DeltaNet** (`NVlabs/GatedDeltaNet`) | article + dépôt | base de l'état récurrent (R-46, §5.6) |
| OpenAI API | documentation officielle `platform.openai.com/docs/api-reference` | date de la version consommée | Annexe B |
| Anthropic API | documentation officielle `docs.anthropic.com` (`/v1/messages`) | date de la version consommée | Annexe B |
| OCP Microscaling | `OCP Microscaling Formats (MX) Specification v1.0` | version 1.0 | sémantique E8M0, MXFP4/FP8 (R-61) |
| CUDA | NVIDIA CUDA 12.8 | version datée | comparaison de graphes et de transferts |

MUST utiliser les versions épinglées dans les rapports de références. **MUST NOT écrire « version courante »** dans une table annoncée comme contenant des révisions épinglées : une référence destinée à un agent doit être immuable ou datée.

**Règle des identifiants invérifiables** : une source non récupérable sous l'identifiant indiqué MUST être remplacée par son identifiant exact ou supprimée. NEVER fonder une décision d'implémentation sur une référence non vérifiable. Identifiants retirés de l'index : `BenoitMalo/xLLaMA.cpp`, `ggermet/ggml`, `rustwasm/wgpu`, `OpenAI/openai`, `Anthropic/anthropic`, `google-gemma` (nom d'organisation, pas un identifiant de dépôt).

**Règles de traçabilité** :
- MUST rattacher chaque exigence à sa source et à sa nature : `Source`, `Adaptation`, `Hypothèse de recherche` (Annexe A).
- MUST nommer les sources par repository, nom d'article, référence API et révision épinglée dans la matrice §13.2.
- **Strata** : le dépôt de référence est **`Niko1221/Strata`**. `ro99/strata` est un **projet homonyme distinct** (moteur C++20/CUDA de placement d'experts) et n'est **pas** une référence d'Alethe.ia. MUST nommer le dépôt et l'organisation dans chaque mention de Strata ; NEVER laisser deux projets homonymes sous une seule référence « Strata ».
- MUST distinguer, dans chaque exigence, ce qui vient d'un article, ce qui vient d'un dépôt, ce qui est une décision d'Alethe.ia, ce qui est une hypothèse.
- MUST citer les sections, équations ou algorithmes source pour les formules précises (R-21, R-22, R-25, §5.5, §5.6).
- MUST déclarer les préconditions des mécanismes empruntés (WeDLM, FLARE, n-gram, dual-stream).
- MUST employer `checkpoint dual-stream` (flux `clean` / `noisy`), NEVER « checkpoint bilingue ».

### 13.2 Matrice de traçabilité Nature → exigences

`Nature = Source | Adaptation | Hypothèse de recherche` (Annexe A). MUST utiliser un seul nom : `WeDLM` (pas « WeDLLM »).

| ID | Nature | Source précise | Exigences | Zone d'application |
|---|---|---|---|---|
| N01 | Source | WeDLM | R-25, R-26, R-27 | décodage causal strict, sélection d'entropie pénalisée, fenêtre glissante |
| N02 | Source | FLUID | R-21, R-22, R-19 | horizon élastique, seuil par position, stride adaptatif |
| N03 | Source | FLARE | §4.3 logit shift | logit shift |
| N04 | Source | Repr-Align (REPR-ALIGN) | R-72 | supervision d'alignement cosinus par couche |
| N05 | Source | AR-Trust (spéculatif clean-stream) | R-14, §5.5 | acceptation de Bernoulli, `qᵢ` exact, correction `[pᵢ − qᵢ]₊` |
| N06 | Source | Diffusion-Trust | R-13, §5.4 | bloc de taille B, `Aₛ = {i ∈ Rₛ : cᵢ ≥ γₛ}`, causal state replay |
| N07 | Source | GDN (**Gated DeltaNet**, `arxiv:2412.06464`, `NVlabs/GatedDeltaNet`) | R-46, §5.6 | état récurrent, rewind à `S⁽ʳ⁾` |
| N08 | Source | Mamba / SSM | §5.6, R-9 | restauration d'état récurrent |
| N09 | Source | MoE top-k + 1 partagé | R-37..R-41 | LRU d'experts, offload, calibrage par mesures |
| N10 | Source | FreeToken | R-36 | chargement d'experts pendant la phase de calcul |
| N11 | Source | CUDA (transferts DMA) | R-42, §4.4 | DMA, transferts pendant le calcul |
| N12 | Source | WebGPU | §4.5, C-15 | limites, atomics, `f16`, mapping |
| N13 | Source | OpenAI | R-48..R-50, Annexe B | endpoints, streaming, erreurs |
| N14 | Source | Anthropic | R-80..R-82, Annexe B | endpoints, messages, streaming |
| N15 | Source | NVFP4, MXFP4, FP4, FP8, INT4, INT8 | R-60, R-61 | formats prioritaires |
| N16 | Source | GGML `q<N>` et i-quants | R-60, R-64 | formats et n-gram |
| N17 | Source | DiffusionGemma (Google DeepMind) | R-12, §7.2, §7.3 | cible de référence dLLM |
| N18 | Source | Clover (DiffuSpace) | R-37, R-46, R-9, §5.6, §7.2, §7.3 | cible de référence dLLM |
| N19 | Adaptation | ordonnanceur de ressources séparé de l'ordonnanceur d'inférence | R-84..R-89, R-24, R-18, R-19, R-26 | architecture du moteur |
| N20 | Adaptation | séparation des validateurs | §6.3, R-57 | interface, préconditions, granularité |
| N21 | Adaptation | placement par rôle, pas par classe d'objet | R-29..R-31, R-33, R-34 | politique de placement |
| N22 | Adaptation | mémoire polymorphe par rôle et budgets | §4.1, R-29, R-30 | couches de mémoire |
| N23 | Adaptation | plan d'exécution à forme fixe | §4.4 | chemin WebGPU portable |
| N24 | Adaptation | mode CPU complet | R-90, N-7, §4.6 | backend CPU |
| N25 | Adaptation | sécurité de persistance | R-75..R-79 | interface, journalisation, validation |
| N26 | Hypothèse de recherche | Δ du stride | R-22 | définition, conversion en positions entières, validation |
| N27 | Hypothèse de recherche | prédiction de routage | R-40 | protocole de mesure, seuil d'abandon |
| N28 | Hypothèse de recherche | τ et λ | R-25, R-27 | plage initiale [0.3, 0.6], λ > 0, mesure par cible |
| N29 | Hypothèse de recherche | arrêt adaptatif | §5.8 | entropie moyenne sous un seuil + prédictions identiques sur 2 pas |
| N30 | Source | `GSAI-ML/LLaDA-8B-Instruct` (`arxiv:2502.09992`, MIT, dépôt `ML-GSAI/LLaDA`) | R-58, R-60, §8 | checkpoint réduit de P0 / P1a, exercé en GGUF Q8_0 ↔ INT8 |

---

## 14. Zones non spécifiées

**Processus de décision des zones** — Ces zones ne sont pas spécifiées. L'agent de développement MUST, pour chaque zone qu'il touche : (1) signaler la zone, (2) **proposer** une note de décision (options, invariants respectés, conséquences sur les exigences, coûts), (3) **soumettre la note au mainteneur**, (4) consigner la décision tranchée dans la ligne correspondante de §14 (décision retenue + date) **avant** d'écrire le code. MUST NOT implémenter une zone non tranchée ; MUST NOT trancher seul un arbitrage d'architecture (§0, règle 8). NEVER une zone résolue par un choix d'implémentation non consigné.

Le dépôt ne spécifie pas ces points. La ligne d'une zone MUST rester à `A TRANCHER` tant que la décision n'est pas consignée.
1. **Format de checkpoint canonique** (R-58) : nom, versioning, encodage, en-tête, tables n-gram. — **A TRANCHER**
2. **ABI Rust ↔ Zig** (I-12, §4.7) : signatures, limites de types, conventions d'erreurs, modes d'appels, transferts de buffers, formats de sérialisation, règles d'alignement. — **A TRANCHER**
3. **Modèle de concurrence** : threading Rust vs threads Zig, propriété des buffers, synchronisation avec les files de commandes WebGPU. — **A TRANCHER**
4. **Multi-GPU et multi-device** : `N-9` (hors périmètre). La mémoire polymorphe est mono-device ; NEVER étendre la sémantique à une distribution multi-machine sans décision explicite. — **A TRANCHER**
5. **Support d'entrée multimodale (images)** : `N-4` jusqu'à décision. — **A TRANCHER**
6. **Surface d'API exacte** (R-48, R-80, Annexe B) : versions d'API, endpoints, schémas de tool calls, format de streaming. — **A TRANCHER**
7. **Stratégie d'échantillonnage complète** (R-65) : temperature / top-p / top-k, afin que la validation par logits de référence soit déterministe. — **A TRANCHER**
8. **Plafond d'horizon K, taille de bloc B, nombre de pas S, taille de fenêtre W** : valeurs par défaut, plages, dépendance au checkpoint. — **A TRANCHER**
9. **Seuil de tolérance de logits par format de poids** (R-66) : valeurs chiffrées à mesurer en P0 (formats F-1) et en P1a / P1b (cibles). — **A TRANCHER**
10. **Politique de réallocation de VRAM** (R-24, R-87, R-42) : critère, granularité, coût. — **A TRANCHER**
11. **Plancher de capacité du pool d'experts** (R-41) : formule de la capacité d'un pas. — **A TRANCHER**
12. **Prediction routing** (R-40) : horizon de prédiction, coût, politique de déclenchement. — **A TRANCHER**
13. **Contrôles d'admission** (R-34) : critère d'admission pour une requête, une fenêtre, un pool d'experts. — **A TRANCHER**
14. **Format des mesures reproductibles** (§9) : format d'artefact, granularité, procédure de reproduction. — **A TRANCHER**
15. **Convention de versioning** : version de l'ABI, des formats de poids, des métadonnées canoniques, règle de compatibilité. — **A TRANCHER**
16. **Durée de rétention de la persistance** (R-76) et **procédure de suppression** des sessions. — **A TRANCHER**
17. **Chiffrement au repos** (R-79) : mécanisme retenu et avertissement. — **A TRANCHER**
18. **Interfaces de sécurité** (R-77) : clé d'API, emplacements autorisés, permissions des fichiers. — **A TRANCHER**
19. **Matrice d'équivalence numérique** (§6.3) : tolérances chiffrées par format de poids. — **A TRANCHER**
20. **Mode CPU complet** (R-90) : granularité des kernels CPU, formats de poids supportés, formes de workgroups. — **A TRANCHER**
21. **Périmètre WASM / navigateur** (`N-8`) : trancher (A) hors périmètre, (B) cible de portage, (C) produit complet. — **A TRANCHER**
22. **Checkpoint réduit de P0 / P1a** (§8) : **décision consignée** — `GSAI-ML/LLaDA-8B-Instruct`, GGUF Q8_0. Reste à consigner : la **taille de bloc** retenue en configuration, le **commit de la quantization GGUF** exercée, la machine de référence (§9).
23. **Complétion de la matrice d'API Annexe B** : schémas, en-têtes, événements SSE, tool calls, format d'erreurs, champs acceptés / ignorés / refusés, avant P4.
24. **Cible de validation dual-stream** (R-97) : identification d'un checkpoint réel avec model card publique, avant toute implémentation de `R-13` / `R-14`.
25. **Rattachement des mécanismes empruntés** : chaque mécanisme repris de Strata, FreeToken, DwarfStar4 MUST être rattaché à des fichiers précis du dépôt source (chemin + fonction) ; à défaut, MUST être déclaré **adaptation d'Alethe.ia**, jamais présenté comme reproduction.

---

## 15. Structure de dépôt imposée

La structure MUST refléter les zones de responsabilité des couches.

```
Alethe.IA/
  _Alethe.IA.md          # ce document — source unique
  docs/fiches_techniques/
  core/                  # Rust : état, ordonnanceur d'inférence, plans, admission, validation, formats
  resources/             # Rust : ordonnanceur de ressources, budgets, coût de transfert
  mem/                   # Zig : layouts, indexation, pools, décodage de poids, staging, ABI
  gpu/                   # WGSL : kernels, plans d'exécution, découverte des limites
  cpu/                   # Zig : kernels CPU, offload, mode CPU-only
  cli/                   # Rust : CLI, invocations non-interactive, sortie JSON
  serve/                 # Rust : API OpenAI, API Anthropic, API de contrôle (R-92..R-96), file de workers, streaming, tool calls
  persist/               # Rust + Zig : checkpoints d'états, sessions, radix cache, rétention, permissions
  tests/                 # unitaires, régression de capacités, mesures, validation de logits
```

Règles de structure :
- NEVER placer de la représentation mémoire dans `core/` ; NEVER placer de la décision d'ordonnancement dans `mem/`.
- NEVER placer de la sémantique GPU dans `core/` : `core/` consomme les limites du device, ne les invente pas.
- NEVER placer de décision de placement dans `core/` sans passer par l'interface typée de l'ordonnanceur de ressources.
- `tests/` MUST être organisé par jalon, chaque test pointant une exigence par son ID.

---

## 16. Matrice de traçabilité exigences → jalons

| Jalon | Exigences couvertes |
|---|---|
| P0 | R-1, R-4, R-34, R-58..R-64, R-83, R-85, R-86, R-90, §4.5, C-15, checkpoint réduit (F-1) |
| P1a | R-2..R-6, R-7..R-12, R-25, R-28, R-29..R-35, R-41, R-42, R-84..R-89, R-18, §7.2.1 (sur le checkpoint réduit) |
| P1b | R-16, R-91, cible DiffusionGemma (mémoire polymorphe, canvas 256, arrêt adaptatif) |
| P2 | R-19..R-24, R-26, R-27, R-36..R-40, R-87, R-89, R-37 |
| P3 | R-15, R-17, R-37..R-41, R-46, R-52..R-57, R-65..R-68, R-70, §5.5, §5.6, §6.3 |
| P4 | R-48..R-51, R-80..R-82, R-92..R-96, Annexe B complétée |
| P5 | R-43..R-47, R-75..R-79, R-67..R-69, R-64 |
| FUTURE (hors jalons du moteur) | R-13, R-14, R-71..R-74, R-97 |

**Lecture de la matrice** : une exigence qui figure dans plusieurs jalons se lit comme **première livraison** (jalons antérieurs) puis **validation finale** (dernier jalon de sa ligne). Un jalon MUST NOT être déclaré complété si une exigence de sa ligne n'est pas exercée par un test ou une mesure ; la validation d'une exigence multi-jalons se consigne à la ligne la plus avancée. NEVER déclarer une exigence couverte par sa seule première livraison.

---

## 17. Critères d'acceptation globale

1. Le moteur sert un dLLM natif, un checkpoint AR adapté, un checkpoint AR natif — par le même contrat d'exécution typé par famille de génération, topologie d'architecture et capacités servies (`I-1`, `N-11`). **Les modes `dual-stream` (`R-13`, `R-14`) ne figurent dans ce critère que lorsqu'un checkpoint réel est identifié (R-97)** ; NEVER les considérer comme acquis.
2. Les décisions d'horizon, de fenêtre, de commit, de réallocation viennent de deux ordonnanceurs séparés (I-2, R-84..R-89, R-18).
3. KV, états récurrents, états de blocs, poids et tables n-gram vivent dans une seule hiérarchie de placement par rôle (I-3, R-29, R-30).
4. La validation de sortie, de couches, de jetons, de structures suit un seul mécanisme de référence, avec des validateurs typés (§6.3).
5. Les formats de poids et de checkpoint conservent la sémantique des checkpoints (I-5, R-59, C-9).
6. Les responsabilités Rust / Zig / WebGPU / CPU sont délimitées par des contrats (I-6, I-7, I-12).
7. Le moteur s'adapte aux capacités du device WebGPU, sans lock-in (I-8, R-34, C-8, R-86).
8. Les dégradations sont explicites (R-34, §9, N-5) ; les fonctionnalités non prises en charge sont documentées.
9. Chaque optimisation est justifiée par des mesures explicites (§9), sur la **machine de référence** consignée.
10. Le code base est compact, avec interfaces étroites (C-1..C-15).
11. Les interfaces d'API sont documentées par une matrice de compatibilité, y compris les fonctionnalités non prises en charge (Annexe B, R-81) ; l'API de contrôle (`R-92..R-96`) est documentée par les mêmes règles (endpoints, schémas, versions, en-têtes).
12. La persistance est explicite, bornée, supprimable, journalisée et sécurisée (R-75..R-79, §6.4).
13. Les exigences de recherche sont mesurées par un protocole et un seuil d'abandon (Annexe A, R-40, R-22).
14. Les cibles de référence sont nommées, leurs préconditions sont déclarées, leur état de support est publié (§7.2, §7.3) ; le checkpoint réduit de P0 / P1a est nommé (§8).
15. L'état d'implémentation est consigné dans `docs/etat_implementation.md` avec l'artefact de mesure ; NEVER un rapport de jalon sans cet état.
16. Une zone de §14 marquée **A TRANCHER** n'est jamais implantée sans décision consignée (§0, règle 8).
17. Le mode CPU complet est un mode normal de production (I-11, R-90, N-7).
18. La conversion de checkpoints est un outillage externe (I-9, N-1, R-71..R-74 FUTURE).

---

## Annexe A — Axes de maturité et état d'implémentation

### A.1 Axe 1 — maturité de la spécification

| Valeur | Usage | Preuve |
|---|---|---|
| **DEFINED** | périmètre et préconditions écrits | définition + critère d'acceptation |
| **EXPERIMENTAL_SPEC** | chemin de production à valider | prototype fonctionnel, mesure sur la cible |
| **RESEARCH** | hypothèse à prouver | protocole de mesure, seuil d'abandon, journal de gains |
| **FUTURE** | intention non implémentée | justification, critère d'entrée en périmètre |

### A.2 Axe 2 — état d'implémentation

La matrice d'état vit dans `docs/etat_implementation.md`, **hors de ce document normatif** : la spécification change lentement, l'état d'implémentation change à chaque validation. MUST y être mise à jour à chaque validation consignée (règle 21 de §11).

Valeurs : `NOT_STARTED` → `IN_PROGRESS` → `IMPLEMENTED` → `VALIDATED`. `VALIDATED` exige une mesure consignée (§9) sur le couple (checkpoint déclaré + machine de référence). NEVER `IMPLEMENTED` sans chemin complet exercé, NEVER `VALIDATED` sans mesure consignée.

### A.3 Nature des reprises

| `Nature` | Usage |
|---|---|
| **Source** | équation, politique, protocole, format, API, nom ou limite reproduits depuis la source |
| **Adaptation** | adaptation d'une source aux structures d'Alethe.ia |
| **Hypothèse de recherche** | politique, seuil, plage, heuristique ou prédiction proposée par Alethe.ia, à valider |

---

## Annexe B — Matrice de compatibilité des APIs (spécification à compléter avant P4)

**Lignes vides = non spécifié.** MUST être complétée, validée par endpoint, avant P4 ; une ligne non complétée est un refus explicite de la fonctionnalité correspondante (R-82).

| Dimension | OpenAI (`/v1/chat/completions`) | Anthropic (`/v1/messages`) |
|---|---|---|
| Endpoint exact et version consommée | A COMPLETER | A COMPLETER |
| Schéma de requête acceptée (champs, types, valeurs par défaut) | A COMPLETER | A COMPLETER |
| Champs **acceptés** | A COMPLETER | A COMPLETER |
| Champs **ignorés** (acceptés sans effet) | A COMPLETER | A COMPLETER |
| Champs **refusés** (réponse d'erreur, R-82) | A COMPLETER | A COMPLETER |
| En-têtes requis (`Authorization`, `api-key`, `anthropic-version`) | A COMPLETER | A COMPLETER |
| Streaming SSE : format des événements, `data: [DONE]` / `message_stop` | A COMPLETER | A COMPLETER |
| Contenus (`content` blocks, `system`, `tool_use`, `tool_result`) | A COMPLETER | A COMPLETER |
| Tool calls / tool use (schéma, choix de tool, streaming) | A COMPLETER | A COMPLETER |
| Arrêts (`finish_reason` / `stop_reason`) | A COMPLETER | A COMPLETER |
| Format des erreurs (`type`, `code`, `message`, HTTP) | A COMPLETER | A COMPLETER |
| Contenus multimodaux (images) | refusé (`N-4`) | refusé (`N-4`) |
| API de contrôle (R-92..R-96) | hors API OpenAI | hors API Anthropic |
| Tests d'acceptation par endpoint | A COMPLETER | A COMPLETER |

MUST la remplir et la tester par endpoint (R-81, R-82).
+
