# Arquitectura por dominio

## Contenido

1. Árbol objetivo
2. Responsabilidad de cada capa
3. Dirección de dependencias
4. Entidades, contratos, schemas y mappers
5. Hooks y componentes
6. Shared y límites entre features
7. Errores comunes

## Árbol objetivo

Usar el dominio como primer nivel visible:

```text
src/
├── features/
│   └── patients/
│       ├── components/
│       ├── constants/
│       ├── entities/
│       ├── hooks/
│       ├── mappers/
│       ├── schemas/
│       ├── server/       # o actions/, queries/ según el framework
│       └── types/
├── routes/               # TanStack Start
├── app/                  # Next.js App Router
└── shared/
    ├── components/
    │   └── ui/           # primitivas shadcn
    ├── constants/
    ├── helpers/
    ├── hooks/
    ├── layout/
    ├── lib/
    ├── providers/
    ├── schemas/
    └── types/
```

Considerar todas las subcarpetas opcionales. Crear una cuando el feature tenga esa responsabilidad, no para completar mecánicamente el árbol.

Nombrar el feature con lenguaje del negocio, normalmente plural y en inglés si esa es la convención del código. Evitar carpetas horizontales globales como `src/components/patients`, `src/hooks/patients` y `src/services/patients`, porque dispersan un solo cambio por todo el repositorio.

## Responsabilidad de cada capa

| Capa | Contiene | No debe contener |
|---|---|---|
| `entities/` | Modelo estable consumido por UI y workflows | Axios responses, Prisma payloads, Zod schemas |
| `types/` | Contratos de fuentes externas y tipos técnicos no inferibles | Duplicados de `z.infer`, props de un único componente |
| `schemas/` | Validación/coerción runtime de forms, URL y server inputs | I/O, toasts, navegación |
| `mappers/` | Transformaciones puras entre contratos y entidades/payloads | Fetch, DB, estado React |
| `server/` o `actions/` | I/O, auth, validación, mapper, política de errores/cache | JSX, hooks React, estado visual |
| `hooks/` | Formularios, mutaciones, filtros, caché e interacción reutilizable | JSX, secretos, acceso directo a DB |
| `components/` | Render y estado visual local | Contratos Axios/ORM, lógica de transporte |
| route/page | Composición, params, loaders y boundaries | Formularios o workflows CRUD completos |

Mantener props pequeñas junto al componente. Moverlas a `types/` solo si son un contrato técnico compartido y no un detalle de presentación.

## Dirección de dependencias

Mantener este flujo conceptual:

```text
external source
      │
      ▼
types + schemas ──► mappers ──► entities
      │                            │
      ▼                            ▼
server functions/actions ──► hooks/loaders ──► components ──► route/page
```

Permitir que `constants/` sea usado por entities, types, schemas y UI. Permitir dependencias hacia `shared/`, nunca hacer que `shared/` importe un feature.

Evitar imports desde componentes hacia server cuando exista un hook que sea dueño del workflow. Permitir que routes/loaders importen server functions para la carga inicial. Evitar ciclos entre features; si dos dominios se necesitan, coordinar desde la route/page o extraer un contrato pequeño y realmente compartido.

## Entidades, contratos, schemas y mappers

### Entidades

Modelar lo que la aplicación entiende, no lo que una fuente casualmente devuelve:

```ts
export type Patient = {
  id: string
  fullName: string
  birthDate: string
  age: number
  isActive: boolean
}
```

Elegir una representación interna consistente para fechas y dinero. Usar ISO strings cuando los datos cruzan límites cliente/servidor; usar `Date` solo cuando el runtime y el límite de serialización lo permitan y el proyecto ya lo haga.

Incluir campos derivados como `age` o `fullName` en la entidad si varias vistas los necesitan. Calcularlos en el mapper o en una función de dominio pura, no repetirlos en componentes.

### Contratos externos en `types/`

Nombrar el origen cuando pueda haber más de uno:

```ts
export interface PatientRestResponse {
  id: string
  name: string
  lastname: string
  birthDate: string
  isActive: boolean
  createdAt: string
}

export interface PatientDbRow {
  id: string
  first_name: string
  last_name: string
  birth_date: Date
  active: boolean
}
```

No exportar tipos generados por un ORM o SDK hacia componentes. Definir el select/payload mínimo en la capa server y mapearlo. Si una API cambia, limitar el impacto al contrato y mapper.

Mantener envelopes y paginación como contratos técnicos reutilizables cuando sean realmente uniformes:

```ts
export type PaginatedResponse<T, K extends string> = {
  [P in K]: T[]
} & { pagination: Pagination }
```

### Schemas

Usar Zod como fuente de verdad para valores que llegan en runtime:

```ts
export const patientCreateSchema = z.object({
  name: z.string().trim().min(1),
  birthDate: z.iso.date(),
  email: z.email(),
})

export type PatientCreateInput = z.input<typeof patientCreateSchema>
export type PatientCreateData = z.output<typeof patientCreateSchema>
```

Usar `z.input` para el valor que recibe un form o boundary antes de coerción y `z.output` para el valor validado. Si no hay transformaciones, ambos pueden coincidir. No escribir otra interface manual con los mismos campos.

Separar schemas por caso de uso cuando sus invariantes difieran. Extender/componer schemas compartidos, pero asegurar que IDs y campos no editables sean requeridos en el boundary que realmente los necesita.

Validar también filtros, page/limit, params dinámicos y payloads parciales. Añadir refinements de relaciones entre campos, por ejemplo `minAge <= maxAge`, cuando el backend o UI dependa de esa regla.

Mantener responses externas como contratos TypeScript cuando esa sea la política del proyecto. Añadir schemas de response en integraciones no confiables, datos persistidos por terceros o flujos de alto riesgo; parsearlas antes del mapper. No confundir tipado estático con validación runtime.

### Mappers

Crear funciones o métodos explícitos por origen:

```ts
export const patientFromRest = (source: PatientRestResponse): Patient => ({
  id: source.id,
  fullName: `${source.name} ${source.lastname}`.trim(),
  birthDate: new Date(source.birthDate).toISOString(),
  age: calculateAge(source.birthDate),
  isActive: source.isActive,
})

export const patientFromDb = (source: PatientDbRow): Patient => ({
  id: source.id,
  fullName: `${source.first_name} ${source.last_name}`.trim(),
  birthDate: source.birth_date.toISOString(),
  age: calculateAge(source.birth_date),
  isActive: source.active,
})
```

Mapear arrays en la capa server inmediatamente después de recibirlos. Añadir mappers de salida como `patientCreateDataToApiPayload` solo si el destino difiere del schema/form. No crear un mapper identidad sin una razón de evolución clara.

Usar funciones puras o una clase con métodos estáticos según la convención local. Conservar nombres que hagan visible el origen y destino, como `patientFromRest` o `PatientsMapper.fromResponseToEntity`.

Probar mappers cuando normalicen fechas, nullables, enums, dinero o campos derivados.

## Hooks y componentes

### Taxonomía de hooks

Usar un hook por workflow y responsabilidad:

- `usePatients`: orquestar el listado cuando la estrategia local use Query/client fetching.
- `usePatientsFilters`: leer/escribir búsqueda, filtros, orden y paginación; preferir URL cuando deban ser compartibles y sobrevivir refresh.
- `useCreatePatient`: poseer schema resolver, mutation, feedback, invalidación y callback de éxito de la creación; devolver `{ createForm, handleCreate, isCreating }`.
- `useUpdatePatient`: sincronizar defaults de edición y mutation de actualización; devolver `{ updateForm, handleUpdate, isUpdating }`.
- `usePatientStatus`: cambiar solamente estado activo/inactivo.

No crear un hook monolítico con list/create/update/delete y todos los formularios. No mover automáticamente `isDialogOpen` a un hook de dominio: mantener estado puramente visual dentro del dialog o trigger.

Permitir que un componente genérico dentro del feature, como `PatientForm`, sea reutilizado por create y update. Mantenerlo en el feature mientras su lenguaje y schema sean del dominio; moverlo a `shared/` solo si pierde toda dependencia de pacientes y tiene un segundo consumidor real.

### Formularios

Conectar React Hook Form al mismo schema usado en el boundary de mutación. Mantener defaults y conversión `entity -> form values` en el hook o mapper de formulario, especialmente para fechas, null y campos ocultos.

Pasar al componente solo `form` (`createForm` o `updateForm`), `onSubmit`, `onCancel`, `isPending` y labels visuales (`submitLabel`). Evitar que el componente importe directamente la función de servidor. Seguir [forms.md](forms.md) para los hooks, el componente y la anatomía de campos (`Controller` + `Field`, sin `FieldDescription`, `InputGroup` para íconos).

### Listados

Separar búsqueda, filtros, tabla, paginación, empty state y skeleton cuando tengan suficiente lógica o sean reutilizados dentro del módulo. Mantener una única fuente de verdad para page/filter/sort. Al cambiar filtros que reducen resultados, volver a la primera página.

## Shared y límites entre features

Mover a `shared/` únicamente:

- primitivas shadcn y composiciones visuales sin vocabulario de dominio;
- helpers puros usados por varios features;
- cliente HTTP/DB y manejo uniforme de errores;
- contratos técnicos realmente globales, por ejemplo paginación;
- schemas base agnósticos al dominio.
- layouts, navegación y providers que forman el app shell.

No mover prematuramente forms, cards, filters o hooks que todavía hablan de un solo dominio. Duplicación breve es preferible a una abstracción que mezcle reglas de negocio distintas.

Mantener `shared/components/ui` como primitive layer, `shared/components` como composition layer y `shared/layout` como shell layer. Consultar `shared-ui-layouts.md` para criterios, flujo CLI de shadcn y límites de layouts.

## Errores comunes

- Organizar primero por tecnología global en lugar de dominio.
- Usar la response externa como entity porque hoy tienen la misma forma.
- Declarar interfaces que duplican un schema Zod.
- Hacer fetch o parsear fechas dentro de un componente.
- Poner toasts, navegación o invalidación dentro de una server action.
- Confiar en un guard visual para autorizar un endpoint.
- Exponer secretos o clientes DB en código importable por el cliente.
- Usar Server Actions para listar datos en Next.js.
- Consultar DB directamente desde un loader isomórfico de TanStack Start.
- Mover todo a `shared/` y borrar el lenguaje del feature.
- Colocar bloques compuestos o componentes de negocio dentro de `shared/components/ui`.
- Sobrescribir una primitiva shadcn adaptada al ejecutar el CLI.
- Hacer que un layout importe hooks o server functions de un feature.
- Crear barrels circulares o un `index.ts` que reexporte toda la aplicación.
