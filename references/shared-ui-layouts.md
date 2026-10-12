# Shared, shadcn y layouts

## Contenido

1. Propósito de shared
2. Árbol y responsabilidades
3. Dirección de dependencias
4. Jerarquía de componentes
5. Integración de shadcn
6. Tokens y personalización
7. Layouts y shells
8. Providers y boundaries globales
9. Checklist

## Propósito de shared

Usar `shared/` como kernel técnico y visual reutilizable de la aplicación, no como carpeta para archivos sin dueño. Aceptar un elemento cuando cumpla al menos una de estas condiciones:

- ser infraestructura agnóstica al dominio;
- ser una primitiva del design system;
- tener consumidores reales en varios features;
- formar parte del app shell, navegación o providers globales;
- representar un contrato técnico uniforme, no una regla de negocio.

Mantener dentro del feature cualquier archivo que conserve nombres, reglas, schemas o casos de uso del dominio. No mover `PatientForm`, `PatientFilters` o `usePatients` a shared aunque varios componentes del mismo feature los reutilicen.

Permitir excepciones deliberadas para piezas estables y claramente transversales, como paginación, manejo uniforme de errores o un helper de fecha genérico. Documentar la razón si aún existe un solo consumidor.

## Árbol y responsabilidades

Seguir la convención local de singular/plural. En este repositorio usar `shared/layout`, no crear además `shared/layouts`.

```text
src/shared/
├── components/
│   ├── ui/                  # primitivas shadcn editables
│   ├── date-picker.tsx      # composición neutral
│   ├── header-page.tsx
│   └── not-found.tsx
├── constants/               # navegación/config estática del shell
├── helpers/                 # funciones puras agnósticas
├── hooks/                   # hooks técnicos reutilizables, si existen
├── layout/                  # app bar, sidebar y shells visuales
├── lib/                     # adapters/config: cn, HTTP, auth client
├── providers/               # providers React transversales, si existen
├── schemas/                 # schemas técnicos base
└── types/                   # contratos técnicos globales
```

Mantener integraciones grandes de terceros en `src/integrations/` cuando el proyecto ya use esa convención, como TanStack Query. No duplicarlas en `shared/providers`.

Aplicar estas responsabilidades:

| Ubicación | Responsabilidad | Ejemplos |
|---|---|---|
| `components/ui` | Primitivas accesibles y variantes base | Button, Dialog, Field, InputGroup, Table |
| `components` | Composiciones neutrales sobre primitivas | DatePicker, HeaderPage, EmptyState |
| `layout` | Regiones persistentes del shell | AppBar, AppSidebar, AuthShell |
| `constants` | Config estática transversal | items de navegación, metadata de app |
| `helpers` | Transformaciones puras | formatters y cálculos genéricos |
| `hooks` | Estado/comportamiento técnico reusable | media query, debounce, disclosure |
| `lib` | Configuración y adapters | `cn`, API client, storage adapter |
| `providers` | Contextos React globales o por shell | theme, auth, query provider |
| `schemas/types` | Contratos técnicos comunes | filtros base, paginación, ActionResult |

No guardar props de un único componente en `shared/types`. Mantenerlas junto al componente.

## Dirección de dependencias

Mantener el grafo acíclico:

```text
shared/lib + shared/helpers + shared/types + shared/schemas
                         │
                         ▼
             shared/components/ui
                         │
                         ▼
               shared/components
                         │
                         ▼
                 shared/layout

features/* ─────────────► shared/*
routes/app ─────────────► shared/layout + features/*
```

Prohibir imports desde `shared/` hacia `features/`. Si un shell necesita datos de sesión, recibirlos por props, consumir un provider transversal o usar una capa auth compartida; no importar `usePatients`, `Patient` o `getPatients`.

Permitir que route layouts compongan shared y features porque actúan como composition roots. Evitar que un componente UI base importe un componente compuesto o layout.

Evitar barrels que reexporten todo shared. Usar imports directos como `@/shared/components/ui/button` para conservar límites, tree-shaking y trazabilidad.

## Jerarquía de componentes

Clasificar antes de crear o mover:

1. Colocar en `features/<domain>/components` si contiene lenguaje, entity, schema o workflow del dominio.
2. Colocar en `shared/components` si compone primitivas, es neutral y tiene alcance transversal o de shell.
3. Colocar en `shared/components/ui` solo si es una primitiva del design system comparable a Button, Input, Dialog o Table.
4. Colocar en `shared/layout` si define una región persistente alrededor de múltiples routes/pages.

Ejemplos:

- `PatientForm` permanece en patients.
- `DatePicker` vive en shared/components porque combina Button, Calendar y Popover sin reglas de pacientes.
- `Button` vive en shared/components/ui.
- `AppSidebar` vive en shared/layout.
- Un bloque `PatientDashboard` no pertenece a ui ni layout aunque use muchas primitivas.

Permitir reutilización dentro de un solo feature sin promover a shared. Promover solo cuando la abstracción ya sea neutral; no borrar diferencias de dominio mediante flags y props excesivos.

## Integración de shadcn

Tratar shadcn como código fuente que el proyecto posee y adapta, no como una caja negra. Inspeccionar siempre `components.json` antes de usar el CLI.

En este repositorio respetar:

```json
{
  "style": "new-york",
  "rsc": false,
  "tailwind": {
    "css": "src/styles.css",
    "config": "",
    "cssVariables": true
  },
  "aliases": {
    "components": "@/shared/components",
    "ui": "@/shared/components/ui",
    "utils": "@/shared/lib/utils",
    "lib": "@/shared/lib",
    "hooks": "@/shared/hooks"
  },
  "iconLibrary": "hugeicons"
}
```

No hardcodear estos valores en otros proyectos. Leer su `components.json`, resolver aliases en TypeScript/bundler y conservar style, base color, CSS variables, biblioteca base e iconos existentes.

Usar el package manager detectado por el lockfile. Desde la raíz de la app:

```bash
# package-lock.json
npx shadcn@latest add card --dry-run
npx shadcn@latest add card

# pnpm-lock.yaml
pnpm dlx shadcn@latest add card --dry-run
pnpm dlx shadcn@latest add card
```

Seguir este flujo:

1. Confirmar que el componente no existe ya y buscar una composición equivalente.
2. Ejecutar `add <component> --dry-run` para revisar archivos y dependencias.
3. Si existe una primitiva local modificada, usar `--diff` o comparar el registry antes de decidir.
4. Instalar sin `--overwrite` por defecto.
5. Revisar todos los archivos y dependencias cambiados por el CLI.
6. Ajustar imports al alias configurado solo si el CLI no lo hizo correctamente.
7. Ejecutar typecheck/lint/build y probar teclado, focus, labels y estados disabled/error.

No ejecutar `init` en un proyecto ya configurado para “arreglar” un componente. No cambiar `style`, `baseColor` o `cssVariables` a mitad del proyecto sin una migración explícita. No usar `--overwrite` sobre `button.tsx`, `dialog.tsx` u otra primitiva adaptada sin preservar variantes, tokens, iconos y accesibilidad locales.

Crear `shared/hooks` cuando el CLI instale un hook o exista comportamiento realmente transversal. No crear la carpeta solo porque el alias esté declarado.

### Personalizar en la capa correcta

Editar `components/ui` cuando el cambio deba afectar globalmente a la primitiva:

- variantes CVA base;
- focus ring, aria y estados disabled/invalid;
- tokens semánticos y spacing del design system;
- correcciones de accesibilidad o compatibilidad.

Crear una composición en `shared/components` cuando combine varias primitivas con una API reusable. Mantener una variante de dominio en su feature. Preferir `className`, `variant` o una wrapper del feature para una diferencia local; no inflar la primitiva global con props como `patientMode` o `invoiceMode`.

Preservar `data-slot`, forwarding de props, `cn`, variantes CVA, portales y asociaciones accesibles generadas por shadcn. No eliminar `DialogTitle`/`DialogDescription`, labels o texto `sr-only` sin ofrecer alternativa accesible.

No añadir ni quitar `"use client"` mecánicamente. Respetar el framework, el valor `rsc` y la necesidad real de hooks/eventos. En Next.js mantener el client boundary lo más estrecho posible.

## Tokens y personalización

Usar clases semánticas ligadas a variables CSS:

```tsx
<section className="border-border bg-background text-foreground">
  <p className="text-muted-foreground">Descripción</p>
  <Button variant="destructive">Eliminar</Button>
</section>
```

Evitar colores hex/rgb y escalas Tailwind concretas para superficies de producto cuando exista un token como `background`, `card`, `popover`, `primary`, `muted`, `destructive`, `success`, `warning` o `sidebar`.

Cambiar el aspecto global en los tokens de `src/styles.css` y en las variantes base, no con overrides repetidos en features. Mantener paridad light/dark. Preservar `cn` con `clsx` + `tailwind-merge` como utility central.

Reservar utilidades CSS de layout como `container`, `sidebar` o `flex-center` para patrones globales estables. Preferir Tailwind local cuando una regla solo aparece en un componente.

Usar la biblioteca de iconos configurada en composiciones nuevas. No mezclar icon libraries por comodidad si ya existe un estándar; tolerar imports internos requeridos por una primitiva instalada y migrarlos solo con verificación visual y funcional.

## Layouts y shells

Separar tres niveles:

1. Root document/layout: `html`, `head`, `body`, CSS global, providers globales, toaster, scripts y devtools.
2. Route-group shell: admin, auth, marketing u otra zona con navegación y geometría persistentes.
3. Page/route: header de página, carga de datos y contenido del feature.

Mantener la parte visual reusable en `shared/layout` y el registro específico del framework en `routes/` o `app/`.

### TanStack Start

Usar `routes/__root.tsx` para el documento raíz. Componer shells desde layout routes:

```tsx
export const Route = createFileRoute("/admin")({
  component: AdminRouteLayout,
})

function AdminRouteLayout() {
  return (
    <div className="flex min-h-svh">
      <AppSidebar />
      <div className="min-w-0 flex-1">
        <AppBar />
        <main><Outlet /></main>
      </div>
    </div>
  )
}
```

Mantener guards de UX en route `beforeLoad`, pero no confundirlos con autorización de server functions.

### Next.js App Router

Usar `app/layout.tsx` para root providers/document y route groups para shells distintos:

```text
app/
├── layout.tsx
├── (admin)/
│   └── layout.tsx
└── (auth)/
    └── layout.tsx
```

Hacer que esos layouts compongan `shared/layout/AppSidebar`, `AppBar` o shells equivalentes y reciban `children`. Evitar marcar el root layout completo con `"use client"`; extraer providers o regiones interactivas a wrappers cliente pequeños.

### Responsabilidades del shell

Permitir que el shell posea:

- sidebar/app bar y navegación global;
- responsive navigation y disclosure móvil;
- límites de viewport, scroll y contenedor principal;
- theme switcher, cuenta/sesión y acciones globales;
- breadcrumbs globales cuando su fuente sea router/config transversal;
- skip links y landmarks accesibles.

Mantener fuera del shell:

- fetching de listados de un feature;
- formularios y mutaciones de dominio;
- mappers, entities y schemas de dominio;
- filtros de una página concreta;
- estados empty/error específicos del feature.

Usar un solo landmark `main` por shell y jerarquía semántica válida. Hacer que sidebar/nav/header sean navegables por teclado. Mantener la región de scroll deliberada para no crear scroll anidado accidental.

Modelar navegación en `shared/constants` o config equivalente. Mantener rutas, labels, iconos y capacidades requeridas; filtrar por permisos usando sesión/capabilities transversales, no imports desde features.

## Providers y boundaries globales

Montar cada provider en el boundary más estrecho que cubra a sus consumidores:

- Query, router y theme globales en root si toda la app los usa.
- Auth/admin-only providers en el shell admin si el resto no los necesita.
- Contexto exclusivo de un feature dentro del feature, no en shared/root.

Mantener Toaster, error boundary global, not-found neutral, analytics y devtools en root o integración correspondiente. Mantener skeleton/error/empty específicos dentro del feature o route.

Evitar que `shared/providers` se convierta en un único `AppProviders` opaco con dependencias circulares. Componer providers de forma explícita y documentar el orden solo cuando sea significativo.

## Checklist

### Shared

- Cada archivo es agnóstico al dominio o forma parte explícita del app shell.
- No existen imports desde shared hacia features.
- No se duplican `layout` y `layouts`, `component` y `components`.
- Props locales permanecen junto a su componente.
- Helpers son puros y lib contiene adapters/configuración, no casos de uso.

### shadcn

- `components.json` fue inspeccionado antes del CLI.
- Aliases, style, tokens, base library e iconos fueron respetados.
- Se ejecutó dry-run/diff cuando había riesgo de overwrite.
- Las personalizaciones existentes permanecen en el diff.
- Primitivas conservan accesibilidad, slots, variants y states.
- Composiciones y bloques no fueron colocados dentro de `components/ui`.

### Layouts

- Root, route shell y page tienen responsabilidades separadas.
- Layouts no importan lógica de features.
- Providers están en el boundary mínimo correcto.
- Navegación, landmarks, scroll y responsive behavior funcionan.
- Loading/error/not-found globales son neutrales; los de dominio permanecen en el feature.

### Verificación

- Typecheck, lint y build aplicables pasan.
- Se probaron light/dark y breakpoints relevantes.
- Se verificaron keyboard, focus, labels, dialogs y menus.
- El diff no contiene overwrites accidentales del CLI ni cambios ajenos.

Documentación oficial relevante:

- https://ui.shadcn.com/docs/components-json
- https://ui.shadcn.com/docs/cli
- https://ui.shadcn.com/docs/installation/vite
