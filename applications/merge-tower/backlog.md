# Backlog de Desarrollo — `merge-tower`

> **Cómo usar este backlog con Copilot/OpenCode:**
> Pide las tareas de una en una. Ejemplo: *"@workspace Implementa la tarea T2.1: CustomUpdateManager"*.
> Nunca pidas más de una tarea a la vez para garantizar código limpio y sin allocations.

---

## Fase 1: Configuración del Entorno de Desarrollo

### T1.1: Configuración de DevContainer para C# Dev Kit

**Objetivo:** Crear un entorno de desarrollo reproducible y aislado del host para editar y analizar código C# de Unity sin depender de la instalación local de .NET.

**Guía de implementación técnica:**
- Crear `.devcontainer/devcontainer.json` con imagen base `mcr.microsoft.com/dotnet/sdk:8.0`
- Instalar extensiones: `ms-dotnettools.csharp`, `ms-dotnettools.csdevkit`, `editorconfig.editorconfig`
- Montar carpeta del proyecto Unity como volumen en `/workspace`
- Agregar `.editorconfig` con reglas de estilo C# (indentación, naming conventions)
- Agregar `omnisharp.json` apuntando al `.sln` del proyecto Unity
- El editor Unity corre en el host nativo (GPU); el DevContainer solo maneja C# LSP y linting

**Definition of Done (DoD):**
- [ ] `Dev Containers: Reopen in Container` abre el proyecto sin errores
- [ ] IntelliSense resuelve referencias de Unity (UnityEngine, VContainer) dentro del contenedor
- [ ] Linter reporta violaciones de estilo en archivos `.cs`
- [ ] Unity Editor en host puede compilar el mismo código sin conflictos

**GC Checkpoint:** No aplica directamente. Configurar el analizador `Roslynator` para detectar en tiempo de edición patrones que generan allocations (LINQ en hot paths, boxing de structs, string concatenation en Update).

---

### T1.2: Estructura de Carpetas del Proyecto Unity

**Objetivo:** Establecer una estructura de carpetas que refleje la arquitectura en capas y facilite la navegación y el trabajo con VContainer LifetimeScopes.

**Guía de implementación técnica:**
```
Assets/
├── _Project/
│   ├── Scripts/
│   │   ├── Core/           # CustomUpdateManager, interfaces base
│   │   ├── Systems/        # MergeGrid, Tower, Economy, Spawn, DailyGoal
│   │   ├── Services/       # Ads, IAP, Tenjin, Save
│   │   ├── Data/           # Structs, ScriptableObjects de configuración
│   │   ├── Events/         # GameEventChannel ScriptableObjects
│   │   ├── UI/             # Presenters, Views (sin lógica de negocio)
│   │   └── DI/             # LifetimeScopes de VContainer
│   ├── Prefabs/
│   ├── ScriptableObjects/
│   └── Scenes/
├── Plugins/                # AppLovin MAX SDK, Tenjin SDK
└── Tests/
    ├── EditMode/           # Tests de sistemas puros (Economy, MergeGrid)
    └── PlayMode/           # Tests de integración
```

**Definition of Done (DoD):**
- [ ] Estructura de carpetas creada y commiteada
- [ ] `.gitignore` configurado para Unity (Library/, Temp/, Build/, *.csproj generados)
- [ ] Assembly Definitions (`.asmdef`) creados por capa para reducir tiempos de compilación
- [ ] VContainer importado via Package Manager (com.annulusgames.vcontainer o scoped registry)

**GC Checkpoint:** Configurar Assembly Definitions correctamente evita recompilaciones innecesarias de todo el proyecto. No es un checkpoint de GC en runtime pero sí de productividad crítica.

---

### T1.3: Configuración de GitHub Actions CI/CD

**Objetivo:** Pipeline automatizado que compile el proyecto con IL2CPP para Android (AAB) e iOS (Xcode project) y ejecute los tests unitarios en cada push a `main`.

**Guía de implementación técnica:**
- Crear `.github/workflows/build.yml`
- Usar `game-ci/unity-builder@v4` action con licencia Unity personal/pro via secrets
- Jobs paralelos: `test` (EditMode + PlayMode), `build-android`, `build-ios`
- Cachear `Library/` con `actions/cache` usando hash de `Assets/` y `Packages/`
- Build Android: `buildTarget: Android`, `androidExportType: androidAppBundle` (AAB)
- Build iOS: `buildTarget: iOS` → genera Xcode project (no compila Xcode en CI por costo)
- Subir artefactos con `actions/upload-artifact`
- Secret `UNITY_LICENSE`, `UNITY_EMAIL`, `UNITY_PASSWORD` en GitHub repo settings

**Definition of Done (DoD):**
- [ ] Push a `main` dispara pipeline sin errores
- [ ] Tests EditMode pasan en CI
- [ ] AAB de Android generado como artefacto descargable
- [ ] Xcode project de iOS generado como artefacto
- [ ] Cache de `Library/` reduce tiempo de build en segunda ejecución >50%

**GC Checkpoint:** IL2CPP elimina el JIT de Mono; las allocations en runtime son más costosas de recuperar. El pipeline debe incluir el paso de tests con el Profiler en modo `Deep Profile` desactivado pero con `Allocation Tracking` activo para detectar regresiones de memoria en PlayMode tests.

---

## Fase 2: Arquitectura del Motor de Juego (Foundation)

### T2.1: Custom Update Manager via PlayerLoopSystem

**Objetivo:** Reemplazar todos los métodos `Update()` nativos de Unity con un sistema centralizado que itera arrays contiguos de `ITickable`, eliminando el overhead del mensaje de Unity por objeto.

**Guía de implementación técnica:**
```csharp
// Core/ITickable.cs
public interface ITickable { void Tick(float deltaTime); }
public interface IFixedTickable { void FixedTick(float fixedDeltaTime); }

// Core/CustomUpdateManager.cs
public sealed class CustomUpdateManager
{
    private readonly List<ITickable> _tickables = new(64);
    // Usar List<T> pre-allocada; NO usar LINQ para iterar

    public void Register(ITickable tickable) => _tickables.Add(tickable);
    public void Unregister(ITickable tickable) => _tickables.Remove(tickable);

    // Inyectado en PlayerLoopSystem en el slot Update
    internal void Tick(float deltaTime)
    {
        // Iterar con for clásico, NO foreach sobre List<T> (evita enumerator allocation)
        for (int i = 0; i < _tickables.Count; i++)
            _tickables[i].Tick(deltaTime);
    }
}

// DI/ProjectLifetimeScope.cs (VContainer)
public class ProjectLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.RegisterEntryPoint<CustomUpdateManagerInstaller>();
        builder.Register<CustomUpdateManager>(Lifetime.Singleton);
    }
}
```
- Registrar `CustomUpdateManager.Tick` en `PlayerLoopSystem` en el slot `Update` usando `PlayerLoopHelper` o manualmente con `PlayerLoop.SetPlayerLoop`

**Definition of Done (DoD):**
- [ ] Ningún sistema de juego tiene método `Update()` propio
- [ ] `CustomUpdateManager` registrado en `PlayerLoopSystem` y ejecutándose cada frame
- [ ] VContainer inyecta `CustomUpdateManager` como singleton en `ProjectLifetimeScope`
- [ ] Test EditMode verifica que `Register`/`Unregister` funcionan correctamente

**GC Checkpoint:** Usar `for` clásico en lugar de `foreach` sobre `List<T>` evita la allocación del `Enumerator`. Pre-alocar la lista con capacidad inicial (`new List<ITickable>(64)`) evita resize allocations. Nunca usar LINQ (`_tickables.Where(...)`) en el path de tick.

---

### T2.2: VContainer LifetimeScopes y Registro de Sistemas

**Objetivo:** Configurar el grafo de dependencias completo del juego usando VContainer, con inyección por constructor en todos los POCOs.

**Guía de implementación técnica:**
```csharp
// DI/GameLifetimeScope.cs
public class GameLifetimeScope : LifetimeScope
{
    [SerializeField] private GameConfig _gameConfig; // ScriptableObject con parámetros

    protected override void Configure(IContainerBuilder builder)
    {
        // Sistemas como singletons
        builder.Register<EconomySystem>(Lifetime.Singleton).AsImplementedInterfaces();
        builder.Register<MergeGridSystem>(Lifetime.Singleton).AsImplementedInterfaces();
        builder.Register<TowerProgressionSystem>(Lifetime.Singleton).AsImplementedInterfaces();
        builder.Register<SpawnSystem>(Lifetime.Singleton).AsImplementedInterfaces();
        builder.Register<DailyGoalSystem>(Lifetime.Singleton).AsImplementedInterfaces();
        builder.Register<SaveSystem>(Lifetime.Singleton).AsImplementedInterfaces();

        // Servicios externos como wrappers
        builder.RegisterComponentInHierarchy<AdsService>().AsImplementedInterfaces();
        builder.RegisterComponentInHierarchy<IAPService>().AsImplementedInterfaces();

        // Config
        builder.RegisterInstance(_gameConfig);

        // Entry points (ITickable se registra automáticamente con VContainer)
        builder.RegisterEntryPoint<SpawnSystem>();
        builder.RegisterEntryPoint<EconomySystem>();
    }
}
```
- Todos los sistemas reciben dependencias via constructor, nunca via `[Inject]` en campos
- `GameConfig` es un `ScriptableObject` con todos los parámetros configurables (spawn interval, rare chance, upgrade costs base, multiplicador)

**Definition of Done (DoD):**
- [ ] Todos los sistemas resueltos por VContainer sin errores en Play Mode
- [ ] Ningún sistema usa `FindObjectOfType`, `GetComponent` o Singleton manual
- [ ] `GameConfig` ScriptableObject con valores por defecto configurado en editor
- [ ] Test EditMode verifica resolución del contenedor con `IContainerBuilder` mock

**GC Checkpoint:** VContainer usa constructor injection que no genera allocations adicionales en runtime (solo en setup). Evitar `Resolve<T>()` en métodos `Tick()`; todas las dependencias deben estar inyectadas en construcción.

---

### T2.3: Event Channels basados en ScriptableObjects

**Objetivo:** Implementar el sistema de comunicación desacoplada entre sistemas, UI y VFX mediante canales de eventos tipados como ScriptableObjects.

**Guía de implementación técnica:**
```csharp
// Events/GameEventChannel.cs
[CreateAssetMenu]
public class GameEventChannel<T> : ScriptableObject
{
    // Usar Action en lugar de UnityEvent para evitar overhead de serialización
    public event Action<T> OnEventRaised;

    public void Raise(T value) => OnEventRaised?.Invoke(value);
}

// Canales concretos (necesarios por limitación de serialización de Unity):
// Events/MergeCompletedChannel.cs
[CreateAssetMenu(menuName = "Events/MergeCompleted")]
public class MergeCompletedChannel : GameEventChannel<MergeResult> { }

// Repetir para: CoinsChangedChannel, RareElementChannel,
//               DailyGoalChannel, AdRewardChannel, PurchaseChannel
```
- Crear assets de cada canal en `Assets/_Project/ScriptableObjects/Events/`
- Los sistemas emiten via referencia al canal inyectada por VContainer
- Los Presenters de UI se suscriben en `OnEnable` y desuscriben en `OnDisable`

**Definition of Done (DoD):**
- [ ] 6 canales de eventos creados como assets en el editor
- [ ] `MergeGridSystem` emite `OnMergeCompleted` al ejecutar merge válido
- [ ] UI HUD se actualiza al recibir `OnCoinsChanged` sin referencia directa a `EconomySystem`
- [ ] Test EditMode verifica que Raise() notifica a suscriptores correctamente

**GC Checkpoint:** Usar `Action<T>` en lugar de `UnityEvent<T>` evita la allocación de `InvocationList` de UnityEvent. Suscribir/desuscribir en `OnEnable`/`OnDisable` (no en `Start`/`OnDestroy`) para evitar suscriptores muertos que causan memory leaks.

---

### T2.4: Object Pooling para Elementos de Cuadrícula y VFX

**Objetivo:** Implementar pools para los elementos visuales de la cuadrícula, textos flotantes de ganancias y partículas de merge, eliminando allocations de `Instantiate`/`Destroy` en runtime.

**Guía de implementación técnica:**
```csharp
// Core/ElementPool.cs
public sealed class ElementPool
{
    private readonly IObjectPool<ElementView> _pool;

    public ElementPool(ElementView prefab, Transform parent)
    {
        _pool = new ObjectPool<ElementView>(
            createFunc: () => Object.Instantiate(prefab, parent),
            actionOnGet: view => view.gameObject.SetActive(true),
            actionOnRelease: view => view.gameObject.SetActive(false),
            actionOnDestroy: view => Object.Destroy(view.gameObject),
            collectionCheck: false, // false en release para performance
            defaultCapacity: 20,
            maxSize: 50
        );
    }

    public ElementView Get() => _pool.Get();
    public void Release(ElementView view) => _pool.Release(view);
}
```
- Pool separado para: `ElementView` (piezas de cuadrícula), `FloatingTextView` (ganancias), `MergeParticleView` (VFX)
- Registrar pools en VContainer como singletons
- `SpawnSystem` usa `ElementPool.Get()` en lugar de `Instantiate`

**Definition of Done (DoD):**
- [ ] Profiler muestra 0 allocations de `GC.Alloc` durante spawn y merge en Play Mode
- [ ] Pool de `ElementView` pre-calienta 20 instancias al arrancar
- [ ] `FloatingTextView` retorna al pool automáticamente al terminar su animación
- [ ] Test PlayMode verifica que el pool no crece indefinidamente en 100 merges consecutivos

**GC Checkpoint:** `collectionCheck: false` en builds de release evita la verificación de doble-release que usa un `HashSet` interno. Pre-calentar el pool en `Awake` con `defaultCapacity` evita allocations en el primer frame de gameplay. Nunca llamar `_pool.Get()` dentro de un loop sin garantizar `Release()` correspondiente.

---

## Fase 3: Lógica de Negocio y Sistemas de Progresión

### T3.1: Modelo de Datos — Structs y PlayerSaveData

**Objetivo:** Definir todos los tipos de datos del juego como structs de C# puro (zero-allocation) y la clase de guardado serializable.

**Guía de implementación técnica:**
```csharp
// Data/GameDataTypes.cs
public readonly struct ElementData
{
    public readonly int Level;
    public readonly bool IsRare;
    public ElementData(int level, bool isRare) { Level = level; IsRare = isRare; }
}

public readonly struct MergeResult
{
    public readonly int ResultLevel;
    public readonly long BonusCoins;
}

public readonly struct GoalProgress
{
    public readonly int Current;
    public readonly int Target;
    public readonly GoalType Type;
}

public readonly struct AdRevenuePayload
{
    public readonly double eCPM;
    public readonly string NetworkName;
    public readonly string AdUnitId;
}

// Data/PlayerSaveData.cs — clase (no struct) para serialización JSON
[Serializable]
public class PlayerSaveData
{
    public long Coins;
    public int[] GridState = new int[16]; // 4x4, 0 = vacía
    public int TowerLevel;
    public long CoinsPerSecond;
    public int DailyGoalProgress;
    public string LastGoalResetDateUtc;
    public bool AdsRemoved;
    public List<string> PurchasedProductIds = new();
}
```

**Definition of Done (DoD):**
- [ ] Todos los structs son `readonly` para garantizar inmutabilidad
- [ ] `PlayerSaveData` serializa/deserializa correctamente con `JsonUtility`
- [ ] Test EditMode verifica round-trip de serialización sin pérdida de datos
- [ ] No existen clases mutables para datos volátiles de gameplay (usar structs)

**GC Checkpoint:** Structs `readonly` pasados por valor no generan heap allocations. Evitar boxing de structs (no asignar a `object` o interfaces no genéricas). `JsonUtility` no genera allocations intermedias a diferencia de `Newtonsoft.Json`.

---

### T3.2: EconomySystem — Balance de Monedas y Costos Exponenciales

**Objetivo:** Implementar el sistema de economía con soporte `long`, acumulación idle de coins/sec y fórmula exponencial de costos de mejora.

**Guía de implementación técnica:**
```csharp
// Systems/EconomySystem.cs
public sealed class EconomySystem : IEconomySystem, ITickable
{
    private long _coins;
    private long _coinsPerSecond;
    private float _accumulatedTime;
    private readonly CoinsChangedChannel _coinsChangedChannel;
    private readonly GameConfig _config;

    // Constructor injection via VContainer
    public EconomySystem(CoinsChangedChannel coinsChangedChannel, GameConfig config)
    {
        _coinsChangedChannel = coinsChangedChannel;
        _config = config;
    }

    public void Tick(float deltaTime)
    {
        _accumulatedTime += deltaTime;
        if (_accumulatedTime < 1f) return; // Tick cada segundo

        _accumulatedTime -= 1f;
        AddCoins(_coinsPerSecond);
    }

    public void AddCoins(long amount)
    {
        _coins += amount;
        _coinsChangedChannel.Raise(_coins);
    }

    public bool SpendCoins(long amount)
    {
        if (_coins < amount) return false;
        _coins -= amount;
        _coinsChangedChannel.Raise(_coins);
        return true;
    }

    // Costo(nivel) = CostoBase × Multiplicador^(nivel-1)
    public long GetUpgradeCost(int currentLevel)
    {
        return (long)(_config.UpgradeCostBase * Math.Pow(_config.CostMultiplier, currentLevel - 1));
    }

    public long GetBalance() => _coins;
    public void SetCoinsPerSecond(long cps) => _coinsPerSecond = cps;
}
```

**Definition of Done (DoD):**
- [ ] `GetUpgradeCost(nivel)` retorna valores correctos para niveles 1-20 verificados en test
- [ ] `AddCoins` con `long.MaxValue - 1` no produce overflow (verificar con test)
- [ ] Coins se acumulan correctamente en idle (verificar con tiempo simulado en test)
- [ ] `OnCoinsChanged` se emite solo cuando el balance cambia, no en cada tick sin cambio

**GC Checkpoint:** El tick de economía no debe generar ninguna allocation. Evitar `string.Format` o interpolación de strings en `Tick()`. El canal de eventos usa `Action<long>` (value type), no genera boxing si los suscriptores usan `long` directamente.

---

### T3.3: MergeGridSystem — Lógica de Cuadrícula y Merge

**Objetivo:** Implementar la lógica de la cuadrícula 4x4, validación de merge, ejecución de merge y cálculo de preview de resultado.

**Guía de implementación técnica:**
```csharp
// Systems/MergeGridSystem.cs
public sealed class MergeGridSystem : IMergeGridSystem
{
    // Array contiguo en memoria para mejor cache performance
    private readonly ElementData[] _grid = new ElementData[16]; // 4x4
    private readonly MergeCompletedChannel _mergeChannel;
    private readonly TowerProgressionSystem _tower;
    private readonly EconomySystem _economy;

    public bool ValidateMerge(int cellA, int cellB)
    {
        if (cellA == cellB) return false;
        if (_grid[cellA].Level == 0 || _grid[cellB].Level == 0) return false;
        return _grid[cellA].Level == _grid[cellB].Level;
    }

    public MergeResult ExecuteMerge(int cellA, int cellB)
    {
        var result = new MergeResult(
            resultLevel: _grid[cellA].Level + 1,
            bonusCoins: _grid[cellA].IsRare || _grid[cellB].IsRare
                ? _grid[cellA].Level * 3L
                : _grid[cellA].Level * 1L
        );

        _grid[cellB] = new ElementData(result.ResultLevel, false);
        _grid[cellA] = default; // celda vacía

        _economy.AddCoins(result.BonusCoins);
        _tower.AddMergedElement(result.ResultLevel);
        _mergeChannel.Raise(result);

        return result;
    }

    public int GetMergeResult(int cellLevel) => cellLevel + 1;

    public bool HasEmptyCell(out int cellIndex)
    {
        for (int i = 0; i < _grid.Length; i++)
        {
            if (_grid[i].Level == 0) { cellIndex = i; return true; }
        }
        cellIndex = -1;
        return false;
    }
}
```

**Definition of Done (DoD):**
- [ ] `ValidateMerge` retorna false para celdas vacías, misma celda, o niveles distintos
- [ ] `ExecuteMerge` actualiza grid, notifica tower y economy correctamente
- [ ] `GetMergeResult` retorna nivel correcto para preview en UI
- [ ] Test EditMode cubre: merge válido, merge inválido, merge con elemento raro, grid llena

**GC Checkpoint:** Array `ElementData[]` fijo en tamaño (no `List<T>`) garantiza memoria contigua y sin resize allocations. `ElementData` es struct, asignación a array no genera heap allocation. Evitar `new ElementData(...)` en `Tick()`; solo en `ExecuteMerge` que es event-driven.

---

### T3.4: TowerProgressionSystem — Progresión Visual y Coins/Sec

**Objetivo:** Mantener el estado de la torre, recalcular coins/sec al agregar elementos mergeados y notificar a la UI para actualización visual.

**Guía de implementación técnica:**
```csharp
// Systems/TowerProgressionSystem.cs
public sealed class TowerProgressionSystem : ITowerProgressionSystem
{
    private int _totalTowerLevel;
    private long _coinsPerSecond;
    private readonly EconomySystem _economy;
    private readonly GameConfig _config;

    public void AddMergedElement(int elementLevel)
    {
        _totalTowerLevel += elementLevel;
        RecalculateCoinsPerSecond();
    }

    private void RecalculateCoinsPerSecond()
    {
        // CoinsPerSecond = SumaNiveles × MultiplicadorBase
        _coinsPerSecond = (long)(_totalTowerLevel * _config.CoinsPerSecondMultiplier);
        _economy.SetCoinsPerSecond(_coinsPerSecond);
    }

    public long GetCoinsPerSecond() => _coinsPerSecond;
    public int GetTowerLevel() => _totalTowerLevel;
}
```

**Definition of Done (DoD):**
- [ ] `AddMergedElement` actualiza coins/sec correctamente verificado en test
- [ ] `EconomySystem` refleja nuevo coins/sec en el siguiente tick
- [ ] Test EditMode verifica progresión de coins/sec con 10 merges consecutivos

**GC Checkpoint:** `RecalculateCoinsPerSecond` es llamado solo en eventos de merge (no en tick), por lo que el cast `(long)` no es un problema de performance. Sin allocations en este sistema.

---

### T3.5: SpawnSystem — Spawn Periódico y Elementos Raros

**Objetivo:** Generar elementos en celdas vacías de la cuadrícula a intervalos configurables, con probabilidad de spawn de elemento raro.

**Guía de implementación técnica:**
```csharp
// Systems/SpawnSystem.cs
public sealed class SpawnSystem : ISpawnSystem, ITickable
{
    private float _timer;
    private readonly MergeGridSystem _grid;
    private readonly ElementPool _pool;
    private readonly RareElementChannel _rareChannel;
    private readonly GameConfig _config;
    // System.Random pre-instanciado, no new Random() en Tick
    private readonly System.Random _random = new();

    public void Tick(float deltaTime)
    {
        _timer += deltaTime;
        if (_timer < _config.SpawnIntervalSeconds) return;
        _timer -= _config.SpawnIntervalSeconds;
        TrySpawn();
    }

    public void TrySpawn()
    {
        if (!_grid.HasEmptyCell(out int cellIndex)) return;

        bool isRare = _random.NextDouble() < _config.RareSpawnChance;
        int level = 1; // Siempre nivel 1 en spawn base

        var element = _pool.Get();
        element.Initialize(new ElementData(level, isRare), cellIndex);

        if (isRare) _rareChannel.Raise(new ElementData(level, true));
    }
}
```

**Definition of Done (DoD):**
- [ ] Spawn ocurre cada `SpawnIntervalSeconds` configurado en `GameConfig`
- [ ] Probabilidad de raro verificada estadísticamente en test (1000 spawns, ~X% raros)
- [ ] No spawna si no hay celdas vacías
- [ ] `ElementPool` provee elementos sin `Instantiate` en runtime

**GC Checkpoint:** `System.Random` pre-instanciado evita `new Random()` en cada tick. `_random.NextDouble()` no genera allocations. El timer usa resta (`_timer -= interval`) en lugar de reset a 0 para evitar drift acumulado.

---

### T3.6: DailyGoalSystem — Objetivos Diarios y Recompensas

**Objetivo:** Trackear el progreso de merges diarios, detectar cambio de día UTC y emitir recompensa al completar el objetivo.

**Guía de implementación técnica:**
```csharp
// Systems/DailyGoalSystem.cs
public sealed class DailyGoalSystem : IDailyGoalSystem
{
    private int _mergeCount;
    private readonly DailyGoalChannel _goalChannel;
    private readonly SaveSystem _save;
    private readonly GameConfig _config;

    public void Initialize(PlayerSaveData saveData)
    {
        CheckDayReset(saveData);
        _mergeCount = saveData.DailyGoalProgress;
    }

    private void CheckDayReset(PlayerSaveData saveData)
    {
        string todayUtc = DateTime.UtcNow.ToString("yyyy-MM-dd");
        if (saveData.LastGoalResetDateUtc == todayUtc) return;

        _mergeCount = 0;
        saveData.DailyGoalProgress = 0;
        saveData.LastGoalResetDateUtc = todayUtc;
    }

    public void RecordMerge()
    {
        _mergeCount++;
        var progress = new GoalProgress(_mergeCount, _config.DailyGoalTarget, GoalType.Merge);
        _goalChannel.Raise(progress);

        if (_mergeCount >= _config.DailyGoalTarget)
            OnGoalCompleted();
    }

    private void OnGoalCompleted() { /* Emitir evento de recompensa disponible */ }
}
```

**Definition of Done (DoD):**
- [ ] Objetivo se resetea al cambiar de día UTC (verificado con fecha simulada en test)
- [ ] Progreso persiste entre sesiones via `PlayerSaveData`
- [ ] UI recibe `OnDailyGoalProgress` y actualiza barra de progreso
- [ ] Test EditMode verifica reset de día y acumulación de merges

**GC Checkpoint:** `DateTime.UtcNow.ToString("yyyy-MM-dd")` genera una string allocation. Aceptable porque ocurre solo en `Initialize` (una vez por sesión), no en tick. Evitar llamar `DateTime.UtcNow` en `RecordMerge()` si se puede evitar.

---

### T3.7: SaveSystem — Guardado y Carga Local

**Objetivo:** Serializar y deserializar `PlayerSaveData` a JSON local con backup automático para prevenir corrupción.

**Guía de implementación técnica:**
```csharp
// Services/SaveSystem.cs
public sealed class SaveSystem : ISaveSystem
{
    private static readonly string SavePath =
        Path.Combine(Application.persistentDataPath, "save.json");
    private static readonly string BackupPath =
        Path.Combine(Application.persistentDataPath, "save.backup.json");

    public void Save(PlayerSaveData data)
    {
        // Backup antes de sobreescribir
        if (File.Exists(SavePath))
            File.Copy(SavePath, BackupPath, overwrite: true);

        string json = JsonUtility.ToJson(data, prettyPrint: false);
        File.WriteAllText(SavePath, json);
    }

    public PlayerSaveData Load()
    {
        if (!File.Exists(SavePath))
            return new PlayerSaveData(); // Estado inicial

        try
        {
            string json = File.ReadAllText(SavePath);
            return JsonUtility.FromJson<PlayerSaveData>(json);
        }
        catch
        {
            // Intentar recuperar desde backup
            if (!File.Exists(BackupPath)) return new PlayerSaveData();
            string backupJson = File.ReadAllText(BackupPath);
            return JsonUtility.FromJson<PlayerSaveData>(backupJson);
        }
    }
}
```
- Llamar `Save()` en `OnApplicationPause(true)` y `OnApplicationFocus(false)` desde un `MonoBehaviour` wrapper
- Llamar `Save()` cada 60 segundos como autosave via `CustomUpdateManager`

**Definition of Done (DoD):**
- [ ] `Save` + `Load` round-trip preserva todos los campos de `PlayerSaveData`
- [ ] Backup se crea antes de cada escritura
- [ ] Si `save.json` está corrupto, carga desde `save.backup.json`
- [ ] Test EditMode verifica round-trip y recuperación desde backup

**GC Checkpoint:** `JsonUtility.ToJson` genera una string allocation (aceptable, ocurre en save, no en tick). `File.WriteAllText` es síncrono; para sesiones largas considerar `async/await` con `File.WriteAllTextAsync` para no bloquear el hilo principal en guardados grandes.

---

## Fase 4: Integración del SDK de Monetización y Atribución

### T4.1: AppLovin MAX — Inicialización Asíncrona y Configuración

**Objetivo:** Integrar el SDK de AppLovin MAX con inicialización asíncrona que no bloquee el hilo principal, con caché preventivo de ads al arrancar.

**Guía de implementación técnica:**
```csharp
// Services/AdsService.cs
public sealed class AdsService : MonoBehaviour, IAdsService
{
    [SerializeField] private string _rewardedAdUnitId;
    [SerializeField] private string _interstitialAdUnitId;

    private bool _isInitialized;
    private readonly AdRewardChannel _rewardChannel;

    private void Awake()
    {
        MaxSdkCallbacks.OnSdkInitializedEvent += OnSdkInitialized;
        MaxSdk.SetSdkKey("YOUR_SDK_KEY");
        MaxSdk.InitializeSdk(); // Asíncrono, no bloquea
    }

    private void OnSdkInitialized(MaxSdkBase.SdkConfiguration config)
    {
        _isInitialized = true;
        InitializeRewardedAds();
        InitializeInterstitialAds();
    }

    private void InitializeRewardedAds()
    {
        MaxSdkCallbacks.Rewarded.OnAdLoadedEvent += OnRewardedAdLoaded;
        MaxSdkCallbacks.Rewarded.OnAdHiddenEvent += OnRewardedAdHidden;
        MaxSdkCallbacks.Rewarded.OnAdReceivedRewardEvent += OnRewardReceived;
        MaxSdkCallbacks.Rewarded.OnAdRevenuePaidEvent += OnAdRevenuePaid;
        MaxSdk.LoadRewardedAd(_rewardedAdUnitId);
    }

    public void ShowRewarded(RewardType rewardType)
    {
        if (!MaxSdk.IsRewardedAdReady(_rewardedAdUnitId)) return;
        _pendingRewardType = rewardType;
        MaxSdk.ShowRewardedAd(_rewardedAdUnitId);
    }

    private void OnAdRevenuePaid(string adUnitId, MaxSdkBase.AdInfo adInfo)
    {
        // Captura eCPM y delega a TenjinService
        var payload = new AdRevenuePayload(
            adInfo.Revenue * 1000, // eCPM
            adInfo.NetworkName,
            adUnitId
        );
        _tenjinService.TrackAdRevenue(payload);
    }
}
```

**Definition of Done (DoD):**
- [ ] SDK inicializa sin bloquear el hilo principal (verificar con Profiler)
- [ ] Rewarded ad cargado antes de que el jugador pueda tocar el botón
- [ ] Botón de rewarded deshabilitado hasta `OnAdLoaded` callback
- [ ] Intersticial se muestra solo en transiciones (no durante gameplay activo)
- [ ] `OnAdRevenuePaid` captura `Revenue` y `NetworkName` correctamente

**GC Checkpoint:** Los callbacks de MAX son invocados en el hilo principal de Unity. Evitar allocations en `OnAdRevenuePaid` (struct `AdRevenuePayload` pasado por valor). No crear strings en callbacks de ads; usar constantes o campos pre-asignados.

---

### T4.2: Tenjin — Tracking de Revenue y ROAS

**Objetivo:** Enviar el payload de revenue de cada ad completado a Tenjin para el cálculo de ROAS, serializado en JSON.

**Guía de implementación técnica:**
```csharp
// Services/TenjinService.cs
public sealed class TenjinService : ITenjinService
{
    private BaseTenjin _instance;

    public void Initialize(string apiKey)
    {
        _instance = Tenjin.getInstance(apiKey);
        _instance.Connect();
    }

    public void TrackAdRevenue(AdRevenuePayload payload)
    {
        // Tenjin SDK método de ad revenue
        _instance.TrackAdRevenue(
            adPlatform: "applovin_max",
            adSource: payload.NetworkName,
            adUnitName: payload.AdUnitId,
            adFormat: "rewarded",
            value: payload.eCPM / 1000.0, // Convertir eCPM a revenue
            currency: "USD"
        );
    }
}
```

**Definition of Done (DoD):**
- [ ] `Connect()` llamado en cada sesión (requerido por Tenjin para atribución)
- [ ] `TrackAdRevenue` invocado en cada `OnAdRevenuePaid` de MAX
- [ ] Verificar en Tenjin dashboard que eventos llegan correctamente (sandbox mode)
- [ ] No se envían datos PII (Personally Identifiable Information) a Tenjin

**GC Checkpoint:** `TrackAdRevenue` del SDK de Tenjin puede generar allocations internas (SDK de tercero). Aceptable porque ocurre en evento de ad completado, no en tick de gameplay. Verificar en Profiler que no hay spike de GC durante el callback.

---

### T4.3: Unity IAP — Compras Integradas

**Objetivo:** Integrar Unity IAP para compras de slots extra y moneda premium, con validación delegada a Google Play / App Store.

**Guía de implementación técnica:**
```csharp
// Services/IAPService.cs
public sealed class IAPService : MonoBehaviour, IAdsService,
    IStoreListener
{
    private IStoreController _storeController;
    private readonly PurchaseChannel _purchaseChannel;

    // IDs de producto (deben coincidir con Google Play Console / App Store Connect)
    private const string ExtraSlotsProductId = "merge_tower_extra_slots";
    private const string PremiumCoinsProductId = "merge_tower_premium_coins_1";
    private const string RemoveAdsProductId = "merge_tower_remove_ads";

    public void Initialize()
    {
        var builder = ConfigurationBuilder.Instance(StandardPurchasingModule.Instance());
        builder.AddProduct(ExtraSlotsProductId, ProductType.NonConsumable);
        builder.AddProduct(PremiumCoinsProductId, ProductType.Consumable);
        builder.AddProduct(RemoveAdsProductId, ProductType.NonConsumable);
        UnityPurchasing.Initialize(this, builder);
    }

    public PurchaseProcessingResult ProcessPurchase(PurchaseEventArgs args)
    {
        _purchaseChannel.Raise(args.purchasedProduct.definition.id);
        return PurchaseProcessingResult.Complete;
    }

    public void OnInitializeFailed(InitializationFailureReason error) { /* Log */ }
    public void OnPurchaseFailed(Product product, PurchaseFailureReason reason) { /* Log */ }
    public void OnInitialized(IStoreController controller, IExtensionProvider extensions)
        => _storeController = controller;
}
```

**Definition of Done (DoD):**
- [ ] 3 productos configurados en Google Play Console (Internal Testing track)
- [ ] Compra de `RemoveAdsProductId` persiste en `PlayerSaveData.AdsRemoved`
- [ ] Compra de `ExtraSlotsProductId` desbloquea 2 celdas extra en la cuadrícula
- [ ] Restore purchases funciona en iOS
- [ ] Test con sandbox account completa ciclo de compra sin errores

**GC Checkpoint:** `ProcessPurchase` es event-driven (no en tick). La string del product ID puede ser comparada con `==` directamente (string interning para constantes). Evitar `string.Contains` o LINQ en el procesamiento de compras.

---

## Fase 5: Testing y Validación

### T5.1: Tests Unitarios de Sistemas de Economía y Progresión

**Objetivo:** Cubrir con tests unitarios los sistemas puros de C# (Economy, MergeGrid, Tower, DailyGoal) usando Unity Test Framework (NUnit).

**Guía de implementación técnica:**
```csharp
// Tests/EditMode/EconomySystemTests.cs
[TestFixture]
public class EconomySystemTests
{
    private EconomySystem _economy;
    private GameConfig _config;

    [SetUp]
    public void SetUp()
    {
        _config = ScriptableObject.CreateInstance<GameConfig>();
        _config.UpgradeCostBase = 100;
        _config.CostMultiplier = 1.8f;
        _economy = new EconomySystem(null, _config); // null channel para unit test
    }

    [Test]
    public void GetUpgradeCost_Level1_ReturnsBase()
        => Assert.AreEqual(100L, _economy.GetUpgradeCost(1));

    [Test]
    public void GetUpgradeCost_Level2_ReturnsBaseTimesMultiplier()
        => Assert.AreEqual(180L, _economy.GetUpgradeCost(2));

    [Test]
    public void AddCoins_LongMaxValue_DoesNotOverflow()
    {
        _economy.AddCoins(long.MaxValue - 1);
        Assert.AreEqual(long.MaxValue - 1, _economy.GetBalance());
    }

    [Test]
    public void SpendCoins_InsufficientBalance_ReturnsFalse()
    {
        _economy.AddCoins(50L);
        Assert.IsFalse(_economy.SpendCoins(100L));
    }
}
```
- Tests similares para `MergeGridSystem`, `TowerProgressionSystem`, `DailyGoalSystem`
- Tests de `SaveSystem` con archivos temporales en `Path.GetTempPath()`

**Definition of Done (DoD):**
- [ ] Cobertura >80% en sistemas de lógica pura (Economy, MergeGrid, Tower, DailyGoal, Save)
- [ ] Todos los tests pasan en CI (GitHub Actions)
- [ ] Tests corren en <5 segundos en EditMode
- [ ] Ningún test usa `Thread.Sleep` o tiempo real; usar tiempo simulado

**GC Checkpoint:** Los tests en EditMode no tienen el overhead del runtime de Unity. Verificar que los sistemas bajo test no generan allocations inesperadas usando `GC.GetTotalMemory(false)` antes y después de operaciones críticas en tests de performance.

---

### T5.2: Test de Integración del Loop Principal (PlayMode)

**Objetivo:** Verificar que el loop completo (spawn → merge → tower → economy → save) funciona correctamente en Play Mode sin crashes ni memory leaks.

**Guía de implementación técnica:**
```csharp
// Tests/PlayMode/GameLoopIntegrationTest.cs
[UnityTest]
public IEnumerator MergeLoop_100Merges_NoGCAllocation()
{
    // Setup escena de test con VContainer
    yield return new WaitForSeconds(0.1f); // Esperar init

    long gcBefore = GC.GetTotalMemory(false);

    for (int i = 0; i < 100; i++)
    {
        // Simular merge via sistema directamente
        _mergeGrid.ExecuteMerge(0, 1);
        yield return null;
    }

    long gcAfter = GC.GetTotalMemory(false);
    long allocated = gcAfter - gcBefore;

    // Tolerancia de 0 bytes en path crítico
    Assert.AreEqual(0, allocated, $"GC allocated {allocated} bytes during merge loop");
}
```

**Definition of Done (DoD):**
- [ ] 100 merges consecutivos sin GC allocation en path crítico
- [ ] Pool de elementos no crece más allá de `maxSize` configurado
- [ ] `PlayerSaveData` persiste correctamente entre `LoadScene` en test
- [ ] No hay `NullReferenceException` en 10 minutos de gameplay simulado

**GC Checkpoint:** Este test ES el checkpoint de GC. Si falla, usar el Profiler con `Deep Profile` para identificar el origen de la allocation y corregirlo antes de marcar la tarea como completada.

---

## Fase 6: Empaquetado Final y Distribución

### T6.1: Configuración de Build para Android (AAB con IL2CPP)

**Objetivo:** Configurar el proyecto Unity para generar un Android App Bundle (AAB) optimizado con IL2CPP listo para Google Play Internal Testing.

**Guía de implementación técnica:**
- `Player Settings > Android`:
  - Scripting Backend: `IL2CPP`
  - Target Architectures: `ARM64` (obligatorio para Google Play desde 2019) + `ARMv7`
  - Build System: `Gradle`
  - Export as Google Android Project: `false` (AAB directo)
  - Minify: `Release` con R8
- Configurar `keystore` para firma del AAB (secrets en GitHub Actions)
- `Build Settings`: `Development Build: false`, `Strip Engine Code: true`
- Subir AAB a Google Play Console > Internal Testing track

**Definition of Done (DoD):**
- [ ] AAB generado sin errores con IL2CPP en CI
- [ ] AAB instalable en dispositivo Android físico (ARM64)
- [ ] Tamaño de APK base <100MB (sin assets de alta resolución en MVP)
- [ ] Subido a Google Play Console Internal Testing sin errores de validación

**GC Checkpoint:** IL2CPP convierte C# a C++ nativo. Las allocations que en Mono son recuperadas por GC en IL2CPP tienen mayor costo de recuperación. Todos los checkpoints de GC anteriores son críticos para IL2CPP. Verificar con `IL2CPP Managed Stripping Level: High` que no se eliminan tipos necesarios (agregar `link.xml` si es necesario).

---

### T6.2: Configuración de Build para iOS (Xcode Project)

**Objetivo:** Configurar el proyecto Unity para generar un Xcode project válido para iOS, listo para compilar y subir a TestFlight.

**Guía de implementación técnica:**
- `Player Settings > iOS`:
  - Scripting Backend: `IL2CPP` (único soportado en iOS)
  - Target minimum iOS version: `14.0`
  - Architecture: `ARM64`
  - Enable Bitcode: `false` (deprecado por Apple)
- Configurar `Bundle Identifier`, `Team ID` y provisioning profile via `xcconfig`
- En CI: generar Xcode project (no compilar Xcode en CI Linux; usar macOS runner o Xcode Cloud)
- Post-process build script (`PostProcessBuild`) para agregar capabilities de iOS (In-App Purchase, Push Notifications si aplica)

**Definition of Done (DoD):**
- [ ] Xcode project generado sin errores en CI
- [ ] Proyecto abre en Xcode sin errores de configuración
- [ ] Build de Xcode en macOS local genera IPA instalable via TestFlight
- [ ] AppLovin MAX y Unity IAP configurados correctamente en Xcode (frameworks, plist)

**GC Checkpoint:** Mismo que T6.1. En iOS, el GC de IL2CPP es el único mecanismo de recuperación de memoria. Los checkpoints de fases anteriores son la garantía de rendimiento en producción.

---

## Resumen de Dependencias entre Tareas

```
T1.1 → T1.2 → T2.1 → T2.2 → T2.3 → T2.4
                              ↓
                    T3.1 → T3.2 → T3.3 → T3.4 → T3.5 → T3.6 → T3.7
                                                                   ↓
                                                    T4.1 → T4.2 → T4.3
                                                                   ↓
                                                    T5.1 → T5.2
                                                                   ↓
T1.3 ──────────────────────────────────────────────────── T6.1 → T6.2
```

## Estimación de Esfuerzo (referencial)

| Fase | Tareas | Esfuerzo estimado |
|---|---|---|
| Fase 1: Entorno | T1.1, T1.2, T1.3 | 2-3 días |
| Fase 2: Foundation | T2.1, T2.2, T2.3, T2.4 | 3-4 días |
| Fase 3: Lógica | T3.1 – T3.7 | 5-7 días |
| Fase 4: Monetización | T4.1, T4.2, T4.3 | 3-4 días |
| Fase 5: Testing | T5.1, T5.2 | 2-3 días |
| Fase 6: Build | T6.1, T6.2 | 1-2 días |
| **Total MVP** | **20 tareas** | **~16-23 días** |
