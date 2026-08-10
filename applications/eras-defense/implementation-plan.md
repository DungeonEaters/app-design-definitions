# Plan de Implementacion de Eras Defense

## Objetivo

Construir un vertical slice de defensa de tablero con fusion, oleadas, combate, tienda y progresion por Eras. El plan prioriza integracion temprana y contratos estables antes de ampliar contenido, monetizacion o escala.

## Principios de ejecucion

- Implementar y validar un flujo completo antes de producir contenido masivo.
- Mantener la logica de dominio independiente de Unity y de SDKs externos.
- Integrar cada milestone en una escena ejecutable y en tests automatizados.
- Medir rendimiento con carga representativa desde el primer motor de combate.
- Mantener configuracion, balance y contenido fuera de reglas rigidas cuando sea viable.
- Tratar persistencia, economia y monetizacion como sistemas transaccionales y trazables.

## Dependencias criticas

```text
Contratos de dominio y composicion
              |
Board + Catalogo de unidades
              |
Oleadas + Combate + Pooling
              |
Tienda + Buffs espaciales + Sinergias
              |
Economia + Persistencia + Recompensas offline
              |
Monetizacion + Analytics + Escala de contenido
```

La presentacion puede evolucionar en paralelo, pero solo consume contratos estables. Analytics debe observar eventos de dominio; no puede ser una dependencia para resolver una accion de juego.

## Fase 0: Alineacion de producto y fundacion tecnica

### Milestone 0: Contrato del vertical slice

**Objetivo:** Cerrar las decisiones que condicionan todos los sistemas.

**Entregables:**

- Definicion de la primera Era y su fantasia visual.
- Alcance del tablero, unidades, familias, niveles y fusion.
- Definicion de una ruta o escenario de combate.
- Reglas de una oleada representativa y condiciones de victoria/derrota.
- Modelo de monedas, gemas, recompensas y costes basicos.
- Politica inicial de persistencia, monetizacion y analitica.
- Contratos de comandos, lecturas y eventos entre dominios.
- Composition root con DI, Event Channels y Tick Manager definidos.
- Estrategia de tests y criterios de rendimiento del vertical slice.

**Dependencias:** Ninguna.

**Criterio de salida:** El equipo puede describir una sesion completa con estados, eventos, datos y responsabilidades sin decisiones criticas abiertas.

## Fase 1: Core de tablero y fusion

### Milestone 1: Core de Fusion y Posicionamiento

**Objetivo:** Producir un tablero jugable con reglas de unidades y fusion.

**Entregables:**

- Estado de tablero y validacion de casillas.
- Catalogo inicial de unidades y reglas de compatibilidad.
- Flujo de colocar, mover, retirar y fusionar.
- Resultado atomico de fusion y eventos de dominio.
- Presentacion inicial del tablero y feedback de acciones.
- Persistencia provisional del tablero para pruebas de continuidad.
- Tests deterministas de reglas y tests de integracion del flujo de fusion.

**Dependencias:** Milestone 0.

**Criterio de salida:** El jugador puede preparar un tablero, fusionar unidades y restaurar el estado sin inconsistencias.

## Fase 2: Oleadas y simulacion de combate

### Milestone 2: Motor de Combate y Oleadas

**Objetivo:** Conectar el tablero con una simulacion de defensa automatica.

**Entregables:**

- Maquina de estados de preparacion, combate, resultado y transicion.
- `WaveManager` con configuracion de oleadas y escalado inicial.
- Modelo de enemigos, rutas, objetivos y condiciones de victoria.
- `CombatSystem` con movimiento, alcance, objetivos, ataques, dano y bajas.
- Visualizacion de entidades y resultados de combate.
- Pooling de entidades y efectos recurrentes.
- Tick centralizado con orden de actualizacion documentado.
- Semillas de simulacion para reproducibilidad y herramientas de balance.
- Tests de oleada, combate y escenario integrado.

**Dependencias:** Milestone 1, contrato de Tick Manager y pooling.

**Criterio de salida:** Una unidad desplegada puede defender el objetivo contra una oleada completa, con resultado reproducible y rendimiento medido.

## Fase 3: Profundidad tactica del tablero

### Milestone 3: Monolitos Temporales y Sinergias

**Objetivo:** Introducir decisiones espaciales y composiciones de tablero que diferencien las Eras.

**Entregables:**

- Reglas de colocacion, area, duracion y expiracion de Monolitos Temporales.
- Sistema de modificadores espaciales consumible por combate y tablero.
- Catalogo inicial de sinergias por Era y familia.
- Evaluacion y recalculo incremental de sinergias.
- Feedback visual y explicacion de buffs y bonos activos.
- Tests de stacking, prioridad, expiracion y cambios de composicion.

**Dependencias:** Milestone 2, modelo estable de unidades y eventos de tablero.

**Criterio de salida:** El jugador puede comparar al menos dos composiciones validas y observar impactos tacticos medibles durante una oleada.

## Fase 4: Metajuego, tienda y economia

### Milestone 4: Metajuego, Tienda y Sinergias

**Objetivo:** Crear decisiones entre oleadas y progresion persistente por Eras.

**Entregables:**

- `ShopSystem` con inventario, compra y reroll.
- Expansion del tablero con limites y progresion.
- Ledger de monedas blandas, gemas y recursos de Era.
- Costes y recompensas de fusion, oleada, tienda y Monolitos.
- Progreso de Era, descubrimientos y oleada maxima.
- Flujo de recompensa tras victoria o derrota.
- HUD de balances y estado de progresion.
- Simulaciones economicas para detectar inflacion y bloqueos.

**Dependencias:** Milestone 3, reglas de economia aprobadas y persistencia base.

**Criterio de salida:** El jugador completa una oleada, recibe recompensa, toma decisiones en tienda y comienza la siguiente con estado consistente.

## Fase 5: Persistencia y continuidad de sesion

### Milestone 5: Guardado, Migraciones y Offline

**Objetivo:** Hacer que el progreso sea resistente y que la reentrada tenga valor.

**Entregables:**

- Snapshot persistente versionado.
- Carga inicial y restauracion de tablero, economia, Eras y oleada maxima.
- Backup, checksum y recuperacion ante corrupcion.
- Migraciones entre versiones de esquema.
- Guardado en puntos seguros y ciclo de vida de la aplicacion.
- Recompensa offline con limites, calculo y reclamacion idempotente.
- Tests de round-trip, corrupcion, migracion y doble reclamacion.

**Dependencias:** Milestone 4 y modelo de datos aprobado.

**Criterio de salida:** Una sesion puede cerrarse, reabrirse y continuar sin perder progreso ni duplicar recompensas.

## Fase 6: Presentacion y experiencia de producto

### Milestone 6: Vertical Slice Presentable

**Objetivo:** Sustituir la representacion tecnica por una experiencia clara y evaluable.

**Entregables:**

- Flujo de pantallas de preparacion, combate, resultado y tienda.
- Identidad visual de la primera Era.
- Feedback de fusion, dano, bajas, buffs, recompensas y progreso.
- Navegacion, accesibilidad basica y estados de carga/error.
- Tutorial minimo que explique tablero, combate y tienda.
- Telemetria de funnel de primera sesion.
- Prueba de usabilidad interna y correcciones de friccion.

**Dependencias:** Milestones 2, 3 y 4; contratos de presentacion estabilizados.

**Criterio de salida:** Una persona nueva puede completar el loop sin asistencia externa y el equipo puede evaluar la fantasia y la propuesta de valor.

## Fase 7: Monetizacion y analitica

### Milestone 7: Monetizacion Complementaria

**Objetivo:** Añadir ingresos sin convertirlos en requisito del core loop.

**Entregables:**

- Adaptador de Rewarded Ads y estados de disponibilidad.
- Recompensa de revivir, reroll patrocinado y duplicacion de offline income.
- Politica de interstitials en transiciones.
- Catalogo inicial de IAP de moneda dura.
- Validacion, idempotencia y restauracion de compras.
- Adaptadores fake para tests sin SDK real.
- Eventos de ads, compras, revenue y conversion.
- Dashboards o consultas iniciales para evaluar el funnel.

**Dependencias:** Milestone 5, eventos de dominio estables y experiencia de Milestone 6.

**Criterio de salida:** Las integraciones fallan de forma segura, no bloquean la partida y cada recompensa o compra se acredita una sola vez.

## Fase 8: Escala de contenido y distribucion

### Milestone 8: Segunda Era y Preparacion de Produccion

**Objetivo:** Verificar que la arquitectura permite expandir la fantasia sin duplicar sistemas.

**Entregables:**

- Segunda Era con unidades, sinergias, Monolitos y enemigos diferenciados.
- Herramientas o flujos de configuracion para contenido y balance.
- Comparativa de rendimiento entre Eras y poblaciones objetivo.
- Pruebas de regresion de saves al introducir contenido.
- Build automatizado de la plataforma inicial.
- Decision documentada sobre segunda plataforma y necesidades de backend.
- Checklist de calidad, observabilidad y lanzamiento de prueba.

**Dependencias:** Milestone 7 y validacion del vertical slice.

**Criterio de salida:** Una nueva Era puede agregarse principalmente mediante contenido y configuracion, sin alterar los contratos nucleares.

## Estrategia de validacion por fase

- **Dominio:** tests deterministas para fusion, oleadas, combate, buffs, tienda, sinergias y economia.
- **Integracion:** composicion DI, canales, tick manager, persistencia y adaptadores.
- **Experiencia:** recorridos completos desde primera sesion hasta varias oleadas.
- **Rendimiento:** perfiles con poblacion objetivo, sin allocations por frame en el camino critico y medicion de picos.
- **Persistencia:** round-trip, corrupcion, migracion e idempotencia.
- **Monetizacion:** disponibilidad, cancelacion, fallo de SDK, compra repetida y entrega unica.
- **Producto:** retencion temprana, comprension de reglas, dificultad, conversion y economia.

## Riesgos de secuenciacion

- No implementar monetizacion antes de estabilizar economia y persistencia.
- No producir multiples Eras antes de probar que el catalogo y las sinergias son configurables.
- No optimizar con Jobs/Burst antes de medir el coste de una simulacion normal.
- No fijar la UI a estructuras internas del dominio.
- No usar analytics o SDKs como mecanismo de coordinacion entre sistemas.

## Decision gates

1. **Gate de core:** fusion y tablero son divertidos y comprensibles sin combate.
2. **Gate de combate:** una oleada es legible, tactica y estable bajo carga.
3. **Gate de metajuego:** tienda, buffs y sinergias crean decisiones sin inflacion inmediata.
4. **Gate de continuidad:** guardar, volver y recibir offline income funciona sin perdida ni duplicacion.
5. **Gate de negocio:** monetizacion complementa el loop y produce datos accionables.
6. **Gate de escala:** una segunda Era demuestra reutilizacion de arquitectura y contenido.
