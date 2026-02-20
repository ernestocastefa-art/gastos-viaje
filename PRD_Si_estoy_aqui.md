# PRD — “Sí, estoy aquí”

## 1) Rol del asistente (meta prompting)
Eres un desarrollador de software experto en React, TypeScript, Tailwind CSS, Supabase, VITE y en el entorno nativo de Lovable. Tu enfoque debe ser construir la aplicación con la máxima precisión y adherencia estricta a todas las restricciones de diseño y técnicas proporcionadas.

## 2) Descripción general y visión
Aplicación móvil‑first para **registro horario** de una empresa (~10 trabajadores) en **teletrabajo**, donde los trabajadores fichan de forma ultra simple al entrar y salir, y el administrador (único) puede auditar, controlar cumplimiento del horario, gestionar usuarios y exportar informes para inspección.

**Objetivo principal (MVP):**
Entregar autenticación completa (autorregistro/login), flujo de fichaje **Entrada/Salida** con guardado de **GPS**, histórico del trabajador, panel de administrador con estadísticas (día/semana/mes + horas extra), gestión de perfiles/estado de cuentas, flujo de **solicitud y aprobación de correcciones**, y exportación **CSV/PDF** con retención de datos **5 años**.

## 3) Supuestos del MVP (decisiones ante ambigüedades)
- **Sin pausas en MVP**: el flujo de fichaje se limita a entrada/salida, pero el modelo de datos queda preparado para pausas futuras.
- **Geolocalización interna, no exportada por defecto**: se guardan coordenadas exactas en cada fichaje, pero **no** se incluyen en exportaciones salvo que el admin active un toggle.
- **Horario base 08:00–15:00 L–V**: las “recuperaciones” se reflejan como fichajes fuera de ventana y se computan como **horas extra** (parametrizable por admin).

## 4) Stack y restricciones técnicas
- **Frontend**: React, TypeScript, Tailwind CSS.
- **Diseño**: Mobile‑first con breakpoints estándar de Tailwind y ShadCN.
- **Backend**: Supabase (PostgreSQL + Auth + Storage).

**Restricciones clave:**
- App muy orientada a móvil: acciones principales a 1 toque.
- Idioma: **Español**.
- Sin integraciones externas en MVP.
- Captura de **geolocalización** (coordenadas exactas) en cada fichaje.
- Retención: **5 años**.

## 5) Arquitectura de datos y flujo

### Modelos de datos clave (Supabase/Postgres)

**`profiles`**
- `id` (uuid, PK, = auth.users.id)
- `full_name` (text)
- `dni` (text, único)
- `role` (enum: `worker` | `admin`)
- `is_active` (bool, default true)
- `created_at`
- Opcionales: `job_title`, `notes`

**`work_schedule_rules`**
- `id` (uuid, PK)
- `day_of_week` (int 1–5)
- `start_time` (time) default 08:00
- `end_time` (time) default 15:00
- `tolerance_minutes` (int) default 0 o 5 (definir)
- `active` (bool)

> Nota: reglas globales (empresa) para MVP.

**`time_entries`**
- `id` (uuid, PK)
- `user_id` (uuid, FK profiles.id)
- `type` (enum: `clock_in` | `clock_out` | `break_start` | `break_end`) *(MVP usa solo `clock_in`/`clock_out` pero el enum queda preparado)*
- `timestamp` (timestamptz)
- `latitude` (numeric)
- `longitude` (numeric)
- `source` (enum: `manual` | `correction_approved`) default `manual`
- `created_at`

**`correction_requests`**
- `id` (uuid, PK)
- `user_id` (uuid, FK)
- `date` (date)
- `requested_changes` (jsonb)
- `reason` (text, obligatorio)
- `status` (enum: `pending` | `approved` | `rejected`)
- `admin_comment` (text, opcional)
- `reviewed_by` (uuid, FK profiles.id)
- `created_at`, `reviewed_at`

**`audit_log`**
- `id`
- `actor_id` (uuid)
- `action` (text) (ej: “APPROVE_CORRECTION”, “DEACTIVATE_USER”, “EDIT_PROFILE”)
- `entity_type` (text)
- `entity_id` (uuid)
- `metadata` (jsonb)
- `created_at`

### Cálculo de horas (lógica)
- **Por día**: emparejar primer `clock_in` y último `clock_out` del día (MVP simple), con validaciones para evitar inconsistencias.
- **Horas normales vs extra**: ventana base 08:00–15:00 L–V según `work_schedule_rules`; fichajes fuera de ventana computan como extra.
- **Cumplimiento**: comparar `clock_in` con `start_time` (+tolerancia) y `clock_out` con `end_time`.

### RLS (Row Level Security) propuesta
**`profiles`**
- Worker: puede leer/editar solo su perfil (excepto `role`, `is_active`).
- Admin: puede leer/editar todos.

**`time_entries`**
- Worker: CRUD solo de sus entradas (MVP: crear y leer; editar/borrar solo vía corrección aprobada).
- Admin: leer todo.

**`correction_requests`**
- Worker: crear y leer las suyas.
- Admin: leer y actualizar estado de todas.

**`audit_log`**
- Solo admin puede leer; escritura desde funciones/servidor o políticas controladas.

## 6) Flujo de usuario detallado

### Pantallas/Páginas (Worker)
- Onboarding/Bienvenida (breve)
- Registro (autorregistro) + Login
- Home “Fichaje” (pantalla principal)
- Histórico (lista por días + detalle)
- Solicitar corrección (formulario)
- Perfil (ver datos; editar básicos si se permite)

### Pantallas/Páginas (Admin)
- Admin Dashboard (KPIs + filtros día/semana/mes)
- Trabajadores (lista + activar/desactivar + editar perfil + ver detalle)
- Detalle trabajador (calendario/lista + horas + incidencias)
- Correcciones (bandeja pending + aprobar/rechazar + comentarios)
- Exportación (rango de fechas + formato CSV/PDF + selección de campos)
- Ajustes (horario base, tolerancia, retención/avisos)

### Navegación (paso a paso)
**Worker**
1. Abre app → (si no logueado) Login/Registro.
2. Home muestra botón principal contextual:
   - Sin `clock_in` hoy: **“Fichar entrada”**.
   - Con entrada y sin salida: **“Fichar salida”**.
3. Al pulsar: solicita permisos de ubicación → obtiene coordenadas → guarda `time_entries`.
4. Histórico → ver días/horas.
5. Corrección → “Solicitar corrección” → motivo + cambios → enviar.

**Admin**
1. Login → Dashboard (hoy/semana/mes).
2. Trabajadores → detalle semanal/mensual + incidencias.
3. Correcciones → aprobar/rechazar con comentario.
4. Exportación → rango + CSV/PDF → descargar.

### Estados (vacío/carga/error)
- Sin fichajes hoy → CTA “Fichar entrada”.
- Sin histórico → vacío con explicación.
- GPS denegado → aviso con pasos; MVP recomendado: **bloquear** fichaje.
- Sin conexión → error y reintentar (cola local opcional).

## 7) Funcionalidades clave y orden de implementación

**1) Diseño frontend (Mobile‑first)**
- Home minimalista con botón grande (Entrada/Salida).
- Histórico (lista + detalle).
- Admin (dashboard, tablas, filtros).
- Formularios: registro/login, correcciones, perfil.

**2) Conexión backend (Supabase)**
- Auth (registro/login/logout).
- Tablas y RLS.
- CRUD controlado para fichajes y solicitudes.

**3) Lógica de negocio**
- Captura de GPS en cada fichaje.
- Validaciones: no permitir salida sin entrada, evitar dobles entradas, etc.
- Correcciones: workflow `pending → approved/rejected`.
  - Límite: **máx 4 solicitudes/mes** por trabajador (enforced server‑side).
  - Motivo obligatorio.
  - Aprobación admin crea/ajusta `time_entries` con `source = correction_approved` y registra en `audit_log`.
- Cálculo de horas por día/semana/mes + horas extra + cumplimiento.
- Exportación CSV/PDF (rango, por trabajador o global).

## 8) Integraciones y lógica externa
- **Sin integraciones externas** en MVP.
- Supabase:
  - Auth: email/password (o magic link si se decide).
  - DB: Postgres.
  - (Opcional) Edge Functions para PDF y cálculos.

## 9) Lineamientos de diseño UI/UX
- Estilo: **minimalista**, elegante.
- Paleta: base **crema** + acentos **dorado** + texto oscuro.
- Componentes: ShadCN (botones grandes, cards, tabs, tables).
- Home: 1 acción principal por pantalla, accesible con el pulgar.
- Admin: layout tipo dashboard con cards métricas + tabla + filtros.
- Accesibilidad: texto legible, estados de error claros, confirmaciones suaves.
- Microcopy en español (ej. “Fichar entrada”, “Fichar salida”, “Solicitud enviada”).

## 10) Alcance del proyecto (scope)

**Incluido:**
- Autorregistro/login.
- Roles: worker y **un único admin**.
- Fichaje entrada/salida con GPS exacto.
- Histórico personal.
- Solicitud de correcciones con motivo obligatorio y **límite 4/mes**.
- Aprobación/rechazo por admin + auditoría.
- Panel admin: vistas día/semana/mes, horas extra, cumplimiento.
- Gestión de usuarios: editar perfil + desactivar/reactivar.
- Exportación **CSV y PDF** (sin ubicación por defecto).
- Retención: **5 años**.

**Excluido:**
- Integraciones externas.
- Multi‑admin.
- Gestión avanzada de turnos por centro/empleado.
- Pausas operativas en UI (queda preparada en datos).
- Firma digital/biometría.

## 11) Nota final
Antes de generar código, el modelo debe leer este documento y confirmar su entendimiento en modo conversación (Chat Mode).
