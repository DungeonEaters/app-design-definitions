# Template de Arquitectura — `merge-tower`

## 1) Resumen Ejecutivo
- **Objetivo:** Prototipo de monetización hyper-casual. Validar engagement y revenue mediante Rewarded Ads, Intersticiales e IAP en un loop de juego Merge + Stack.
- **Aplicación o módulo:** merge-tower
- **Estado:** Draft
- **Responsable técnico:** Por definir
- **Última actualización:** 2026-05-24

## 2) Contexto y Alcance

### 2.1 Problema a resolver
Validar si un loop de juego simple (merge + progresión visual de torre) genera suficiente retención y engagement para monetizar mediante anuncios e IAP antes de invertir en un producto más complejo.

### 2.2 Alcance incluido
- Loop de juego: cuadrícula de merge (4x4) + torre de progresión visual
- Sistema de spawn de elementos (normales y raros)
- Economía idle: coins/sec generados por nivel de torre
- Sistema de mejoras con curva exponencial de costos
- Mecánicas de enganche: preview de siguiente merge, elemento raro aleatorio, objetivo diario
- Monetización: Rewarded Ads (AppLovin MAX) + IAP básico (Unity IAP)
- Analítica de ROAS: Tenjin con captura de eCPM
- Guardado local (JSON en `Application.persistentDataPath`)
- Build pipeline CI/CD para Android (APK/AAB) e iOS
- Entorno de desarrollo contenedorizado (DevContainer + VS Code)

### 2.3 Fuera de alcance
- Backend propio / guardado en la nube
- Multijugador o features sociales
- Sistema de prestige o reset
- Combo multiplier
- Más de una escena de juego activa
- Evolución hacia RPG u otros géneros

## 3) Requisitos

### 3.1 Funcionales
- El jugador puede arrastrar dos elementos iguales en la cuadrícula para mergearlos
- Al mergear, el elemento resultante se suma visualmente a la torre
- La torre genera coins/sec de forma pasiva según su nivel acumulado
- Cada N segundos aparece un nuevo elemento en una celda vacía
- Con baja probabilidad configurable, el elemento spawneado es "raro" (vale 3x al mergearlo)
- El jugador siempre ve qué elemento resultará del próximo merge posible
- Existe un objetivo diario visible con recompensa al completarse
- El jugador puede ver un Rewarded Ad para obtener un elemento extra o duplicar ganancias por 2h
- El jugador puede comprar slots extra o moneda premium mediante IAP
- El progreso se guarda localmente de forma automática

### 3.2 No funcionales
- **Rendimiento:** Cero métodos `Update()` nativos en objetos duplicables; Custom Update Manager via `PlayerLoopSystem`
- **Memoria:** Zero-allocation en runtime crítico; Object Pooling para elementos de cuadrícula, textos flotantes y partículas
- **Escalabilidad:** Arquitectura desacoplada mediante Event Channels; lógica de negocio en POCOs sin dependencia de Unity
- **Mantenibilidad:** Inyección de dependencias exclusivamente via VContainer; sin Singletons tradicionales
- **Seguridad:** Validación de IAP delegada a Google Play / App Store; sin backend propio en MVP
- **Experiencia de uso:** Init de SDK de ads asíncrona para no bloquear el hilo principal al arrancar

## 4) Arquitectura Propuesta

### 4.1 Diagrama de alto nivel

```
┌─────────────────────────────────────────────┐
│             Presentation Layer              │
│   Unity uGUI / UI Toolkit                   │
│   Event Channels (ScriptableObjects)        │
├─────────────────────────────────────────────┤
│             Game Logic Layer                │
│   POCOs puros en C# (sin MonoBehaviour)     │
│   VContainer (DI via LifetimeScopes)        │
│   Custom Update Manager (PlayerLoopSystem)  │
├─────────────────────────────────────────────┤
│             Data Layer                      │
│   Structs serializables (C# puro)           │
│   JSON local (persistentDataPath)           │
├─────────────────────────────────────────────┤
│             Services Layer                  │
│   AppLovin MAX  │  Unity IAP  │  Tenjin     │
└─────────────────────────────────────────────┘
```

### 4.2 Componentes principales

| Componente | Responsabilidad | Tecnología |
|---|---|---|
| `MergeGridSystem` | Lógica de cuadrícula, validación de merge, preview de resultado | POCO + VContainer |
| `TowerProgressionSystem` | Altura de torre, cálculo de coins/sec por nivel acumulado | POCO + VContainer |
| `EconomySystem` | Balance de monedas (`long`), aplicación de costos exponenciales | POCO + VContainer |
| `SpawnSystem` | Spawn periódico de elementos normales y raros con probabilidad configurable | POCO + Object Pool |
| `DailyGoalSystem` | Tracking de merges diarios, emisión de recompensa al completar objetivo | POCO + VContainer |
| `AdsService` | Init asíncrona de AppLovin MAX, rewarded y intersticiales, callback de eCPM | MonoBehaviour wrapper |
| `IAPService` | Compras de slots y moneda premium, validación via tienda | MonoBehaviour wrapper |
| `TenjinService` | Captura de eCPM desde callback MAX, serialización JSON, envío a Tenjin | POCO |
| `SaveSystem` | Serialización/deserialización de `PlayerSaveData` a JSON local | POCO |
| `CustomUpdateManager` | Loop central de tick; reemplaza `Update()` en todos los sistemas | PlayerLoopSystem |

### 4.3 Flujo principal

1. App arranca → `CustomUpdateManager` se registra en `PlayerLoopSystem`
2. `AdsService.InitializeAsync()` corre en background; cachea rewarded e intersticial
3. `SaveSystem` carga `PlayerSaveData` desde disco; VContainer inyecta estado en sistemas
4. `SpawnSystem` emite elementos en celdas vacías cada N segundos via tick
5. Jugador arrastra elemento A sobre elemento B igual → `MergeGridSystem` valida merge
6. Merge válido → `OnMergeCompleted` event channel notifica a `TowerProgressionSystem`, VFX y UI
7. `TowerProgressionSystem` recalcula coins/sec → `EconomySystem` acumula en tick
8. `OnCoinsChanged` event channel actualiza HUD
9. `DailyGoalSystem` recibe `OnMergeCompleted`, incrementa contador, emite `OnDailyGoalProgress`
10. Al cerrar app → `SaveSystem` serializa estado a JSON local

## 5) Diseño Funcional y Técnico

### 5.1 Casos de uso clave

**UC-01: Merge de elementos**
- Precondición: dos celdas con elementos del mismo nivel
- Flujo: drag & drop → validación → pool recicla elemento origen → elemento destino sube de nivel → event `OnMergeCompleted`
- Preview: `MergeGridSystem` expone `GetMergeResult(elementLevel)` consultado por UI en hover/drag

**UC-02: Spawn de elemento raro**
- `SpawnSystem` evalúa `RareSpawnChance` (configurable via ScriptableObject) en cada spawn
- Si raro: elemento spawneado con flag `IsRare = true`, emite `OnRareElementSpawned`
- UI aplica highlight visual; al mergearlo, `EconomySystem` aplica multiplicador 3x

**UC-03: Objetivo diario**
- `DailyGoalSystem` persiste fecha UTC del último reset en `PlayerSaveData`
- Al arrancar, compara fecha actual; si es día nuevo, resetea contador y genera nuevo objetivo
- Al completar: emite `OnDailyGoalCompleted` → UI ofrece Rewarded Ad para duplicar recompensa

**UC-04: Rewarded Ad**
- Jugador toca botón "Ver ad" → `AdsService.ShowRewarded(RewardType)`
- Callback `OnAdRewardGranted` emite `RewardType` (ExtraElement / DoubleIncome2h)
- `SpawnSystem` o `EconomySystem` aplica efecto según tipo

**UC-05: IAP**
- `IAPService` inicializa catálogo al arrancar
- Jugador compra → validación via Google Play / App Store → `OnPurchaseCompleted` event
- `EconomySystem` o `MergeGridSystem` aplica beneficio (slots extra, moneda premium)

### 5.2 Contratos e interfaces

**Interfaces de sistemas (POCOs):**
```
ITickable              → tick por frame via CustomUpdateManager
IMergeGridSystem       → ValidateMerge(), ExecuteMerge(), GetMergeResult()
ITowerProgressionSystem → AddMergedElement(level), GetCoinsPerSecond()
IEconomySystem         → AddCoins(long), SpendCoins(long), GetBalance()
ISpawnSystem           → TrySpawn(), SetSpawnInterval(float)
IDailyGoalSystem       → RecordMerge(), GetCurrentGoal(), ClaimReward()
ISaveSystem            → Save(PlayerSaveData), Load() : PlayerSaveData
IAdsService            → InitializeAsync(), ShowRewarded(RewardType), ShowInterstitial()
IIAPService            → Initialize(), Purchase(productId)
ITenjinService         → TrackAdRevenue(AdRevenuePayload)
```

**Event Channels (ScriptableObjects):**
```
GameEventChannel<MergeResult>     OnMergeCompleted
GameEventChannel<long>            OnCoinsChanged
GameEventChannel<ElementData>     OnRareElementSpawned
GameEventChannel<GoalProgress>    OnDailyGoalProgress
GameEventChannel<RewardType>      OnAdRewardGranted
GameEventChannel<string>          OnPurchaseCompleted
```

### 5.3 Modelo de datos

**Structs serializables (C# puro, zero-allocation):**
```csharp
struct ElementData    { int Level; bool IsRare; }
struct MergeResult    { int ResultLevel; long BonusCoins; }
struct GoalProgress   { int Current; int Target; GoalType Type; }
struct AdRevenuePayload { double eCPM; string networkName; string adUnitId; }

// Clase serializable para guardado (no struct, necesita serialización JSON)
class PlayerSaveData  {
    long Coins;
    int[] GridState;       // nivel de cada celda, 0 = vacía
    int TowerLevel;
    long CoinsPerSecond;
    int DailyGoalProgress;
    string LastGoalResetDate; // UTC ISO 8601
    bool AdsRemoved;
    Dictionary<string, bool> PurchasedProducts;
}
```

**Fórmula de costos de mejora:**
```
Costo(nivel) = CostoBase × Multiplicador^(nivel - 1)
```
Valores iniciales sugeridos: `CostoBase = 100`, `Multiplicador = 1.8`

**Fórmula de coins/sec de torre:**
```
CoinsPerSecond = SumaDeTodosLosNivelesEnTorre × MultiplicadorBase
```

**Retención y trazabilidad:**
- `PlayerSaveData` se guarda en `Application.persistentDataPath/save.json`
- Backup automático en `save.backup.json` antes de cada escritura
- Tenjin recibe payload de revenue en cada callback de ad completado

## 6) Decisiones de Arquitectura (ADR breve)

| ID | Decisión | Alternativas evaluadas | Motivo | Fecha |
|---|---|---|---|---|
| ADR-001 | VContainer para DI | Zenject, Singletons manuales | Zero-allocation, constructor injection en POCOs, mejor rendimiento de GC | 2026-05-24 |
| ADR-002 | Custom Update Manager via PlayerLoopSystem | Update() nativo, Coroutines | Elimina overhead de Update() en objetos duplicables; itera arrays contiguos | 2026-05-24 |
| ADR-003 | ScriptableObject Event Channels | UnityEvents, C# events directos, MessagePipe | Desacopla UI/VFX de lógica sin referencias directas; configurable en editor | 2026-05-24 |
| ADR-004 | UnityEngine.Pool para pooling | Custom pool, Instantiate/Destroy | API nativa Unity 2021+; zero-allocation en runtime crítico | 2026-05-24 |
| ADR-005 | Guardado JSON local | PlayerPrefs, SQLite, Unity Gaming Services | Sin backend en MVP; structs serializables son suficientes; fácil de migrar a nube después | 2026-05-24 |
| ADR-006 | AppLovin MAX como mediador de ads | AdMob directo, IronSource | Mayor fill rate por mediación; rewarded + intersticiales; callback de eCPM para Tenjin | 2026-05-24 |
| ADR-007 | `long` para monedas | `int`, `double`, `BigInteger` | Previene overflow en idle tardío sin overhead de BigInteger | 2026-05-24 |
| ADR-008 | IL2CPP para builds móviles | Mono | Rendimiento nativo en Android/iOS; requerido para App Store | 2026-05-24 |
| ADR-009 | Sin backend propio en MVP | Cloud Run + GCP, Firebase | Reduce tiempo de desarrollo; validación de IAP delegada a tiendas; expandible después | 2026-05-24 |

## 7) Riesgos y Mitigaciones

| Riesgo | Impacto | Mitigación | Dueño |
|---|---|---|---|
| AppLovin MAX no cachea ad a tiempo en primer arranque | Alto: jugador ve botón de ad sin ad disponible | Deshabilitar botón hasta `OnAdLoaded` callback; mostrar spinner | Dev |
| Guardado local corrupto (crash durante escritura) | Alto: pérdida de progreso | Escribir a `save.backup.json` antes de sobreescribir `save.json` | Dev |
| Overflow de `long` en coins (teórico) | Bajo: `long` soporta ~9.2 × 10^18 | Suficiente para MVP; documentar límite | Dev |
| IL2CPP build time lento en CI | Medio: pipeline lento | Cachear Library/ en GitHub Actions; usar Unity Build Automation si escala | DevOps |
| Fill rate bajo de ads en LATAM | Alto: revenue bajo | AppLovin MAX con múltiples redes; configurar floor price | Dev |
| Rechazo de App Store por ads agresivos | Alto: no publicación | Intersticiales solo en transiciones naturales (no durante gameplay) | Dev |

## 8) Preguntas Abiertas y Próximos Pasos

- Definir nombre final del juego y assets visuales mínimos para prototipo
- Decidir tamaño de cuadrícula inicial (4x4 vs 5x5)
- Definir cuántos niveles de elemento existen (sugerido: 8-10 para MVP)
- Configurar floor price de ads en AppLovin MAX dashboard
- Definir catálogo de IAP (IDs de producto, precios)
- Evaluar si agregar backend (Unity Gaming Services) tras validar retención

## 9) Criterios de Aceptación

- [ ] Loop de merge funciona sin crashes en 10 minutos de sesión continua
- [ ] Torre crece visualmente con cada merge completado
- [ ] Coins/sec se acumula correctamente en background (idle)
- [ ] Elemento raro aparece con probabilidad configurable y aplica bonus 3x
- [ ] Preview de resultado de merge visible durante drag
- [ ] Objetivo diario se resetea correctamente al cambiar de día UTC
- [ ] Rewarded Ad se muestra y entrega recompensa correcta
- [ ] Intersticial se muestra solo en transiciones (no durante gameplay)
- [ ] IAP completa ciclo de compra y aplica beneficio
- [ ] eCPM capturado y enviado a Tenjin en cada ad completado
- [ ] Guardado local persiste entre sesiones sin corrupción
- [ ] Build de Android (AAB) e iOS (Xcode project) generados por CI/CD sin errores
- [ ] Cero allocations en Profiler durante tick de gameplay (merge, spawn, economy)

## 10) Referencias

- Documentos relacionados: `backlog.md` (tareas de implementación)
- Repositorios: Por definir (Unity project repo separado)
- Tickets/Issues: Por definir
