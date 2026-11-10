# ADR-002 — BF16 comme format numérique de référence de P0

**Statut** : accepté (10 novembre 2026).
**Régime** : décision de validation numérique ; NE MODIFIE PAS les tolérances de §6.3, les ajoute par format.
**Preuves lues à la source** : API Hugging Face (champs `safetensors.parameters`), 10 novembre 2026.

## Contexte

P0 validait **un seul** format (GGUF Q8_0) et reportait BF16 à P5. Fait constaté (API Hugging Face, 10 novembre 2026) : les distributions de référence des checkpoints dLLM sont publiées en **BF16** — `UT-IFML/dQwen3.5-0.8B-Base` : **752 393 024 paramètres BF16**, `model.safetensors` = **1 504 822 808 octets** ; `UT-IFML/dQwen3.5-2B-Base` : **1 881 825 088 paramètres BF16**, `model.safetensors` = **3 763 687 248 octets** ; `Diffuspace/Clover-35B-A3B-Preview` : **34 661 350 848 paramètres BF16**, repo complet (tous fichiers, plusieurs formats de shards) = **138 685 741 760 octets**. La conversion AR → dLLM de la base dQwen est donc **exercée en BF16** : si BF16 n'est pas validé en P0, la comparaison de déquantification de P0 ne couvre pas le format dans lequel la cible est publiée.

Deux faits de précision, consignés :
1. BF16 n'est **pas** FP16 : mantisse 8 bits contre 11. Une tolérance fixée pour FP16 **NEVER** appliquée à BF16.
2. `shader-f16` est une **fonctionnalité WebGPU à lire sur le device** (§4.5), pas une garantie : BF16 MUST être calculable sans `shader-f16` (accumulation en `f32`, conversion explicite) sur un device qui n'expose pas la fonctionnalité. NEVER rendre P0 inatteignable par une dépendance à `shader-f16`.

## Options

| Option | Couverture de P0 | Coût | Verdict |
|---|---|---|---|
| Q8_0 seul | format de la quantization, pas le format de publication | faible | rejeté : ne couvre pas la cible |
| **BF16 (format de poids de référence) + FP32 (oracle de comparaison) + Q8_0 (corpus GGUF LLaDA) en P0** | format de publication + référence de précision + format quantisé | un jeu de vecteurs de référence par format exercé | **retenu** |
| BF16 en P5 | P0 valide un format absent de la cible | report du risque numérique | rejeté |

## Décision

**BF16 MUST être exercé en P0 comme format numérique de référence.** MUST valider en P0, chacun avec ses **vecteurs de référence propres** et ses tolérances chiffrées (§6.3) : **BF16** (format de poids de la distribution de référence), **Q8_0** (corpus `diffuse-cpp/LLaDA-8B-Instruct-GGUF`, commit de la quantization épinglé), et MUST utiliser **FP32 comme oracle de comparaison** — **jamais** comme format de poids que le moteur doit servir en production. **FP16 n'est pas une obligation de P0** : c'est un format `F-3` de R-60 ; MUST être exercé à son propre jalon avec sa ligne de tolérance, NEVER exigé en P0 sans justification **mesurée**. MUST séparer BF16 de FP16 dans les types, NEVER les confondre sous un même alias.

MUST calculer BF16 en accumulant en `f32` (R-28) et convertir aux extrémités. MAY utiliser `shader-f16` **si et seulement si** le device l'expose (§4.5), NEVER comme précondition de P0.

MUST consigner, pour chaque format : la structure réelle lue dans le code source de la déquantification (le facteur d'échelle de Q8_0 est `float16`, pas `float32` — ADR-003), la tolérance mesurée contre la référence FP32, la tolérance entre backends (CPU ↔ WebGPU), et la liste des formats F-1 **non** couverts par la validation de P0. NEVER présenter la validation d'un format comme couvrant les formats F-1.

MUST mesurer l'empreinte BF16 (§8.1) : c'est l'empreinte de la distribution de référence, la seule base d'une décision d'admission (R-32). NEVER décider une admissibilité depuis une taille de modèle annoncée.

## Conséquences

- §6.3 : une ligne par format exercé, tolérance chiffrée par format, NEVER une tolérance globale.
- §8.1 : P0 MUST produire un rapport de références numériques **par format exercé** ; NEVER une clôture de P0 sans BF16, Q8_0 et l'oracle FP32 consigné. FP16 MUST être reporté à son jalon (`F-3`), NEVER présenté comme acquis.
- LLaDA n'est plus la seule validation de format : il valide Q8_0 et la session CPU-only sur le corpus GGUF ; dQwen valide BF16 et la session CPU-only BF16 — **deux exercices distincts, NEVER fusionnés** (R-90).
- La sémantique E8M0 (`0x00` = `2^-127`, `0xFF` = NaN réservé) est consignée pour les formats MX, NEVER appliquée à Q8_0 (ADR-003).
