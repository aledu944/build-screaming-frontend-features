---
name: build-screaming-frontend-features
description: >-
  Crear, extender, refactorizar y revisar frontends React con TypeScript organizados por módulos de dominio mediante Screaming Architecture. Usar al iniciar un proyecto o implementar features CRUD, listados, detalles, formularios, filtros y mutaciones que necesiten separar components, hooks, Zod schemas, entities, mappers, contratos externos en types y server functions/actions; también al diseñar shared, integrar primitivas shadcn, componentes reutilizables, providers y layouts. Aplicar tanto a TanStack Start como a Next.js App Router, adaptándose primero a las convenciones reales del repositorio.
---

# Build Screaming Frontend Features

Construir slices verticales cuyo árbol y vocabulario expresen el negocio. Mantener separados el modelo interno, los contratos externos, la validación en runtime, la adaptación de datos, el acceso al servidor, la lógica reutilizable y la presentación.

## Preparar el trabajo

1. Leer todas las instrucciones `AGENTS.md` aplicables antes de editar.
2. Inspeccionar `package.json`, alias, TypeScript, router, fetching, formularios, validación, tests y manejo de errores.
3. Buscar el módulo análogo más completo y tratarlo como convención local si es que existe. No asumir que la convención del proyecto es la misma que la de un ejemplo genérico.
4. Identificar el framework sin adivinar:
   - Para TanStack Start, leer [tanstack-start.md](references/tanstack-start.md).
   - Para Next.js App Router, leer [nextjs-app-router.md](references/nextjs-app-router.md).
   - Para otro framework, conservar las capas de dominio y adaptar únicamente sus límites de transporte y routing.
5. Leer siempre [architecture.md](references/architecture.md). Leer [feature-blueprint.md](references/feature-blueprint.md) al crear un módulo o una vertical completa.
6. Leer [shared-ui-layouts.md](references/shared-ui-layouts.md) al iniciar el proyecto o tocar `shared/`, shadcn, estilos/tokens, providers, navegación, shells o layouts.

Si algún archivo de `references/` no existe en la instalación local (instalación parcial), no continuar sin él: obtenerlo desde `https://raw.githubusercontent.com/aledu944/build-screaming-frontend-features/main/references/<archivo>` y avisar al usuario de que reinstale con `npx skills add aledu944/build-screaming-frontend-features`.

No reemplazar dependencias, estilos globales, primitivas UI o estrategia de datos sin que la tarea lo exija. Preservar cambios ajenos y limitar la edición al slice solicitado.

## Diseñar el slice

Definir antes de crear archivos:

- nombre plural del dominio, por ejemplo `patients`, `appointments` o `invoices`;
- casos de uso concretos: listar, filtrar, obtener detalle, crear, actualizar, cambiar estado;
- fuentes de datos: REST, GraphQL, ORM/DB, SDK, storage o varias a la vez;
- entidad estable que consumirá la UI;
- contratos de entrada/salida y schemas que protegen cada límite;
- política de errores, autorización, caché e invalidación ya usada por el proyecto.
- impacto sobre shared, primitivas UI, navegación, providers y layouts; mantenerlo vacío si no existe impacto real.

Crear solamente carpetas con una responsabilidad real. No generar directorios vacíos ni capas de una sola línea que no protejan un límite, permitan sustitución o eliminen duplicación.

## Implementar de adentro hacia afuera

Seguir este orden y mantener cada archivo enfocado:

1. Crear `constants/` para valores de dominio compartidos dentro del feature.
2. Crear `entities/` con el modelo normalizado y estable que usa la aplicación.
3. Crear `types/` con contratos externos o técnicos no inferibles: responses REST, filas de DB/SDK, paginación o envelopes.
4. Crear `schemas/` para validar entradas reales en runtime. Inferir sus tipos con `z.input`/`z.output`; no duplicarlos manualmente en `types/`.
5. Crear `mappers/` puros para convertir cada contrato de origen a la entidad y, cuando haga falta, la entidad/input al payload de destino.
6. Crear `server/`, `actions/` o la convención equivalente para I/O, autorización, validación, mapeo y errores.
7. Crear hooks por workflow: plural para listado/orquestación y hooks separados para crear, actualizar, eliminar o cambiar estado.
8. Crear componentes de feature delgados que rendericen entidades y deleguen workflows a hooks.
9. Mantener routes/pages como composition roots: cargar datos, validar parámetros de URL y componer componentes; evitar lógica de negocio extensa.
10. Reutilizar primitivas desde `shared/components/ui`; crear una composición en `shared/components` o `shared/layout` solo cuando sea neutral al dominio y tenga alcance compartido o de app shell.

Completar una vertical funcional antes de abrir otra. Conectar lectura, formulario, mutación, feedback, invalidación y estados pending/error/empty cuando apliquen.

## Aplicar reglas no negociables

- Evitar que un response de API, fila ORM o tipo de SDK llegue directamente a componentes.
- Ejecutar mappers en el límite de entrada, no dentro de la vista.
- Mantener mappers deterministas, síncronos y sin I/O.
- Validar siempre forms, URL params y entradas de mutación en runtime; validar también responses externas cuando la fuente, el riesgo o la convención del proyecto lo requieran. TypeScript por sí solo no valida.
- Tratar cada server function/action como endpoint público: autenticar y autorizar dentro del handler o middleware.
- Mantener los hooks libres de JSX y los componentes libres de detalles de transporte.
- Nombrar hooks por intención: `usePatients`, `usePatientsFilters`, `useCreatePatient`, `useUpdatePatient`, `usePatientStatus`.
- Conservar estado de UI efímero local al componente; mover a hook la lógica reutilizable o el workflow con efectos.
- Subir código a `shared/` solo cuando sea verdaderamente agnóstico al dominio y usado por más de un feature.
- Mantener la dirección `features -> shared`; impedir que `shared` importe entities, hooks, server functions o componentes de un feature.
- Tratar `shared/components/ui` como la capa de primitivas shadcn configurada por `components.json`; no instalar componentes en rutas arbitrarias ni sobrescribir personalizaciones sin revisar el diff.
- Mantener layouts como shells de navegación, providers y regiones persistentes; no mover workflows de dominio al layout.
- Evitar barrels globales que oculten ciclos. Mantener imports explícitos o seguir el patrón local.
- No usar `any`, non-null assertions o casts para silenciar contratos mal modelados.

## Verificar

Revisar la dirección de dependencias y ejecutar los comandos disponibles de typecheck, lint, tests y build. Añadir pruebas focalizadas para schemas, mappers y workflows con lógica relevante.

Confirmar antes de entregar:

- árbol del feature legible por dominio;
- inputs y search params validados;
- responses externos tipados y mapeados;
- fechas, nullables, enums y campos derivados normalizados;
- autorización presente en límites privados;
- formularios conectados al schema correcto;
- invalidación o revalidación después de mutaciones;
- estados de carga, error, vacío y éxito tratados;
- routes/pages sin absorber responsabilidades de hooks o servidor.
- componentes shadcn instalados en los aliases configurados y personalizaciones previas conservadas;
- shared libre de imports desde features y layouts libres de lógica de dominio;
- providers, toaster, theme y devtools montados en el boundary mínimo correcto.

Resumir los archivos creados, el flujo `source -> contract -> schema -> mapper -> entity -> UI`, las decisiones específicas del framework y las verificaciones ejecutadas.
