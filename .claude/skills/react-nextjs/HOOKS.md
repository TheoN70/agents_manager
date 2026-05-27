# Hooks Convention

## Emplacements

```
# Hook lié à UNE SEULE feature
features/[feature-name]/hooks/
└── [hook-name].hook.ts

# Hook générique (réutilisable partout)
lib/hooks/
└── [hook-name].hook.ts
```

## Hook générique (lib/hooks/)

```tsx
// lib/hooks/use-click-outside.hook.ts
import { useEffect, useRef } from "react";
import type { RefObject } from "react";

export function useClickOutside<T extends HTMLElement>(
  handler: () => void,
  isActive: boolean = true
): RefObject<T | null> {
  const ref = useRef<T>(null);

  useEffect(() => {
    if (!isActive) return;

    const handleClickOutside = (event: MouseEvent) => {
      if (ref.current && !ref.current.contains(event.target as Node)) {
        handler();
      }
    };

    document.addEventListener("mousedown", handleClickOutside);

    return () => {
      document.removeEventListener("mousedown", handleClickOutside);
    };
  }, [handler, isActive]);

  return ref;
}
```

## Hook feature-specific

```tsx
// features/player-search/hooks/use-player-search.hook.ts
import { useState, useCallback } from "react";
import type { Player } from "@/lib/services/player/player.types";

export function usePlayerSearch() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState<Player[]>([]);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const search = useCallback(async (searchQuery: string) => {
    if (!searchQuery.trim()) {
      setResults([]);
      return;
    }

    setIsLoading(true);
    setError(null);

    try {
      const { searchPlayersAction } = await import(
        "../actions/search-players.action"
      );
      const data = await searchPlayersAction(searchQuery);
      setResults(data);
    } catch (err) {
      setError(err instanceof Error ? err : new Error("Erreur de recherche"));
      setResults([]);
    } finally {
      setIsLoading(false);
    }
  }, []);

  const reset = useCallback(() => {
    setQuery("");
    setResults([]);
    setError(null);
  }, []);

  return {
    query,
    setQuery,
    results,
    isLoading,
    error,
    search,
    reset,
  };
}
```

## Hook avec debounce

```tsx
// lib/hooks/use-debounce.hook.ts
import { useState, useEffect } from "react";

export function useDebounce<T>(value: T, delay: number = 300): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => {
      clearTimeout(timer);
    };
  }, [value, delay]);

  return debouncedValue;
}
```

## Hook avec localStorage

```tsx
// lib/hooks/use-local-storage.hook.ts
import { useState, useEffect, useCallback } from "react";

export function useLocalStorage<T>(
  key: string,
  initialValue: T
): [T, (value: T | ((prev: T) => T)) => void, () => void] {
  const [storedValue, setStoredValue] = useState<T>(initialValue);

  // Load from localStorage on mount
  useEffect(() => {
    try {
      const item = window.localStorage.getItem(key);
      if (item) {
        setStoredValue(JSON.parse(item));
      }
    } catch (error) {
      console.error(`Error reading localStorage key "${key}":`, error);
    }
  }, [key]);

  // Save to localStorage
  const setValue = useCallback(
    (value: T | ((prev: T) => T)) => {
      try {
        const valueToStore =
          value instanceof Function ? value(storedValue) : value;
        setStoredValue(valueToStore);
        window.localStorage.setItem(key, JSON.stringify(valueToStore));
      } catch (error) {
        console.error(`Error setting localStorage key "${key}":`, error);
      }
    },
    [key, storedValue]
  );

  // Remove from localStorage
  const removeValue = useCallback(() => {
    try {
      window.localStorage.removeItem(key);
      setStoredValue(initialValue);
    } catch (error) {
      console.error(`Error removing localStorage key "${key}":`, error);
    }
  }, [key, initialValue]);

  return [storedValue, setValue, removeValue];
}
```

## Hook avec media query

```tsx
// lib/hooks/use-media-query.hook.ts
import { useState, useEffect } from "react";

export function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(false);

  useEffect(() => {
    const mediaQuery = window.matchMedia(query);
    setMatches(mediaQuery.matches);

    const handler = (event: MediaQueryListEvent) => {
      setMatches(event.matches);
    };

    mediaQuery.addEventListener("change", handler);

    return () => {
      mediaQuery.removeEventListener("change", handler);
    };
  }, [query]);

  return matches;
}

// Hooks dérivés pratiques
export function useIsMobile(): boolean {
  return useMediaQuery("(max-width: 768px)");
}

export function useIsDesktop(): boolean {
  return useMediaQuery("(min-width: 1024px)");
}
```

## Hook avec intersection observer

```tsx
// lib/hooks/use-intersection-observer.hook.ts
import { useEffect, useRef, useState } from "react";
import type { RefObject } from "react";

type Options = {
  threshold?: number;
  rootMargin?: string;
  triggerOnce?: boolean;
};

export function useIntersectionObserver<T extends HTMLElement>(
  options: Options = {}
): [RefObject<T | null>, boolean] {
  const { threshold = 0, rootMargin = "0px", triggerOnce = false } = options;
  const ref = useRef<T>(null);
  const [isIntersecting, setIsIntersecting] = useState(false);

  useEffect(() => {
    const element = ref.current;
    if (!element) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        const isVisible = entry.isIntersecting;
        setIsIntersecting(isVisible);

        if (isVisible && triggerOnce) {
          observer.unobserve(element);
        }
      },
      { threshold, rootMargin }
    );

    observer.observe(element);

    return () => {
      observer.disconnect();
    };
  }, [threshold, rootMargin, triggerOnce]);

  return [ref, isIntersecting];
}
```

## Règles

### Naming

```tsx
// ✅ Fichier: kebab-case.hook.ts
use-click-outside.hook.ts
use-player-search.hook.ts

// ✅ Fonction: camelCase avec préfixe "use"
export function useClickOutside() { }
export function usePlayerSearch() { }
```

### Export

```tsx
// ✅ Named export (pas default)
export function useMyHook() { }

// ❌ Default export
export default function useMyHook() { }
```

### Types génériques

```tsx
// ✅ Utiliser des génériques quand pertinent
export function useLocalStorage<T>(key: string, initial: T): [T, (v: T) => void] { }

// ✅ Contraintes de type
export function useClickOutside<T extends HTMLElement>(): RefObject<T | null> { }
```

### Return type

```tsx
// ✅ Object pour plusieurs valeurs
return {
  value,
  setValue,
  isLoading,
  error,
};

// ✅ Tuple pour valeurs simples
return [value, setValue] as const;

// ✅ RefObject pour les refs
return ref;
```

### Cleanup

```tsx
// ✅ TOUJOURS cleanup les effects
useEffect(() => {
  const handler = () => { };
  window.addEventListener("resize", handler);

  return () => {
    window.removeEventListener("resize", handler);
  };
}, []);
```

### Dépendances

```tsx
// ✅ useCallback pour les fonctions retournées
const search = useCallback(async (query: string) => {
  // ...
}, [/* deps */]);

// ✅ Dépendances correctes dans useEffect
useEffect(() => {
  // ...
}, [handler, isActive]); // Toutes les deps listées
```

## Décision: Feature vs Générique

| Critère | Feature-specific | Générique (lib/) |
|---------|------------------|------------------|
| Utilisé dans 1 feature | ✅ | ❌ |
| Utilisé dans 2+ features | ❌ | ✅ |
| Contient logique métier | ✅ | ❌ |
| Utilitaire pur (DOM, storage) | ❌ | ✅ |
| Appelle des actions de la feature | ✅ | ❌ |
