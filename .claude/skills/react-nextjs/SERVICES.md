# Services Convention

## Principe

Les Services encapsulent TOUTE la logique d'appel API. Ils sont appelés par les Server Actions ou directement dans les Server Components.

```
Server Component/Action → Service → ApiConfig → API externe
```

## Emplacement

```
lib/services/
└── [service-name]/
    ├── [service-name].service.ts    # Classe principale
    ├── [service-name].types.ts      # Types
    └── mocks/                       # Mocks pour tests (optionnel)
        └── [endpoint].json
```

## Structure de base

```tsx
// lib/services/player/player.service.ts
import ApiConfig from "../api.config";
import type {
  Player,
  PlayerListResponse,
  CreatePlayerPayload,
  UpdatePlayerPayload,
} from "./player.types";

export default class PlayerService {
  // ✅ Toutes les méthodes sont static
  static async getAll(): Promise<PlayerListResponse> {
    const url = "players";

    return ApiConfig.poona(url, {
      next: { tags: ["players"] },
    }).then((res) => res.json());
  }

  static async getById(playerId: number): Promise<Player | null> {
    try {
      const url = `players/${playerId}`;

      return ApiConfig.poona(url, {
        next: { tags: [`player-${playerId}`] },
      }).then((res) => res.json());
    } catch {
      return null;
    }
  }

  static async create(payload: CreatePlayerPayload): Promise<Player> {
    const url = "players";

    return ApiConfig.poona(url, {
      method: "POST",
      body: JSON.stringify(payload),
    }).then((res) => res.json());
  }

  static async update(
    playerId: number,
    payload: UpdatePlayerPayload
  ): Promise<Player> {
    const url = `players/${playerId}`;

    return ApiConfig.poona(url, {
      method: "PUT",
      body: JSON.stringify(payload),
    }).then((res) => res.json());
  }

  static async delete(playerId: number): Promise<void> {
    const url = `players/${playerId}`;

    await ApiConfig.poona(url, {
      method: "DELETE",
    });
  }
}
```

## Fichier Types

```tsx
// lib/services/player/player.types.ts

export type Player = {
  id: number;
  name: string;
  email: string;
  level: PlayerLevel;
  createdAt: string;
  updatedAt: string;
};

export type PlayerLevel = "beginner" | "intermediate" | "advanced" | "pro";

export type PlayerListResponse = {
  items: Player[];
  total: number;
  page: number;
  pageSize: number;
};

export type CreatePlayerPayload = {
  name: string;
  email: string;
  level: PlayerLevel;
};

export type UpdatePlayerPayload = Partial<CreatePlayerPayload>;

export type PlayerStatsResponse = {
  gamesPlayed: number;
  wins: number;
  losses: number;
  winRate: number;
};
```

## Service avec cache tags

```tsx
// lib/services/player/player.service.ts
import { revalidateTag } from "next/cache";
import ApiConfig from "../api.config";

export default class PlayerService {
  // ✅ Lecture avec cache tag
  static async getById(playerId: number): Promise<Player | null> {
    return ApiConfig.poona(`players/${playerId}`, {
      next: { tags: [`player-${playerId}`] },
    }).then((res) => res.json());
  }

  // ✅ Mutation avec revalidation
  static async update(playerId: number, payload: UpdatePlayerPayload) {
    const result = await ApiConfig.poona(`players/${playerId}`, {
      method: "PUT",
      body: JSON.stringify(payload),
    }).then((res) => res.json());

    // Invalide le cache
    revalidateTag(`player-${playerId}`);
    revalidateTag("players");

    return result;
  }
}
```

## Service avec gestion d'erreur

```tsx
export default class PlayerService {
  static async getById(playerId: number): Promise<Player | null> {
    try {
      const url = `players/${playerId}`;

      const response = await ApiConfig.poona(url, {
        next: { tags: [`player-${playerId}`] },
      });

      if (!response.ok) {
        return null;
      }

      return response.json();
    } catch (error) {
      console.error("[PlayerService][getById]", error);
      return null;
    }
  }

  static async getStats(playerId: number): Promise<PlayerStatsResponse | null> {
    try {
      const url = `players/${playerId}/stats`;

      const response = await ApiConfig.poona(url);

      if (!response.ok) {
        return null;
      }

      return response.json();
    } catch {
      return null;
    }
  }
}
```

## Service avec méthodes dépendantes

```tsx
export default class PlayerService {
  // Méthode publique qui utilise d'autres méthodes
  static async getFullProfile(playerId: number) {
    const [player, stats, history] = await Promise.all([
      PlayerService.getById(playerId),
      PlayerService.getStats(playerId),
      PlayerService.getHistory(playerId),
    ]);

    if (!player) return null;

    return {
      ...player,
      stats,
      history,
    };
  }

  static async getById(playerId: number) { /* ... */ }
  static async getStats(playerId: number) { /* ... */ }
  static async getHistory(playerId: number) { /* ... */ }
}
```

## ApiConfig (référence)

```tsx
// lib/services/api.config.ts
export default class ApiConfig {
  private static readonly API_URL = process.env.API_URL!;
  private static readonly API_KEY = process.env.API_KEY!;

  static async poona(
    url: string,
    options: RequestInit & { next?: NextFetchRequestConfig } = {}
  ) {
    const headers = {
      "Content-Type": "application/json",
      Authorization: `Bearer ${ApiConfig.API_KEY}`,
      ...options.headers,
    };

    // Auto-enable cache si tags fournis
    const fetchOptions: RequestInit = {
      ...options,
      headers,
    };

    if (options.next?.tags && !options.cache) {
      fetchOptions.cache = "force-cache";
    }

    const response = await fetch(`${ApiConfig.API_URL}/${url}`, fetchOptions);

    if (!response.ok) {
      const error = await response.json().catch(() => ({}));
      throw new Error(error.message || "API Error");
    }

    return response;
  }
}
```

## Règles

### Export

```tsx
// ✅ TOUJOURS export default class
export default class PlayerService { }

// ❌ Pas de named export
export class PlayerService { }
```

### Méthodes static

```tsx
// ✅ TOUJOURS static
static async getById(id: number) { }

// ❌ Pas de méthodes d'instance
async getById(id: number) { }
```

### Types

```tsx
// ✅ Import type séparé
import type { Player, CreatePlayerPayload } from "./player.types";

// ✅ Return types explicites
static async getById(id: number): Promise<Player | null> { }
```

### Naming

| Type | Convention | Exemple |
|------|------------|---------|
| Dossier | kebab-case | `player-stats/` |
| Fichier service | [name].service.ts | `player.service.ts` |
| Fichier types | [name].types.ts | `player.types.ts` |
| Classe | PascalCase + Service | `PlayerService` |
| Méthodes | camelCase (verbe) | `getById`, `create`, `update` |
