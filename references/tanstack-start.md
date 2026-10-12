# Variante TanStack Start

## Contenido

1. Límites del framework
2. Organización
3. Lecturas y filtros
4. Mutaciones y hooks
5. Seguridad y errores
6. Checklist

## Límites del framework

Usar `createServerFn` para todo acceso a DB, filesystem, secretos o APIs privadas. Tratar sus handlers como endpoints invocables directamente.

Recordar que los loaders de TanStack Router son isomórficos: pueden ejecutarse en cliente y servidor. Hacer que el loader invoque una server function; no importar el cliente DB o secretos dentro de la route.

Usar imports estáticos de server functions. TanStack reemplaza su implementación por stubs RPC en bundles cliente.

Aplicar `createServerFn({ method: "GET" })` a lecturas y `POST` a mutaciones. Validar todo input con `.validator(schema)` antes de entrar al handler.

## Organización

Conservar la convención local si ya existe:

```text
features/patients/
├── server/
│   ├── get-patients.ts
│   ├── get-patient.ts
│   ├── create-patient.ts
│   └── update-patient.ts
├── hooks/
├── schemas/
├── mappers/
├── entities/
└── types/
```

En un proyecto nuevo también es válido separar wrappers y helpers exclusivos del servidor:

```text
features/patients/server/
├── patients.functions.ts  # createServerFn, seguro de importar desde cliente
└── patients.server.ts     # DB/API helpers server-only
```

No renombrar todo un proyecto existente solo para adoptar el sufijo nuevo. Mantener explícito cuál archivo crea el RPC y cuál nunca debe entrar al bundle cliente.

## Lecturas y filtros

Validar search params en la route con el schema del feature y hacerlos dependencias del loader:

```ts
export const Route = createFileRoute("/admin/patients/")({
  validateSearch: patientsFilterOptionsSchema,
  loaderDeps: ({ search }) => search,
  loader: ({ deps }) =>
    Promise.all([
      getPatients({ data: deps }),
      getPatientStats(),
    ]),
  component: PatientsPage,
})
```

Implementar la server function con contrato y mapper:

```ts
export const getPatients = createServerFn({ method: "GET" })
  .validator(patientsFilterOptionsSchema)
  .handler(async ({ data: filters }) => {
    await requirePatientsReadAccess()
    const response = await apiClient.get<GetPatientsResponse>("/patients", {
      params: filters,
    })

    return {
      patients: response.data.patients.map(patientFromRest),
      pagination: response.data.pagination,
    }
  })
```

Guardar filtros de listados en search params cuando deban sobrevivir refresh, soportar back/forward y producir enlaces compartibles. Encapsular escritura de URL en `usePatientsFilters`. Reiniciar `page` cuando cambie búsqueda o un filtro que pueda recortar resultados.

Usar loader para datos iniciales ligados a una route. Usar TanStack Query cuando existan necesidades cliente como polling, cache compartida fuera de routes, optimistic updates o datos independientes del ciclo del router. No mantener dos cachés para el mismo recurso sin definir cuál es la autoridad.

## Mutaciones y hooks

Crear una server function por intención de escritura:

```ts
export const updatePatient = createServerFn({ method: "POST" })
  .validator(updatePatientSchema)
  .handler(async ({ data: { id, ...input } }) => {
    await requirePatientWriteAccess(id)
    const { data } = await apiClient.patch<UpdatePatientResponse>(
      `/patients/${id}`,
      input,
    )

    return {
      message: data.message,
      patient: patientFromRest(data.patient),
    }
  })
```

Envolverla en un hook enfocado cuando la invoque UI interactiva:

```ts
export const useUpdatePatient = ({ patient, onSuccess }: Options) => {
  const router = useRouter()
  const updateForm = useForm<PatientFormValues>({
    resolver: zodResolver(patientFormSchema),
    defaultValues: patientToFormValues(patient),
  })

  const mutation = useMutation({
    mutationFn: updatePatient,
    onSuccess: async () => {
      await router.invalidate()
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

Seguir [forms.md](forms.md) para el contrato completo de formularios (hook de creación, `PatientForm` y anatomía de campos).

Usar el resultado y la estrategia de errores ya establecidos por el repositorio. Elegir una sola política por área:

- lanzar errores tipados y resolverlos en route/query boundaries; o
- devolver una unión/envelope consistente como `{ data, error }`.

No mezclar ambas sin adaptación. No devolver stack traces, errores Axios/DB ni información sensible.

Invocar directamente una server function desde `useMutation` si devuelve datos normales. Usar `useServerFn` cuando lance `redirect()` o `notFound()` para que el router procese correctamente el control de flujo; usarlo también como opción segura si esa conducta puede añadirse.

Después de una mutación ligada a loader data, usar `router.invalidate()`. Si la autoridad es TanStack Query, invalidar sus query keys. No invalidar indiscriminadamente ambas capas.

## Seguridad y errores

Aplicar auth middleware o un check dentro de cada handler privado. Un `beforeLoad` protege la experiencia de la route, no el endpoint de `createServerFn`.

No cachear como `public` una respuesta dependiente de sesión, tenant, cookie o Authorization. Usar `private` con `Vary` apropiado o `no-store` para datos sensibles.

Evitar estos patrones:

```ts
// Incorrecto: DB en loader isomórfico
loader: () => db.patient.findMany()

// Incorrecto: patrón de otro framework
"use server"

// Incorrecto: la función no fue invocada
const patients = await getPatients
```

## Checklist

- Usar `createServerFn` en el límite server.
- Validar input con `.validator`.
- Autorizar dentro del handler/middleware.
- Mapear responses antes de devolverlos a la UI.
- Hacer que loaders llamen funciones, no DB/API secreta.
- Conectar `loaderDeps` con filtros validados.
- Definir una sola autoridad de caché por dato.
- Invalidar router o query cache después de mutar.
- Manejar redirect/notFound mediante `useServerFn` cuando corresponda.
