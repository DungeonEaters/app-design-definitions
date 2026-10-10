# Decisiones de Arquitectura (ADR) — Logística y Recolección de Plásticos

Fecha de todas las decisiones: 2026-10-04 (origen: [conversation-gemini.md](conversation-gemini.md)).

## ADR-001 — Backend en Python + FastAPI
- **Alternativas:** Django + DRF, Streamlit/Reflex.
- **Motivo:** Asíncrono (llamadas a Maps sin bloquear), integración con Docker, ecosistema Python para OR-Tools. Django daría admin automático pero es más pesado.

## ADR-002 — PostgreSQL
- **Motivo:** Relacional, robusto, escalable; PostGIS disponible si se requieren consultas geoespaciales.

## ADR-003 — Frontend desacoplado (Astro o React)
- **Alternativas:** Streamlit/Reflex (todo Python).
- **Motivo:** Libertad de UI B2B. Astro vs. React queda **abierto**.

## ADR-004 — Móvil con Capacitor + Ionic UI
- **Alternativas:** PWA pura, React Native + Expo, Flet, Kivy/BeeWare.
- **Motivo:** Reutiliza código web; plugins nativos resuelven GPS en segundo plano, BLE y almacenamiento offline seguro. Python móvil rinde mal en hardware.
- **Riesgo:** IndexedDB puede ser purgado por el SO; usar almacenamiento nativo.

## ADR-005 — Ruteo MVP con Google Maps Routes API
- **Alternativas:** Mapbox Optimization API; OR-Tools + OSRM/GraphHopper.
- **Motivo:** Más rápido para validar. Considera tráfico. Limitaciones: ~25 waypoints, sin peso/ventanas horarias, costo por consulta.
- **Evolución:** migrar a OR-Tools + OSRM autoalojado al crecer la flota.

## ADR-006 — Hosting: VPS con Docker Compose + frontend en CDN
- **Alternativas:** Render/Railway, GCP Cloud Run (Cloud SQL ~$10–25/mes lo encarece), AWS/Azure.
- **Motivo:** Mejor rendimiento por dólar (~$5–10/mes); un VPS puede alojar OSRM después.

## ADR-007 — Estilos: Tailwind CSS + shadcn/ui (web) + Ionic UI (móvil)
- **Alternativas:** DaisyUI, Tremor (útil para dashboards REP), Bootstrap.

## ADR-008 — Oracle Cloud Always Free como base del MVP
- **Decisión:** Usar una instancia Ubuntu Ampere A1 ARM64 Always Free, hasta 4 OCPUs y 24 GB RAM, como base de los ambientes de test y producción del MVP.
- **Motivo:** Costo de infraestructura fijo cero dentro de la cuota, capacidad suficiente para API, PostgreSQL, frontends estáticos y reverse proxy en la etapa de validación.
- **Alternativas:** VPS Hetzner/DigitalOcean, Render/Railway, GCP Cloud Run + Cloud SQL.
- **Consecuencias:** Test y producción comparten host, por lo que no tienen aislamiento fuerte ni alta disponibilidad. Solo Caddy expone 80/443; el despliegue usa imágenes ARM64 desde GHCR. Se requiere backup de PostgreSQL, monitoreo y migración a infraestructura separada cuando el riesgo/carga lo justifique.
- **Operación:** Ver [oracle-cloud-provisioning.md](oracle-cloud-provisioning.md). Terraform se mantiene en [DungeonEaters/recycling-logistics-infrastructure](https://github.com/DungeonEaters/recycling-logistics-infrastructure).
