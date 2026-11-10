# ADR-013 — Découverte de la configuration, descripteur interne et adaptateurs architecturaux (zone 30)

**Statut** : proposé (10 novembre 2026) — **non tranché**, à soumettre au mainteneur (§14, processus de décision).
**Régime** : décision d'architecture de chargement ; NE MODIFIE PAS les contrats du cycle d'exécution, AJOUTE les exigences R-98, R-101, R-102.
**Preuves lues à la source** : `config.json` et `README.md` de `UT-IFML/dQwen3.5-0.8B-Base` (10 novembre 2026), `tokenizer_config.json` + `added_tokens` de `tokenizer.json` (dQwen 0.8B et 2B), `config.json` de `GSAI-ML/LLaDA-8B-Base`, `google/diffusiongemma-26B-A4B-it`, `yuchen-zhu-zyc/HybridDiffusion-2B`.

## Contexte

La spécification mentionne `config.json` et `tokenizer_config.json` comme **sources** (§13.1) et exige des faits lus dans ces fichiers (masque `248061`, `full_attention_interval = 4`, `layer_types`), mais **ne définit aucun système normatif de découverte** : pas de lecture obligatoire des champs, pas de validation de cohérence, pas de descripteur interne, pas de sélection d'adaptateur, pas de distinction entre un modèle **reconnu** et un modèle **exécutable**.

Fait constaté sur `UT-IFML/dQwen3.5-0.8B-Base` (`config.json`, 10 novembre 2026) : `architectures = ["DiffuQwen35"]`, `model_type = diffuqwen35`, **`auto_map = { "AutoConfig": "modeling_dqwen3_5.DiffuQwen35Config", "AutoModel": "modeling_dqwen3_5.DiffuQwen35" }`**, `num_hidden_layers = 24`, `layer_types` = 18 `linear_attention` + 6 `full_attention`, `full_attention_interval = 4`, `hidden_size = 1024`, `num_attention_heads = 8`, `num_key_value_heads = 2`, `head_dim = 256`, `linear_num_key_heads = linear_num_value_heads = 16`, `linear_key_head_dim = linear_value_head_dim = 128`, `linear_conv_kernel_dim = 4`, `partial_rotary_factor = 0.25`, `mamba_ssm_dtype = "float32"`, `max_position_embeddings = 262144`, `vocab_size = 248320`, `tie_word_embeddings = true`, `use_cache = false`, `dtype = "bfloat16"`, `transformers_version = "5.13.0"`.

**La carte impose `trust_remote_code=True`** pour son exemple (`AutoModel.from_pretrained(..., trust_remote_code=True)`) et exige un GPU CUDA : l'architecture **n'est pas prise en charge par la bibliothèque de référence**, elle est fournie **par le dépôt du modèle**. Un moteur qui découvre une architecture de cette manière doit décider **explicitement** s'il l'exécute avec son propre adaptateur, sans importer le code du dépôt (R-100, `ADR-014`).

Fait complémentaire : `GSAI-ML/LLaDA-8B-Base` (`model_type = llada`, `block_type = llama`) ne déclare **aucune** taille de bloc ; `yuchen-zhu-zyc/HybridDiffusion-2B` publie la chaîne `generation_family = Hybrid`, qui **n'est pas** une valeur de l'énumération canonique de §7.1. La découverte **doit** normaliser, et consigner la chaîne source à côté de la valeur normalisée ; à défaut, la specification écrit des valeurs inventées.

## Options

| Option | Découverte | Coût | Risque | Verdict |
|---|---|---|---|---|
| Chargeur écrit par nom de modèle (`dQwen`, `LLaDA`, `Clover`) | aucune lecture normative | faible | la topologie est devinée depuis un nom ; un checkpoint renommé ou dérivé est chargé sur le mauvais chemin | rejetée |
| **Descripteur interne lu + table d'adaptateurs par architecture reconnue** | lecture des champs canoniques, validation de cohérence, mappage vers §7.1, refus nommant l'architecture | une table à tenir, un journal par chargement | la table est un objet normatif à faire vivre | **retenue** |
| Exécution du code fourni par le dépôt (`auto_map`, `trust_remote_code`) | délégation de la définition de l'architecture au dépôt | nulle | code distant exécuté, dépendance à `transformers`, réseau requis ; contredit `ADR-014` | rejetée |

## Décision

1. **Descripteur interne** : MUST lire les champs canoniques listés en R-98, MUST valider la cohérence interne (couches vs `layer_types`, dimensions vs index de poids), MUST produire un descripteur typé par les **trois dimensions de §7.1** + `profil_attention` + `granularité_déclarée` + `token_de_masque` + `contexte_max` + `format_de_poids` + `adaptateur_retenu` + `etat = {reconnu, exécutable, refus}`.
2. **Table des adaptateurs** : MUST associer un adaptateur à chaque architecture **reconnue**. MUST refuser une architecture inconnue **par nom** (R-83), MUST refuser une architecture reconnue **sans adaptateur écrit** en nommant l'architecture (jamais un chargement partiel).
3. **Mappage** : MUST consigner la chaîne **brute** lue (`"DiffuQwen35"`, `"Hybrid"`, `"llada"`) **et** la valeur normalisée, côte à côte, dans le journal de découverte. MUST signaler un mappage non établi comme un refus d'admission.
4. **Statut `reconnu` ≠ `exécutable`** : MUST afficher les deux dans le rapport de chargement ; NEVER présenter `reconnu` comme une capacité d'exécution.
5. **Premier artefact de P0** : MUST produire le journal de découverte du checkpoint de référence **avant** tout kernel (§8.1).

## Conséquences

- §7.4 (R-98, R-101, R-102) : les exigences sont écrites ; **zone 30** : le schéma exact du descripteur (champs obligatoires, types, table des adaptateurs publiée) reste **à trancher** par le mainteneur.
- §8 P0 / §8.1 : le journal de découverte est un artefact de clôture de P0.
- `ADR-014` : la découverte s'exerce **sans réseau** ; le code du dépôt n'est jamais exécuté.
- NEVER étendre la table pour une architecture non exercée.
