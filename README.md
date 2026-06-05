# ♻️ EcoClean Connect

**Plataforma con IA generativa para el reporte, cotización subsidiada y recolección de residuos voluminosos.**

Construida en un día durante la **Hackathon de la Beca Ser ANDI** (Universidad EAFIT, Medellín — 5 de junio de 2026).

> Los camiones de basura no recogen colchones, muebles ni escombros. Sin un canal formal, estos residuos terminan en esquinas, parques y ríos. EcoClean Connect es el puente digital entre el ciudadano y las empresas recolectoras autorizadas — con la IA haciendo el trabajo difícil.

## 🚀 Demo en producción

| Vista | URL | Rol |
|---|---|---|
| Portal ciudadano | `ecocleanbsa.netlify.app/ciudadano.html` | Cualquier usuario registrado |
| Panel del despachador | `ecocleanbsa.netlify.app/panel.html` | Solo administradores |
| Ruta del conductor | `ecocleanbsa.netlify.app/conductor.html` | Solo conductores |
| Login / registro | `ecocleanbsa.netlify.app/login.html` | — |

## ✨ Qué hace

1. **El ciudadano fotografía el residuo** desde su celular (GPS + pin ajustable + dirección automática).
2. **La IA de visión (Claude) lo clasifica**: tipo, peso y volumen estimados, descripción — y **rechaza reportes falsos** explicando el motivo.
3. **El estrato se detecta automáticamente** con la capa de estratificación por manzana del DANE (el usuario confirma).
4. **Cotización instantánea con subsidio**: la IA clasifica, pero el precio lo calculan **reglas de negocio deterministas** — la IA nunca decide cuánto paga un ciudadano.
5. **El despachador optimiza la ruta**: orden de paradas por vecino más cercano + recorrido real por las calles (OSRM) con km y minutos.
6. **El conductor ejecuta** desde su celular: paradas en orden, navegación a Google Maps y marcación de recogidas.
7. **EcoBot**, asistente conversacional con IA, resuelve las dudas del ciudadano sobre el servicio.

## 🏗️ Arquitectura

```
Ciudadano (foto + GPS) ──► n8n (webhook) ──► Claude Vision (clasifica y valida)
                                │                    │
                                ▼                    ▼
                        Reglas de negocio ──► Supabase (PostgreSQL + Auth + RLS)
                        (tarifa + subsidio)          ▲
                                                     │
        Panel admin (ruta: vecino más cercano + OSRM) ┤
        Vista conductor (paradas ordenadas) ──────────┘
```

**Principio rector:** la IA *clasifica y estima*; las reglas *deciden el precio*. Seguridad por capas: el ciudadano nunca escribe directo en la base de datos — todo reporte pasa por el backend, donde la IA lo valida primero.

## 🧰 Stack

- **IA generativa:** Claude (Anthropic) — visión para clasificación/antifraude y chat para EcoBot
- **Backend sin servidor:** n8n (workflows `ecoclean-clasificador-n8n.json` y `ecoclean-chatbot-n8n.json`)
- **Datos y auth:** Supabase (PostgreSQL, Auth con roles ciudadano/conductor/admin, RLS) — ver [`modelo-de-datos.md`](modelo-de-datos.md)
- **Mapas:** Leaflet + OpenStreetMap · ruteo por calles con OSRM · geocodificación con Nominatim · estratos con capa DANE/Esri Colombia
- **Frontend:** HTML/CSS/JS vanilla en Netlify (sin frameworks, carga instantánea en cualquier celular)

## ⚙️ Cómo correrlo

1. **Supabase:** crea un proyecto y ejecuta el SQL de [`modelo-de-datos.md`](modelo-de-datos.md) (tablas, políticas y trigger). Desactiva "Confirm email" en Authentication para registro instantáneo.
2. **n8n:** importa los dos workflows JSON, pon tu API key de Anthropic en los headers `x-api-key`, conecta la credencial de Supabase y actívalos.
3. **Frontend:** edita las constantes `SUPABASE_URL`, `SUPABASE_KEY` (anon) y las URLs de webhook en los HTML, y súbelos junto a `logo.png` y `video.mp4` a Netlify (o cualquier hosting estático).
4. Promueve los roles: `update perfiles set rol='admin' where email='...';`

> ⚠️ La API key de Anthropic vive **solo en n8n** — nunca en el frontend ni en este repo.

## 🛡️ Ética y salvaguardas

- Antifraude con IA (rechazo de fotos que no muestran residuos reales)
- Humano en el control: la IA y los datos abiertos sugieren, el usuario confirma
- Precios deterministas y auditables — la IA no fija tarifas
- Minimización de datos: solo la ubicación del punto de recogida, con permiso explícito
- EcoBot con límites declarados: no acepta residuos peligrosos, no inventa coberturas

## 👥 Equipo

Delia Rosa Vargas Gómez · Fernando José Arveláez Pérez · Juan Pablo Blandón Martínez · Sofía Valdés Alvear · John Fredy Rojas Ceballos
**Mentora:** Laura Alejandra Sánchez Giraldo

Reto propuesto por **EcoClean Connect** · Hackathon **Beca Ser ANDI** — Fondo Social SER ANDI, ANDI Seccional Antioquia, Más País, NODO y EAFIT.
