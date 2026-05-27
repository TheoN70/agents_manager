# Pages Convention (Next.js App Router)

## Principe fondamental

**Les pages sont UNIQUEMENT des Server Components qui :**
1. Récupèrent les données (fetch)
2. Passent les données aux composants de features
3. Ne contiennent AUCUNE logique UI

## Structure

```
app/
├── (main)/                    # Route group (layout partagé)
│   ├── layout.tsx            # Layout du groupe
│   ├── page.tsx              # Home page
│   ├── my-page/
│   │   ├── page.tsx          # Server Component
│   │   ├── loading.tsx       # Suspense fallback
│   │   ├── error.tsx         # Error boundary
│   │   └── @modal/           # Parallel route (slot)
│   │       ├── page.tsx
│   │       └── default.tsx   # Fallback (return null)
│   └── [id]/                 # Dynamic route
│       └── page.tsx
└── (auth)/                   # Autre route group
    ├── layout.tsx
    └── connexion/
        └── page.tsx
```

## Page basique

```tsx
// app/(main)/players/page.tsx
import PlayersList from "@/features/players-list";
import PlayersService from "@/lib/services/players/players.service";

// ✅ Force dynamic pour les pages avec data fetching API
export const dynamic = "force-dynamic";

export default async function PlayersPage() {
  const players = await PlayersService.getAll();

  return (
    <main data-testid="players-page">
      <PlayersList players={players} />
    </main>
  );
}
```

## Page avec paramètres

```tsx
// app/(main)/player/[id]/page.tsx
import PlayerProfile from "@/features/player-profile";
import PlayerService from "@/lib/services/player/player.service";
import { notFound } from "next/navigation";

export const dynamic = "force-dynamic";

type Props = {
  params: Promise<{ id: string }>;
};

export default async function PlayerPage({ params }: Props) {
  const { id } = await params;
  const player = await PlayerService.getById(Number(id));

  if (!player) {
    notFound();
  }

  // ✅ CORRECT - Passer uniquement ce qui est nécessaire
  return (
    <main data-testid="player-page">
      <PlayerProfile
        id={player.id}
        fullName={`${player.firstName} ${player.lastName}`}
        avatar={player.avatar}
        level={player.level}
      />
    </main>
  );
}
```

## Props minimalistes (IMPORTANT)

```tsx
// ❌ INCORRECT - Passer tout l'objet
export default async function PlayerPage({ params }: Props) {
  const player = await PlayerService.getById(id);  // 50 propriétés

  return <PlayerProfile player={player} />;  // NON !
}

// ✅ CORRECT - Passer uniquement le nécessaire
export default async function PlayerPage({ params }: Props) {
  const player = await PlayerService.getById(id);

  return (
    <PlayerProfile
      id={player.id}
      fullName={`${player.firstName} ${player.lastName}`}
      avatar={player.avatar}
      stats={{
        gamesPlayed: player.stats.total,
        winRate: player.stats.wins / player.stats.total,
      }}
    />
  );
}
```

**Avantages** :
- Props explicites = contrat clair
- Transformation côté serveur (pas de logique client)
- Moins de data transférée au client
- Composant client plus simple à tester

## Page avec authentification

```tsx
// app/(main)/dashboard/page.tsx
import Dashboard from "@/features/dashboard";
import AuthService from "@/lib/services/auth/auth.service";
import DashboardService from "@/lib/services/dashboard/dashboard.service";
import { redirect } from "next/navigation";

export const dynamic = "force-dynamic";

export default async function DashboardPage() {
  const isAuthenticated = await AuthService.isAuthenticated();

  if (!isAuthenticated) {
    redirect("/connexion");
  }

  const data = await DashboardService.getData();

  return (
    <main data-testid="dashboard-page">
      <Dashboard data={data} />
    </main>
  );
}
```

## Layout avec Parallel Routes

```tsx
// app/(main)/(home)/layout.tsx
import MobileSafeArea from "@/components/mobile-safe-area";
import AuthService from "@/lib/services/auth/auth.service";
import { redirect } from "next/navigation";

export const dynamic = "force-dynamic";

type Props = {
  carousel: React.ReactNode;
  widgets: React.ReactNode;
  news: React.ReactNode;
  children: React.ReactNode;
};

export default async function HomeLayout({
  carousel,
  widgets,
  news,
  children,
}: Props) {
  const isAuthenticated = await AuthService.isAuthenticated();

  if (!isAuthenticated) {
    redirect("/connexion");
  }

  return (
    <main data-testid="home-layout">
      {carousel}
      <MobileSafeArea>
        {widgets}
        {news}
        {children}
      </MobileSafeArea>
    </main>
  );
}
```

## Parallel Route (Slot)

```tsx
// app/(main)/(home)/@carousel/page.tsx
import Carousel from "@/features/home/components/carousel";
import NewsService from "@/lib/services/news/news.service";

export default async function CarouselSlot() {
  const news = await NewsService.getForCarousel();

  if (!news || news.length === 0) {
    return null;
  }

  return <Carousel news={news} />;
}
```

```tsx
// app/(main)/(home)/@carousel/default.tsx
// ✅ TOUJOURS créer default.tsx pour les parallel routes
export default function CarouselDefault() {
  return null;
}
```

## Loading UI

```tsx
// app/(main)/players/loading.tsx
import Loading from "@/components/loading";

export default function PlayersLoading() {
  return (
    <div data-testid="players-loading" className="flex justify-center p-8">
      <Loading />
    </div>
  );
}
```

## Error Boundary

```tsx
// app/(main)/players/error.tsx
"use client";

import ErrorMessage from "@/components/error-message";

type Props = {
  error: Error & { digest?: string };
  reset: () => void;
};

export default function PlayersError({ error, reset }: Props) {
  return (
    <div data-testid="players-error" className="p-8">
      <ErrorMessage
        message="Impossible de charger les joueurs"
        error={error}
      />
      <button onClick={reset} className="mt-4 underline">
        Réessayer
      </button>
    </div>
  );
}
```

## Not Found

```tsx
// app/(main)/player/[id]/not-found.tsx
import Empty from "@/components/empty";
import AppLink from "@/components/link";

export default function PlayerNotFound() {
  return (
    <div data-testid="player-not-found" className="p-8">
      <Empty>Joueur introuvable</Empty>
      <AppLink href="/players" className="mt-4 underline">
        Retour à la liste
      </AppLink>
    </div>
  );
}
```

## Règles

### Ce que fait une page

```tsx
// ✅ OUI
const data = await MyService.getData();     // Fetch data
redirect("/login");                          // Redirect
notFound();                                  // 404
return <MyFeature data={data} />;           // Render feature
```

### Ce que NE fait PAS une page

```tsx
// ❌ NON - Pas de logique UI
const [state, setState] = useState();        // Pas de state
onClick={() => {}}                           // Pas de handlers
<div className="complex-layout">...</div>    // Pas de markup complexe
```

### Metadata

```tsx
// app/(main)/players/page.tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: "Joueurs | MyApp",
  description: "Liste des joueurs",
};

export default async function PlayersPage() { }
```

### Dynamic Metadata

```tsx
// app/(main)/player/[id]/page.tsx
import type { Metadata } from "next";

type Props = {
  params: Promise<{ id: string }>;
};

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { id } = await params;
  const player = await PlayerService.getById(Number(id));

  return {
    title: player ? `${player.name} | MyApp` : "Joueur | MyApp",
  };
}
```
