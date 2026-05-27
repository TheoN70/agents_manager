# Tailwind CSS Convention

## Règle d'or

**TOUJOURS utiliser `twMerge()` pour combiner les classes.**

## twMerge obligatoire

```tsx
import { twMerge } from "tailwind-merge";

// ✅ CORRECT - twMerge avec pattern standard
className={twMerge(
  "base-classes",
  condition && "conditional-classes",
  className
)}

// ❌ INCORRECT - Template literals
className={`base ${condition ? "active" : ""}`}

// ❌ INCORRECT - Concaténation
className={"base " + (condition ? "active" : "")}

// ❌ INCORRECT - Array join
className={["base", condition && "active"].filter(Boolean).join(" ")}
```

## Pattern standard

```tsx
// ✅ Pattern complet
className={twMerge(
  "flex items-center gap-4 p-4",           // Classes de base
  VARIANTS[variant],                        // Variante (si applicable)
  size === "lg" && "p-6 text-lg",          // Conditionnelle
  isActive && "bg-primary text-white",     // État
  isDisabled && "opacity-50 cursor-not-allowed",
  className                                 // Override externe (TOUJOURS en dernier)
)}
```

## Valeurs Tailwind (pas de valeurs arbitraires)

```tsx
// ✅ CORRECT - Utiliser les valeurs Tailwind définies
<div className="text-xl p-4 mt-8 rounded-lg" />
<div className="w-64 h-32 gap-6" />

// ❌ INCORRECT - Valeurs arbitraires
<div className="text-[20px] p-[16px] mt-[32px] rounded-[8px]" />
<div className="w-[256px] h-[128px] gap-[24px]" />
```

### Référence des valeurs courantes

| Arbitraire | → | Tailwind |
|------------|---|----------|
| `text-[12px]` | → | `text-xs` |
| `text-[14px]` | → | `text-sm` |
| `text-[16px]` | → | `text-base` |
| `text-[18px]` | → | `text-lg` |
| `text-[20px]` | → | `text-xl` |
| `text-[24px]` | → | `text-2xl` |
| `p-[4px]` | → | `p-1` |
| `p-[8px]` | → | `p-2` |
| `p-[16px]` | → | `p-4` |
| `p-[24px]` | → | `p-6` |
| `p-[32px]` | → | `p-8` |
| `gap-[8px]` | → | `gap-2` |
| `gap-[16px]` | → | `gap-4` |
| `rounded-[4px]` | → | `rounded` |
| `rounded-[8px]` | → | `rounded-lg` |
| `rounded-[16px]` | → | `rounded-2xl` |

### Quand utiliser les valeurs arbitraires

```tsx
// ✅ OK - Valeur vraiment spécifique (design system)
<div className="w-[347px]" />  // Largeur exacte du design

// ✅ OK - Couleur custom du projet
<div className="bg-[#FF5733]" />  // Couleur non définie dans config

// ❌ ÉVITER - Valeur proche d'une valeur Tailwind
<div className="p-[15px]" />  // Utiliser p-4 (16px)
<div className="text-[13px]" />  // Utiliser text-sm (14px)
```

## Shorthand size

```tsx
// ✅ CORRECT - Utiliser size-X pour width + height égaux
<div className="size-10" />      // = w-10 h-10
<div className="size-full" />    // = w-full h-full
<div className="size-8" />       // = w-8 h-8

// ❌ INCORRECT - w-X h-X séparés quand égaux
<div className="w-10 h-10" />
<div className="w-full h-full" />
```

## Composant avec variants

```tsx
// components/button/button.config.ts
export const BUTTON_VARIANTS = {
  primary: "bg-primary hover:bg-secondary text-white",
  secondary: "bg-secondary hover:bg-primary text-white",
  danger: "bg-danger hover:bg-red-700 text-white",
  ghost: "bg-transparent hover:bg-gray-100 text-gray-700",
} as const;

export const BUTTON_SIZES = {
  sm: "px-3 py-1.5 text-sm",
  md: "px-4 py-2 text-base",
  lg: "px-6 py-3 text-lg",
} as const;
```

```tsx
// components/button/index.tsx
import { twMerge } from "tailwind-merge";
import { BUTTON_VARIANTS, BUTTON_SIZES } from "./button.config";

type Props = {
  variant?: keyof typeof BUTTON_VARIANTS;
  size?: keyof typeof BUTTON_SIZES;
  className?: string;
  children: React.ReactNode;
};

export default function Button({
  variant = "primary",
  size = "md",
  className,
  children,
  ...props
}: Props) {
  return (
    <button
      className={twMerge(
        "inline-flex items-center justify-center rounded-lg font-medium transition-colors",
        BUTTON_VARIANTS[variant],
        BUTTON_SIZES[size],
        props.disabled && "opacity-50 cursor-not-allowed",
        className
      )}
      {...props}
    >
      {children}
    </button>
  );
}
```

## Conditional classes

```tsx
// ✅ CORRECT - Boolean condition avec &&
className={twMerge(
  "p-4",
  isActive && "bg-primary",
  isDisabled && "opacity-50"
)}

// ✅ CORRECT - Ternaire pour alternatives
className={twMerge(
  "p-4",
  isActive ? "bg-primary" : "bg-gray-100"
)}

// ❌ INCORRECT - Conditions dans template literal
className={`p-4 ${isActive ? "bg-primary" : ""}`}
```

## Responsive design

```tsx
// ✅ Mobile-first approach
className={twMerge(
  "flex flex-col gap-4",           // Mobile (default)
  "md:flex-row md:gap-6",          // Tablet
  "lg:gap-8"                       // Desktop
)}

// ✅ Grid responsive
className={twMerge(
  "grid grid-cols-1 gap-4",
  "sm:grid-cols-2",
  "lg:grid-cols-3",
  "xl:grid-cols-4"
)}
```

## États et pseudo-classes

```tsx
// ✅ États groupés logiquement
className={twMerge(
  "bg-white text-gray-900",
  "hover:bg-gray-50",
  "focus:ring-2 focus:ring-primary focus:outline-none",
  "active:bg-gray-100",
  "disabled:opacity-50 disabled:cursor-not-allowed"
)}

// ✅ Groupe hover/focus sur parent
<div className="group">
  <span className="group-hover:text-primary">Text</span>
</div>
```

## Dark mode (si utilisé)

```tsx
// ✅ Pattern dark mode
className={twMerge(
  "bg-white text-gray-900",
  "dark:bg-gray-900 dark:text-white"
)}
```

## Spacing cohérent

```tsx
// ✅ Utiliser gap plutôt que margin sur les enfants
<div className="flex flex-col gap-4">
  <div>Item 1</div>
  <div>Item 2</div>
</div>

// ❌ Margin sur chaque enfant
<div className="flex flex-col">
  <div className="mb-4">Item 1</div>
  <div className="mb-4">Item 2</div>
</div>
```

## Animation et transition

```tsx
// ✅ Transitions explicites
className="transition-colors duration-200"
className="transition-all duration-300 ease-in-out"

// ✅ Animation prédéfinie
className="animate-spin"
className="animate-pulse"
```

## Anti-patterns à éviter

```tsx
// ❌ Styles inline
style={{ padding: "16px" }}

// ❌ Classes en double (twMerge résout ça, mais éviter)
className="p-4 p-8"  // twMerge garde p-8

// ❌ !important via arbitrary
className="!p-4"  // Éviter, repenser l'architecture

// ❌ Trop de classes (extraire dans config)
className="bg-blue-500 hover:bg-blue-600 active:bg-blue-700 focus:ring-2..."
// → Créer un variant dans config
```
