# ADR-003 — Structure réelle de Q8_0 et validation par format

**Statut** : accepté (10 novembre 2026).
**Régime** : décision de validation numérique ; corrige une description fausse, ne modifie aucune exigence.
**Preuves lues à la source** : spec `ggml` `docs/gguf.md`, API Hugging Face, 10 novembre 2026.

## Contexte

La spécification décrivait Q8_0 comme « bloc de 32 éléments avec un facteur d'échelle `float32` et un exposant E8M0 ». C'est **faux** : la structure de Q8_0 (ggml) est `(ggml_half d, int8_t qs[32])` = **34 octets pour 32 éléments**. Le facteur d'échelle est **`float16`** (`ggml_half`), **pas `float32`**, et il n'y a **aucun** exposant E8M0 dans Q8_0.

E8M0 est la sémantique d'exposant des formats **MX** (MXFP4, MXFP8) : `0x00` = `2^-127`, `0xFF` = NaN réservé. NEVER appliquer la sémantique E8M0 à Q8_0 : la structure est différente, la tolérance est différente.

Fait constaté sur le corpus disponible : `diffuse-cpp/LLaDA-8B-Instruct-GGUF` publie `llada-8b-q8_0.gguf` et `llada-8b-q4km.gguf` (carte déclarant `apache-2.0` pour le dépôt dérivé, base MIT, `context_length = 4096`, 8 015 581 184 paramètres). C'est un dépôt **dérivé** : sa revision est **distincte** de la revision des poids du modèle d'origine ; MUST épingler les deux.

## Décision

MUST décrire la structure **exacte** dans le code : bloc de **34 octets pour 32 éléments**, facteur d'échelle `float16`, quantes `int8`. MUST NOT appeler le facteur d'échelle `float32` ; MUST NOT parler d'E8M0 pour Q8_0.

MUST valider la déquantification de Q8_0 contre une référence FP32 avec tolérance **chiffrée pour Q8_0** (§6.3), NEVER une tolérance héritée d'un autre format. MUST valider la sémantique E8M0 (`0x00` / `0xFF`) **uniquement** pour les formats MX.

MUST épingler, par format exercé, la revision du dépôt de quantization **et** la revision des poids d'origine ; MUST les consigner comme deux champs distincts dans l'index des sources (§13.1, zone 23). NEVER absorber la quantization dans la revision du modèle.

MUST déclarer, dans le rapport de P0, la liste des formats F-1 **non** couverts par la validation exercée. NEVER présenter « Q8_0 validé » comme « formats F-1 validés ».

## Conséquences

- §8, §6.3, R-60 : la structure est consignée telle que lue ; chaque format supplémentaire (BF16, FP16, FP32, FP8 E4M3, FP8 E5M2, MXFP4, NVFP4, INT8, INT4, GGUF `q<N>`) MUST disposer de ses vecteurs de référence et de ses tests.
- Le dépôt de quantization devient une source à épingler au même titre que le modèle.
- Une tolérance par format est un **prérequis d'entrée** de chaque jalon (§8, critères d'entrée).
