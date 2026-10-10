# Arquitectura — Plataforma de Logística y Recolección de Plásticos (MVP)

## 1) Resumen Ejecutivo
- **Objetivo:** Optimizar la recolección punto a punto de plásticos/material reciclable, digitalizando pesaje, rutas y trazabilidad (Ley REP, Chile).
- **Aplicación:** `recycling-logistics` (API + panel web B2B + app móvil de choferes).
- **Estado:** Draft
- **Responsable técnico:** Por definir
- **Última actualización:** 2026-10-04
- **Origen:** Conversación con Gemini, ver [conversation-gemini.md](conversation-gemini.md) y [sources/](sources/).

## 2) Contexto y Alcance
### 2.1 Problema a resolver
Un camión llega a un punto (edificio, condominio, empresa), pesa los kilos de plástico y sigue al siguiente. Hoy hay papeleo, rutas manuales y poca trazabilidad. Se necesita: rutas eficientes, registro rápido de pesaje (incluso sin señal) y datos para certificación Ley REP.

### 2.2 Alcance incluido (MVP)
- Panel web: CRUD de clientes/condominios/puntos con GPS, asignación de rutas optimizadas, dashboard de progreso y kilos, exportación CSV de trazabilidad.
- App móvil de choferes: login, ruta activa con navegación a Waze/Google Maps, registro de pesaje, tolerancia offline con sincronización.
- Optimización de ruteo vía Google Maps Routes API (`optimizeWaypointOrder`), lotes de hasta 25 puntos.

### 2.3 Fuera de alcance (MVP)
Ideas evaluadas y diferidas (ver [backlog.md](backlog.md)):
- App de usuarios finales con gamificación (EcoPuntos) y solicitudes on-demand.
- Pesaje IoT por Bluetooth con balanzas.
- Validación QR/NFC en contenedores.
- Marketplace de lotes en tránsito y control de capacidad del camión.
- Motor VRP propio (OR-Tools + OSRM/GraphHopper).
- Tracking GPS en segundo plano.
- Categorización de material (PET 1, HDPE 2…) y certificados automáticos.

## 3) Requisitos
### 3.1 Funcionales
| ID | Requisito | Módulo |
|---|---|---|
| RF-01 | CRUD de clientes, condominios y puntos de recolección con coordenadas GPS | Web |
| RF-02 | Seleccionar puntos del día, optimizar orden vía API y asignar ruta a camión/chofer | Web |
| RF-03 | Dashboard en tiempo real: puntos visitados vs. pendientes, kilos por camión/jornada | Web |
| RF-04 | Exportar planillas CSV para auditoría y certificados | Web |
| RF-05 | Autenticación de operarios (y diferenciación chofer/administrador) | Móvil/API |
| RF-06 | Lista cronológica de visitas con botón de navegación a Waze/Google Maps | Móvil |
| RF-07 | Formulario de pesaje in-situ (kilos recibidos) | Móvil |
| RF-08 | Guardado local de pesajes sin red y sincronización posterior | Móvil |

### 3.2 No funcionales
- **Rendimiento:** API asíncrona; muchas peticiones simultáneas de choferes.
- **Seguridad:** Autenticación segura; roles; HTTPS; almacenamiento local seguro de pesajes pendientes.
- **Escalabilidad:** Esquema relacional con camino a PostGIS; camino a motor VRP propio.
- **Mantenibilidad:** Contenedores Docker; frontend y backend desacoplados vía API.
- **Experiencia de uso:** UI táctil nativa (Ionic) para choferes; UI B2B limpia (shadcn/ui).
- **Resiliencia offline:** Ningún pesaje se pierde por falta de señal o limpieza de caché.
- **Costo:** Costos fijos mensuales < ~$30 USD en los primeros meses.

## 4) Arquitectura Propuesta
### 4.1 Diagrama de alto nivel
```text
 [App Móvil Choferes]            [Panel Web B2B]
 Capacitor + Ionic UI            Astro/React + Tailwind + shadcn/ui
        │  HTTPS / JSON                 │  HTTPS / JSON
        └──────────────┬────────────────┘
                       ▼
              [API FastAPI (Python)]  ── Routes API ──► [Google Maps Platform]
                       │
                       ▼
                 [PostgreSQL] (PostGIS a futuro)
```

### 4.2 Componentes principales
| Componente | Responsabilidad | Tecnología |
|---|---|---|
| Backend / API | Usuarios, clientes, puntos, rutas, pesajes, exportes; integra ruteo | Python, FastAPI |
| Base de datos | Persistencia relacional y futura capa geoespacial | PostgreSQL (+PostGIS futuro) |
| Panel Admin | Gestión comercial, asignación de rutas, dashboard, trazabilidad | Astro o React, Tailwind, shadcn/ui (Tremor opcional para gráficos) |
| App Móvil | Ruta activa, pesaje, offline sync | Capacitor + Ionic UI |
| Motor de ruteo (MVP) | Ordenar paradas (VRP simple) | Google Maps Routes API, `optimizeWaypointOrder` |
| Infra | Contenedores del backend + DB; CDN para frontend | Docker Compose en VPS; Vercel / Cloudflare Pages |

### 4.3 Flujo principal
1. Operaciones registra clientes/puntos con GPS en el panel.
2. Selecciona los puntos del día; el backend los envía a Routes API y obtiene el orden óptimo.
3. La ruta ordenada se asigna a un camión/chofer.
4. El chofer ve la ruta en la app, navega con Waze/Maps y registra el pesaje en cada punto.
5. Sin señal, el pesaje se guarda localmente y se sincroniza al recuperar red.
6. El dashboard muestra el progreso; se exportan CSV para auditoría/Ley REP.

## 5) Diseño Funcional y Técnico
### 5.1 Casos de uso clave
- Despachar ruta del día (admin).
- Ejecutar ruta y registrar pesaje (chofer).
- Sincronizar pesajes offline (app).
- Exportar trazabilidad por cliente/periodo (admin).

### 5.2 Contratos e interfaces
- **APIs:** REST/JSON FastAPI (usuarios/auth, clientes, puntos, rutas, pesajes, exportes). Detalle pendiente (ver [backlog.md](backlog.md)).
- **Integraciones externas:** Google Maps Routes API (límite ~25 waypoints intermedios por consulta); deep links a Waze/Google Maps.
- **Sync offline:** pesajes con ID cliente (UUID) idempotente para evitar duplicados al reenviar.

### 5.3 Modelo de datos (preliminar)
- **Entidades:** Usuario (rol), Cliente, Punto de recolección (lat/lng), Camión, Ruta, Parada de ruta (orden, estado), Pesaje (kilos, timestamp, chofer, punto).
- **Relaciones:** Cliente 1—N Punto; Ruta 1—N Parada; Parada → Punto; Ruta → Camión/Chofer; Pesaje → Parada.
- **Trazabilidad:** pesajes inmutables con timestamp y autor; exportables por cliente/periodo.

## 6) Decisiones de Arquitectura
Resumen; detalle en [decisions.md](decisions.md).
| ID | Decisión | Motivo |
|---|---|---|
| ADR-001 | Python + FastAPI backend | Async, rápido, ecosistema de ruteo (OR-Tools) |
| ADR-002 | PostgreSQL | Relacional, PostGIS a futuro |
| ADR-003 | Frontend desacoplado (React/Astro) | Libertad de UI B2B vs. Streamlit/Django admin |
| ADR-004 | Capacitor + Ionic para móvil | Reutilizar código web, acceso a plugins nativos |
| ADR-005 | Google Maps Routes API en MVP | Rapidez; migrar a OR-Tools + OSRM al crecer |
| ADR-006 | VPS + Docker Compose; frontend en CDN | Costo mínimo |
| ADR-007 | Tailwind + shadcn/ui + Ionic UI | Estilos transversales |

## 7) Riesgos y Mitigaciones
| Riesgo | Impacto | Mitigación |
|---|---|---|
| Pérdida de pesajes por caché borrada del WebView | Alto | Almacenamiento nativo vía Capacitor (SQLite/Preferences), no solo IndexedDB |
| GPS se apaga en segundo plano | Medio | Plugin nativo de geolocalización en background (post-MVP) |
| Google no considera peso/capacidad ni ventanas horarias | Medio | Aceptado en MVP; migrar a OR-Tools |
| Límite de ~25 paradas por ruta | Medio | Dividir lotes; migrar a motor propio |
| Costo de Routes API al escalar | Medio | Monitorear cuota de $200/mes; caché de matrices; OSRM |
| Web Bluetooth limitado en iOS | Bajo (post-MVP) | Plugin nativo de BLE |
| Un VPS único sin alta disponibilidad | Medio | Backups de DB; migrar a gestionado al validar |

## 8) Preguntas Abiertas y Próximos Pasos
- ¿Modelo de negocio: B2B (empresas) o residencial (edificios/casas)? (Gemini lo preguntó; sin respuesta.)
- ¿Astro o React para el panel?
- Estrategia de autenticación y roles (chofer/admin/cliente).
- Modelo de datos y contrato del endpoint de pesaje (pendiente de detallar).
- ¿Requisitos exactos de certificado/planilla Ley REP?
- Tamaño de flota inicial y volumen diario de puntos.

## 9) Criterios de Aceptación del MVP
- [ ] Admin crea puntos con GPS y despacha una ruta optimizada a un chofer.
- [ ] Chofer completa ruta y registra pesajes desde la app.
- [ ] Pesajes sin señal se sincronizan sin pérdida ni duplicados (prueba física en subterráneo).
- [ ] Dashboard refleja progreso y kilos de la jornada.
- [ ] CSV de trazabilidad exportable por cliente/periodo.
- [ ] Desplegado en producción con costo fijo < ~$30 USD/mes.

## 10) Referencias
- [implementation-plan.md](implementation-plan.md) · [backlog.md](backlog.md) · [decisions.md](decisions.md)
- [oracle-cloud-provisioning.md](oracle-cloud-provisioning.md) — provisión de la base Oracle Cloud Always Free, DNS, red y operación
- [conversation-gemini.md](conversation-gemini.md) — contexto original
- Fuentes originales: [sources/](sources/)
- Repositorio de implementación: por crear.
