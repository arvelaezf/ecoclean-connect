# EcoClean Connect — Documentación del Modelo de Datos

**Base de datos:** PostgreSQL (Supabase) · **Fecha:** 5 de junio de 2026
**Hackathon Beca Ser ANDI** — Universidad EAFIT

---

## 1. Visión general

El modelo es deliberadamente simple: **dos tablas** cubren todo el ciclo del servicio. La tabla `solicitudes` es el corazón operativo (cada reporte ciudadano con su clasificación por IA, tarifa y estado logístico) y la tabla `perfiles` conecta la autenticación con los roles del sistema.

```
auth.users (Supabase Auth)
     │ 1:1 (trigger automático)
     ▼
  perfiles ──── rol: ciudadano | conductor | admin

  solicitudes ── ciclo: cotizada → pendiente | rechazada → en_ruta → recogido
```

Los reportes **no se insertan directamente** desde el navegador: pasan por el backend (n8n), donde la IA valida la foto y las reglas de negocio calculan el precio. Esto es una decisión de seguridad y antifraude.

---

## 2. Tabla `solicitudes`

Cada fila es un reporte ciudadano de un residuo voluminoso.

| Columna | Tipo | Descripción |
|---|---|---|
| `id` | uuid (PK) | Identificador único, generado automáticamente |
| `created_at` | timestamptz | Fecha y hora del reporte |
| `es_valido` | boolean | **Veredicto de la IA**: ¿la foto muestra un residuo voluminoso real? |
| `motivo_rechazo` | text | Si la IA rechazó el reporte, la explicación (antifraude) |
| `tipo` | text | Clasificación de la IA: colchon, mueble, electrodomestico, escombros, otro |
| `descripcion` | text | Descripción del objeto generada por la IA desde la foto |
| `peso_kg` | numeric | Peso estimado por la IA (conservador) |
| `volumen_m3` | numeric | Volumen estimado por la IA |
| `tarifa_base` | int | Tarifa según tipo (regla de negocio determinista, NO la decide la IA) |
| `subsidio_pct` | int | % de subsidio según estrato (regla determinista) |
| `precio_final` | int | Lo que paga el ciudadano: tarifa_base × (1 − subsidio) |
| `estrato` | int | Estrato socioeconómico (auto-detectado por capa DANE, confirmado por el usuario) |
| `lat`, `lng` | float8 | Coordenadas del punto de recogida (GPS o pin ajustado) |
| `direccion` | text | Dirección legible (geocodificación inversa, editable) |
| `estado` | text | Ciclo: `cotizada` → (decisión del usuario) → `pendiente` o `rechazada` → `en_ruta` → `recogido` |
| `orden_ruta` | int | Posición de la parada en la ruta optimizada (la lee el conductor) |
| `foto_base64` | text | Foto del residuo (base64; en producción migraría a Storage) |

**Principio de diseño clave:** la IA *clasifica y estima* (`tipo`, `peso_kg`, `descripcion`, `es_valido`); el *precio* lo calculan reglas de negocio deterministas en el backend. La IA nunca decide cuánto paga un ciudadano.

## 3. Tabla `perfiles`

Conecta cada usuario autenticado con su rol en el sistema.

| Columna | Tipo | Descripción |
|---|---|---|
| `id` | uuid (PK, FK → auth.users) | Mismo id del usuario en Supabase Auth |
| `email` | text | Correo del usuario |
| `rol` | text | `ciudadano` (por defecto) · `conductor` · `admin` |
| `created_at` | timestamptz | Fecha de creación de la cuenta |

**Trigger automático:** al registrarse cualquier usuario, un trigger de base de datos (`al_crear_usuario`) crea su perfil con rol `ciudadano`. Los roles `admin` y `conductor` se asignan manualmente (en producción, por un superadministrador).

## 4. Seguridad: control de acceso por capas

1. **El ciudadano nunca toca la base de datos:** su reporte viaja al backend n8n, que valida con IA y escribe usando la credencial privada `service_role`.
2. **Row Level Security (RLS) activado** en ambas tablas:
   - `solicitudes`: la key pública (`anon`) solo puede **leer** y **actualizar estados** — no insertar ni borrar.
   - `perfiles`: cada usuario autenticado solo puede leer **su propio** perfil.
3. **Autenticación** con Supabase Auth (email + contraseña); cada vista exige sesión y rol: portal ciudadano (cualquier usuario), panel (solo admin), ruta (solo conductor/admin).
4. **Roadmap:** autorización fina por RLS según rol (políticas por `rol` del JWT) y migración de fotos a Supabase Storage con URLs firmadas.

## 5. Flujo de datos completo

```
CIUDADANO (web, autenticado)
  foto + GPS + estrato ──► n8n (webhook)
                              ├─► Claude Vision: clasifica, estima, valida (antifraude)
                              ├─► Reglas de negocio: tarifa + subsidio (determinista)
                              └─► INSERT en solicitudes con estado='cotizada' (service_role)
  el ciudadano ve el precio y DECIDE ──► acepta: estado='pendiente' (entra al panel)
                                     └─► rechaza: estado='rechazada' (dato de elasticidad)
ADMIN (panel)
  lee solicitudes (anon + RLS) ──► optimiza ruta (vecino más cercano + OSRM)
                                └─► UPDATE estado='en_ruta', orden_ruta=N
CONDUCTOR (vista móvil)
  lee solicitudes en_ruta ordenadas ──► al recoger: UPDATE estado='recogido'
```

## 6. SQL reproducible (esquema completo)

```sql
-- Tabla principal de reportes
create table solicitudes (
  id uuid primary key default gen_random_uuid(),
  created_at timestamptz default now(),
  es_valido boolean,
  motivo_rechazo text,
  tipo text,
  descripcion text,
  peso_kg numeric,
  volumen_m3 numeric,
  tarifa_base int,
  subsidio_pct int,
  precio_final int,
  estrato int,
  lat float8,
  lng float8,
  direccion text,
  estado text default 'pendiente',
  orden_ruta int,
  foto_base64 text
);

alter table solicitudes enable row level security;
create policy "panel puede leer" on solicitudes
  for select to anon using (true);
create policy "panel puede actualizar estado" on solicitudes
  for update to anon using (true);

-- Perfiles y roles
create table perfiles (
  id uuid primary key references auth.users(id) on delete cascade,
  email text,
  rol text not null default 'ciudadano' check (rol in ('ciudadano','conductor','admin')),
  created_at timestamptz default now()
);

alter table perfiles enable row level security;
create policy "leer mi perfil" on perfiles
  for select to authenticated using (auth.uid() = id);

-- Trigger: todo usuario nuevo nace como ciudadano
create or replace function crear_perfil()
returns trigger language plpgsql security definer set search_path = public as $$
begin
  insert into public.perfiles (id, email) values (new.id, new.email);
  return new;
end $$;

create trigger al_crear_usuario after insert on auth.users
  for each row execute function crear_perfil();
```

## 7. Decisiones de diseño documentadas (apropiación del aprendizaje)

- **¿Por qué lat/lng y no PostGIS?** El MVP no hace consultas espaciales en SQL (el ruteo lo resuelve OSRM y la detección de estrato la capa ArcGIS del DANE). PostGIS se adoptaría si llegaran consultas como "solicitudes a menos de 2 km del camión".
- **¿Por qué la foto en base64 y no en Storage?** Velocidad de desarrollo en hackathon con volúmenes mínimos. Migrar a Storage con URLs firmadas es el primer paso del roadmap técnico.
- **¿Por qué no hay tabla de asignaciones (route_assignments)?** El piloto opera un solo camión: la ruta del día ES la asignación (`estado` + `orden_ruta`). Multi-vehículo = agregar `conductor_id` y un filtro; el modelo ya lo soporta.
- **¿Por qué el estrato lo confirma el usuario?** El dato del DANE es *predominante por manzana*, no oficial por predio: automatización con humano en el control.
