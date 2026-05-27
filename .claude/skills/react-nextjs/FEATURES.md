# Features Convention

## Structure

```
features/[feature-name]/
├── index.tsx                          # Main component (client)
├── components/                        # Feature-specific components
│   └── [component-name]/
│       └── index.tsx
├── actions/                           # Server actions
│   └── [action-name].action.ts
├── hooks/                             # Feature-specific hooks
│   └── [hook-name].hook.ts
├── types/                             # Type definitions
│   └── [feature-name].types.ts
├── schemas/                           # Zod validation schemas
│   └── [schema-name].schema.ts
└── CLAUDE.md                          # Feature documentation (optionnel)
```

## Main Component (index.tsx)

```tsx
"use client";

import Empty from "@/components/empty";
import ErrorMessage from "@/components/error-message";
import Loading from "@/components/loading";
import type { MyDataType } from "@/lib/services/my-service/my-service.types";
import { useEffect, useState } from "react";

type Props = {
  entityId: number;
  showDetails?: boolean;
};

export default function MyFeature({ entityId, showDetails = false }: Props) {
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  const [data, setData] = useState<MyDataType | null>(null);

  useEffect(() => {
    setIsLoading(true);

    // ✅ Import dynamique du server action
    import("./actions/get-data.action").then(({ getDataAction }) => {
      getDataAction(entityId)
        .then((result) => {
          if (typeof result === "string") throw new Error(result);
          setData(result);
          setIsLoading(false);
        })
        .catch((err) => {
          setError(err);
          setIsLoading(false);
        });
    });
  }, [entityId]);

  if (error) {
    return (
      <ErrorMessage
        message="Impossible de charger les données"
        error={error}
      />
    );
  }

  if (isLoading) return <Loading />;

  const hasResults = data && data.length > 0;

  return (
    <div data-testid="my-feature" className="flex flex-col gap-4">
      {!hasResults && <Empty>Aucune donnée disponible.</Empty>}

      {hasResults && (
        <div data-testid="my-feature-content">
          {/* Contenu */}
        </div>
      )}
    </div>
  );
}
```

## Feature-specific Component

```
features/[feature-name]/components/[component-name]/index.tsx
```

```tsx
"use client";

import { twMerge } from "tailwind-merge";

type Props = {
  title: string;
  isActive?: boolean;
};

export default function FeatureSubComponent({ title, isActive = false }: Props) {
  return (
    <div
      data-testid="feature-sub-component"
      className={twMerge(
        "p-4 rounded-lg",
        isActive && "bg-primary text-white"
      )}
    >
      <h3>{title}</h3>
    </div>
  );
}
```

## Server Action (dans feature)

```
features/[feature-name]/actions/[action-name].action.ts
```

```tsx
"use server";

import MyService from "@/lib/services/my-service/my-service.service";

// ✅ TOUJOURS export function (jamais default)
export async function getDataAction(entityId: number) {
  return MyService.getData(entityId);
}

// ✅ Avec validation
export async function updateDataAction(entityId: number, payload: UpdatePayload) {
  // Validation si nécessaire
  if (!entityId) throw new Error("Entity ID requis");

  return MyService.updateData(entityId, payload);
}
```

## Règles

### Isolation

```tsx
// ✅ Feature importe depuis lib/, components/, types/
import Button from "@/components/button";
import MyService from "@/lib/services/my-service/my-service.service";

// ❌ Feature n'importe JAMAIS depuis une autre feature
import Something from "@/features/other-feature"; // INTERDIT
```

### Import dynamique des actions

```tsx
// ✅ TOUJOURS import dynamique dans useEffect/handlers
useEffect(() => {
  import("./actions/my-action.action").then(({ myAction }) => {
    myAction(id).then(setData);
  });
}, [id]);

// ✅ Dans un handler
const handleSubmit = async () => {
  const { submitAction } = await import("./actions/submit.action");
  await submitAction(formData);
};

// ❌ Import statique en haut du fichier
import { myAction } from "./actions/my-action.action"; // NON
```

### Naming

| Type | Convention | Exemple |
|------|------------|---------|
| Dossier feature | kebab-case | `player-stats` |
| Component | PascalCase | `PlayerStats` |
| Action file | kebab-case.action.ts | `get-stats.action.ts` |
| Action function | camelCase + Action | `getStatsAction` |
| Hook file | kebab-case.hook.ts | `use-stats.hook.ts` |
| Type file | kebab-case.types.ts | `player-stats.types.ts` |

### Props

```tsx
// ✅ Type Props local, non exporté
type Props = {
  playerId: number;
  showHistory?: boolean;
};

// ✅ Destructuring avec defaults
export default function PlayerStats({
  playerId,
  showHistory = false
}: Props) { }
```

### État standard

```tsx
// ✅ Pattern standard pour data fetching
const [isLoading, setIsLoading] = useState(true);
const [error, setError] = useState<Error | null>(null);
const [data, setData] = useState<DataType | null>(null);
```

### SRP : Toujours des sous-composants

```tsx
// ❌ INCORRECT - Tout dans un seul composant
export default function PlayerStats({ stats }: Props) {
  return (
    <div>
      {stats.map((stat) => (
        <div key={stat.id}>
          <span>{stat.label}</span>
          <span>{stat.value}</span>
          <span>{stat.trend > 0 ? "↑" : "↓"}</span>
        </div>
      ))}
    </div>
  );
}

// ✅ CORRECT - Sous-composant dans le map
export default function PlayerStats({ stats }: Props) {
  return (
    <div data-testid="player-stats">
      {stats.map((stat) => (
        <StatCard key={stat.id} stat={stat} />
      ))}
    </div>
  );
}
```

```tsx
// features/player-stats/components/stat-card/index.tsx
type Props = {
  stat: Stat;
};

export default function StatCard({ stat }: Props) {
  return (
    <div data-testid={`stat-${stat.id}`} className="...">
      <span>{stat.label}</span>
      <span>{stat.value}</span>
      <TrendIndicator trend={stat.trend} />
    </div>
  );
}
```

**Règle absolue** : Si `.map()` → sous-composant obligatoire dans le map.
