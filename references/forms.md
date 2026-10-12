# Formularios con React Hook Form

## Contenido

1. Reparto de responsabilidades
2. Schema y tipos compartidos
3. Hook de creación
4. Hook de actualización
5. Componente de formulario
6. Anatomía de un campo
7. Campos con íconos (InputGroup)
8. Otros controles
9. Composición en dialogs
10. Requisitos de shadcn

## Reparto de responsabilidades

| Pieza | Posee | No posee |
| --- | --- | --- |
| `useCreatePatient` | `createForm`, `handleCreate`, mutation, `isCreating`, feedback, invalidación, reset | JSX, estado open/close del dialog |
| `useUpdatePatient` | `updateForm`, `handleUpdate`, mutation, `isUpdating`, defaults desde la entidad, resincronización | JSX, estado open/close del dialog |
| `PatientForm` | campos, errores visuales y botones | mutations, server functions, schema resolver |
| dialog / page | open/close, elegir qué hook usar, pasar labels y callbacks | lógica de formulario |

Mantener create y update en hooks separados: cada workflow crece por su lado (pasos, campos condicionales, permisos) sin contaminar al otro. Ambos exponen el mismo contrato de formulario para que `PatientForm` sirva a los dos.

## Schema y tipos compartidos

```ts
// schemas/patient.schema.ts
export const patientFormSchema = z.object({
  name: z.string().min(1, "Nombre requerido"),
  email: z.string().email("Correo inválido"),
})

export type PatientFormValues = z.input<typeof patientFormSchema>
```

Usar `PatientFormValues` en `createForm` y `updateForm`. Si update necesita el ID, añadirlo en el schema del boundary de la server function, no en el formulario.

## Hook de creación

```ts
// hooks/use-create-patient.ts
export const useCreatePatient = ({ onSuccess }: { onSuccess?: () => void } = {}) => {
  const createForm = useForm<PatientFormValues>({
    resolver: zodResolver(patientFormSchema),
    defaultValues: patientEmptyFormValues,
  })

  const mutation = useMutation({
    mutationFn: createPatient,
    onSuccess: async () => {
      // invalidar la autoridad de datos correcta del proyecto
      createForm.reset()
      onSuccess?.()
    },
  })

  return {
    createForm,
    isCreating: mutation.isPending,
    handleCreate: createForm.handleSubmit((values) =>
      mutation.mutateAsync({ data: values }),
    ),
  }
}
```

## Hook de actualización

```ts
// hooks/use-update-patient.ts
export const useUpdatePatient = ({ patient, onSuccess }: Options) => {
  const updateForm = useForm<PatientFormValues>({
    resolver: zodResolver(patientFormSchema),
    defaultValues: patientToFormValues(patient),
  })

  useEffect(() => {
    updateForm.reset(patientToFormValues(patient))
  }, [patient, updateForm])

  const mutation = useMutation({
    mutationFn: updatePatient,
    onSuccess: async () => {
      // invalidar la autoridad de datos correcta del proyecto
      onSuccess?.()
    },
  })

  return {
    updateForm,
    isUpdating: mutation.isPending,
    handleUpdate: updateForm.handleSubmit((values) =>
      mutation.mutateAsync({ data: { ...values, id: patient.id } }),
    ),
  }
}
```

Tomar el ID de `patient.id` (props/route), nunca de un campo del formulario. Convertir entidad a valores de formulario en `patientToFormValues` (fechas, `null`, enums).

Adaptar las llamadas de mutation y la invalidación a la convención del repositorio (ver la variante del framework).

## Componente de formulario

`PatientForm` es genérico: recibe el formulario ya construido, los textos y las acciones.

```tsx
type PatientFormProps = {
  form: UseFormReturn<PatientFormValues>
  submitLabel: string
  onSubmit: () => void
  onCancel?: () => void
  isPending?: boolean
}

export function PatientForm({ form, submitLabel, onSubmit, onCancel, isPending }: PatientFormProps) {
  return (
    <form onSubmit={onSubmit} noValidate>
      <FieldGroup>
        {/* campos, ver anatomía */}
      </FieldGroup>
      <div className="flex justify-end gap-2">
        {onCancel && (
          <Button type="button" variant="outline" onClick={onCancel} disabled={isPending}>
            Cancelar
          </Button>
        )}
        <Button type="submit" disabled={isPending}>
          {submitLabel}
        </Button>
      </div>
    </form>
  )
}
```

No importar server functions ni hooks de mutation en este componente. Si los textos de los botones deben ser configurables, exponerlos por props (`submitLabel`, `cancelLabel`).

## Anatomía de un campo

Seguir esta estructura para cada campo. No usar `FieldDescription`.

```tsx
<Controller
  name="name"
  control={form.control}
  render={({ field, fieldState }) => (
    <Field data-invalid={fieldState.invalid}>
      <FieldLabel htmlFor={field.name}>Nombre</FieldLabel>
      <Input
        {...field}
        id={field.name}
        aria-invalid={fieldState.invalid}
        placeholder="María Pérez"
        autoComplete="off"
      />
      {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
    </Field>
  )}
/>
```

Reglas:

- `Field` lleva `data-invalid={fieldState.invalid}`.
- `FieldLabel` usa `htmlFor={field.name}` y el control usa `id={field.name}`.
- El control recibe `{...field}` y `aria-invalid={fieldState.invalid}`.
- `FieldError` solo se renderiza cuando `fieldState.invalid`.

## Campos con íconos (InputGroup)

Cuando el campo lleve un ícono, prefijo o botón, usar `InputGroup` en lugar de `Input`:

```tsx
<Field data-invalid={fieldState.invalid}>
  <FieldLabel htmlFor={field.name}>Correo</FieldLabel>
  <InputGroup>
    <InputGroupAddon>
      <MailIcon />
    </InputGroupAddon>
    <InputGroupInput
      {...field}
      id={field.name}
      aria-invalid={fieldState.invalid}
      placeholder="correo@ejemplo.com"
      autoComplete="off"
    />
  </InputGroup>
  {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
</Field>
```

## Otros controles

Mantener `Controller`, `Field`, `FieldLabel` y `FieldError`; solo cambia cómo se conecta el control:

- Select: `value={field.value}` y `onValueChange={field.onChange}`.
- Checkbox / Switch: `checked={field.value}` y `onCheckedChange={field.onChange}`.
- Textarea: igual que Input, con `{...field}`.

## Composición en dialogs

```tsx
export function NewPatientDialog() {
  const [open, setOpen] = useState(false)
  const { createForm, handleCreate, isCreating } = useCreatePatient({
    onSuccess: () => setOpen(false),
  })

  return (
    <Dialog open={open} onOpenChange={setOpen}>
      {/* trigger y contenido */}
      <PatientForm
        form={createForm}
        submitLabel="Crear paciente"
        onSubmit={handleCreate}
        onCancel={() => setOpen(false)}
        isPending={isCreating}
      />
    </Dialog>
  )
}
```

`EditPatientDialog` es análogo: `useUpdatePatient({ patient, onSuccess })`, `form={updateForm}`, `onSubmit={handleUpdate}`, `isPending={isUpdating}` y `submitLabel="Guardar cambios"`.

## Requisitos de shadcn

Las primitivas `field` e `input-group` deben estar instaladas en `shared/components/ui` (alias configurado en `components.json`). Si faltan, instalarlas con la CLI de shadcn sin sobrescribir personalizaciones existentes.
