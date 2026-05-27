# React Hook Form Conventions

Guide pour l'utilisation de react-hook-form avec zod dans les projets Next.js.

## Setup standard

```tsx
"use client";

import { useForm, FormProvider, Controller } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { mySchema, type MyFormData } from "./schemas/my.schema";

export default function MyForm() {
  const methods = useForm<MyFormData>({
    resolver: zodResolver(mySchema),
    defaultValues: {
      name: "",
      email: "",
    },
  });

  const {
    register,
    control,
    handleSubmit,
    watch,
    reset,
    formState: { errors, isSubmitting },
  } = methods;

  async function onSubmit(data: MyFormData) {
    // Action dynamique
    await import("./actions/submit.action")
      .then((module) => module.submitAction(data));
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* fields */}
    </form>
  );
}
```

## Patterns de champs

### 1. Champ simple avec `register`

```tsx
<div>
  <label htmlFor="name">Nom</label>
  <Input
    id="name"
    data-testid="form-name"
    placeholder="Saisissez votre nom"
    error={errors.name}
    {...register("name")}
  />
  {errors.name && <span className="text-danger">{errors.name.message}</span>}
</div>
```

### 2. Champ complexe avec `Controller`

Pour les composants custom (select, radio, phone input, etc.):

```tsx
<Controller
  name="gender"
  control={control}
  render={({ field }) => (
    <RadioGroup
      data-testid="form-gender"
      label="Civilité"
      items={GENDER_ITEMS}
      selectedKey={field.value}
      onChange={field.onChange}
      onBlur={field.onBlur}
      error={errors.gender}
      required
    />
  )}
/>
```

### 3. Select avec Controller

```tsx
<Controller
  name="countryId"
  control={control}
  render={({ field }) => (
    <Select
      id="country"
      data-testid="form-country"
      items={countries}
      value={field.value}
      onChange={field.onChange}
      onBlur={field.onBlur}
      error={errors.countryId}
    />
  )}
/>
```

### 4. Checkbox avec register

```tsx
<Checkbox
  data-testid="form-accept-terms"
  label="J'accepte les conditions"
  {...register("acceptTerms")}
/>
```

## FormProvider pour formulaires complexes

Quand le formulaire a des sous-composants:

```tsx
// Parent
export default function WizardStep() {
  const methods = useForm<StepFormData>({
    resolver: zodResolver(stepSchema),
  });

  return (
    <FormProvider {...methods}>
      <div className="space-y-6">
        <PersonalInfo />   {/* Sous-composant */}
        <AddressInfo />    {/* Sous-composant */}
      </div>
    </FormProvider>
  );
}

// Enfant - utilise useFormContext
import { useFormContext } from "react-hook-form";

export default function PersonalInfo() {
  const { register, formState: { errors } } = useFormContext<StepFormData>();

  return (
    <div>
      <label>Prénom</label>
      <Input {...register("firstName")} error={errors.firstName} />
      {errors.firstName && <span>{errors.firstName.message}</span>}
    </div>
  );
}
```

## Watch et rendu conditionnel

```tsx
const isParaBadminton = watch("isParaBadminton");

return (
  <>
    <Checkbox
      label="Je suis joueur para-badminton"
      {...register("isParaBadminton")}
    />

    {isParaBadminton && <HandicapSelector />}
  </>
);
```

## Synchronisation avec Zustand Store

Pattern pour wizard multi-étapes:

```tsx
export default function StepForm() {
  const setFormData = useWizardStore((state) => state.setFormData);
  const initialData = useRef(useWizardStore.getState().formData);
  const isResettingRef = useRef(false);

  const methods = useForm<FormData>({
    resolver: zodResolver(schema),
    defaultValues: initialData.current,
  });

  const { watch, reset } = methods;

  // Sync store → form au mount
  useEffect(() => {
    isResettingRef.current = true;
    reset(initialData.current);
    setTimeout(() => { isResettingRef.current = false; }, 0);
  }, [reset]);

  // Sync form → store à chaque changement
  useEffect(() => {
    const subscription = watch((value) => {
      if (!isResettingRef.current) {
        setFormData(value as Partial<FormData>);
      }
    });
    return () => subscription.unsubscribe();
  }, [watch, setFormData]);

  // ...
}
```

## Soumission avec Server Action

```tsx
async function onSubmit(data: FormData) {
  try {
    const result = await import("./actions/submit.action")
      .then((module) => module.submitAction(data));

    if (!result.success) {
      openErrorSnackBar(result.error);
      return;
    }

    openSuccessSnackBar("Enregistré avec succès");
    router.push("/next-page");
  } catch {
    openErrorSnackBar("Une erreur est survenue");
  }
}

return (
  <form onSubmit={handleSubmit(onSubmit)}>
    {/* fields */}
    <Button type="submit" disabled={isSubmitting}>
      {isSubmitting ? <Loading /> : "Valider"}
    </Button>
  </form>
);
```

## Composant Input avec état erreur

```tsx
type Props = ComponentProps<"input"> & {
  error?: FieldError;
};

export default function Input({ className, error, ...props }: Props) {
  return (
    <input
      className={twMerge(
        "px-3 py-2 text-sm border-2 border-gray-200 rounded",
        "focus:border-primary focus:outline-none",
        error && "border-danger",
        props.disabled && "bg-gray-100 cursor-not-allowed",
        className
      )}
      {...props}
    />
  );
}
```

## Règles

| Règle | Description |
|-------|-------------|
| `zodResolver` | Toujours utiliser avec le schema zod |
| `Controller` | Pour tout composant custom (pas juste input natif) |
| `FormProvider` | Dès qu'il y a des sous-composants |
| `data-testid` | Sur tous les champs |
| Import dynamique | Pour les server actions dans onSubmit |

## Anti-patterns

```tsx
// ❌ Controller pour un input simple
<Controller
  name="name"
  render={({ field }) => <input {...field} />}
/>
// ✅ Utiliser register
<input {...register("name")} />

// ❌ Accéder au store directement dans le form
const data = useStore.getState().data;
// ✅ Utiliser useRef pour la valeur initiale
const initialData = useRef(useStore.getState().data);

// ❌ Submit sans loading state
<Button type="submit">Valider</Button>
// ✅ Avec isSubmitting
<Button disabled={isSubmitting}>
  {isSubmitting ? <Loading /> : "Valider"}
</Button>
```
