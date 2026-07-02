# Pacas de Bogotá — Requerimientos v1 (para generador de repo)

<aside>
🌱

App web que mapea y mide las pacas digestoras activas de Bogotá: dónde están, cuánto residuo orgánico procesan y cuántas semanas llevan. Cualquiera contribuye sin login; el admin aprueba antes de publicar.

</aside>

## Stack

- **Frontend:** Next.js (App Router) + TypeScript en **Vercel**
- **Backend:** **Supabase** (Postgres + PostGIS + Storage + Auth)
- **Mapa:** Leaflet + tiles OpenStreetMap
- **Gráficas:** Chart.js
- **Estilos:** Tailwind CSS
- **i18n / TZ:** UI en español (Bogotá), timezone `America/Bogota`

## Modelo de datos (Supabase / Postgres + PostGIS)

### `places` (parques y lugares públicos)

Entidad de primera clase para agrupar, navegar y filtrar pacas por parque.

- `id` (uuid, pk)
- `name` (text) — ej. "Parque Portugal"
- `slug` (text, unique) — para rutas `/parque/parque-portugal`
- `localidad` (text) — ej. "Chapinero", "Suba"
- `center` (geography(Point)) — centro del parque para encuadrar el mapa
- `boundary` (geography(Polygon), nullable) — opcional, para validar que una paca cae dentro
- `description` (text, nullable)
- `created_at` (timestamptz, default now())

### `pacas`

- `id` (uuid, pk)
- `place_id` (uuid, fk → places)
- `location` (geography(Point)) — punto exacto dentro del parque
- `exact_location_visible` (bool, default true) — si false, el mapa usa `places.center`
- `steward_alias` (text, nullable) — alias opcional, nunca datos personales
- `start_date` (date) — para calcular edad en semanas
- `status` (text, check in `active` | `harvested` | `abandoned`, default `active`)
- `harvest_date` (date, nullable)
- `method` (text, nullable)
- `notes` (text, nullable)
- `moderation_status` (text, check in `pending` | `approved` | `rejected`, default `pending`)
- `created_at` / `updated_at` (timestamptz)

### `updates` (bitácora de eventos por paca)

Cada entrada es un evento inmutable: peso añadido, cambio de estado o nota.

- `id` (uuid, pk)
- `paca_id` (uuid, fk → pacas)
- `type` (text, check in `weight` | `status_change` | `note`)
- `weight_kg` (numeric, nullable) — requerido cuando `type = 'weight'`
- `new_status` (text, nullable) — usado cuando `type = 'status_change'`
- `note` (text, nullable)
- `moderation_status` (text, check in `pending` | `approved` | `rejected`, default `pending`)
- `created_at` (timestamptz, default now())

### `photos`

- `id` (uuid, pk)
- `paca_id` (uuid, fk → pacas)
- `update_id` (uuid, fk → updates, nullable) — foto ligada a un evento concreto
- `storage_url` (text) — Supabase Storage
- `caption` (text, nullable)
- `created_at` (timestamptz)

### Índices y campos derivados

- **Índices:** GIST en `pacas.location` y `places.center`/`boundary`; btree en `pacas.status`, `pacas.moderation_status`, `updates.paca_id`, `updates.created_at`.
- **Derivados (en query, no almacenados):** edad en semanas = `now() - start_date`; kg totales de una paca = `SUM(updates.weight_kg)` con `type='weight'` y `moderation_status='approved'`.
- Todo lo público filtra siempre `moderation_status = 'approved'`.

---

## Requerimientos funcionales

### RF-1 · Mapa interactivo (home)

- Mapa de Bogotá con un dot por paca **activa y aprobada**.
- Color del dot = edad en semanas; tamaño = kg totales acumulados.
- Click en un dot → popup con nombre del parque, edad, kg y enlace a la ficha.
- Si `exact_location_visible = false`, el dot se ubica en `places.center`.
- El mapa reacciona a los filtros activos (RF-3).
- **Aceptación:** al cargar, solo se ven pacas `status='active'` y `moderation_status='approved'`.

### RF-2 · Score cards

1. **Hero semanal:** *"Esta semana salvamos ## kg de llegar al Doña Juana!"* = `SUM(updates.weight_kg)` de updates aprobados con `created_at` dentro de la semana ISO actual (reset lunes 00:00, hora Bogotá).
2. **Pacas activas:** conteo de pacas `status='active'` y aprobadas.
3. **kg de residuos en proceso:** suma all-time de `weight_kg` de updates aprobados **solo** de pacas activas (las cosechadas se excluyen por completo).
- Las cards respetan los filtros activos.
- **Aceptación:** marcar una paca como cosechada la retira al instante de las cards 2 y 3.

### RF-3 · Filtros globales

- Filtrar por **parque/localidad**, **estado** y **rango de fechas**.
- Afectan simultáneamente mapa y cards.
- Estado por defecto: solo activas aprobadas.
- **Aceptación:** cambiar un filtro actualiza mapa y cards sin recargar la página.

### RF-4 · Navegación por parque

- Ruta `/parque/[slug]`: ficha del parque con lista de sus pacas activas, kg agregados, conteo y mini-mapa centrado en `places.center`.
- Autocomplete/búsqueda de parques.
- **Aceptación:** desde un dot o desde la búsqueda se puede llegar a la vista del parque y ver solo sus pacas aprobadas.

### RF-5 · Ficha de paca `/paca/[id]`

- Timeline de updates **aprobados** (peso, notas, cambios de estado), galería de fotos, kg totales, edad en semanas, estado actual, parque y alias (si existe).
- **Aceptación:** updates pendientes no se muestran en la ficha pública.

### RF-6 · Contribuir (público, sin login → entra como `pending`)

- **6.1 Añadir paca:** elegir parque (autocomplete sobre `places`; si no existe → "sugerir parque nuevo", también `pending`) → poner pin dentro del parque → fecha de inicio → foto y/o peso inicial → toggle "mostrar ubicación exacta". Confirmación "en revisión".
- **6.2 Registrar actualización:** desde la ficha → añadir peso (kg) y/o foto, o nota.
- **6.3 Marcar como cosechada:** crea un update `type='status_change'`, `new_status='harvested'`.
- El público **solo crea**, nunca edita ni borra registros existentes.
- **Campos requeridos al crear paca:** parque, `start_date`, y al menos una foto o un peso.
- **Aceptación:** todo envío público nace con `moderation_status='pending'` y no aparece en mapa/cards/fichas hasta ser aprobado.

### RF-7 · Admin (protegido con Supabase Auth — solo el admin)

- **7.1 Login** con Supabase Auth; rutas `/admin/*` inaccesibles sin sesión.
- **7.2 Cola de revisión:** lista de pacas y updates `pending` con foto y detalle; botones **Aprobar** / **Rechazar** (aprobar publica al instante).
- **7.3 Editar cualquier registro:** parque, ubicación, fecha, peso, estado, alias, notas.
- **7.4 Cambiar estado** (active / harvested / abandoned) y fijar `harvest_date`.
- **7.5 Gestionar parques:** crear/editar `places`, aprobar parques sugeridos.
- **Aceptación:** un usuario anónimo que intente entrar a `/admin` es redirigido al login.

---

## RD · Requerimientos de datos y validación

- Pesos: `weight_kg > 0`; rechazar vacíos o texto.
- Pin obligatorio dentro del parque (si hay `boundary`, validar contención con PostGIS; si no, aceptar el `center` del parque).
- Fechas no futuras (`start_date <= today`, `harvest_date >= start_date`).
- Fotos: subir a Supabase Storage; límite de tamaño (p. ej. ≤ 5 MB), formatos jpg/png/webp.
- Edad y kg totales se calculan en query, no se almacenan.

## RNF · No funcionales

- **Seguridad (RLS):** público puede `INSERT` en `pacas`/`updates`/`photos` como `pending` y `SELECT` solo `approved`; cambios de `moderation_status` y edición/borrado solo para el rol admin.
- **Rendimiento:** índice GIST en columnas geográficas; carga inicial del mapa < 2 s con cientos de pacas.
- **Responsive:** móvil primero (mucha gente contribuye desde el celular en el parque).
- **Accesibilidad e idioma:** textos en español, contraste legible, botones grandes para móvil.
- **Deploy:** frontend en Vercel; variables de entorno para claves Supabase.

## Fuera de alcance v1 (iterar luego)

- Refinamiento del dashboard (tiempo promedio de cosecha, series temporales, leaderboards).
- Bot de WhatsApp + parsing con IA.
- Identidad anónima-pero-trazable (hash de teléfono) y log de auditoría.
- Integración Wikimedia Commons y ángulo de financiación.
- Detección automática de outliers y merge de duplicados.
