# Blueprint de un feature

## Contenido

1. Inventario del slice
2. Plantilla de archivos
3. Flujo de lectura
4. Flujo de creación/actualización
5. Revisión final

## Inventario del slice

Usar este inventario como menú, no como obligación. Para un módulo equivalente a `patients`:

```text
features/patients/
├── components/
│   ├── patient-form.tsx
│   ├── patient-table.tsx
│   ├── patient-filters.tsx
│   ├── new-patient-dialog.tsx
│   └── edit-patient-dialog.tsx
├── constants/
│   └── gender.ts
├── entities/
│   ├── patient.entity.ts
│   └── patient-stats.entity.ts
├── hooks/
│   ├── use-patients-filters.ts
│   ├── use-create-patient.ts
│   ├── use-update-patient.ts
│   └── use-patient-status.ts
├── mappers/
│   └── patient.mapper.ts
├── schemas/
│   ├── patient.schema.ts
│   └── patients-filter-options.schema.ts
├── server/ o actions/
│   ├── get-patients.ts
│   ├── get-patient.ts
│   ├── create-patient.ts
│   └── update-patient.ts
└── types/
    ├── patient-response.interface.ts
    └── get-patients-response.interface.ts
```

Conservar nombres de archivo en singular para una unidad/caso de uso y plural para colecciones: `patient.entity.ts`, `get-patients.ts`, `use-patients-filters.ts`.

## Plantilla de archivos

### Constante y tipo cerrado

```ts
export const STATUS_VALUES = ["ACTIVE", "INACTIVE"] as const
export type Status = (typeof STATUS_VALUES)[number]

export const STATUS_LABELS: Record<Status, string> = {
  ACTIVE: "Activo",
  INACTIVE: "Inactivo",
}
```

Usar una única lista `as const` para alimentar schema, selects y labels.

### Entity

```ts
export type Patient = {
  id: string
  name: string
  lastname: string
  birthDate: string
  age: number
  status: Status
  createdAt: string
}
```

### Contrato REST

```ts
export interface PatientResponse {
  id: string
  name: string
  lastname: string
  birthDate: string
  status: Status
  createdAt: string
  updatedAt: string
}

export interface GetPatientResponse {
  patient: PatientResponse
}
```

Representar exactamente la fuente aunque use nombres o tipos incómodos. Corregir esa incomodidad en el mapper.

### Schemas

```ts
export const patientFieldsSchema = z.object({
  name: z.string().trim().min(1, "El nombre es requerido"),
  lastname: z.string().trim().min(1, "El apellido es requerido"),
  birthDate: z.iso.date(),
})

export const patientCreateSchema = patientFieldsSchema

export const patientUpdateSchema = patientFieldsSchema.extend({
  id: z.uuid(),
})

export type PatientCreateInput = z.input<typeof patientCreateSchema>
export type PatientCreateData = z.output<typeof patientCreateSchema>
export type PatientUpdateInput = z.input<typeof patientUpdateSchema>
```

Mantener el ID requerido en el schema del boundary de update aunque no sea editable en el formulario. Si el form comparte un schema base donde ID es opcional, extender nuevamente con ID requerido en la server function/action.

### Mapper

```ts
export const patientFromResponse = (response: PatientResponse): Patient => ({
  id: response.id,
  name: response.name,
  lastname: response.lastname,
  birthDate: new Date(response.birthDate).toISOString(),
  age: calculateAge(response.birthDate),
  status: response.status,
  createdAt: new Date(response.createdAt).toISOString(),
})
```

Separar `patientFromResponse`, `patientFromDbRow` o `patientFromCsvRow` cuando existan fuentes distintas. Evitar un mapper con ramas implícitas basadas en presencia de campos.

### Result de mutación

Usar la forma local. Si no existe, preferir una unión discriminada:

```ts
export type ActionResult<T> =
  | { ok: true; data: T }
  | { ok: false; error: string; fieldErrors?: Record<string, string[] | undefined> }
```

Una unión evita estados imposibles como `data` y `error` presentes simultáneamente. No reemplazar un envelope consolidado en el proyecto sin una migración explícita.

## Flujo de lectura

Implementar y verificar el recorrido:

```text
URL/search params
  -> Zod filter schema
  -> route/page loader
  -> getPatients server boundary
  -> REST/DB typed contract
  -> patientFromResponse/patientFromDb
  -> Patient[] entity
  -> PatientTable
```

Para listados:

1. Validar `search`, `page`, `limit`, filters y sort.
2. Conservar filtros en URL cuando deban ser compartibles.
3. Pasar filtros validados al boundary server.
4. Tipar el envelope externo.
5. Mapear cada item.
6. Devolver entities y metadata técnica necesaria.
7. Renderizar loading, error, empty y success.
8. Conectar paginación y filtros a la misma fuente de verdad.

Para detalle, validar ID, mapear la response y transformar ausencia al mecanismo not-found del framework.

## Flujo de creación/actualización

Implementar y verificar el recorrido:

```text
PatientForm
  -> useCreatePatient/useUpdatePatient
  -> React Hook Form + Zod
  -> createPatient/updatePatient server boundary
  -> auth + server-side validation
  -> REST/DB write
  -> response contract
  -> mapper
  -> entity result
  -> cache invalidation + toast/navigation/dialog callback
```

Asignar responsabilidades:

- `PatientForm`: campos, errores visuales y botones; recibe por props `form`, `submitLabel`, `onSubmit`, `onCancel` e `isPending` (ver [forms.md](forms.md)).
- `useCreatePatient`: defaults de alta, `createForm`, `handleCreate`, `isCreating`, feedback e invalidación.
- `useUpdatePatient`: conversión entity a form defaults, resincronización, `updateForm`, `handleUpdate` e `isUpdating`.
- server function/action: auth, validación no confiable, I/O, mapper y error sanitizado.
- dialog: estado open/close puramente visual.

No confiar en IDs o ownership que provienen del form. Tomar el ID canónico de props/route y verificar ownership en servidor.

Después de éxito:

1. invalidar la autoridad de datos correcta;
2. mostrar feedback desde UI/hook;
3. resetear el form cuando corresponda;
4. cerrar dialog mediante callback;
5. navegar solo si el caso de uso lo requiere.

## Revisión final

### Arquitectura

- El nombre del feature expresa negocio.
- Cada carpeta creada tiene una responsabilidad usada.
- Routes/pages componen; no concentran el slice.
- `shared/` no importa desde `features/`.
- No hay ciclos ni barrels opacos.

### Contratos

- Entities no importan Axios, Prisma ni SDKs.
- Responses/rows externos viven en `types/` o server-only.
- Types inferidos no duplican schemas.
- Mappers cubren cada fuente y no hacen I/O.
- Fechas, enums, nullables y derivados tienen representación interna consistente.

### Runtime y seguridad

- URL, forms y server inputs se validan con Zod.
- Server functions/actions autentican y autorizan.
- Errores externos se sanitizan.
- Datos privados no usan caché pública.
- El cliente no recibe secretos ni objetos no serializables.

### UX y estado

- Hooks están separados por workflow.
- Formularios usan el schema correcto.
- Mutaciones manejan pending/error/success.
- Invalidación/revalidación actualiza listados y detalles afectados.
- Listados contemplan loading, error, empty y pagination.

### Calidad

- No se introdujo `any` ni casts evasivos.
- Typecheck, lint, tests y build aplicables pasan.
- Schemas y mappers no triviales tienen pruebas focalizadas.
- El diff se limita al alcance pedido.
