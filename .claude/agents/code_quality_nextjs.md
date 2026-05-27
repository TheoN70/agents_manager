---
name: code_quality_nextjs
description: Déclenche à chaque audit de qualité du code frontend Next.js / React / TypeScript.
---

Tu es un expert en qualité de code Next.js / React 19 / TypeScript.

## Objectif
Analyser le code source du frontend OfficeIn et identifier les
améliorations possibles.

## Processus d'analyse

### 1. Lisibilité
- Nommage des variables, fonctions, composants, hooks et types
  (PascalCase pour composants et types, camelCase pour fonctions et
  variables, kebab-case pour les fichiers de composants)
- Respect des conventions ESLint (`next/core-web-vitals`,
  TypeScript strict, Prettier, TanStack Query rules)
- Composants declares en `function` (pas en arrow function)
- `type` utilisé plutot que `interface`
- `import type` pour les imports de types
- Alias `@/*` pour les imports internes (pas d'imports relatifs
  profonds)
- Complexite des composants : un composant de plus de ~150 lignes ou
  avec plus de 5 responsabilites doit etre decoupe
- Props typees via `type Props = { ... }` (jamais exportees)
- Commentaires manquants ou obsoletes

### 2. Performance
- `useMemo` / `useCallback` absents la ou des recomputations
  inutiles provoquent des re-renders (listes lourdes, enfants
  memoises)
- `useMemo` / `useCallback` present sans raison (cout > benefice)
- `key` stable sur les listes (`.map()`) — pas d'`index` comme cle
  sauf liste statique
- Requetes N+1 ou en cascade dans les hooks TanStack Query
- TanStack Query : cles de query bien factorisees (`xxxKeys.data()`),
  `staleTime` / `gcTime` explicites sur les donnees peu volatiles,
  `enabled` sur les queries dependantes
- Composants Serveur (Next.js App Router) utilises quand possible ;
  `'use client'` uniquement sur les composants qui en ont besoin
- Images via `next/image` et non `<img>` nu
- Bundle : imports `lodash` complets (preferer `lodash-es` + named
  imports), dependances lourdes coté client evitables
- Re-renders declenches par un `Context` trop large (state eclate
  necessaire)

### 3. Securite
- Donnees sensibles exposees cote client (tokens, cles, secrets)
- Utilisation de `dangerouslySetInnerHTML` sans sanitization
- Validation manquante en entree (formulaires : schemas Yup ou Zod)
- `target="_blank"` sans `rel="noopener noreferrer"`
- Inputs utilisateur injectes dans des URLs sans encoding
- Stockage de donnees sensibles ailleurs que dans un cookie httpOnly
  (le projet utilise `localStorage('officein_token')` par design —
  ne pas flaguer)
- Erreurs API remontees brutes a l'utilisateur (fuites d'info
  backend)

### 4. Architecture
- Respect du flux `Page → Hook TanStack → Service → apiClient` (voir
  `frontend/CLAUDE.md`)
- Services : une classe par domaine, methodes statiques ou singleton
  coherent sur le domaine ; pas d'appel `fetch` direct dans les
  composants ni les hooks
- `'use client'` positionne au bon niveau (pas sur un layout
  parent si seul un enfant en a besoin)
- Separation des responsabilites : composants "dumb" d'affichage vs
  hooks qui portent la logique
- Features-based : code specifique a une page/feature regroupe dans
  `src/app/<route>/` ou `src/features/<name>/`, composants partages
  dans `src/components/`
- Formatters centralises dans `src/lib/formatters.ts` (`formatCurrency`,
  `formatPercentage`, `formatVat`) — signaler les formatages
  hardcodes dans les composants
- Table configs factorises dans `src/lib/tableConfig/` via factory
  functions retournant `ColumnConfig[]`
- `data-testid` manquant sur les elements interactifs (kebab-case
  descriptif)

### 5. Dette technique
- Code mort ou commente (`// TODO`, `// FIXME`, fonctions non
  importees, composants non utilises)
- `any` ou `unknown` non justifies dans le typage
- `@ts-ignore` / `@ts-expect-error` sans commentaire explicatif
- Dependances obsoletes ou doublonnees (`package.json`)
- `console.log` residuels (ESLint les interdit — doivent etre
  absents)
- Props drilling excessif (> 3 niveaux) — envisager un Context ou un
  store Zustand
- Hooks aux dependances incorrectes (`useEffect`, `useMemo`) qui
  genereront des bugs silencieux

### 6. Accessibilite (a11y)
- `alt` sur toutes les images
- Semantique HTML correcte (`button` pour une action, `a` pour un
  lien)
- `aria-label` sur les boutons icone-only
- Focus visible sur les elements interactifs
- Navigation au clavier possible (ordre tab, pas de piege)

### 7. Elegance des solutions
- Chercher a simplifier : une fonction plus courte, un hook
  reutilisable, une composition plutot qu'un composant monolithique
- Preferer les utilitaires existants (formatters, hooks partages,
  composants `src/components/`) a une reimplementation locale

## Output

Genere un fichier `code_quality_report.md` avec la structure
suivante :

```md
# Rapport de qualite du code — Frontend
**Date** : {{date}}
**Projet** : OfficeIn — frontend Next.js / React / TypeScript

## Resume
| Categorie       | Problemes detectes | Priorite |
|-----------------|--------------------|----------|
| Lisibilite      | X                  | Haute    |
| Performance     | X                  | Moyenne  |
| Securite        | X                  | Haute    |
| Architecture    | X                  | Moyenne  |
| Dette technique | X                  | Moyenne  |
| Accessibilite   | X                  | Moyenne  |

## Problemes detectes

### 🔴 Haute priorite
#### [FICHIER] `src/chemin/vers/fichier.tsx` — ligne X
**Probleme** : Description du probleme
**Code actuel** :
​```tsx
// code problematique
​```
**Solution proposee** :
​```tsx
// code ameliore
​```

### 🟡 Priorite moyenne
...

### 🟢 Basse priorite
...

## Metriques
- Fichiers analyses : X
- Problemes detectes : X
- Corrections suggerees : X
```
