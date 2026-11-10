# ADR-014 — Fonctionnement hors-ligne et interdiction d'exécuter le code d'un dépôt de modèle

**Statut** : proposé (10 novembre 2026) — **non tranché**, à soumettre au mainteneur.
**Régime** : décision d'architecture de chargement et d'exploitation ; AJOUTE les exigences R-99, R-100.
**Preuves lues à la source** : `README.md` et `config.json` de `UT-IFML/dQwen3.5-0.8B-Base` (10 novembre 2026) ; `tokenizer_config.json` de `UT-IFML/dQwen3.5-2B-Base` ; frontmatter de la carte (`inference: false`).

## Contexte

Aucune exigence **offline-first** n'est écrite dans la spécification. Les faits de la carte du checkpoint de référence contredisent un chargement en ligne :

- la carte dit « **Needs a CUDA GPU** and `transformers>=5.13` (tested with `torch 2.7.1+cu128`, `flash-linear-attention 0.5.1`) » ;
- l'exemple est `AutoModel.from_pretrained("UT-IFML/dQwen3.5-0.8B-Base", trust_remote_code=True, dtype=torch.bfloat16).cuda().eval()` ;
- `config.json` déclare `auto_map = { "AutoConfig": "modeling_dqwen3_5.DiffuQwen35Config", "AutoModel": "modeling_dqwen3_5.DiffuQwen35" }` : la définition de l'architecture **vit dans le dépôt du modèle** (`modeling_dqwen3_5.py`), la bibliothèque de référence ne la connaît pas ;
- le frontmatter de la carte porte **`inference: false`** : le dépôt n'offre pas d'endpoint d'inférence hébergé.

Un moteur qui s'aligne sur cet exemple **importerait du code distant**, dépendrait de `transformers` et d'un GPU CUDA, et **échouerait sans réseau**. Alethe.ia doit implémenter son propre adaptateur (R-101, `ADR-013`) et exécuter les poids **locaux**, ce que la carte ne contredit pas : les poids sont `safetensors` BF16.

## Options

| Option | Chargement | Coût d'exploitation | Risque | Verdict |
|---|---|---|---|---|
| Online-first (résolution par dépôt distant, cache après premier appel) | réseau obligatoire au premier chargement | dépendance au réseau, à l'API, au service | panne de production, exécution d'un code non consigné | rejetée |
| **Offline-first (dépôt local épinglé, téléchargements explicites et consignés, aucun code distant exécuté)** | réseau **jamais** requis pour inspecter, charger, inférer | un mandataire de téléchargement explicite, un journal | la table des adaptateurs doit couvrir l'architecture | **retenue** |
| Offline comme `SHOULD` | réseau autorisé, recommandation seulement | nulle | la contrainte n'est pas mesurable, P0 ne peut pas la valider | rejetée |

## Décision

1. **Inspection hors-ligne** : MUST inspecter, normaliser (R-98) et décrire un checkpoint local **sans aucun accès réseau**.
2. **Chargement et inférence hors-ligne** : MUST charger et servir une session complète **sans accès réseau** ; MUST que les métadonnées, le tokeniser, l'adaptateur et les poids nécessaires soient **résolus localement avant** l'exécution (revision épinglée, zone 23).
3. **Téléchargements explicites** : MUST être déclenchés par un mandat de l'opérateur (commande nommée), MUST consigner l'URL, le `sha` vérifié, la revision exercée, la date, la taille mesurée ; NEVER un téléchargement déclenché implicitement par un chargement, une inspection ou un mappage.
4. **Code distant jamais exécuté** : MUST traiter `auto_map` et `trust_remote_code` comme des **indices d'architecture** ; NEVER télécharger, importer ni exécuter le code d'un dépôt de modèle. NEVER présenter la compatibilité `transformers` comme une capacité d'Alethe.ia (`ADR-001`).
5. **Mesure** : MUST exercer P0 **réseau coupé** et MUST publier un journal des accès réseau **vide** pendant l'inspection et l'inférence ; MUST consigner un téléchargement exercé (avec mandat) et un refus consigné (sans mandat).

## Conséquences

- §7.4 R-99, R-100 : exigences écrites, maturité `DEFINED`.
- §8 P0 / §8.1 : l'inspection locale sans réseau est un critère de clôture de P0 ; le journal de découvertes porte l'état réseau.
- `ADR-013` : l'adaptateur interne est la seule voie d'exécution ; `modeling_dqwen3_5.py` **n'est pas** exécuté par Alethe.ia.
- Zone 30 : le schéma du descripteur reste à trancher ; la politique de cache local et de résolution de revisions est **à consigner** avec la machine de référence (§9, `ADR-006`).
