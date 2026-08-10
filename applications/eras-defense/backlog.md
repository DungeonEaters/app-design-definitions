# Backlog de Eras Defense

## Como usar este backlog

El backlog organiza el trabajo por dominios de producto y arquitectura. Cada tarea debe implementarse contra los contratos definidos en `architecture.md` y validarse con tests de dominio, integracion o experiencia segun corresponda.

Las tareas estan expresadas a nivel de sistema. Los detalles de clases, variables y estructura de scripts pertenecen al repositorio de implementacion y no se fijan en este documento.

## Priorizacion

- **MVP:** necesario para el vertical slice jugable.
- **Post-MVP:** habilita profundidad, escalabilidad o monetizacion posterior.
- **Exploracion:** requiere validacion de producto antes de comprometer arquitectura o contenido.

## Epica 1: Core Board & Merge

### Objetivo
Crear el tablero de defensa, las reglas de colocacion y el sistema de fusion que sostienen la toma de decisiones del jugador.

### Tareas MVP

- Definir el modelo abstracto de casillas, capacidad, ocupacion y estados validos.
- Definir el catalogo de unidades con Era, familia, nivel, capacidades y reglas de compatibilidad.
- Implementar comandos de colocar, mover, retirar y fusionar con validacion centralizada.
- Implementar resultado de fusion, consumo de unidades origen y actualizacion atomica del tablero.
- Implementar feedback de casillas validas, fusiones posibles y bloqueos.
- Integrar pooling para representaciones visuales de unidades y efectos recurrentes.
- Crear una vista de tablero para el vertical slice con una Era y un conjunto reducido de unidades.

### Criterios de aceptacion

- El jugador puede colocar y mover unidades solo en casillas validas.
- Una fusion valida produce una unidad resultado consistente y no duplica ni pierde recursos.
- Una fusion invalida no muta el tablero y comunica la razon al flujo de presentacion.
- El estado del tablero puede serializarse y restaurarse sin perder niveles ni ocupacion.
- La vista refleja cambios emitidos por el dominio sin contener reglas de fusion.

### Restricciones arquitectonicas

- El Board System no depende de MonoBehaviours, UI ni SDKs.
- Las reglas de fusion son deterministas y testeables sin escena.
- Las vistas no acceden directamente a estructuras mutables del dominio.
- La representacion visual usa pooling para entidades repetidas.

### Tareas Post-MVP / Exploracion

- Expansiones de tablero con formas no rectangulares.
- Fusion contextual influida por terreno, Era o Monolitos.
- Reglas de fusion en cadena y capacidades heredadas.

## Epica 2: Wave & Combat Engine

### Objetivo
Construir la simulacion de oleadas y combate automatico con escalado controlado y rendimiento medible.

### Tareas MVP

- Definir el ciclo de preparacion, combate, resultado y transicion.
- Implementar `WaveManager` con configuracion de oleadas, dificultad y condiciones de victoria.
- Definir rutas, objetivos defensivos y comportamiento minimo de enemigos.
- Implementar `CombatSystem` con movimiento, alcance, seleccion de objetivos, ataques, dano, estados y bajas.
- Implementar escalado de stats por oleada y por Era.
- Implementar pooling de enemigos, proyectiles, impactos y efectos temporales.
- Crear un flujo de resultado para victoria, derrota y revivir.
- Agregar simulaciones reproducibles para balance y tests de regresion.

### Criterios de aceptacion

- Una oleada puede iniciarse, progresar y terminar con un resultado unico y observable.
- Las unidades atacan objetivos validos y el dano respeta las reglas configuradas.
- Los enemigos alcanzan o son detenidos por el objetivo segun el estado de combate.
- La dificultad aumenta de forma configurable sin romper limites de rendimiento.
- Una derrota puede entrar en un flujo de recuperacion sin mutar dos veces las recompensas.
- El vertical slice soporta la poblacion objetivo de entidades con mediciones documentadas.

### Restricciones arquitectonicas

- `WaveManager` orquesta el ciclo pero no resuelve dano por entidad.
- `CombatSystem` no crea ni destruye objetos visuales directamente.
- El tick de combate se centraliza y se organiza por fases predecibles.
- El camino critico evita allocations y permite evolucionar a Jobs/Burst tras profiling.
- Los resultados de combate se comunican mediante snapshots y eventos agregados.

### Tareas Post-MVP / Exploracion

- Multiples rutas y objetivos defensivos.
- Jefes, habilidades activas y comportamientos especiales.
- Simulacion acelerada para recompensas offline y herramientas de balance.
- Procesamiento paralelo de grandes poblaciones.

## Epica 3: Tienda & Sinergias

### Objetivo
Dar al jugador decisiones entre oleadas y construir identidades de tablero basadas en Eras y familias de unidades.

### Tareas MVP

- Definir inventario de tienda por fase, Era y reglas de disponibilidad.
- Implementar compras con validacion de saldo, capacidad y estado del tablero.
- Implementar reroll con coste, limite y origen de recompensa trazable.
- Implementar expansion del grid con costes y limites de progresion.
- Definir `SpatialBuffSystem` para Monolitos Temporales: colocacion, area, duracion y expiracion.
- Definir `SynergySystem` para bonos por Era y familia de unidades.
- Mostrar al jugador las sinergias activas, sus condiciones y su impacto.
- Garantizar que las modificaciones se recalculan despues de cambios en tablero o buffs.

### Criterios de aceptacion

- Las compras consumen exactamente el coste correspondiente y actualizan el inventario.
- Un reroll produce un inventario valido y respeta su coste y limite.
- La expansion aumenta la capacidad del tablero sin invalidar el estado existente.
- Un Monolito Temporal afecta solo su area y deja de afectar al expirar.
- Las sinergias se activan y desactivan de forma consistente con el tablero.
- La UI puede explicar por que una sinergia o un buff esta activo.

### Restricciones arquitectonicas

- `ShopSystem`, `SpatialBuffSystem` y `SynergySystem` no se llaman directamente entre si.
- Los costes y recompensas pasan por un ledger de economia.
- Los buffs se modelan como modificadores composables con prioridad y stacking definidos.
- El recalculo debe ser incremental o acotado para evitar reconstrucciones costosas en cada frame.

### Tareas Post-MVP / Exploracion

- Ofertas temporales y tienda por Era.
- Sinergias negativas, trade-offs y builds especializadas.
- Monolitos con interacciones entre regiones o rutas.
- Eventos de tienda y decisiones de riesgo/recompensa.

## Epica 4: Economia Idle & Persistencia

### Objetivo
Sostener la progresion entre sesiones con una economia legible, auditable y resistente a errores.

### Tareas MVP

- Definir monedas blandas, gemas, recursos de Era y sus fuentes y sinks.
- Definir costes de unidades, tienda, reroll, expansion y Monolitos.
- Implementar ledger de transacciones con motivo, origen, destino y correlacion.
- Implementar progreso de jugador, Eras desbloqueadas y oleada maxima.
- Implementar guardado local versionado con snapshot, backup y recuperacion.
- Implementar carga, validacion y migracion de saves.
- Implementar recompensa offline con limites de tiempo, calculo y reclamacion idempotente.
- Crear HUD de balances, progreso y estado de recompensa.

### Criterios de aceptacion

- Ninguna operacion puede producir saldo negativo o crear recursos sin una transaccion valida.
- El jugador puede cerrar y reabrir la aplicacion sin perder el progreso confirmado.
- Un guardado corrupto se recupera mediante backup o se rechaza sin sobrescribir datos validos.
- Los saves de una version anterior migran o fallan de forma explicita y segura.
- Las recompensas offline respetan limites y no se pueden reclamar dos veces.
- Las simulaciones de economia muestran fuentes, sinks y crecimiento esperado.

### Restricciones arquitectonicas

- La economia es el unico propietario de balances y transacciones.
- Persistencia no puede depender de la UI ni de un SDK de monetizacion.
- Todas las operaciones importantes deben ser idempotentes o tener una clave de deduplicacion.
- El calculo offline no debe ejecutar una oleada arbitrariamente larga sin limites.

### Tareas Post-MVP / Exploracion

- Cloud save opcional.
- Migracion de perfil entre dispositivos.
- Prestige o reinicio con recompensas permanentes.
- Economia asimetrica por Era y temporadas.

## Epica 5: Integracion de Monetizacion y Analitica

### Objetivo
Integrar monetizacion complementaria y medicion sin acoplarla al dominio ni comprometer la experiencia.

### Tareas MVP

- Definir el puerto de Rewarded Ads y estados de disponibilidad.
- Implementar recompensa de revivir con limites, confirmacion y deduplicacion.
- Implementar recompensa de reroll patrocinado.
- Implementar duplicacion de ingresos offline mediante una ventana controlada.
- Definir politica de interstitials, cooldown y consentimiento.
- Definir catalogo de IAP de moneda dura y flujo de validacion de compra.
- Implementar adaptadores de plataforma y sustitutos para tests.
- Instrumentar eventos de oleada, fusion, economia, anuncios, compras y conversion.
- Medir revenue y resultados de monetizacion sin bloquear la simulacion.

### Criterios de aceptacion

- Un rewarded completado entrega una sola recompensa del tipo autorizado.
- Un rewarded cancelado, fallido o no disponible no altera el estado del jugador.
- Los interstitials solo aparecen en transiciones y respetan la frecuencia configurada.
- Una compra validada acredita moneda dura una sola vez y queda trazada.
- El juego sigue siendo jugable si un SDK externo no esta disponible.
- Los eventos de analitica contienen contexto suficiente para evaluar el funnel sin datos sensibles.

### Restricciones arquitectonicas

- El dominio recibe resultados validados mediante puertos, nunca callbacks directos del SDK.
- Las recompensas monetizadas se liquidan como transacciones idempotentes.
- Los SDKs pueden sustituirse por dobles de test en EditMode.
- Analytics es asincrono, tolerante a fallos y no controla el flujo principal.

### Tareas Post-MVP / Exploracion

- Experimentos A/B de ofertas y frecuencia.
- Paquetes de progresion y productos por Era.
- Remote configuration para balance y monetizacion.
- Attribution y cohortes avanzadas.

## Dependencias transversales

- El Board System y el catalogo de unidades preceden a combate, sinergias y tienda.
- `WaveManager` y `CombatSystem` requieren un modelo estable de tablero y unidades.
- Economia y persistencia deben existir antes de compras, recompensas offline y monetizacion.
- Los Event Channels y el Tick Manager deben definirse antes de integrar presentacion y pooling.
- Analytics se instrumenta despues de estabilizar los eventos de dominio, no como mecanismo de coordinacion.

## Definition of Done transversal

- Cada sistema tiene responsabilidad unica y contrato documentado.
- Las reglas de dominio tienen tests deterministas sin dependencia de Unity.
- Los flujos de integracion tienen tests de composicion y de ciclo completo.
- El runtime no depende de singletons globales ni referencias no configuradas.
- Las allocations del camino critico se miden con una carga representativa.
- La documentacion de cada epic se mantiene alineada con la arquitectura aprobada.
