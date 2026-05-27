# Zod Schema Conventions

Guide pour la création et l'utilisation des schemas Zod.

## Structure des fichiers

```
features/
└── my-feature/
    └── schemas/
        ├── user.schema.ts           # Schema principal
        ├── address.schema.ts         # Schema secondaire (si besoin)
        └── step-01.schema.ts         # Schema composite (si besoin)
```

## Schema de base

```typescript
import { z } from "zod";

export const userSchema = z.object({
  firstName: z
    .string({ required_error: "Le prénom est obligatoire" })
    .min(1, "Le prénom est obligatoire"),
  lastName: z
    .string({ required_error: "Le nom est obligatoire" })
    .min(1, "Le nom est obligatoire"),
  email: z
    .string({ required_error: "L'email est obligatoire" })
    .email("L'email n'est pas valide"),
  age: z
    .number({ required_error: "L'âge est obligatoire" })
    .min(1, "L'âge doit être supérieur à 0")
    .max(120, "L'âge doit être inférieur à 120"),
  phone: z.string().optional(),
});

// ✅ Type inféré du schema
export type UserFormData = z.infer<typeof userSchema>;
```

## Validations courantes

### Strings

```typescript
z.string()                              // Requis
z.string().optional()                   // Optionnel
z.string().min(1, "Champ obligatoire")  // Non vide
z.string().max(100, "Trop long")        // Max length
z.string().email("Email invalide")      // Email
z.string().url("URL invalide")          // URL
z.string().regex(/pattern/, "Invalide") // Regex
z.string().trim()                       // Trim whitespace
```

### Numbers

```typescript
z.number()                              // Requis
z.number().positive("Doit être positif")
z.number().min(0, "Min 0")
z.number().max(100, "Max 100")
z.number().int("Doit être entier")
z.coerce.number()                       // Coerce string → number
```

### Booleans

```typescript
z.boolean()                             // Requis
z.boolean().default(false)              // Valeur par défaut
z.literal(true)                         // Doit être true
```

### Arrays

```typescript
z.array(z.string())                     // Array de strings
z.array(z.string()).min(1, "Au moins 1")
z.array(z.string()).optional()          // Optionnel
z.array(z.string()).default([])         // Défaut vide
```

### Enums

```typescript
z.enum(["option1", "option2", "option3"])
z.enum(["M", "F"] as const)

// Avec constantes
const GENDERS = ["M", "F"] as const;
z.enum(GENDERS)
```

## Validation custom avec refine

```typescript
import { isValidBirthDate } from "@/lib/utils/date.utils";

export const personSchema = z.object({
  birthDate: z
    .string({ required_error: "Date de naissance obligatoire" })
    .min(1, "Date de naissance obligatoire")
    .refine(isValidBirthDate, "Date invalide (entre 3 et 100 ans)"),

  phone: z
    .string()
    .refine(
      (val) => !val || PHONE_REGEX.test(val.replace(/\s/g, "")),
      "Numéro de téléphone invalide"
    ),
});
```

## Validation cross-field avec superRefine

```typescript
export const registrationSchema = z
  .object({
    password: z.string().min(8, "8 caractères minimum"),
    confirmPassword: z.string(),
    birthDate: z.string(),
    gender: z.string(),
    maidenName: z.string().optional(),
  })
  .superRefine((data, ctx) => {
    // Validation password match
    if (data.password !== data.confirmPassword) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: "Les mots de passe ne correspondent pas",
        path: ["confirmPassword"],
      });
    }

    // Validation conditionnelle: maidenName requis pour femmes adultes
    const isFemale = data.gender === "F";
    const isAdult = !isMinor(data.birthDate);

    if (isFemale && isAdult && !data.maidenName) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: "Le nom de naissance est obligatoire",
        path: ["maidenName"],
      });
    }
  });
```

## Schema composite

Combiner plusieurs schemas pour un formulaire multi-sections:

```typescript
// schemas/step-01.schema.ts
import { z } from "zod";
import { informationsSchema } from "./informations.schema";
import { coordsSchema } from "./coords.schema";

export const step01Schema = z.object({
  informations: informationsSchema,
  coords: coordsSchema,
});

export type Step01FormData = z.infer<typeof step01Schema>;
```

## Fonction de validation

```typescript
export function validateStep01(data: unknown) {
  const result = step01Schema.safeParse(data);

  if (result.success) {
    return { success: true as const, data: result.data };
  }

  const errors = result.error.flatten().fieldErrors;
  return { success: false as const, errors };
}
```

## Utilitaire pour récupérer la première erreur

```typescript
// lib/utils/zod.utils.ts
import type { SafeParseReturnType } from "zod";

export function getFirstZodError(
  ...results: SafeParseReturnType<unknown, unknown>[]
): string | null {
  for (const result of results) {
    if (!result.success) {
      return result.error.errors[0]?.message ?? null;
    }
  }
  return null;
}

// Usage dans un hook de validation
function handleNext() {
  const store = useWizardStore.getState();

  const error = getFirstZodError(
    informationsSchema.safeParse(store.informations),
    coordsSchema.safeParse(store.coords)
  );

  if (error) {
    openErrorSnackBar(error);
    return;
  }

  nextStep();
}
```

## Patterns de regex courants

```typescript
// lib/core/validation.constants.ts
export const PHONE_REGEX = /^(?:(?:\+|00)33|0)\s*[1-9](?:[\s.-]*\d{2}){4}$/;
export const POSTAL_CODE_FRANCE_REGEX = /^[0-9]{5}$/;
export const LICENCE_REGEX = /^[0-9]{8}$/;

// Usage dans schema
const phoneSchema = z
  .string()
  .refine(
    (val) => !val || PHONE_REGEX.test(val.replace(/\s/g, "")),
    "Format invalide"
  );
```

## Schema pour Server Action

```typescript
// features/contact/schemas/contact.schema.ts
export const contactSchema = z.object({
  name: z.string().min(1, "Nom requis"),
  email: z.string().email("Email invalide"),
  message: z.string().min(10, "Message trop court"),
});

export type ContactFormData = z.infer<typeof contactSchema>;

// features/contact/actions/send-contact.action.ts
"use server";

import { contactSchema } from "../schemas/contact.schema";

export async function sendContactAction(data: unknown) {
  const result = contactSchema.safeParse(data);

  if (!result.success) {
    return {
      success: false as const,
      error: result.error.errors[0]?.message
    };
  }

  // Utiliser result.data (typé)
  await sendEmail(result.data);

  return { success: true as const };
}
```

## Conventions de nommage

| Élément | Pattern | Exemple |
|---------|---------|---------|
| Fichier schema | `<domain>.schema.ts` | `user.schema.ts` |
| Schema composite | `<step>.schema.ts` | `step-01.schema.ts` |
| Export schema | `<domain>Schema` | `userSchema` |
| Export type | `<Domain>FormData` | `UserFormData` |
| Fonction validation | `validate<Domain>` | `validateUser` |

## Anti-patterns

```typescript
// ❌ Interface au lieu de z.infer
interface UserFormData {
  name: string;
  email: string;
}
// ✅ Type inféré
type UserFormData = z.infer<typeof userSchema>;

// ❌ Message d'erreur générique
z.string().min(1)
// ✅ Message explicite
z.string().min(1, "Le champ est obligatoire")

// ❌ Validation dans le composant
if (!email.includes("@")) { ... }
// ✅ Validation dans le schema
z.string().email("Email invalide")

// ❌ required_error sans min
z.string({ required_error: "Requis" })
// ✅ Avec min pour les champs texte
z.string({ required_error: "Requis" }).min(1, "Requis")

// ❌ Coerce partout
z.coerce.string()
// ✅ Coerce seulement quand nécessaire (ex: number depuis input)
z.coerce.number()
```

## Intégration avec react-hook-form

Voir [FORMS.md](FORMS.md) pour l'intégration complète avec `zodResolver`.
