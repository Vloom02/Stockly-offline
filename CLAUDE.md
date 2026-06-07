# Stockly — Guía para Claude

> ⭐ **Stockly es la app PRIORITARIA.** Perfeccionarla antes de avanzar con la app de ventas
> (`almacen-ventas-final`). El objetivo: que funcione sólida para generar datos reales que
> después sirvan de referencia en ventas.

## Qué es
App **offline-first** multi-tenant de control de inventario y vencimientos para comercios.
Comparte el proyecto Supabase con la app de ventas.

## Stack
- Ionic React 8 · TypeScript 5.9 · Vite 5 · Capacitor 8 · React 19
- Backend: Supabase (Postgres 17). Proyecto id `diytllvtweghnagqjnrc`.
- Persistencia local: IndexedDB (`src/lib/localDb.ts`). Sync en `src/lib/sync.ts`.
- Estado: Context + useReducer (`src/context/StoreContext.tsx`, `AuthContext.tsx`).

## Comandos
- `npm run dev` — servidor de desarrollo (http://localhost:5173)
- `npm run build` — tsc + vite build
- `/apk` — generar APK Android (build → cap sync → abrir Android Studio)

## Arquitectura de datos
- Tablas: `comercios`, `miembros` (rol dueno|empleado), `sucursales`, `productos`
  (catálogo compartido por comercio), `lotes` (stock real + vencimiento por sucursal),
  `movimientos` (audit log).
- **Tipos de la DB**: `src/types/database.types.ts` (generados, NO editar a mano).
  Tipos de dominio camelCase en `src/types/index.ts`, mapeados en `sync.ts`.
- RLS multi-tenant vía `comercios_del_usuario()`. Ya auditado y endurecido (2026-06):
  escritura de catálogo solo rol `dueno`, search_path fijo, sin policy de auto-inserción
  en `miembros`. El `schema.sql` refleja el estado seguro.

## Convenciones
- Responder en español (es-AR).
- Offline-first: toda escritura va primero a IndexedDB y se encola en `outbox`.
- Nunca tocar `.env` (bloqueado por permisos).
- Subagentes disponibles: `revisor-sync`, `revisor-rls`, `revisor-ionic`, `escritor-tests`.

## ⚠️ Deuda técnica conocida (auditada 2026-06) — pendiente de arreglar
1. **Sync sin `updated_at` real** → el merge es "el último que sincroniza gana", un device
   con datos viejos pisa datos nuevos. Falta columna `updated_at` + comparación.
2. **`bajarCambios` hace full-pull** de productos/lotes en cada sync (ineficiente al crecer).
3. **`sincronizar` sin guarda anti-solapamiento** (puede correr 2 veces a la vez).
4. **Conflicto cruzado con ventas**: Stockly hace upsert del lote con `cantidad` absoluta y
   puede pisar el descuento de stock hecho por la app de ventas. Tocar `cantidad` por delta.
5. Sin tests, sin Sentry, sin CI, IndexedDB crudo (candidato a Dexie), fetching manual
   (candidato a TanStack Query).

## Roadmap (orden acordado)
1. ✅ Tipos generados desde Supabase (`src/types/database.types.ts`)
2. ✅ Tests (Vitest): lógica pura en `src/lib/vencimientos.ts` + `plan.ts` (17 tests). `npm test`
3. ⏳ Arreglar sync (updated_at + incremental + guarda anti-solapamiento + delta de stock) ← SIGUIENTE
4. Migrar IndexedDB → Dexie / fetching → TanStack Query
5. Sentry + CI (GitHub Actions)
6. Funcionalidades: liquidación automática FEFO, push notifications reales (FCM)
7. (Mercado Pago / cobro: lo ÚLTIMO de todo)

## Testing
- `npm test` (vitest run) · `npm run test:watch`
- Solo lógica PURA por ahora. La lógica testeable se extrae a `src/lib/*.ts` sin React/Supabase.
