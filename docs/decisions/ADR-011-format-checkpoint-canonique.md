# ADR-011 — Format de checkpoint canonique

**Statut** : **proposé — A TRANCHER** (10 novembre 2026). Zone 1 de §14.
**Régime** : décision de format d'échange ; ne modifie aucune exigence, fixe la convention de sérialisation.
**Preuves lues à la source** : spec ggml `docs/gguf.md` (structure de Q8_0 : `ggml_half d` + `int8_t qs[32]`, 34 octets pour 32 éléments), 10 novembre 2026.

## Contexte

R-58 est normative : un checkpoint canonique MUST transporter la topologie **lisible** (R-160 : architecture, couches, têtes, dimensions, GDN / Mamba / SSM, **token de masque natif**, granularité déclarée), les formats de poids, la conversion AR → dLLM appliquée (R-16), la provenance et la licence. La **convention de format** n'est pas tranchée.

Trois faits contraignent la convention :
1. Les distributions de référence sont publiées en **safetensors** (dQwen, Clover, SDAR, LLaDA, DiffusionGemma), BF16 pour dQwen et Clover.
2. Les formats quantisés sont publiés en **GGUF** (LLaDA, FUGU, LFM2, LightOn, SFW), avec des structures **spécifiques par format** (Q8_0 = `float16` d + `int8` qs, 34 octets pour 32 éléments — ADR-003).
3. Les métadonnées canoniques MUST être **lues avant** la lecture des poids (R-58, R-98, R-101 : une topologie non déclarée est refusée, NEVER devinée ; un checkpoint **reconnu sans adaptateur** est refusé en nommant l'architecture). NEVER une topologie reconstruite depuis des noms de tensors.

## Options

| Option | Effet | Verdict |
|---|---|---|
| safetensors seul | topologie via `config.json` séparé ; pas de formats quantisés | rejeté : couvre F-1, pas F-2 / F-5 / F-6 / F-7 / F-9 / F-10 |
| GGUF seul | formats quantisés couverts ; la topologie d'un modèle à couches récurrentes n'est pas représentée dans les clés ggml standard | rejeté : ne couvre pas R-160 |
| **Format canonique d'Alethe.ia (conteneurs + clé de topologie), GGUF comme format d'import / export** | la topologie est une clé **du format**, pas une lecture de `config.json` ; NEVER une topologie devinée | **retenu comme proposition** |

## Décision proposée

Le format canonique MUST être un conteneur **à clé de topologie explicite** (la granularité de bloc, l'architecture par couche, le token de masque natif, les formats de poids, la politique d'admission, la conversion AR → dLLM appliquée) ; MUST transporter la **licence** et la provenance (dépôt + revision épinglée). MUST être **refusé à l'import** si une clé de topologie est absente, NEVER complétée par inférence (R-58, R-98, §7.2, `I-4`).

MUST respecter les structures **réelles** des formats importés (ADR-003 : facteur d'échelle de Q8_0 = `float16`, **pas** `float32`, **pas** d'E8M0). MUST traiter les formats F-1 à F-10 (poids, activations, quantization, format de cache) comme **distincts** ; NEVER appliquer à un format la tolérance d'un autre (§6.3, R-60, R-66). MUST déclarer le format canonique en **versioning** (zone 15, `ADR-005`) ; NEVER un format canonique sans version consignée.

MUST fournir l'import GGUF (F-2, F-5 à F-7, F-9, F-10) et l'import safetensors (F-1), NEVER les fusionner sous un seul lecteur : la structure d'un bloc GGUF et la structure d'un tensor safetensors ne sont pas la même. NEVER convertir en silence.

## Conséquences

- Zone 1 de §14 : **reste A TRANCHER** jusqu'à la décision du mainteneur ; cette ADR consigne l'obligation d'une clé de topologie explicite.
- §7.2, R-58, R-98 : la clé de topologie est un objet typé, exercé par P0 (clé construite depuis des fixtures, NEVER depuis des poids réels) ; le journal de découverte (`ADR-013`) en est l'artefact.
- §13.1 : chaque format exercé MUST avoir ses vecteurs de référence et ses tolérances chiffrées (ADR-002, ADR-003).
