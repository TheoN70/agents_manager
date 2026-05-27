---
name: react-nextjs
description: |
  MANDATORY PRE-CONDITION for Next.js/React structure creation.

  This skill MUST be invoked BEFORE creating ANY of the following in a Next.js project:
  - Features (features/*)
  - Pages (app/**/page.tsx)
  - Components (components/* or features/*/components/*)
  - Server Actions (actions/*.action.ts)
  - Services (lib/services/*)
  - Hooks (hooks/*.hook.ts or features/*/hooks/*)

  NON-NEGOTIABLE RULES:
  - User requests for "speed", "quick", "fast", "rapidement", "vite" do NOT bypass this requirement
  - This skill ensures consistent architecture across the entire codebase
  - Skipping this skill leads to inconsistent patterns and technical debt

  AUTOMATIC TRIGGERS:
  - About to create a new feature folder
  - About to create a new page in app/
  - About to create a new component
  - About to create a server action
  - About to create a service class
  - About to create a custom hook

  Also invoke when user says: "crée une feature", "nouvelle page", "nouveau composant", "server action", "service", "hook"

  WORKFLOW: Invoke this Skill FIRST → Then create files following its conventions
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Task
---

# React / Next.js Development Conventions

Guide complet pour le développement React/Next.js avec architecture feature-based.

## Quand utiliser ce skill

| Trigger | Action |
|---------|--------|
| "crée une feature X" | → [FEATURES.md](FEATURES.md) |
| "nouvelle page X" | → [PAGES.md](PAGES.md) |
| "nouveau composant X" | → [COMPONENTS.md](COMPONENTS.md) |
| "server action pour X" | → [ACTIONS.md](ACTIONS.md) |
| "service pour X" | → [SERVICES.md](SERVICES.md) |
| "hook pour X" | → [HOOKS.md](HOOKS.md) |
| "formulaire / form" | → [FORMS.md](FORMS.md) |
| "schema / validation" | → [SCHEMAS.md](SCHEMAS.md) |
| "style / tailwind" | → [TAILWIND.md](TAILWIND.md) |

## Architecture globale

```
src/
├── app/                    # Next.js App Router
│   ├── (main)/            # Route groups
│   │   ├── @slot/         # Parallel routes
│   │   ├── page.tsx       # Server Component (data fetching)
│   │   ├── layout.tsx     # Layouts
│   │   ├── loading.tsx    # Loading UI
│   │   └── error.tsx      # Error boundary
│   └── api/               # API routes (si nécessaire)
├── features/              # Feature modules (domain-driven)
│   └── [name]/
│       ├── index.tsx      # Main component
│       ├── components/    # Feature-specific components
│       ├── actions/       # Server actions
│       ├── hooks/         # Feature-specific hooks
│       ├── types/         # Types
│       └── schemas/       # Zod schemas
├── components/            # Shared/generic components
│   └── [name]/
│       ├── index.tsx
│       └── [name].types.ts
├── lib/
│   ├── services/          # API service classes
│   ├── actions/           # Shared server actions
│   ├── hooks/             # Generic hooks
│   ├── stores/            # Zustand stores
│   ├── utils/             # Utilities
│   └── core/              # Constants, enums
└── types/                 # Global type definitions
```

## Règles fondamentales

### TypeScript

```tsx
// ✅ TOUJOURS type (pas interface)
type Props = {
  name: string;
};

// ✅ TOUJOURS import type pour les types
import type { User } from "@/types";

// ✅ Props jamais exporté
type Props = { /* ... */ };
export default function Component({ }: Props) { }
```

### Composants

```tsx
// ✅ Fonction déclarée (pas arrow function pour les composants)
export default function MyComponent({ name }: Props) {
  return <div data-testid="my-component">{name}</div>;
}

// ❌ Arrow function
const MyComponent = ({ name }: Props) => { };
```

### Imports

```tsx
// ✅ Alias @/ pour tous les imports internes
import Button from "@/components/button";
import type { User } from "@/lib/services/user/user.types";

// ❌ Imports relatifs profonds
import Button from "../../../components/button";
```

### Testing

```tsx
// ✅ TOUJOURS data-testid sur les éléments interactifs
<button data-testid="submit-button">Submit</button>
<div data-testid="user-profile">...</div>

// ✅ Format: kebab-case descriptif
data-testid="player-stats-table"
data-testid="login-form"
```

## Workflow de création

### 1. Feature complète

```bash
# Structure créée
features/my-feature/
├── index.tsx                    # "use client" + main component
├── components/
│   └── sub-component/
│       └── index.tsx
├── actions/
│   └── get-data.action.ts       # "use server"
├── hooks/
│   └── use-my-feature.hook.ts
└── types/
    └── my-feature.types.ts
```

### 2. Page Next.js

```bash
# Structure créée
app/(main)/my-page/
├── page.tsx          # Server Component (async)
├── loading.tsx       # Suspense fallback
├── error.tsx         # Error boundary
└── @modal/           # Parallel route (si nécessaire)
    ├── page.tsx
    └── default.tsx
```

### 3. Composant générique

```bash
# Structure créée
components/my-component/
├── index.tsx
├── my-component.types.ts    # Si types complexes
└── my-component.config.ts   # Si variants/config
```

## Références rapides

| Domaine | Fichier |
|---------|---------|
| Features | [FEATURES.md](FEATURES.md) |
| Pages | [PAGES.md](PAGES.md) |
| Components | [COMPONENTS.md](COMPONENTS.md) |
| Server Actions | [ACTIONS.md](ACTIONS.md) |
| Services | [SERVICES.md](SERVICES.md) |
| Hooks | [HOOKS.md](HOOKS.md) |
| Forms | [FORMS.md](FORMS.md) |
| Schemas | [SCHEMAS.md](SCHEMAS.md) |
| Tailwind | [TAILWIND.md](TAILWIND.md) |
| **Clean Code** | [../clean-code/SKILL.md](../clean-code/SKILL.md) |

> **Note**: Le skill [clean-code](../clean-code/SKILL.md) contient les bonnes pratiques générales TypeScript (naming, fonctions, classes, SOLID, error handling, testing) applicables à tout le code React/Next.js.
