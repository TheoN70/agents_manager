# Server Actions Convention

## Principe

Les Server Actions font le pont entre les Client Components et les Services SERVER.

```
Client Component → (dynamic import) → Server Action → Service → API
```

## Emplacement

```
# Actions liées à une feature
features/[feature-name]/actions/
└── [action-name].action.ts

# Actions partagées
lib/actions/
└── [action-name].action.ts
```

## Structure de base

```tsx
// features/player/actions/get-player.action.ts
"use server";

import PlayerService from "@/lib/services/player/player.service";

// ✅ TOUJOURS export function (JAMAIS default)
export async function getPlayerAction(playerId: number) {
  return PlayerService.getById(playerId);
}
```

## Règles d'export

```tsx
// ✅ CORRECT - Named export
export async function getPlayerAction(id: number) { }
export async function updatePlayerAction(id: number, data: UpdateData) { }

// ❌ INCORRECT - Default export
export default async function getPlayerAction(id: number) { }
```

**Pourquoi ?** Import dynamique avec `.then(mod => mod.actionName)`

## Types : réutiliser ceux du Service

```tsx
// ✅ CORRECT - Réutiliser le type du service
import type { UpdatePlayerPayload } from "@/lib/services/player/player.types";

export async function updatePlayerAction(
  playerId: number,
  payload: UpdatePlayerPayload  // Même type que le service
) {
  return PlayerService.update(playerId, payload);
}

// ✅ CORRECT - Pick/Omit si besoin partiel
import type { CreatePlayerPayload } from "@/lib/services/player/player.types";

export async function quickCreatePlayerAction(
  payload: Pick<CreatePlayerPayload, "name" | "email">
) {
  return PlayerService.create({ ...payload, level: "beginner" });
}

// ❌ INCORRECT - Recréer un type similaire
type UpdateData = {  // NON ! Utiliser celui du service
  name: string;
  email: string;
};
```

**Règle** : Les arguments de l'action = les arguments du service (ou Pick/Omit).

## Import dynamique (OBLIGATOIRE)

```tsx
// Dans un Client Component

// ✅ CORRECT - Import dynamique dans useEffect
useEffect(() => {
  import("./actions/get-player.action").then(({ getPlayerAction }) => {
    getPlayerAction(playerId)
      .then(setData)
      .catch(setError);
  });
}, [playerId]);

// ✅ CORRECT - Import dynamique dans handler
const handleSubmit = async () => {
  const { updatePlayerAction } = await import("./actions/update-player.action");
  await updatePlayerAction(playerId, formData);
};

// ❌ INCORRECT - Import statique
import { getPlayerAction } from "./actions/get-player.action";
```

**Pourquoi ?** Maximise le tree-shaking et réduit le bundle client.

## Action avec validation

```tsx
// features/player/actions/update-player.action.ts
"use server";

import PlayerService from "@/lib/services/player/player.service";
import type { UpdatePlayerPayload } from "@/lib/services/player/player.types";

export async function updatePlayerAction(
  playerId: number,
  payload: UpdatePlayerPayload
) {
  // Validation basique
  if (!playerId) {
    throw new Error("Player ID requis");
  }

  if (!payload.name?.trim()) {
    throw new Error("Nom requis");
  }

  return PlayerService.update(playerId, payload);
}
```

## Action avec Zod

```tsx
// features/player/actions/create-player.action.ts
"use server";

import PlayerService from "@/lib/services/player/player.service";
import { createPlayerSchema } from "../schemas/create-player.schema";

export async function createPlayerAction(formData: FormData) {
  const rawData = {
    name: formData.get("name"),
    email: formData.get("email"),
    level: formData.get("level"),
  };

  // Validation Zod
  const result = createPlayerSchema.safeParse(rawData);

  if (!result.success) {
    return {
      success: false,
      errors: result.error.flatten().fieldErrors,
    };
  }

  const player = await PlayerService.create(result.data);

  return {
    success: true,
    data: player,
  };
}
```

## Action sécurisée (avec auth)

```tsx
// lib/actions/secure-server-action.ts
import AuthService from "@/lib/services/auth/auth.service";

export function secureServerAction<TArgs extends unknown[], TReturn>(
  action: (...args: TArgs) => Promise<TReturn>
) {
  return async (...args: TArgs): Promise<TReturn> => {
    const isAuthenticated = await AuthService.isAuthenticated();

    if (!isAuthenticated) {
      throw new Error("Vous devez être connecté");
    }

    return action(...args);
  };
}
```

```tsx
// Utilisation
import { secureServerAction } from "@/lib/actions/secure-server-action";

export const updateProfileAction = secureServerAction(
  async (data: ProfileData) => {
    return ProfileService.update(data);
  }
);
```

## Action avec revalidation

```tsx
// features/player/actions/delete-player.action.ts
"use server";

import { revalidatePath } from "next/cache";
import PlayerService from "@/lib/services/player/player.service";

export async function deletePlayerAction(playerId: number) {
  await PlayerService.delete(playerId);

  // Revalide les pages affectées
  revalidatePath("/players");
  revalidatePath(`/player/${playerId}`);

  return { success: true };
}
```

## Pattern complet dans un Component

```tsx
"use client";

import { useState } from "react";
import Button from "@/components/button";
import Loading from "@/components/loading";

type Props = {
  playerId: number;
};

export default function PlayerActions({ playerId }: Props) {
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const handleDelete = async () => {
    setIsLoading(true);
    setError(null);

    try {
      const { deletePlayerAction } = await import(
        "./actions/delete-player.action"
      );
      await deletePlayerAction(playerId);
      // Redirect ou update UI
    } catch (err) {
      setError(err instanceof Error ? err : new Error("Erreur inconnue"));
    } finally {
      setIsLoading(false);
    }
  };

  if (isLoading) return <Loading />;

  return (
    <div data-testid="player-actions">
      <Button
        variant="danger"
        onClick={handleDelete}
        disabled={isLoading}
      >
        Supprimer
      </Button>
      {error && <p className="text-danger">{error.message}</p>}
    </div>
  );
}
```

## Naming

| Type | Convention | Exemple |
|------|------------|---------|
| Fichier | kebab-case.action.ts | `get-player.action.ts` |
| Fonction | camelCase + Action | `getPlayerAction` |
| Params | types explicites | `(playerId: number)` |

## Règles résumées

| ✅ DO | ❌ DON'T |
|-------|----------|
| `"use server"` en haut | Oublier la directive |
| `export function` | `export default` |
| Import dynamique | Import statique |
| Appeler un Service | Appeler l'API directement |
| Valider les inputs | Faire confiance aux données |
| Types explicites | `any` |
