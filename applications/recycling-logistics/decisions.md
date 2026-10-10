# Decisiones de Arquitectura (ADR) — Logística y Recolección de Plásticos

Fecha de todas las decisiones: 2026-10-04 (origen: [conversation-gemini.md](conversation-gemini.md)).

## ADR-001 — Backend en Python + FastAPI
- **Alternativas:** Django + DRF, Streamlit/Reflex.
- **Motivo:** Asíncrono (llamadas a Maps sin bloquear), integración con Docker, ecosistema Python para OR-Tools. Django daría admin automático pero es más pesado.

## ADR-002 — PostgreSQL
- **Motivo:** Relacional, robusto, escalable; PostGIS disponible si se requieren consultas geoespaciales.

## ADR-003 — Frontend desacoplado (Astro o React)
- **Alternativas:** Streamlit/Reflex (todo Python); Astro.
- **Motivo:** Libertad de UI B2B.
- **Actualización 2026-10-05:** se eligió **React + Vite** (SPA con panel autenticado, sin necesidad de SSR/contenido estático) y se comparte stack con la app móvil.

## ADR-004 — Móvil con Capacitor + Ionic UI
- **Alternativas:** PWA pura, React Native + Expo, Flet, Kivy/BeeWare.
- **Motivo:** Reutiliza código web; plugins nativos resuelven GPS en segundo plano, BLE y almacenamiento offline seguro. Python móvil rinde mal en hardware.
- **Riesgo:** IndexedDB puede ser purgado por el SO; usar almacenamiento nativo.

## ADR-005 — Ruteo MVP con Google Maps Routes API
- **Alternativas:** Mapbox Optimization API; OR-Tools + OSRM/GraphHopper.
- **Motivo:** Más rápido para validar. Considera tráfico. Limitaciones: ~25 waypoints, sin peso/ventanas horarias, costo por consulta.
- **Evolución:** migrar a OR-Tools + OSRM autoalojado al crecer la flota.

## ADR-006 — Oracle Cloud Always Free ARM64 + Docker Compose + Caddy
- **Alternativas:** Render/Railway, GCP Cloud Run (Cloud SQL ~$10–25/mes lo encarece), AWS/Azure, VPS + CDN para frontend.
- **Motivo:** Costo fijo mínimo; la VM ARM64 puede alojar los cuatro servicios y eventualmente OSRM. Caddy resuelve TLS de Let's Encrypt sin servicio adicional.
- **Consecuencia:** PostgreSQL, API y frontends comparten una VM; se requieren backups y monitoreo. La base de datos no se expone públicamente.

## ADR-007 — Estilos: Tailwind CSS + shadcn/ui (web) + Ionic UI (móvil)
- **Alternativas:** DaisyUI, Tremor (útil para dashboards REP), Bootstrap.

## ADR-008 — Optimizador local como respaldo del ruteo
- **Contexto:** Google Routes exige clave y facturación; puede fallar o rechazar el lote.
- **Decisión:** `POST /routes` siempre ordena; usa Google si hay `GOOGLE_MAPS_API_KEY`, y cae a vecino más cercano + 2-opt (distancia haversine) si falla o no hay clave. La ruta guarda qué optimizador se usó.
- **Consecuencia:** el despacho nunca depende de un tercero; el orden local no considera tráfico. Límite de 25 paradas se mantiene.

## ADR-009 — Pesaje idempotente con UUID del cliente
- **Contexto:** Los choferes registran sin red y reintentan al reconectar.
- **Decisión:** el UUID del pesaje lo genera la app; el servidor lo usa como clave primaria. Reenvíos devuelven `duplicate`. `POST /weighings/batch` responde por ítem para no bloquear el lote por un error.
- **Consecuencia:** sincronización segura ante reintentos; un UUID de otro chofer se rechaza.

## ADR-010 — Monorepo para el MVP
- **Alternativas:** tres repos (backend, web, mobile).
- **Motivo:** una sola persona/equipo pequeño, contrato de API que cambia seguido, `docker compose` único. Se puede separar luego con `git subtree split`.
- **Revisar cuando:** haya equipos o ciclos de release distintos (p. ej. publicación en tiendas).

## ADR-011 — Dev Container y CI en GitHub Actions
- **Decisión:** `.devcontainer/` (Python 3.12, Node 22, Postgres 16 por compose; dependencias en volúmenes para no mezclar binarios Linux/macOS). CI: pytest, build web, test+build móvil y smoke test de `docker compose`.
- **Motivo:** entorno replicable y verificación automática en cada PR; Node 22 por requisito de Vite 8.

## ADR-012 — Borrado lógico
- **Decisión:** clientes, puntos y camiones se desactivan (`active=false`); rutas con pesajes no se eliminan.
- **Motivo:** trazabilidad para auditoría Ley REP.

## ADR-013 — Imágenes multi-stage para producción
- **Decisión:** `web` y `mobile` compilan con Node 22 y su resultado se sirve desde Nginx Alpine; la API ejecuta Uvicorn sin `--reload`.
- **Motivo:** Vite es servidor de desarrollo, no de producción. Nginx reduce tamaño de imagen y soporta las rutas SPA con `try_files`.
- **Consecuencia:** `VITE_API_URL` se define al construir la imagen; cada despliegue genera una imagen para el dominio de producción.

## ADR-014 — GHCR y despliegue ARM64 por GitHub Actions
- **Decisión:** el push a `main` construye imágenes `linux/arm64` para `api`, `web` y `mobile`, las etiqueta con SHA y las publica en GHCR. El job de despliegue se conecta por SSH a Oracle Cloud, actualiza el tag y ejecuta Compose.
- **Motivo:** la VM Always Free es ARM64; etiquetar con SHA permite saber exactamente qué commit corre.
- **Consecuencia:** la VM requiere un token de solo lectura de paquetes para descargar imágenes privadas; secretos permanecen en `.env` de la VM.

## ADR-015 — Caddy como reverse proxy de producción
- **Decisión:** Caddy es el único servicio que publica 80/443. Sirve tres dominios: panel en el dominio principal, app móvil en `driver.` y API en `api.`.
- **Motivo:** certificados Let's Encrypt automáticos y aislamiento de los servicios internos.
- **Consecuencia:** los tres registros DNS deben apuntar a la VM antes del primer despliegue. Compose con una réplica no garantiza cero downtime durante recreación; es una limitación aceptada del MVP.
