# Components Convention (Génériques)

## Emplacement

```
components/                    # UNIQUEMENT composants génériques/réutilisables
└── [component-name]/
    ├── index.tsx             # Composant principal
    ├── [name].types.ts       # Types (si complexes)
    └── [name].config.ts      # Config/variants (si nécessaire)
```

## Structure de base

```tsx
// components/button/index.tsx
"use client";

import { twMerge } from "tailwind-merge";
import { BUTTON_VARIANTS } from "./button.config";

type Props = BaseButtonProps & {
  variant?: Variant | "default";
  rounded?: boolean;
};

export default function Button({
  className,
  variant = "primary",
  rounded = false,
  children,
  ...props
}: Props) {
  return (
    <button
      data-testid="button"
      data-variant={variant}
      className={twMerge(
        "inline-block transition-all duration-300 px-5 py-2 cursor-pointer",
        BUTTON_VARIANTS[variant],
        props.disabled && "bg-slate-300 pointer-events-none",
        rounded && "rounded-lg",
        className
      )}
      aria-disabled={props.disabled}
      {...props}
    >
      {children}
    </button>
  );
}
```

## Fichier Config (variants)

```tsx
// components/button/button.config.ts
export const BUTTON_VARIANTS: Record<Variant | "default", string> = {
  primary: "bg-primary hover:bg-secondary text-white border-2 border-primary",
  secondary: "bg-secondary hover:bg-primary text-white border-2 border-secondary",
  info: "bg-info hover:bg-white text-white hover:text-black",
  danger: "bg-danger hover:bg-white text-white hover:text-secondary",
  warning: "bg-warning hover:bg-white text-white hover:text-black",
  default: "bg-white hover:bg-primary text-secondary hover:text-white border-2",
};
```

## Fichier Types (si complexe)

```tsx
// components/table/table.types.ts
export type Column<T> = {
  label: string;
  accessor?: keyof T;
  render?: (row: T, index: number) => React.ReactNode;
  className?: string;
};

export type TableProps<T> = {
  data: T[];
  columns: Column<T>[];
  emptyMessage?: string;
  isStriped?: boolean;
};
```

## Composant avec types externes

```tsx
// components/table/index.tsx
"use client";

import { twMerge } from "tailwind-merge";
import type { TableProps } from "./table.types";

export default function Table<T>({
  data,
  columns,
  emptyMessage = "Aucune donnée",
  isStriped = true,
}: TableProps<T>) {
  if (!data.length) {
    return <p data-testid="table-empty">{emptyMessage}</p>;
  }

  return (
    <table data-testid="table" className="w-full">
      <thead>
        <tr>
          {columns.map((col, i) => (
            <th key={i} className={col.className}>
              {col.label}
            </th>
          ))}
        </tr>
      </thead>
      <tbody>
        {data.map((row, rowIndex) => (
          <tr
            key={rowIndex}
            className={twMerge(isStriped && rowIndex % 2 === 0 && "bg-gray-50")}
          >
            {columns.map((col, colIndex) => (
              <td key={colIndex} className={col.className}>
                {col.render
                  ? col.render(row, rowIndex)
                  : col.accessor
                    ? String(row[col.accessor])
                    : null}
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

## Composant Input avec forwardRef

```tsx
// components/input/index.tsx
"use client";

import { forwardRef } from "react";
import { twMerge } from "tailwind-merge";

type Props = BaseInputProps & {
  label?: string;
  error?: string;
};

const Input = forwardRef<HTMLInputElement, Props>(
  ({ className, label, error, id, ...props }, ref) => {
    const inputId = id || props.name;

    return (
      <div className="flex flex-col gap-1">
        {label && (
          <label htmlFor={inputId} className="text-sm font-medium">
            {label}
          </label>
        )}
        <input
          ref={ref}
          id={inputId}
          data-testid={`input-${inputId}`}
          className={twMerge(
            "px-4 py-2 border rounded-lg focus:outline-none focus:ring-2",
            error && "border-danger focus:ring-danger",
            className
          )}
          aria-invalid={!!error}
          aria-describedby={error ? `${inputId}-error` : undefined}
          {...props}
        />
        {error && (
          <span id={`${inputId}-error`} className="text-sm text-danger">
            {error}
          </span>
        )}
      </div>
    );
  }
);

Input.displayName = "Input";

export default Input;
```

## Règles

### SRP : Toujours des sous-composants

```tsx
// ❌ INCORRECT - Logique complexe dans un seul composant
export default function UserList({ users }: Props) {
  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>
          <img src={user.avatar} />
          <div>
            <span>{user.name}</span>
            <span>{user.email}</span>
            <button onClick={() => handleDelete(user.id)}>Delete</button>
          </div>
        </li>
      ))}
    </ul>
  );
}

// ✅ CORRECT - Sous-composant pour chaque item
export default function UserList({ users }: Props) {
  return (
    <ul data-testid="user-list">
      {users.map((user) => (
        <UserListItem key={user.id} user={user} />
      ))}
    </ul>
  );
}
```

```tsx
// components/user-list/components/user-list-item/index.tsx
type Props = {
  user: User;
};

export default function UserListItem({ user }: Props) {
  return (
    <li data-testid={`user-item-${user.id}`}>
      <UserAvatar src={user.avatar} alt={user.name} />
      <UserInfo name={user.name} email={user.email} />
      <DeleteButton userId={user.id} />
    </li>
  );
}
```

**Règle absolue** : Si `.map()` → sous-composant obligatoire dans le map.

### data-testid obligatoire

```tsx
// ✅ TOUJOURS ajouter data-testid
<button data-testid="button">Click</button>
<div data-testid="card">...</div>
<input data-testid="input-email" />

// ✅ Avec variante si pertinent
<button data-testid="button" data-variant={variant}>

// ✅ Dynamique si nécessaire
<li data-testid={`list-item-${index}`}>
```

### Pas de HTML brut

```tsx
// ❌ JAMAIS de HTML brut
<button className="...">Click</button>
<a href="/path">Link</a>
<input type="text" />

// ✅ TOUJOURS utiliser les composants partagés
<Button variant="primary">Click</Button>
<AppLink href="/path">Link</AppLink>
<Input name="email" label="Email" />
```

### Spread props

```tsx
// ✅ Toujours spreader les props restantes
export default function Button({ variant, className, ...props }: Props) {
  return <button {...props} className={twMerge(base, className)} />;
}
```

### Composition className

```tsx
// ✅ twMerge avec className en dernier
className={twMerge(
  "base-classes",           // Classes de base
  VARIANTS[variant],        // Variante
  condition && "active",    // Conditionnelles
  className                 // Override externe (TOUJOURS en dernier)
)}
```

### Accessibilité

```tsx
// ✅ ARIA attributes
aria-disabled={props.disabled}
aria-invalid={!!error}
aria-describedby={error ? `${id}-error` : undefined}
aria-label="Description"
aria-expanded={isOpen}

// ✅ Labels liés
<label htmlFor={inputId}>Label</label>
<input id={inputId} />
```

### Props naming

```tsx
// ✅ Props interne (non exporté) → toujours "Props"
type Props = {
  name: string;
};

export default function Button({ name }: Props) { }

// ✅ Props exporté → [ComponentName]Props
export type UserMenuProps = {
  user: User;
  onLogout: () => void;
};

export default function UserMenu({ user, onLogout }: UserMenuProps) { }
```

| Cas | Naming | Exemple |
|-----|--------|---------|
| Props interne | `Props` | `type Props = { }` |
| Props exporté | `[Component]Props` | `export type UserMenuProps = { }` |

### Naming fichiers

| Type | Convention | Exemple |
|------|------------|---------|
| Dossier | kebab-case | `date-picker/` |
| Fichier principal | index.tsx | `index.tsx` |
| Fichier types | [name].types.ts | `date-picker.types.ts` |
| Fichier config | [name].config.ts | `date-picker.config.ts` |
| Composant | PascalCase | `DatePicker` |
