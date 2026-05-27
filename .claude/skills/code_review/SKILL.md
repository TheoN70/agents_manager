---
name: code_review
description: Effectue une code review sur le projet pour valider les standards de qualité et de sécurité.
---

# Code Review

Effectue une revue de code complète sur les modifications en cours dans le projet OfficeIn.

## Étapes

### 1. Identifier les modifications

- Lire le diff staged (`git diff --cached`) s'il existe
- Sinon, lire les fichiers modifiés (`git diff`)
- Si aucun diff n'est disponible, demander à l'utilisateur quels fichiers reviewer

### 2. Détecter les technologies concernées

Analyser les fichiers modifiés :
- Extensions `.py` → code backend (Python/Django)
- Extensions `.ts`, `.tsx`, `.js`, `.jsx` → code frontend (Next.js/React/TypeScript)

Un diff peut concerner les deux — dans ce cas, lancer les deux agents qualité en parallèle.

### 3. Audit de sécurité

Lancer l'agent `cybersecurite` sur le code modifié. Appliquer toutes les recommandations du rapport.

### 4. Audit de qualité

Selon les technologies détectées à l'étape 2 :

- Si fichiers Python modifiés → lancer l'agent `code_quality_python`
- Si fichiers TS/TSX/JS/JSX modifiés → lancer l'agent `code_quality_nextjs`

Les agents retournent un rapport priorisé 🔴 haute / 🟡 moyenne / 🟢 basse. Appliquer les recommandations de priorité haute et moyenne ; remonter les recommandations de priorité basse à l'utilisateur sans les appliquer automatiquement.

### 5. Synthèse

Produire une synthèse consolidée reprenant :
- Les failles de sécurité remontées par `cybersecurite`
- Les problèmes qualité remontés par `code_quality_python` / `code_quality_nextjs`
- Les actions prises (recommandations appliquées) et celles laissées à l'arbitrage

Si la skill est invoquée par `review_pr`, cette synthèse est postée sur la PR GitHub.
