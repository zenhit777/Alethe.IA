# ADR-001 — Checkpoint de référence de P0 / P1a

**Statut** : accepté (10 novembre 2026). Remplace le choix de `GSAI-ML/LLaDA-8B-Instruct` comme checkpoint de référence de P1a.
**Régime** : décision d'architecture de validation ; ne modifie aucune exigence du cycle.
**Preuves lues à la source** : API Hugging Face et fichiers des dépôts, 10 novembre 2026.

## Contexte

P0 / P1a exigent un checkpoint exercé, réel, de taille admissible sur la machine de développement, dont la topologie **déclare** les mécanismes que le moteur doit servir : état récurrent GDN, granularité de bloc, token de masque natif. Le checkpoint choisi précédemment (`GSAI-ML/LLaDA-8B-Instruct`) ne déclare rien de tout cela.

Fait constaté sur `GSAI-ML/LLaDA-8B-Instruct` (`config.json`, `configuration_llada.py`) : 32 couches d'attention **pleine**, `n_heads = 32`, `n_kv_heads = 32` (pas de GQA), `intermediate_size = 12288`, `max_sequence_length = 4096`, vocabulaire 126 464, token de masque `126336`, `weight_tying = false`, `use_cache = false`, poids BF16 (~16 Go), licence MIT, `model_type = llada`, `block_type = llama`. **Aucune couche récurrente. Aucune taille de bloc déclarée.** Servir ce checkpoint en P1a ne prouve ni l'état GDN, ni la granularité déclarée, ni la fenêtre glissante, ni la cross-attention : le cycle dLLM y est exercé sur un chemin **uniquement dense**.

Fait constaté sur `UT-IFML/dQwen3.5-0.8B-Base` : `architectures = ["DiffuQwen35"]`, `model_type = diffuqwen35`, 24 couches dont **18 `linear_attention` (GDN) + 6 `full_attention`** (`full_attention_interval = 4`), `num_attention_heads = 8`, `num_key_value_heads = 2`, `linear_conv_kernel_dim = 4`, `hidden_size = 1024`, `vocab_size = 248 320`, **752 393 024 paramètres BF16 = `model.safetensors` 1 504 822 808 octets**, contexte 262 144, **token de masque natif `<|fim_middle|>` = id `248061`**, licence **Apache-2.0**, dépôt de code `AntonXue/dQwen`, base `Qwen/Qwen3.5-0.8B`, `arxiv:2609.20751` (lus le 10 nov. 2026).

Fait constaté sur `UT-IFML/dQwen3.5-2B-Base` : **1 881 825 088 paramètres**, `model.safetensors` = **3 763 687 248 octets** BF16, base `Qwen/Qwen3.5-2B`, licence `other` = **PolyForm Noncommercial 1.0.0**, `architectures = ["Qwen3_5DLLMForConditionalGeneration"]`, `full_attention_interval = 4`, `hidden_size = 2048`, contexte 262 144, **masque `<|fim_middle|>` = id `248061` (le même id que 0.8B : NEVER affirmer qu'il diffère)**, modes **`AR-Trust`**, **`Diffusion-Trust`**, **causal** déclarés sur le **même** jeu de poids, runtime de référence `sglang` / FlashInfer, `block_size: 3` (bloc logique `B = 4` sous la règle de décodage décalée), seuil `0.95`.

## Options

| Option | Preuve exercée | Empreinte BF16 | Licence | Verdict |
|---|---|---|---|---|
| `GSAI-ML/LLaDA-8B-Instruct` | dense uniquement : pas de GDN, pas de granularité déclarée, contexte 4096 | ~16 Go | MIT | rejeté comme référence de P1a |
| `UT-IFML/dQwen3.5-0.8B-Base` | GDN natif (18 GDN + 6 full attention), masque natif `248061`, contexte 262 144, **752 393 024 paramètres**, `model.safetensors` = **1 504 822 808 octets** ≈ **1,40 GiB** | Apache-2.0 | **retenu** |
| `UT-IFML/dQwen3.5-2B-Base` | modes dual-stream **déclarés** (R-13, R-14) | **1 881 825 088 paramètres**, `model.safetensors` = **3 763 687 248 octets** ≈ **3,50 GiB** | PolyForm Noncommercial 1.0.0 | référence complémentaire, jamais seule |
| DiffusionGemma 26B A4B (`google/diffusiongemma-26B-A4B-it`, sha `f7f5b7f5fa82ffc52addd066915886d497f5517b`) | cible de P1b (cross-attention, canva 256 (`canvas_length`), fenêtre 1024 (`sliding_window`), 30 couches, MoE 128 experts / top-8) | **51 647 701 024 octets mesurés** sur `tree/main` (11 shards) pour **25 823 781 228 paramètres BF16** — la taille « ~50 Go (25,2 B) » était une estimation **fausse** ; **51,65 Go > 47,04 GiB de RAM** | Apache-2.0 (`cardData`) | hors P0 / P1a ; validation CPU-only complète reportée à P5 ; **pipeline `image-text-to-text`** (`vision_config` 27 couches, `gemma4_vision`) → NEVER le décrire comme texte-only |
| `UT-IFML/dQwen3-1.7B-Base` (`sha` `f7a02dcf8fac5f86b2fe3fcbcc990b74acaee36f`, 1 720 574 976 paramètres BF16 = 3 441 149 952 octets ≈ **3,20 GiB**, **28 couches toutes `full_attention`, 0 `linear_attention`**, contexte 40960) | **contrôle iso-famille** : **même méthode d'adaptation**, backbone dense → isole la contribution GDN sans faire varier la méthode, le tokenizer, le contexte ni le masque | Apache-2.0 | **retenu comme contre-épreuve**, jamais comme référence de P1a |

## Décision

Le checkpoint de référence de **P0 / P1a** est **`UT-IFML/dQwen3.5-0.8B-Base`**. MUST être épinglé par commit HF au téléchargement, la revision consignée dans chaque rapport (§13.1, zone 23).

MUST exercer en P0 : **BF16** (format de la distribution de référence, cible de poids de P0), **GGUF Q8_0** (R-60, corpus LLaDA, commit de la quantization épinglé) et **FP32 comme oracle de comparaison** (jamais comme format de poids à servir en production) ; FP16 (`F-3`) est reporté à son jalon. L'empreinte mesurée (§8.1) est la base de toute décision d'admission.

`GSAI-ML/LLaDA-8B-Instruct` **reste dans la spécification, repositionné** : (1) corpus de validation du format GGUF Q8_0 (`diffuse-cpp/LLaDA-8B-Instruct-GGUF`, `llada-8b-q8_0.gguf`, commit de la quantization à épingler) et de la session CPU-only (R-90) ; (2) **contre-épreuve de profil** : prouve que le moteur refuse, sur un checkpoint dense à contexte 4096, toute attribution de cache KV, de fenêtre glissante, d'attention croisée, de double flux, de table n-gram ou d'état GDN. NEVER le classer comme dLLM adapté : c'est un dLLM natif (R-16). NEVER lui attribuer une granularité supposée : sa taille de bloc est un paramètre de configuration consigné.

`UT-IFML/dQwen3.5-2B-Base` est la **référence complémentaire** : sert à exercer les modes `AR` / `Diffusion` / `dual-stream` sur un checkpoint qui les déclare (R-13, R-14, R-97), NEVER un prérequis de la validation de P1a. MUST consigner la licence PolyForm Noncommercial 1.0.0 ; NEVER le considérer comme libre d'usage commercial sans confirmation juridique. MUST NOT présumer une compatibilité du seul fait de l'appartenance à la famille Qwen3.5 : la carte exige `sglang`, les préconditions d'adaptation MUST être consignées avant toute exécution par Alethe.ia (R-16, R-85).

## Conséquences

- §8, §7.2.1, §13.1, N30, zone 22 : le nom change, les faits consignés sont ceux lus à la source.
- P1a est exercé sur un chemin **récurrent** (GDN), pas uniquement dense : l'état récurrent est validé en P1a, NEVER reporté à P3.
- La granularité MUST être lue depuis les métadonnées du checkpoint (R-160), NEVER supposée.
- La validation CPU-only (R-90) est **deux exercices distincts** : (a) session complète **BF16 sur dQwen** (chemin récurrent GDN, canva entier), (b) session complète **Q8_0 sur le corpus GGUF LLaDA** (chemin dense, contexte 4096) ; NEVER fusionner les deux, NEVER n'exercer que l'un des deux.
- L'empreinte BF16 de dQwen-0.8B (**1 504 822 808 octets ≈ 1,40 GiB** de poids) tient dans la machine de développement (`ADR-006`, 32 607 MiB de VRAM) : la contrainte d'admission devient mesurable en P0, NEVER un argument de taille de modèle. MUST mesurer l'empreinte **totale** (§8.1 : poids + tampons + activations + états récurrents + copies), NEVER se limiter aux poids.

## Correction du 10 novembre 2026 — classification de `UT-IFML/dQwen3.5-0.8B-Base`

Une lecture antérieure de la spec 0.2.1 décrivait dQwen3.5-0.8B-Base comme une « **base à convertir** » et écrivait qu'il ne devait **pas** être servi comme dLLM. **C'est faux, contredit par la carte publiée** : « A masked diffusion language model **adapted from** Qwen3.5-0.8B. The backbone is hybrid: only its attention layers are made bidirectional, and the Gated DeltaNet layers stay causal » (`README.md`, lu le 10 novembre 2026), `architectures = ["DiffuQwen35"]`, `model_type = diffuqwen35`. Le checkpoint **est déjà** un dLLM adapté par la méthode **`dQwen35`** (`arxiv:2609.20751`). MUST le charger et l'exécuter **tel quel** ; NEVER exiger de l'opérateur une seconde conversion ; NEVER le classer dans la catégorie « base à convertir » (`Qwen/Qwen3.5-0.8B-Base` est le **modèle source** de l'adaptation, jamais l'artefact exercé de P0 / P1a).

**Conséquences de la correction** : §7.1 catégorie 2 (méthode `dQwen35` ajoutée à la liste), R-16 (un checkpoint adapté est servi tel quel), §7.2.1 (profil requalifié), §8 (rôles de validation), §13.1 ligne « Base de conversion », `docs/CHANGELOG.md`.

**Deuxième fait corrigé** : `block_length = 32` est un **exemple d'option d'appel** de `generate` (la carte : « `generate` decodes the whole canvas at once, committing positions above a confidence threshold (`tau=0.9`) ; pass `block_length=32` for left-to-right block decoding, or `tau=None, steps_per_block=k` for a fixed budget »). **Aucune granularité de conversion n'est déclarée par le checkpoint** ; NEVER figer `B = 32` comme granularité de référence (R-160 corrigé en conséquence).
