# Plan de Implementación — Logística y Recolección de Plásticos (MVP)

Repositorio: [JulioRom/recycling-logistics](https://github.com/JulioRom/recycling-logistics). Duración estimada: **10–12 semanas**, equipo de 2 (1 Full-Stack, 1 Frontend). Diseño en [architecture.md](architecture.md); tareas en [backlog.md](backlog.md).

## Estado de fases
| Fase | Semanas | Estado |
|---|---|---|
| 1. Setup y Modelado | 1–2 | Hecho (2026-10-05) |
| 2. Motor Backend y Rutas | 3–5 | Hecho en local; falta validar Google Routes con clave real |
| 3. Panel Web B2B | 6–8 | Hecho en local |
| 4. App Móvil Híbrida | 9–11 | UI y offline hechos en navegador; falta empaquetado nativo con Capacitor |
| 5. Pruebas QA y Lanzamiento | 12 | Pendiente |

## Fase 1 — Setup y Modelado (Semanas 1–2)
- Modelado de entidades y diagrama de BD.
- Repositorios configurados.
- Docker Compose con FastAPI + PostgreSQL.
- **Entregable:** entorno local reproducible y esquema inicial.

## Fase 2 — Motor Backend y Rutas (Semanas 3–5)
- Endpoints de usuarios/auth, clientes, puntos, camiones, rutas, pesajes.
- Integración con Google Maps Routes API (`optimizeWaypointOrder`).
- **Entregable:** API operativa con integración de rutas validada.

## Fase 3 — Panel Web B2B (Semanas 6–8)
- Gestión comercial (CRUD con GPS), asignación de rutas, dashboard, exportación CSV.
- **Entregable:** frontend consumiendo la API; monitoreo y generación de rutas.

## Fase 4 — App Móvil Híbrida (Semanas 9–11)
- UI Ionic, login, ruta activa, pesaje, deep link a Waze/Maps.
- Empaquetado Capacitor (iOS/Android) y almacenamiento offline con sync.
- **Entregable:** app instalable con flujo completo y offline.

## Fase 5 — QA y Lanzamiento (Semana 12)
- Pruebas físicas sin señal (subterráneos), validación de rutas sugeridas.
- QA final y despliegue a producción (VPS + CDN).
- **Entregable:** MVP en producción.

## Infraestructura y presupuesto mensual
| Ítem | Opción | Costo |
|---|---|---|
| Backend + BD | VPS (Hetzner / DigitalOcean) con Docker | ~$5–10 USD |
| Frontend web | Vercel / Cloudflare Pages | $0 |
| Mapas y rutas | Google Maps Routes API (crédito $200/mes) | $0 |
| Dominio y correo | Namecheap / Google Workspace | ~$15 USD |
| **Total** | | **< ~$30 USD** |

Alternativas evaluadas: Render/Railway ($0–10), GCP Cloud Run (barato) pero Cloud SQL ~$10–25/mes.

## Bloqueadores y riesgos
Ver sección 7 de [architecture.md](architecture.md) y [backlog.md](backlog.md).

## Estado de la implementación (2026-10-05)
- **Hecho:** API FastAPI + PostgreSQL con auth/roles, CRUD, rutas optimizadas, pesaje idempotente, dashboard y CSV; panel web; app de choferes con cola offline; tests (backend 9, móvil 9); Dev Container; CI en GitHub Actions.
- **Verificación:** flujo completo en local, incluido pesaje offline → sincronización, con navegador automatizado.
- **Pendiente para producción:** clave de Google Routes, migraciones (Alembic), endurecimiento de auth (secretos, refresh), `cap add ios/android`, prueba física sin señal, despliegue (VPS + CDN), backups de BD.
