# Arquitectura de Eras Defense

## 1) Resumen Ejecutivo

- **Objetivo:** Definir un juego de defensa de tablero con fusion de unidades, combate por oleadas, progresion de eras y economia idle. La experiencia toma como referencia funcional el genero de *Farm Defense TD*, pero se adapta a una fantasia de evolucion humana desde la Edad de Piedra hasta un futuro Sci-Fi.
- **Aplicacion o modulo:** eras-defense
- **Estado:** En revision
- **Responsable tecnico:** Por definir
- **Ultima actualizacion:** 2026-08-10

El producto debe combinar decisiones tacticas de corto plazo, progresion persistente y una fantasia clara de avance tecnologico. La arquitectura prioriza un nucleo de simulacion determinista, desacoplado de Unity y preparado para gestionar muchas entidades con bajo coste de CPU y memoria.

## 2) Contexto y Alcance

### 2.1 Problema a resolver

Construir un vertical slice jugable que demuestre un loop sostenible:

1. El jugador obtiene y combina unidades.
2. Organiza unidades en un tablero defensivo.
3. Resiste oleadas de enemigos.
4. Usa recursos entre oleadas para mejorar la defensa.
5. Desbloquea nuevas Eras del Hombre con nuevas unidades, reglas y sinergias.
6. Conserva progreso y vuelve para continuar la evolucion.

### 2.2 Alcance incluido

- Tablero de casillas con unidades desplegables y reglas de ocupacion.
- Fusion de unidades compatibles para producir unidades de mayor nivel.
- Combate automatico contra oleadas con rutas, objetivos, dano y supervivencia.
- Escalado de dificultad por oleada y por Era.
- Monolitos Temporales que modifican casillas y crean decisiones de posicionamiento.
- Tienda entre oleadas con compra, reroll y expansion del tablero.
- Sinergias globales derivadas de la combinacion de Eras y familias de unidades.
- Progresion desde Edad de Piedra, Antiguedad, Edad Medieval, Edad Industrial, Edad Digital y futuro Sci-Fi. La lista final de Eras y su contenido se valida durante el diseno de producto.
- Economia de monedas blandas, gemas y recompensas offline.
- Guardado local versionado y preparado para migraciones.
- Rewarded Ads, interstitials e IAP de moneda dura.
- Analitica de progresion, combate, economia, anuncios y compras.
- Tests de dominio, integracion de simulacion y validacion de rendimiento.

### 2.3 Fuera de alcance inicial

- Multijugador en tiempo real.
- PvP y matchmaking.
- Backend obligatorio para ejecutar el loop principal.
- Generacion procedural ilimitada de mapas.
- Editor de niveles para usuarios.
- Guilds, chat y funciones sociales complejas.
- Sistemas de prestige hasta que la progresion de Eras este validada.
- Monetizacion que bloquee la resolucion de una oleada o fuerce compras.

## 3) Requisitos

### 3.1 Funcionales

- El jugador puede colocar, mover y fusionar unidades en casillas validas.
- El tablero informa de las posiciones ocupadas, objetivos de fusion y efectos activos.
- Una fusion valida consume las unidades de origen y produce una unidad resultado.
- Las unidades desplegadas atacan automaticamente cuando una oleada esta activa.
- El combate procesa alcance, objetivos, dano, velocidad de ataque, efectos y bajas.
- Las oleadas tienen configuracion, composicion, ruta, condiciones de victoria y dificultad escalable.
- El jugador puede pausar o acelerar la simulacion cuando la plataforma y el modo de juego lo permitan.
- Al completar o perder una oleada se presenta una transicion controlada hacia la tienda o la recuperacion.
- Los Monolitos Temporales pueden afectar una region del tablero durante una duracion definida.
- La tienda permite comprar ofertas, rerollear su inventario y ampliar el tablero con costes definidos.
- Las sinergias se recalculan cuando cambia el conjunto de unidades o una Era activa.
- El jugador obtiene y gasta monedas blandas, gemas y recursos de progresion.
- El progreso de Eras y la oleada maxima se conserva entre sesiones.
- El jugador puede recibir recompensas offline con limites y reglas de integridad claras.
- Rewarded Ads ofrecen revivir una defensa, rerollear la tienda o duplicar ingresos offline.
- Interstitials aparecen solo en transiciones naturales y con limites de frecuencia.
- IAP permite comprar moneda dura y productos definidos por el catalogo.
- Los eventos importantes se registran para medir dificultad, retencion, conversion y economia.

### 3.2 No funcionales

- **Rendimiento:** La simulacion debe soportar oleadas con muchas entidades sin depender de callbacks por objeto. El trabajo de combate se organiza en fases y puede evolucionar a procesamiento por lotes o Jobs/Burst si las mediciones lo justifican.
- **Memoria:** El camino critico evita allocations por frame. Enemigos, proyectiles, efectos y textos recurrentes usan pooling.
- **Determinismo:** El resultado de una simulacion debe ser reproducible con una semilla y una configuracion equivalentes, especialmente en tests y herramientas de balance.
- **Escalabilidad:** La cantidad de unidades, enemigos, Eras y modificadores puede crecer sin acoplar los sistemas entre si.
- **Mantenibilidad:** La logica de dominio se expresa como POCOs y contratos. Unity se limita a presentacion, configuracion, entrada y adaptadores de plataforma.
- **Inyeccion de dependencias:** VContainer compone los sistemas mediante LifetimeScopes. No se permiten singletons globales ni localizacion de servicios desde el dominio.
- **Comunicacion:** Los cambios de estado entre dominios se distribuyen mediante Event Channels tipados o puertos de aplicacion bien definidos.
- **Persistencia:** Los datos guardados tienen version, validacion, migraciones y recuperacion ante corrupcion.
- **Monetizacion:** Ningun anuncio o compra puede mutar el estado sin pasar por una politica de recompensa validada y trazable.
- **Experiencia de uso:** Las transiciones entre preparacion, combate, resultado y tienda son legibles, reversibles cuando corresponda y no interrumpen la toma de decisiones.
- **Observabilidad:** Los eventos de analitica no pueden bloquear la simulacion ni convertirse en dependencia del core.

## 4) Arquitectura Propuesta

### 4.1 Diagrama de alto nivel

```text
Presentacion e Input
        |
Orquestacion de aplicacion y flujo de pantallas
        |
Sistemas de dominio desacoplados
  Board | Waves | Combat | Spatial Buffs | Shop | Synergies
        |
Estado de partida y progresion persistente
        |
Adaptadores de plataforma
 Ads | IAP | Analytics | Storage | Runtime | Pooling
```

El flujo de dependencias apunta hacia el dominio. Los adaptadores no son invocados directamente por los sistemas de juego; exponen puertos que la composicion de la aplicacion conecta con implementaciones concretas.

### 4.2 Componentes principales

| Componente | Responsabilidad | Tecnologia o patron |
|---|---|---|
| Board System | Estado del tablero, ocupacion, colocacion, movimiento y fusion | POCO, comandos y validacion de reglas |
| Unit Catalog | Definicion de unidades, niveles, Eras, familias y capacidades | Datos inmutables y configuracion externa |
| WaveManager | Ciclo de oleadas, dificultad, composicion, rutas y transiciones | Maquina de estados y configuracion de balance |
| CombatSystem | Resolucion de alcance, objetivos, dano, cooldowns, efectos y bajas | Simulacion por fases, pooling y posible procesamiento por lotes |
| SpatialBuffSystem | Activacion, area, duracion y acumulacion de Monolitos Temporales | Modificadores de dominio y eventos de expiracion |
| ShopSystem | Inventario, compra, reroll, expansion y costes entre oleadas | Politicas de economia y ofertas deterministas |
| SynergySystem | Evaluacion de Eras, familias y condiciones del tablero | Motor de reglas desacoplado y recalculo incremental |
| Progression System | Desbloqueo de Eras, recompensas, oleada maxima y metaprogresion | Estado persistente y politicas de desbloqueo |
| Economy System | Monedas blandas, gemas, costes, recompensas y sinks | Ledger de transacciones y validacion |
| Persistence System | Guardado, carga, backup, versionado y migracion | Puerto de almacenamiento + snapshots |
| Monetization Services | Ads, IAP, validacion de recompensas y disponibilidad | Adaptadores de SDK y contratos de servicio |
| Analytics Service | Eventos de producto, economia, combate y revenue | Event sink asincrono y desacoplado |
| Custom Tick Manager | Actualizacion centralizada de sistemas registrados | Player Loop y contratos de tick |
| Pooling Layer | Reutilizacion de entidades visuales y objetos de combate | Object Pooling |
| Presentation Layer | Vistas, HUD, feedback, transiciones y entrada | Presenter/View, uGUI o UI Toolkit |

### 4.3 Flujo principal de una partida

1. La aplicacion carga el perfil y la configuracion de la Era actual.
2. El flujo de preparacion crea el tablero, aplica sinergias y genera el inventario de tienda.
3. El jugador compra, rerollear o reorganiza unidades y Monolitos dentro de las reglas disponibles.
4. `WaveManager` confirma la configuracion de la oleada y publica el inicio de combate.
5. `CombatSystem` crea o recupera entidades, procesa la simulacion por fases y publica resultados.
6. Los sistemas de tablero, buffs y sinergias reaccionan a cambios relevantes sin referencias directas entre ellos.
7. Al terminar la oleada, el sistema de recompensas liquida monedas, progreso, oleada maxima y analytics.
8. La aplicacion entra en resultado, tienda o recuperacion y permite una nueva decision del jugador.
9. El sistema de persistencia toma un snapshot en puntos seguros y ante eventos de ciclo de vida.

## 5) Diseno Funcional y Tecnico

### 5.1 Casos de uso o flujos clave

#### Preparar el tablero

El jugador observa el inventario disponible, coloca unidades en casillas validas y recibe feedback de ocupacion, compatibilidad de fusion, sinergias activas y efectos de Monolitos. La vista nunca decide reglas; solicita validaciones al dominio.

#### Fusionar unidades

El jugador selecciona dos unidades compatibles. El Board System valida nivel, familia, Era, espacio de resultado y restricciones de combate. Si la operacion es valida, emite un resultado de fusion y los sistemas dependientes recalculan sinergias, capacidades y presentacion.

#### Resolver una oleada

`WaveManager` selecciona una definicion de oleada y publica sus parametros. `CombatSystem` procesa movimiento, seleccion de objetivos, ataques, dano, estados y bajas. Las vistas reciben snapshots o eventos agregados, no controlan la simulacion.

#### Usar un Monolito Temporal

El jugador activa o coloca un Monolito en una region autorizada. `SpatialBuffSystem` valida coste, duracion y compatibilidad; registra el modificador y notifica a combate, tablero y sinergias. La expiracion revierte el efecto sin reconstruir todo el estado cuando sea posible.

#### Comprar y rerollear

`ShopSystem` ofrece un inventario segun la fase, la Era y las reglas de economia. La compra consume recursos mediante `EconomySystem`; el reroll usa monedas blandas o una recompensa autorizada; la expansion del tablero actualiza la capacidad y los limites del Board System.

#### Avanzar de Era

Al cumplir condiciones de progreso, `Progression System` desbloquea una nueva Era, su catalogo y reglas. El cambio debe ser compatible con saves anteriores y no invalidar unidades ya desplegadas sin una politica de migracion definida.

### 5.2 Contratos e interfaces

Los contratos se definen por responsabilidad, sin exponer implementaciones de Unity al dominio.

**Puertos de comandos:**

- Solicitud de colocar, mover o fusionar una unidad.
- Solicitud de iniciar, acelerar, pausar o finalizar una oleada.
- Solicitud de activar un Monolito Temporal.
- Solicitud de comprar, rerollear o ampliar el tablero.
- Solicitud de reclamar una recompensa.
- Solicitud de aplicar una recompensa de monetizacion validada.

**Lecturas del dominio:**

- Snapshot del tablero y sus restricciones.
- Snapshot de oleada, entidades activas y progreso de combate.
- Estado de buffs espaciales y sinergias.
- Inventario y precios de tienda.
- Balances, progreso de Eras y oleada maxima.
- Estado de disponibilidad de anuncios y catalogo IAP.

**Eventos de dominio:**

- Tablero cambiado.
- Fusion completada o rechazada.
- Sinergias recalculadas.
- Oleada iniciada, progresada, completada o perdida.
- Entidad creada, danada, derrotada o llegada al objetivo.
- Monolito activado o expirado.
- Compra, reroll o expansion completados.
- Recompensa economica liquidada.
- Era desbloqueada.
- Save creado, cargado, migrado o rechazado.

**Integraciones externas:**

- Adaptador de Rewarded Ads e Interstitials.
- Adaptador de compras de plataforma.
- Adaptador de almacenamiento local.
- Adaptador de analytics y revenue.
- Adaptador de tiempo y ciclo de vida de la aplicacion.

### 5.3 Modelo de datos

- **Perfil de jugador:** identidad local, configuracion, preferencias y estado de monetizacion.
- **Progreso de Era:** Eras desbloqueadas, Era activa, unidades descubiertas y mejoras permanentes.
- **Estado del tablero:** dimensiones, casillas, unidades, niveles, familias, modificadores y capacidad.
- **Definicion de unidad:** identidad de contenido, Era, familia, estadisticas base, capacidades y reglas de fusion.
- **Estado de oleada:** numero, dificultad, semilla, fase, entidades, objetivos y resultado.
- **Estado de Monolitos:** ubicacion, tipo, magnitud, duracion, origen y reglas de acumulacion.
- **Estado de tienda:** ofertas, inventario, rerolls utilizados, compras y expiraciones.
- **Economia:** monedas blandas, gemas, recursos de progresion, ledger de transacciones y fuentes/sinks.
- **Oleada maxima:** mejor marca global y marcas por Era o nivel de dificultad.
- **Recompensas pendientes:** recompensas obtenidas pero aun no liquidadas, con idempotencia.
- **Snapshot persistente:** version de esquema, timestamp, checksum, estado necesario para reanudar y metadatos de migracion.

La persistencia es local para el primer MVP. Los snapshots se escriben en puntos seguros, mantienen backup y se migran por version. La economia y las recompensas se registran como transacciones para poder auditar duplicados, fallos y reclamaciones.

### 5.4 Estrategia de monetizacion

- **Rewarded Ads:** revivir una defensa perdida bajo limites, obtener un reroll gratuito o duplicar una recompensa offline. Cada recompensa requiere un token de oportunidad, confirmacion del SDK y liquidacion idempotente.
- **Interstitials:** solo en transiciones entre oleadas o pantallas, nunca durante la toma de decisiones ni en mitad de combate. Deben respetar cooldown, frecuencia maxima y estado de consentimiento.
- **IAP:** venta de gemas o moneda dura mediante catalogo de plataforma. El dominio recibe una compra validada, no un callback sin verificar.
- **Analitica:** registrar impresiones, completados, recompensas, compras, abandonos, oleadas, tiempo de combate y cambios economicos sin bloquear el hilo de simulacion.
- **Proteccion de experiencia:** la monetizacion complementa la estrategia y no debe ser requisito para completar el contenido del MVP.

## 6) Decisiones de Arquitectura (ADR breve)

| ID | Decision | Alternativas evaluadas | Motivo | Fecha |
|---|---|---|---|---|
| ADR-001 | Sistemas de dominio desacoplados | Logica en MonoBehaviours | Permite testear, simular y escalar el juego sin acoplar reglas a la escena | 2026-08-10 |
| ADR-002 | VContainer como composition root | Singletons y localizadores globales | Hace explicitas las dependencias y facilita sustituir SDKs y backends | 2026-08-10 |
| ADR-003 | Event Channels tipados | Referencias directas entre sistemas | Reduce acoplamiento y permite reaccionar a cambios de dominio | 2026-08-10 |
| ADR-004 | Tick centralizado | Update por entidad | Controla orden, coste y medicion del runtime critico | 2026-08-10 |
| ADR-005 | Simulacion por fases y pooling | Instanciacion libre por evento | Reduce allocations y hace predecible el coste de oleadas numerosas | 2026-08-10 |
| ADR-006 | Persistencia local versionada | PlayerPrefs sin esquema | Permite backups, migraciones y recuperacion de progreso | 2026-08-10 |
| ADR-007 | Monetizacion mediante puertos | SDKs invocados desde gameplay | Aisla plataformas y permite tests sin dependencias externas | 2026-08-10 |
| ADR-008 | Eras como contenido configurable | Reglas codificadas por Era | Permite balancear y ampliar la fantasia sin duplicar sistemas | 2026-08-10 |

## 7) Riesgos y Mitigaciones

| Riesgo | Impacto | Mitigacion | Duenio |
|---|---|---|---|
| Combate costoso con muchas entidades | Alto | Profiling desde el primer milestone, fases de simulacion, pooling y procesamiento por lotes cuando proceda | Tech Lead |
| Complejidad de sinergias y buffs | Alto | Motor de reglas con prioridad, stacking y trazabilidad definidos antes del contenido | Game Design |
| Economia inflacionaria | Alto | Ledger de fuentes/sinks, simulaciones offline y limites de recompensa | Economy Design |
| Perdida o duplicacion de progreso | Alto | Saves versionados, backups, checksum y recompensas idempotentes | Backend/Client |
| Cambios de Era rompen contenido anterior | Medio | Catalogo versionado y politicas de compatibilidad/migracion | Content Tech |
| Monetizacion intrusiva | Alto | Frecuencia limitada, puntos naturales de transicion y pruebas de retencion | Product |
| Dependencia de SDKs externos | Medio | Puertos, adaptadores fake para tests y degradacion controlada offline | Tech Lead |
| Alcance demasiado amplio | Alto | Vertical slice con una Era, un tablero, pocas unidades y oleadas representativas antes de escalar | Product |

## 8) Preguntas Abiertas y Proximos Pasos

- Definir el nombre final, tono visual y reglas de la fantasia de Eras del Hombre.
- Confirmar las primeras Eras incluidas en el vertical slice.
- Definir si la fusion es el centro del tablero o un recurso limitado durante la preparacion.
- Determinar la forma de las rutas enemigas y si existe mas de un objetivo defensivo.
- Establecer el numero inicial de unidades, familias, niveles y sinergias.
- Definir reglas exactas de Monolitos Temporales: colocacion, coste, duracion y acumulacion.
- Elegir UI Toolkit o uGUI y fijar la plataforma inicial.
- Definir calendario de recompensas offline y limites de rewarded ads.
- Confirmar si el primer MVP requiere cuenta, cloud save o solo persistencia local.
- Crear una matriz de trazabilidad entre requisito, sistema, evento, vista y test.

## 9) Criterios de Aceptacion

- [ ] Un vertical slice con una Era permite preparar un tablero, fusionar unidades y completar varias oleadas.
- [ ] El resultado de una misma simulacion es reproducible con la misma configuracion y semilla.
- [ ] El combate mantiene un coste medible y estable con la poblacion objetivo de entidades.
- [ ] Monolitos Temporales modifican casillas, expiran correctamente y no dejan estado residual.
- [ ] La tienda, el reroll y la expansion validan costes y actualizan el tablero sin duplicar recursos.
- [ ] Las sinergias se recalculan de forma consistente tras colocar, fusionar, retirar o mejorar unidades.
- [ ] El progreso de Era, tablero, oleada maxima y economia sobrevive a cerrar y reabrir la aplicacion.
- [ ] Un save corrupto o de version anterior se recupera o migra mediante una politica verificable.
- [ ] Rewarded Ads, interstitials e IAP estan aislados por adaptadores y sus recompensas son idempotentes.
- [ ] Los eventos de analytics no bloquean la simulacion ni exponen datos sensibles.
- [ ] El camino critico de combate, oleadas y actualizacion de tablero no genera allocations por frame en la carga objetivo.
- [ ] Los tests cubren reglas de dominio, flujos de integracion y un escenario completo de vertical slice.

## 10) Referencias

- Documento base: `template-architecture.md`.
- Backlog: `backlog.md` en esta carpeta.
- Plan de implementacion: `implementation-plan.md` en esta carpeta.
- Repositorio de implementacion Unity: `merge-tower-unity`.
- Referencia funcional de producto: *Harvest King: Farm Defense TD*, utilizada como referencia de genero y loop, no como copia de assets o contenido propietario.
