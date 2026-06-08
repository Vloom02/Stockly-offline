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

## Compilar el APK desde consola (sin abrir Android Studio)
- El `java` del sistema es la v26 (incompatible con Gradle). Usar el JDK de Android Studio:
  `JAVA_HOME = C:\Program Files\Android\Android Studio\jbr` (JDK 21).
- Pasos: `npm run build` → `npx cap sync android` → en `android/`: `.\gradlew.bat assembleDebug --no-daemon`.
- APK queda en `android/app/build/outputs/apk/debug/app-debug.apk`.

## ⚠️ Android: permiso de CÁMARA (escáner)
`android/` está en `.gitignore`. El `AndroidManifest.xml` DEBE incluir, además de INTERNET:
`<uses-permission android:name="android.permission.CAMERA" />` y
`<uses-feature android:name="android.hardware.camera" android:required="false" />`.
Sin eso, el escáner da "Permiso de cámara denegado" (Android ni muestra el cartel).
Si se regenera la carpeta android, volver a agregarlo.

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
2. ✅ Tests (Vitest): `vencimientos.ts` + `plan.ts` + `liquidacion.ts` (27 tests). `npm test`
3. 🟡 Sync: ✅ updated_at + pull incremental + guarda anti-solapamiento.
   ⏳ PENDIENTE 3d: descuento de stock por DELTA atómico (hoy upsert pisa cantidad).
   Requiere RPC transaccional compartida con ventas → hacerlo junto al refactor de ventas.
4. 🟡 Performance/refactor:
   ✅ Code-splitting: páginas con React.lazy + Suspense, vendors en chunks (vite manualChunks).
      Bundle propio 1375→~230 KB; Ionic/Supabase/Sentry/React separados y cacheados.
   ⏳ Migrar IndexedDB → Dexie / fetching → TanStack Query (refactor grande, REQUIERE probar en la app)
5. ✅ CI (GitHub Actions) · ✅ repo en GitHub · ✅ Sentry (`src/lib/sentry.ts`, init en
   main.tsx + ErrorBoundary; solo reporta en PROD/APK). DSN overrideable con VITE_SENTRY_DSN.
6. ✅ Liquidación FEFO: lógica pura (`liquidacion.ts`) + chip en StockPage + sugerencia en
   LotePage + ✅ descuentos % configurables desde Ajustes (settings desc-critico/urgente/aviso,
   getters DESC_* y descuentoConfig() en StoreContext).
   ✅ Push (FCM) COMPLETO:
      - Cliente: @capacitor/push-notifications + src/lib/push.ts (registra token al loguear).
      - Firebase: proyecto stockly-1598d, google-services.json en android/app/ (gitignored).
      - DB: tabla push_tokens con RLS. Secreto FCM_SERVICE_ACCOUNT en Supabase.
      - Envío: Edge Function `supabase/functions/avisos-vencimiento` (OAuth v1 + FCM).
      - Cron: pg_cron 'avisos-vencimiento-diario' 0 12 * * * (9 AM ARG). Probado: OAuth OK.
      Para re-desplegar la function: Management API multipart (el MCP está read-only).
7. (Mercado Pago / cobro: lo ÚLTIMO de todo)

## Estado git
- Repo: **https://github.com/Vloom02/Stockly-offline** (privado, rama `main`).
- CI de GitHub Actions corriendo y en verde (typecheck + tests + build).
- Para subir cambios: `git push`. `gh` está instalado y logueado (cuenta Vloom02).

## Testing
- `npm test` (vitest run) · `npm run test:watch`
- Solo lógica PURA por ahora. La lógica testeable se extrae a `src/lib/*.ts` sin React/Supabase.
