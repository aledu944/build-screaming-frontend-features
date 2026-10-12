# Variante Next.js App Router

## Contenido

1. Límites del framework
2. Organización
3. Lecturas
4. Mutaciones
5. Formularios y hooks
6. Seguridad, caché y checklist

## Límites del framework

Usar Server Components para lecturas iniciales de route. Pueden consultar una API, SDK u ORM del lado servidor sin crear una Server Action intermedia.

Usar Server Functions marcadas con `"use server"` principalmente para mutaciones. Cuando una Server Function se pasa a un form o se usa en contexto de mutación, actúa como Server Action. Tratar cada una como endpoint POST directamente alcanzable.

No usar Server Actions para listados o fetching paralelo: se despachan como mutaciones y pueden serializarse en cliente. Para client fetching frecuente, polling o APIs públicas, usar Route Handlers o la librería de query adoptada por el proyecto.

Marcar helpers de acceso a datos con `server-only` cuando puedan ser importados accidentalmente por Client Components.

## Organización

Mantener el dominio fuera de `app/`; usar `app/` como composition root:

```text
src/
├── app/
│   └── admin/patients/
│       ├── page.tsx
│       ├── loading.tsx
│       └── [id]/page.tsx
└── features/patients/
    ├── actions/
    │   ├── create-patient.ts
    │   └── update-patient.ts
    ├── server/
    │   └── patient-queries.ts
    ├── components/
    ├── hooks/
    ├── schemas/
    ├── mappers/
    ├── entities/
    └── types/
```

Si el repositorio usa solamente `server/`, colocar `*.action.ts` y `*.query.ts` allí. No introducir dos convenciones simultáneas.

Mantener `page.tsx` enfocado en leer `params/searchParams`, lanzar lecturas, componer Suspense/boundaries y renderizar componentes del feature.

## Lecturas

Crear una query server-only que aplique autorización y mapper:

```ts
import "server-only"

export async function getPatients(
  filters: PatientsFilterOptions,
): Promise<PaginatedPatients> {
  await requirePatientsReadAccess()
  const response = await apiClient.get<GetPatientsResponse>("/patients", {
    params: filters,
  })

  return {
    patients: response.data.patients.map(patientFromRest),
    pagination: response.data.pagination,
  }
}
```

Validar search params antes de pasarlos a la query:

```tsx
export default async function PatientsPage({ searchParams }: PageProps) {
  const rawSearch = await searchParams
  const filters = patientsFilterOptionsSchema.parse(rawSearch)
  const result = await getPatients(filters)

  return <PatientTable patients={result.patients} />
}
```

Adaptar strings de URL a boolean/number dentro del schema mediante coerción o preprocess. No afirmar que `searchParams` ya contiene numbers o booleans.

Evitar `z.coerce.boolean()` para strings de query: JavaScript convierte cualquier string no vacío, incluido `"false"`, a `true`. Usar un preprocess o enum `"true" | "false"` y transformarlo explícitamente.

Consultar fuentes independientes con `Promise.all` o separar en Suspense boundaries para evitar waterfalls. Mantener secretos y tipos ORM dentro de server/query/mapper.

## Mutaciones

Crear un archivo de acción con la directiva a nivel módulo cuando deba importarse desde Client Components:

```ts
"use server"

import { revalidatePath } from "next/cache"

export async function updatePatient(rawInput: unknown): Promise<ActionResult<Patient>> {
  const session = await requireSession()
  const parsed = updatePatientSchema.safeParse(rawInput)

  if (!parsed.success) {
    return { data: null, error: "Datos inválidos", fieldErrors: parsed.error.flatten().fieldErrors }
  }

  await authorizePatientUpdate(session.user, parsed.data.id)
  const { id, ...payload } = parsed.data
  const response = await apiClient.patch<UpdatePatientResponse>(
    `/patients/${id}`,
    payload,
  )
  const patient = patientFromRest(response.data.patient)

  revalidatePath("/admin/patients")
  revalidatePath(`/admin/patients/${id}`)
  return { data: patient, error: null }
}
```

Validar dentro de la acción aunque el mismo schema se ejecute en cliente. El cliente es una mejora de UX, no una frontera de confianza.

Autorizar el recurso concreto, no solo comprobar que existe sesión. Evitar recibir `userId`, tenant u ownership confiando en campos ocultos del form; derivarlos de sesión y DB.

Mantener argumentos y retornos serializables cuando crucen hacia Client Components. No devolver instancias Axios, errores, conexiones, clases ORM ni secretos.

## Formularios y hooks

Elegir la integración ya usada por el proyecto:

- Usar `<form action={serverAction}>`, `useActionState` y `useFormStatus` para progressive enhancement y flujo nativo.
- Usar React Hook Form + Zod en un Client Component cuando exista validación interactiva compleja; volver a validar en la acción.
- Encapsular form, estado pending, feedback y callback visual en `useCreatePatient`/`useUpdatePatient` si el patrón de hooks ya existe, con el contrato de [forms.md](forms.md).

No hacer que la acción muestre toast o controle dialogs; devolver un resultado serializable y dejar esos efectos al hook/componente.

No usar `router.refresh()` como sustituto automático de invalidación server-side. `router.refresh()` vuelve a pedir el payload de la route, pero no invalida por sí solo la caché de datos del servidor. Ejecutar `revalidatePath`, `revalidateTag` o `updateTag` en la acción según la política de datos, y usar navegación/refresh cliente solo si todavía hace falta para UX.

## Seguridad, caché y checklist

Considerar cada Server Function pública:

- autenticar dentro de la función;
- autorizar acción y recurso;
- validar input después de cruzar el límite;
- aplicar rate limiting/CSRF/origin policy si el riesgo y despliegue lo requieren;
- evitar datos sensibles en closures, argumentos o retornos;
- devolver errores sanitizados.

Elegir invalidación precisa:

- `revalidatePath` para una page/layout concreta;
- `revalidateTag` para marcar datos compartidos como stale;
- `updateTag` cuando el proyecto necesite expiración inmediata del tag tras una escritura.

Checklist:

- Usar Server Components o queries server-only para lecturas.
- Usar Server Functions/Actions para mutaciones.
- Mantener `app/` como composición y `features/` como dominio.
- Validar params, search params y action input.
- Mapear REST/DB/SDK a entities antes de renderizar.
- Repetir auth y autorización dentro de cada acción/query privada.
- Mantener límites Client/Server explícitos.
- Revalidar paths/tags afectados por cada mutación.
- No devolver contratos no serializables a Client Components.

Documentación oficial relevante:

- https://nextjs.org/docs/app/getting-started/fetching-data
- https://nextjs.org/docs/app/getting-started/mutating-data
- https://nextjs.org/docs/app/api-reference/functions/revalidatePath
- https://nextjs.org/docs/app/guides/backend-for-frontend
