# MergeTower MVP — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> (recommended) or superpowers:executing-plans to implement this plan task-by-task.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construir un prototipo de juego hyper-casual Merge + Tower que valide monetización mediante Rewarded Ads (AppLovin MAX), IAP (Unity IAP) y analítica de ROAS (Tenjin), con arquitectura zero-allocation lista para producción móvil.

**Architecture:** Cuatro capas desacopladas (Presentation / Game Logic / Data / Services). Lógica de negocio en POCOs puros inyectados por VContainer. Comunicación entre capas exclusivamente via Event Channels (ScriptableObjects). Custom Update Manager reemplaza todos los Update() nativos.

**Tech Stack:** Unity 2022 LTS · C# · VContainer · UnityEngine.Pool · AppLovin MAX SDK · Unity IAP · Tenjin SDK · NUnit (Unity Test Framework) · GitHub Actions (game-ci/unity-builder) · IL2CPP · Android AAB · iOS Xcode project

**Spec:** Ver `architecture.md` y `backlog.md` en este mismo directorio.

---

## Estructura de archivos

```
merge-tower-unity/
├── .devcontainer/
│   ├── devcontainer.json
│   └── Dockerfile
├── .github/
│   └── workflows/
│       └── build.yml
├── .editorconfig
├── omnisharp.json
├── Assets/
│   └── _Project/
│       ├── Scripts/
│       │   ├── Core/
│       │   │   ├── ITickable.cs
│       │   │   └── CustomUpdateManager.cs
│       │   ├── DI/
│       │   │   ├── ProjectLifetimeScope.cs
│       │   │   └── GameLifetimeScope.cs
│       │   ├── Data/
│       │   │   ├── GameDataTypes.cs
│       │   │   ├── PlayerSaveData.cs
│       │   │   └── GameConfig.cs
│       │   ├── Events/
│       │   │   ├── GameEventChannel.cs
│       │   │   ├── MergeCompletedChannel.cs
│       │   │   ├── CoinsChangedChannel.cs
│       │   │   ├── RareElementChannel.cs
│       │   │   ├── DailyGoalChannel.cs
│       │   │   ├── AdRewardChannel.cs
│       │   │   └── PurchaseChannel.cs
│       │   ├── Systems/
│       │   │   ├── EconomySystem.cs
│       │   │   ├── MergeGridSystem.cs
│       │   │   ├── TowerProgressionSystem.cs
│       │   │   ├── SpawnSystem.cs
│       │   │   └── DailyGoalSystem.cs
│       │   ├── Services/
│       │   │   ├── SaveSystem.cs
│       │   │   ├── AdsService.cs
│       │   │   ├── IAPService.cs
│       │   │   ├── TenjinService.cs
│       │   │   └── AutoSaveController.cs
│       │   ├── Pool/
│       │   │   └── ElementPool.cs
│       │   └── UI/
│       │       ├── ElementView.cs
│       │       └── HudPresenter.cs
│       └── _Project.asmdef
└── Assets/Tests/
    ├── EditMode/
    │   ├── TickableTests.cs
    │   ├── EventChannelTests.cs
    │   ├── SaveDataTests.cs
    │   ├── SaveSystemTests.cs
    │   ├── EconomySystemTests.cs
    │   ├── MergeGridSystemTests.cs
    │   ├── TowerProgressionSystemTests.cs
    │   ├── DailyGoalSystemTests.cs
    │   └── SpawnSystemTests.cs
    ├── PlayMode/
    │   └── GameLoopIntegrationTests.cs
    ├── Tests.EditMode.asmdef
    └── Tests.PlayMode.asmdef
```

---

## Fase 1 — Configuración del Entorno

### Task 1: DevContainer para C# Dev Kit

**Files:**
- Create: `merge-tower-unity/.devcontainer/Dockerfile`
- Create: `merge-tower-unity/.devcontainer/devcontainer.json`
- Create: `merge-tower-unity/.editorconfig`
- Create: `merge-tower-unity/omnisharp.json`

- [ ] Crear `merge-tower-unity/.devcontainer/Dockerfile`:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0

RUN apt-get update && apt-get install -y \
    git \
    curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /workspace
```

- [ ] Crear `merge-tower-unity/.devcontainer/devcontainer.json`:

```json
{
  "name": "MergeTower C# Dev",
  "build": { "dockerfile": "Dockerfile" },
  "workspaceFolder": "/workspace",
  "workspaceMount": "source=${localWorkspaceFolder},target=/workspace,type=bind",
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-dotnettools.csharp",
        "ms-dotnettools.csdevkit",
        "editorconfig.editorconfig",
        "josefpihrt-vscode.roslynator"
      ],
      "settings": {
        "omnisharp.useModernNet": true,
        "omnisharp.enableRoslynAnalyzers": true
      }
    }
  },
  "postCreateCommand": "dotnet --version"
}
```

- [ ] Crear `merge-tower-unity/.editorconfig`:

```ini
root = true

[*.cs]
indent_style = space
indent_size = 4
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

dotnet_naming_rule.private_fields.symbols = private_fields
dotnet_naming_rule.private_fields.style = underscore_prefix
dotnet_naming_rule.private_fields.severity = warning
dotnet_naming_symbols.private_fields.applicable_kinds = field
dotnet_naming_symbols.private_fields.applicable_accessibilities = private
dotnet_naming_style.underscore_prefix.capitalization = camel_case
dotnet_naming_style.underscore_prefix.required_prefix = _
```

- [ ] Crear `merge-tower-unity/omnisharp.json`:

```json
{
  "solution": "merge-tower-unity.sln",
  "enableRoslynAnalyzers": true,
  "enableEditorConfigSupport": true
}
```

- [ ] Abrir VS Code, ejecutar `Dev Containers: Reopen in Container`, verificar que el terminal muestra `dotnet --version` >= 8.0 sin errores.

- [ ] Commit:

```bash
git add .devcontainer/ .editorconfig omnisharp.json
git commit -m "chore: add devcontainer for C# development"
```

---

### Task 2: Estructura de carpetas Unity y Assembly Definitions

**Files:**
- Create: `merge-tower-unity/Assets/_Project/_Project.asmdef`
- Create: `merge-tower-unity/Assets/Tests/Tests.EditMode.asmdef`
- Create: `merge-tower-unity/Assets/Tests/Tests.PlayMode.asmdef`

- [ ] En Unity Editor, crear la estructura de carpetas bajo `Assets/_Project/Scripts/` con las subcarpetas: `Core`, `DI`, `Data`, `Events`, `Systems`, `Services`, `Pool`, `UI`. Crear también `Assets/_Project/ScriptableObjects/Events/` y `Assets/_Project/Prefabs/`.

- [ ] Crear `Assets/_Project/_Project.asmdef` via Unity Editor (click derecho en `Assets/_Project/` > Create > Assembly Definition):

```json
{
  "name": "MergeTower",
  "rootNamespace": "MergeTower",
  "references": [],
  "includePlatforms": [],
  "excludePlatforms": [],
  "allowUnsafeCode": false,
  "overrideReferences": false,
  "autoReferenced": true
}
```

- [ ] Crear `Assets/Tests/Tests.EditMode.asmdef`:

```json
{
  "name": "MergeTower.Tests.EditMode",
  "rootNamespace": "MergeTower.Tests",
  "references": ["MergeTower", "UnityEngine.TestRunner", "UnityEditor.TestRunner"],
  "includePlatforms": ["Editor"],
  "excludePlatforms": [],
  "allowUnsafeCode": false,
  "overrideReferences": true,
  "precompiledReferences": ["nunit.framework.dll"],
  "autoReferenced": false,
  "defineConstraints": ["UNITY_INCLUDE_TESTS"]
}
```

- [ ] Crear `Assets/Tests/Tests.PlayMode.asmdef`:

```json
{
  "name": "MergeTower.Tests.PlayMode",
  "rootNamespace": "MergeTower.Tests",
  "references": ["MergeTower", "UnityEngine.TestRunner"],
  "includePlatforms": [],
  "excludePlatforms": [],
  "allowUnsafeCode": false,
  "overrideReferences": true,
  "precompiledReferences": ["nunit.framework.dll"],
  "autoReferenced": false,
  "defineConstraints": ["UNITY_INCLUDE_TESTS"]
}
```

- [ ] Instalar VContainer via Package Manager: `Window > Package Manager > + > Add package by name` → `com.annulusgames.vcontainer`.

- [ ] Verificar que Unity compila sin errores en la consola.

- [ ] Commit:

```bash
git add Assets/
git commit -m "chore: add Unity folder structure and assembly definitions"
```

---

### Task 3: GitHub Actions CI/CD Pipeline

**Files:**
- Create: `merge-tower-unity/.github/workflows/build.yml`

- [ ] Crear `merge-tower-unity/.github/workflows/build.yml`:

```yaml
name: Build MergeTower

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    name: Run EditMode Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          lfs: true

      - uses: actions/cache@v4
        with:
          path: Library
          key: Library-${{ hashFiles('Assets/**', 'Packages/**', 'ProjectSettings/**') }}
          restore-keys: Library-

      - uses: game-ci/unity-test-runner@v4
        env:
          UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}
          UNITY_EMAIL: ${{ secrets.UNITY_EMAIL }}
          UNITY_PASSWORD: ${{ secrets.UNITY_PASSWORD }}
        with:
          testMode: editmode
          artifactsPath: test-results

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: test-results

  build-android:
    name: Build Android AAB
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
        with:
          lfs: true

      - uses: actions/cache@v4
        with:
          path: Library
          key: Library-Android-${{ hashFiles('Assets/**', 'Packages/**') }}
          restore-keys: Library-Android-

      - uses: game-ci/unity-builder@v4
        env:
          UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}
          UNITY_EMAIL: ${{ secrets.UNITY_EMAIL }}
          UNITY_PASSWORD: ${{ secrets.UNITY_PASSWORD }}
        with:
          targetPlatform: Android
          androidExportType: androidAppBundle
          androidKeystoreName: merge-tower.keystore
          androidKeystoreBase64: ${{ secrets.ANDROID_KEYSTORE_BASE64 }}
          androidKeystorePass: ${{ secrets.ANDROID_KEYSTORE_PASS }}
          androidKeyaliasName: ${{ secrets.ANDROID_KEY_ALIAS }}
          androidKeyaliasPass: ${{ secrets.ANDROID_KEY_PASS }}

      - uses: actions/upload-artifact@v4
        with:
          name: build-android
          path: build/Android

  build-ios:
    name: Build iOS Xcode Project
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
        with:
          lfs: true

      - uses: actions/cache@v4
        with:
          path: Library
          key: Library-iOS-${{ hashFiles('Assets/**', 'Packages/**') }}
          restore-keys: Library-iOS-

      - uses: game-ci/unity-builder@v4
        env:
          UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}
          UNITY_EMAIL: ${{ secrets.UNITY_EMAIL }}
          UNITY_PASSWORD: ${{ secrets.UNITY_PASSWORD }}
        with:
          targetPlatform: iOS

      - uses: actions/upload-artifact@v4
        with:
          name: build-ios
          path: build/iOS
```

- [ ] Agregar los siguientes secrets en GitHub repo settings (`Settings > Secrets and variables > Actions`):
  - `UNITY_LICENSE` (contenido del archivo `.ulf` de activación de Unity)
  - `UNITY_EMAIL`
  - `UNITY_PASSWORD`
  - `ANDROID_KEYSTORE_BASE64` (keystore en base64: `base64 -i merge-tower.keystore`)
  - `ANDROID_KEYSTORE_PASS`
  - `ANDROID_KEY_ALIAS`
  - `ANDROID_KEY_PASS`

- [ ] Commit y push a `main`, verificar que el pipeline se dispara en GitHub Actions sin errores de configuración.

- [ ] Commit:

```bash
git add .github/
git commit -m "ci: add GitHub Actions pipeline for test, Android AAB and iOS builds"
```

---

## Fase 2 — Foundation del Motor

### Task 4: Interfaces base e ITickable

**Files:**
- Create: `merge-tower-unity/Assets/_Project/Scripts/Core/ITickable.cs`
- Create: `merge-tower-unity/Assets/Tests/EditMode/TickableTests.cs`

- [ ] Crear `Assets/Tests/EditMode/TickableTests.cs`:

```csharp
using NUnit.Framework;
using MergeTower;

namespace MergeTower.Tests
{
    public class TickableTests
    {
        private class FakeTickable : ITickable
        {
            public float LastDelta;
            public int TickCount;
            public void Tick(float deltaTime) { LastDelta = deltaTime; TickCount++; }
        }

        [Test]
        public void ITickable_Tick_ReceivesDeltaTime()
        {
            var t = new FakeTickable();
            t.Tick(0.016f);
            Assert.AreEqual(0.016f, t.LastDelta, 0.0001f);
            Assert.AreEqual(1, t.TickCount);
        }
    }
}
```

- [ ] Ejecutar el test en Unity Test Runner (`Window > General > Test Runner > EditMode`). Verificar que falla con `ITickable not found`.

- [ ] Crear `Assets/_Project/Scripts/Core/ITickable.cs`:

```csharp
namespace MergeTower
{
    public interface ITickable
    {
        void Tick(float deltaTime);
    }

    public interface IFixedTickable
    {
        void FixedTick(float fixedDeltaTime);
    }
}
```

- [ ] Ejecutar el test. Verificar que pasa.

- [ ] Commit:

```bash
git add Assets/_Project/Scripts/Core/ITickable.cs Assets/Tests/EditMode/TickableTests.cs
git commit -m "feat: add ITickable and IFixedTickable interfaces"
```

---

### Task 5: CustomUpdateManager

**Files:**
- Create: `merge-tower-unity/Assets/_Project/Scripts/Core/CustomUpdateManager.cs`
- Modify: `merge-tower-unity/Assets/Tests/EditMode/TickableTests.cs`

- [ ] Agregar tests al final de `Assets/Tests/EditMode/TickableTests.cs`:

```csharp
public class CustomUpdateManagerTests
{
    private class FakeTickable : ITickable
    {
        public float LastDelta;
        public int TickCount;
        public void Tick(float deltaTime) { LastDelta = deltaTime; TickCount++; }
    }

    [Test]
    public void Register_ThenTick_CallsTickable()
    {
        var manager = new CustomUpdateManager();
        var fake = new FakeTickable();
        manager.Register(fake);
        manager.Tick(0.016f);
        Assert.AreEqual(1, fake.TickCount);
    }

    [Test]
    public void Unregister_ThenTick_DoesNotCallTickable()
    {
        var manager = new CustomUpdateManager();
        var fake = new FakeTickable();
        manager.Register(fake);
        manager.Unregister(fake);
        manager.Tick(0.016f);
        Assert.AreEqual(0, fake.TickCount);
    }

    [Test]
    public void Tick_MultipleTickables_AllReceiveDelta()
    {
        var manager = new CustomUpdateManager();
        var a = new FakeTickable();
        var b = new FakeTickable();
        manager.Register(a);
        manager.Register(b);
        manager.Tick(0.033f);
        Assert.AreEqual(0.033f, a.LastDelta, 0.0001f);
        Assert.AreEqual(0.033f, b.LastDelta, 0.0001f);
    }
}
```

- [ ] Ejecutar tests. Verificar que los 3 nuevos fallan con `CustomUpdateManager not found`.

- [ ] Crear `Assets/_Project/Scripts/Core/CustomUpdateManager.cs`:

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.LowLevel;
using UnityEngine.PlayerLoop;

namespace MergeTower
{
    public sealed class CustomUpdateManager
    {
        private readonly List<ITickable> _tickables = new List<ITickable>(64);

        public void Register(ITickable tickable)
        {
            if (!_tickables.Contains(tickable))
                _tickables.Add(tickable);
        }

        public void Unregister(ITickable tickable) =>
            _tickables.Remove(tickable);

        public void Tick(float deltaTime)
        {
            for (int i = 0; i < _tickables.Count; i++)
                _tickables[i].Tick(deltaTime);
        }

        public void InjectIntoPlayerLoop()
        {
            var loop = PlayerLoop.GetCurrentPlayerLoop();
            for (int i = 0; i < loop.subSystemList.Length; i++)
            {
                if (loop.subSystemList[i].type == typeof(Update))
                {
                    var update = loop.subSystemList[i];
                    var list = new List<PlayerLoopSystem>(
                        update.subSystemList ?? new PlayerLoopSystem[0]);
                    list.Add(new PlayerLoopSystem
                    {
                        type = typeof(CustomUpdateManager),
                        updateDelegate = () => Tick(Time.deltaTime)
                    });
                    update.subSystemList = list.ToArray();
                    loop.subSystemList[i] = update;
                    break;
                }
            }
            PlayerLoop.SetPlayerLoop(loop);
        }
    }
}
```

- [ ] Ejecutar los 3 tests. Verificar que pasan.

- [ ] Commit:

```bash
git add Assets/_Project/Scripts/Core/CustomUpdateManager.cs Assets/Tests/EditMode/TickableTests.cs
git commit -m "feat: add CustomUpdateManager with PlayerLoopSystem injection"
```

---

### Task 6: GameConfig y VContainer LifetimeScopes

**Files:**
- Create: `merge-tower-unity/Assets/_Project/Scripts/Data/GameConfig.cs`
- Create: `merge-tower-unity/Assets/_Project/Scripts/DI/ProjectLifetimeScope.cs`
- Create: `merge-tower-unity/Assets/_Project/Scripts/DI/GameLifetimeScope.cs`

- [ ] Crear `Assets/_Project/Scripts/Data/GameConfig.cs`:

```csharp
using UnityEngine;

namespace MergeTower
{
    [CreateAssetMenu(fileName = "GameConfig", menuName = "MergeTower/GameConfig")]
    public sealed class GameConfig : ScriptableObject
    {
        [Header("Economy")]
        public long UpgradeCostBase = 100;
        public float CostMultiplier = 1.8f;
        public float CoinsPerSecondMultiplier = 1f;

        [Header("Spawn")]
        public float SpawnIntervalSeconds = 3f;
        [Range(0f, 1f)]
        public float RareSpawnChance = 0.05f;

        [Header("Daily Goal")]
        public int DailyGoalTarget = 20;

        [Header("Grid")]
        public int GridWidth = 4;
        public int GridHeight = 4;
        public int MaxElementLevel = 10;
    }
}
```

- [ ] En Unity Editor, crear el asset: click derecho en `Assets/_Project/ScriptableObjects/` > Create > MergeTower > GameConfig. Nombrar `GameConfig`.

- [ ] Crear `Assets/_Project/Scripts/DI/ProjectLifetimeScope.cs`:

```csharp
using VContainer;
using VContainer.Unity;

namespace MergeTower.DI
{
    public sealed class ProjectLifetimeScope : LifetimeScope
    {
        protected override void Configure(IContainerBuilder builder)
        {
            builder.Register<CustomUpdateManager>(Lifetime.Singleton);
        }
    }
}
```

- [ ] Crear `Assets/_Project/Scripts/DI/GameLifetimeScope.cs`:

```csharp
using UnityEngine;
using VContainer;
using VContainer.Unity;

namespace MergeTower.DI
{
    public sealed class GameLifetimeScope : LifetimeScope
    {
        [SerializeField] private GameConfig _gameConfig;

        protected override void Configure(IContainerBuilder builder)
        {
            builder.RegisterInstance(_gameConfig);
            // Los sistemas se registrarán en tareas posteriores conforme se crean
        }
    }
}
```

- [ ] En Unity Editor, crear un GameObject vacío en la escena principal llamado `[ProjectScope]`, agregarle el componente `ProjectLifetimeScope`. Crear otro llamado `[GameScope]`, agregarle `GameLifetimeScope` y asignar el asset `GameConfig` en el inspector.

- [ ] Verificar que Unity compila sin errores.

- [ ] Commit:

```bash
git add Assets/_Project/Scripts/Data/GameConfig.cs Assets/_Project/Scripts/DI/
git commit -m "feat: add VContainer LifetimeScopes and GameConfig ScriptableObject"
```

---

### Task 7: GameDataTypes y Event Channels

**Files:**
- Create: `merge-tower-unity/Assets/_Project/Scripts/Data/GameDataTypes.cs`
- Create: `merge-tower-unity/Assets/_Project/Scripts/Events/GameEventChannel.cs`
- Create: `merge-tower-unity/Assets/_Project/Scripts/Events/CoinsChangedChannel.cs`
- Create: `merge-tower-unity/Assets/_Project/Scripts/Events/MergeCompletedChannel.cs`
- Create: `merge-tower-unity/Assets/_Project/Scripts/Events/RareElementChannel.cs`
- Create: `merge-tower-unity/Assets/_Project/Scripts/Events/DailyGoalChannel.cs`
- Create: `merge-tower-unity/Assets/_Project/Scripts/Events/AdRewardChannel.cs`
- Create: `merge-tower-unity/Assets/_Project/Scripts/Events/PurchaseChannel.cs`
- Create: `merge-tower-unity/Assets/Tests/EditMode/EventChannelTests.cs`

- [ ] Crear `Assets/Tests/EditMode/EventChannelTests.cs`:

```csharp
using NUnit.Framework;
using UnityEngine;
using MergeTower;

namespace MergeTower.Tests
{
    public class EventChannelTests
    {
        [Test]
        public void CoinsChangedChannel_Raise_NotifiesSubscriber()
        {
            var channel = ScriptableObject.CreateInstance<CoinsChangedChannel>();
            long received = -1;
            channel.OnEventRaised += val => received = val;
            channel.Raise(500L);
            Assert.AreEqual(500L, received);
        }

        [Test]
        public void CoinsChangedChannel_Raise_NoSubscribers_DoesNotThrow()
        {
            var channel = ScriptableObject.CreateInstance<CoinsChangedChannel>();
            Assert.DoesNotThrow(() => channel.Raise(100L));
        }
    }
}
```

- [ ] Ejecutar tests. Verificar que fallan con `CoinsChangedChannel not found`.

- [ ] Crear `Assets/_Project/Scripts/Data/GameDataTypes.cs`:

```csharp
namespace MergeTower
{
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
        public MergeResult(int resultLevel, long bonusCoins)
        { ResultLevel = resultLevel; BonusCoins = bonusCoins; }
    }

    public readonly struct GoalProgress
    {
        public readonly int Current;
        public readonly int Target;
        public GoalProgress(int current, int target) { Current = current; Target = target; }
    }

    public readonly struct AdRevenuePayload
    {
        public readonly double eCPM;
        public readonly string NetworkName;
        public readonly string AdUnitId;
        public AdRevenuePayload(double ecpm, string networkName, string adUnitId)
        { eCPM = ecpm; NetworkName = networkName; AdUnitId = adUnitId; }
    }

    public enum RewardType { ExtraElement, DoubleIncome2h }
}
```

- [ ] Crear `Assets/_Project/Scripts/Events/GameEventChannel.cs`:

```csharp
using System;
using UnityEngine;

namespace MergeTower
{
    public abstract class GameEventChannel<T> : ScriptableObject
    {
        public event Action<T> OnEventRaised;
        public void Raise(T value) => OnEventRaised?.Invoke(value);
    }
}
```

- [ ] Crear los 6 canales concretos:

`Assets/_Project/Scripts/Events/CoinsChangedChannel.cs`:
```csharp
using UnityEngine;
namespace MergeTower
{
    [CreateAssetMenu(menuName = "MergeTower/Events/CoinsChanged")]
    public sealed class CoinsChangedChannel : GameEventChannel<long> { }
}
```

`Assets/_Project/Scripts/Events/MergeCompletedChannel.cs`:
```csharp
using UnityEngine;
namespace MergeTower
{
    [CreateAssetMenu(menuName = "MergeTower/Events/MergeCompleted")]
    public sealed class MergeCompletedChannel : GameEventChannel<MergeResult> { }
}
```

`Assets/_Project/Scripts/Events/RareElementChannel.cs`:
```csharp
using UnityEngine;
namespace MergeTower
{
    [CreateAssetMenu(menuName = "MergeTower/Events/RareElement")]
    public sealed class RareElementChannel : GameEventChannel<ElementData> { }
}
```

`Assets/_Project/Scripts/Events/DailyGoalChannel.cs`:
```csharp
using UnityEngine;
namespace MergeTower
{
    [CreateAssetMenu(menuName = "MergeTower/Events/DailyGoal")]
    public sealed class DailyGoalChannel : GameEventChannel<GoalProgress> { }
}
```

`Assets/_Project/Scripts/Events/AdRewardChannel.cs`:
```csharp
using UnityEngine;
namespace MergeTower
{
    [CreateAssetMenu(menuName = "MergeTower/Events/AdReward")]
    public sealed class AdRewardChannel : GameEventChannel<RewardType> { }
}
```

`Assets/_Project/Scripts/Events/PurchaseChannel.cs`:
```csharp
using UnityEngine;
namespace MergeTower
{
    [CreateAssetMenu(menuName = "MergeTower/Events/Purchase")]
    public sealed class PurchaseChannel : GameEventChannel<string> { }
}
```

- [ ] En Unity Editor, crear los 6 assets de canales en `Assets/_Project/ScriptableObjects/Events/` via click derecho > Create > MergeTower > Events > [nombre].

- [ ] Ejecutar los 2 tests. Verificar que pasan.

- [ ] Commit:

```bash
git add Assets/_Project/Scripts/Data/GameDataTypes.cs Assets/_Project/Scripts/Events/ Assets/Tests/EditMode/EventChannelTests.cs
git commit -m "feat: add typed Event Channels and GameDataTypes structs"
```

---

### Task 8: Object Pool para ElementView

**Files:**
- Create: `merge-tower-unity/Assets/_Project/Scripts/UI/ElementView.cs`
- Create: `merge-tower-unity/Assets/_Project/Scripts/Pool/ElementPool.cs`

- [ ] Crear `Assets/_Project/Scripts/UI/ElementView.cs`:

```csharp
using UnityEngine;

namespace MergeTower
{
    public sealed class ElementView : MonoBehaviour
    {
        public int CellIndex { get; private set; }
        public ElementData Data { get; private set; }

        public void Initialize(ElementData data, int cellIndex)
        {
            Data = data;
            CellIndex = cellIndex;
            gameObject.SetActive(true);
        }

        public void ReturnToPool(ElementPool pool)
        {
            gameObject.SetActive(false);
            pool.Release(this);
        }
    }
}
```

- [ ] Crear `Assets/_Project/Scripts/Pool/ElementPool.cs`:

```csharp
using UnityEngine;
using UnityEngine.Pool;

namespace MergeTower
{
    public sealed class ElementPool
    {
        private readonly IObjectPool<ElementView> _pool;

        public ElementPool(ElementView prefab, Transform parent,
            int defaultCapacity = 20, int maxSize = 50)
        {
            _pool = new ObjectPool<ElementView>(
                createFunc: () => Object.Instantiate(prefab, parent),
                actionOnGet: view => view.gameObject.SetActive(true),
                actionOnRelease: view => view.gameObject.SetActive(false),
                actionOnDestroy: view => Object.Destroy(view.gameObject),
                collectionCheck: false,
                defaultCapacity: defaultCapacity,
                maxSize: maxSize
            );
        }

        public ElementView Get() => _pool.Get();
        public void Release(ElementView view) => _pool.Release(view);
        public int CountActive => _pool.CountActive;
        public int CountInactive => _pool.CountInactive;
    }
}
```

- [ ] En Unity Editor, crear un prefab `ElementView` en `Assets/_Project/Prefabs/`: nuevo GameObject > agregar componente `ElementView` > agregar hijo con componente `Image` para el visual > arrastrar a carpeta Prefabs para guardarlo.

- [ ] Verificar que Unity compila sin errores.

- [ ] Commit:

```bash
git add Assets/_Project/Scripts/UI/ElementView.cs Assets/_Project/Scripts/Pool/ElementPool.cs
git commit -m "feat: add ElementPool using UnityEngine.Pool and ElementView component"
```

---

## Fase 3 — Lógica de Negocio

### Task 9: PlayerSaveData

**Files:**
- Create: `merge-tower-unity/Assets/_Project/Scripts/Data/PlayerSaveData.cs`
- Create: `merge-tower-unity/Assets/Tests/EditMode/SaveDataTests.cs`

- [ ] Crear `Assets/Tests/EditMode/SaveDataTests.cs`:

```csharp
using NUnit.Framework;
using UnityEngine;
using MergeTower;

namespace MergeTower.Tests
{
    public class SaveDataTests
    {
        [Test]
        public void PlayerSaveData_DefaultValues_AreValid()
        {
            var data = new PlayerSaveData();
            Assert.AreEqual(0L, data.Coins);
            Assert.AreEqual(16, data.GridState.Length);
            Assert.IsFalse(data.AdsRemoved);
        }

        [Test]
        public void PlayerSaveData_JsonRoundTrip_PreservesCoins()
        {
            var data = new PlayerSaveData { Coins = 123456789L };
            string json = JsonUtility.ToJson(data);
            var loaded = JsonUtility.FromJson<PlayerSaveData>(json);
            Assert.AreEqual(123456789L, loaded.Coins);
        }

        [Test]
        public void PlayerSaveData_JsonRoundTrip_PreservesGridState()
        {
            var data = new PlayerSaveData();
            data.GridState[3] = 5;
            string json = JsonUtility.ToJson(data);
            var loaded = JsonUtility.FromJson<PlayerSaveData>(json);
            Assert.AreEqual(5, loaded.GridState[3]);
        }
    }
}
```

- [ ] Ejecutar tests. Verificar que fallan con `PlayerSaveData not found`.

- [ ] Crear `Assets/_Project/Scripts/Data/PlayerSaveData.cs`:

```csharp
using System;
using System.Collections.Generic;

namespace MergeTower
{
    [Serializable]
    public sealed class PlayerSaveData
    {
        public long Coins = 0L;
        public int[] GridState = new int[16];
        public int TowerLevel = 0;
        public long CoinsPerSecond = 0L;
        public int DailyGoalProgress = 0;
        public string LastGoalResetDateUtc = "";
        public bool AdsRemoved = false;
        public List<string> PurchasedProductIds = new List<string>();
    }
}
```

- [ ] Ejecutar los 3 tests. Verificar que pasan.

- [ ] Commit:

```bash
git add Assets/_Project/Scripts/Data/PlayerSaveData.cs Assets/Tests/EditMode/SaveDataTests.cs
git commit -m "feat: add PlayerSaveData with JSON serialization"
```

---

### Task 10: SaveSystem

**Files:**
- Create: `merge-tower-unity/Assets/_Project/Scripts/Services/SaveSystem.cs`
- Create: `merge-tower-unity/Assets/Tests/EditMode/SaveSystemTests.cs`

- [ ] Crear `Assets/Tests/EditMode/SaveSystemTests.cs`:

```csharp
using NUnit.Framework;
using System.IO;
using MergeTower;

namespace MergeTower.Tests
{
    public class SaveSystemTests
    {
        private string _testDir;
        private SaveSystem _saveSystem;

        [SetUp]
        public void SetUp()
        {
            _testDir = Path.Combine(Path.GetTempPath(), "merge-tower-tests");
            Directory.CreateDirectory(_testDir);
            _saveSystem = new SaveSystem(_testDir);
        }

        [TearDown]
        public void TearDown()
        {
            if (Directory.Exists(_testDir))
                Directory.Delete(_testDir, recursive: true);
        }

        [Test]
        public void Load_NoFile_ReturnsDefaultData()
        {
            var data = _saveSystem.Load();
            Assert.AreEqual(0L, data.Coins);
        }

        [Test]
        public void Save_ThenLoad_PreservesCoins()
        {
            var data = new PlayerSaveData { Coins = 9999L };
            _saveSystem.Save(data);
            var loaded = _saveSystem.Load();
            Assert.AreEqual(9999L, loaded.Coins);
        }

        [Test]
        public void Save_CreatesBackupFile()
        {
            var data = new PlayerSaveData { Coins = 1L };
            _saveSystem.Save(data);
            _saveSystem.Save(data);
            Assert.IsTrue(File.Exists(Path.Combine(_testDir, "save.backup.json")));
        }

        [Test]
        public void Load_CorruptFile_LoadsFromBackup()
        {
            var data = new PlayerSaveData { Coins = 777L };
            _saveSystem.Save(data);
            _saveSystem.Save(data);
            File.WriteAllText(Path.Combine(_testDir, "save.json"), "CORRUPTED{{");
            var loaded = _saveSystem.Load();
            Assert.AreEqual(777L, loaded.Coins);
        }
    }
}
```

- [ ] Ejecutar tests. Verificar que fallan con `SaveSystem not found`.

- [ ] Crear `Assets/_Project/Scripts/Services/SaveSystem.cs`:

```csharp
using System.IO;
using UnityEngine;

namespace MergeTower
{
    public sealed class SaveSystem
    {
        private readonly string _savePath;
        private readonly string _backupPath;

        public SaveSystem() : this(Application.persistentDataPath) { }

        public SaveSystem(string directory)
        {
            _savePath = Path.Combine(directory, "save.json");
            _backupPath = Path.Combine(directory, "save.backup.json");
        }

        public void Save(PlayerSaveData data)
        {
            if (File.Exists(_savePath))
                File.Copy(_savePath, _backupPath, overwrite: true);

            string json = JsonUtility.ToJson(data, prettyPrint: false);
            File.WriteAllText(_savePath, json);
        }

        public PlayerSaveData Load()
        {
            if (!File.Exists(_savePath))
                return new PlayerSaveData();

            try
            {
                string json = File.ReadAllText(_savePath);
                return JsonUtility.FromJson<PlayerSaveData>(json);
            }
            catch
            {
                if (!File.Exists(_backupPath))
                    return new PlayerSaveData();

                string backupJson = File.ReadAllText(_backupPath);
                return JsonUtility.FromJson<PlayerSaveData>(backupJson);
            }
        }
    }
}
```

- [ ] Ejecutar los 4 tests. Verificar que pasan.

- [ ] Registrar en `GameLifetimeScope.cs` dentro del método `Configure`:

```csharp
builder.Register<SaveSystem>(Lifetime.Singleton);
```

- [ ] Commit:

```bash
git add Assets/_Project/Scripts/Services/SaveSystem.cs Assets/Tests/EditMode/SaveSystemTests.cs Assets/_Project/Scripts/DI/GameLifetimeScope.cs
git commit -m "feat: add SaveSystem with backup and corruption recovery"
```

---

### Task 11: EconomySystem

**Files:**
- Create: `merge-tower-unity/Assets/_Project/Scripts/Systems/EconomySystem.cs`
- Create: `merge-tower-unity/Assets/Tests/EditMode/EconomySystemTests.cs`

- [ ] Crear `Assets/Tests/EditMode/EconomySystemTests.cs`:

```csharp
using NUnit.Framework;
using UnityEngine;
using MergeTower;

namespace MergeTower.Tests
{
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
            _economy = new EconomySystem(null, _config);
        }

        [Test]
        public void GetUpgradeCost_Level1_ReturnsBase()
            => Assert.AreEqual(100L, _economy.GetUpgradeCost(1));

        [Test]
        public void GetUpgradeCost_Level2_ReturnsBaseTimesMultiplier()
            => Assert.AreEqual(180L, _economy.GetUpgradeCost(2));

        [Test]
        public void GetUpgradeCost_Level5_FollowsExponentialCurve()
        {
            long expected = (long)(100 * System.Math.Pow(1.8, 4));
            Assert.AreEqual(expected, _economy.GetUpgradeCost(5));
        }

        [Test]
        public void AddCoins_IncreasesBalance()
        {
            _economy.AddCoins(500L);
            Assert.AreEqual(500L, _economy.GetBalance());
        }

        [Test]
        public void SpendCoins_SufficientBalance_ReturnsTrue()
        {
            _economy.AddCoins(200L);
            Assert.IsTrue(_economy.SpendCoins(100L));
            Assert.AreEqual(100L, _economy.GetBalance());
        }

        [Test]
        public void SpendCoins_InsufficientBalance_ReturnsFalse()
        {
            _economy.AddCoins(50L);
            Assert.IsFalse(_economy.SpendCoins(100L));
            Assert.AreEqual(50L, _economy.GetBalance());
        }

        [Test]
        public void AddCoins_NearLongMax_DoesNotOverflow()
        {
            _economy.AddCoins(long.MaxValue - 1);
            Assert.AreEqual(long.MaxValue - 1, _economy.GetBalance());
        }

        [Test]
        public void Tick_AccumulatesCoinsPerSecond_AfterOneSecond()
        {
            _economy.SetCoinsPerSecond(10L);
            _economy.Tick(1.0f);
            Assert.AreEqual(10L, _economy.GetBalance());
        }

        [Test]
        public void Tick_SubSecondDeltas_DoNotAccumulateUntilOneSecond()
        {
            _economy.SetCoinsPerSecond(10L);
            _economy.Tick(0.5f);
            _economy.Tick(0.4f);
            Assert.AreEqual(0L, _economy.GetBalance());
            _economy.Tick(0.1f);
            Assert.AreEqual(10L, _economy.GetBalance());
        }
    }
}
```

- [ ] Ejecutar tests. Verificar que fallan con `EconomySystem not found`.

- [ ] Crear `Assets/_Project/Scripts/Systems/EconomySystem.cs`:

```csharp
using System;

namespace MergeTower
{
    public sealed class EconomySystem : ITickable
    {
        private long _coins;
        private long _coinsPerSecond;
        private float _accumulatedTime;
        private readonly CoinsChangedChannel _coinsChangedChannel;
        private readonly GameConfig _config;

        public EconomySystem(CoinsChangedChannel coinsChangedChannel, GameConfig config)
        {
            _coinsChangedChannel = coinsChangedChannel;
            _config = config;
        }

        public void Tick(float deltaTime)
        {
            _accumulatedTime += deltaTime;
            if (_accumulatedTime < 1f) return;
            _accumulatedTime -= 1f;
            AddCoins(_coinsPerSecond);
        }

        public void AddCoins(long amount)
        {
            _coins += amount;
            _coinsChangedChannel?.Raise(_coins);
        }

        public bool SpendCoins(long amount)
        {
            if (_coins < amount) return false;
            _coins -= amount;
            _coinsChangedChannel?.Raise(_coins);
            return true;
        }

        public long GetUpgradeCost(int currentLevel) =>
            (long)(_config.UpgradeCostBase * Math.Pow(_config.CostMultiplier, currentLevel - 1));

        public long GetBalance() => _coins;
        public void SetCoinsPerSecond(long cps) => _coinsPerSecond = cps;
        public void LoadFromSave(long coins) => _coins = coins;
    }
}
```

- [ ] Ejecutar los 9 tests. Verificar que pasan.

- [ ] Agregar en `GameLifetimeScope.Configure`:

```csharp
builder.Register<EconomySystem>(Lifetime.Singleton);
builder.RegisterEntryPoint<EconomySystem>();
```

- [ ] Commit:

```bash
git add Assets/_Project/Scripts/Systems/EconomySystem.cs Assets/Tests/EditMode/EconomySystemTests.cs Assets/_Project/Scripts/DI/GameLifetimeScope.cs
git commit -m "feat: add EconomySystem with exponential upgrade costs and idle accumulation"
```

---

### Task 12: TowerProgressionSystem

**Files:**
- Create: `merge-tower-unity/Assets/_Project/Scripts/Systems/TowerProgressionSystem.cs`
- Create: `merge-tower-unity/Assets/Tests/EditMode/TowerProgressionSystemTests.cs`

- [ ] Crear `Assets/Tests/EditMode/TowerProgressionSystemTests.cs`:

```csharp
using NUnit.Framework;
using UnityEngine;
using MergeTower;

namespace MergeTower.Tests
{
    public class TowerProgressionSystemTests
    {
        private TowerProgressionSystem _tower;
        private EconomySystem _economy;
        private GameConfig _config;

        [SetUp]
        public void SetUp()
        {
            _config = ScriptableObject.CreateInstance<GameConfig>();
            _config.CoinsPerSecondMultiplier = 1f;
            _config.UpgradeCostBase = 100;
            _config.CostMultiplier = 1.8f;
            _economy = new EconomySystem(null, _config);
            _tower = new TowerProgressionSystem(_economy, _config);
        }

        [Test]
        public void AddMergedElement_IncreasesTowerLevel()
        {
            _tower.AddMergedElement(3);
            Assert.AreEqual(3, _tower.GetTowerLevel());
        }

        [Test]
        public void AddMergedElement_UpdatesCoinsPerSecond()
        {
            _tower.AddMergedElement(5);
            Assert.AreEqual(5L, _tower.GetCoinsPerSecond());
        }

        [Test]
        public void AddMergedElement_Multiple_AccumulatesLevel()
        {
            _tower.AddMergedElement(2);
            _tower.AddMergedElement(3);
            Assert.AreEqual(5, _tower.GetTowerLevel());
            Assert.AreEqual(5L, _tower.GetCoinsPerSecond());
        }
    }
}
```

- [ ] Ejecutar tests. Verificar que fallan.

- [ ] Crear `Assets/_Project/Scripts/Systems/TowerProgressionSystem.cs`:

```csharp
namespace MergeTower
{
    public sealed class TowerProgressionSystem
    {
        private int _totalTowerLevel;
        private long _coinsPerSecond;
        private readonly EconomySystem _economy;
        private readonly GameConfig _config;

        public TowerProgressionSystem(EconomySystem economy, GameConfig config)
        {
            _economy = economy;
            _config = config;
        }

        public void AddMergedElement(int elementLevel)
        {
            _totalTowerLevel += elementLevel;
            RecalculateCoinsPerSecond();
        }

        private void RecalculateCoinsPerSecond()
        {
            _coinsPerSecond = (long)(_totalTowerLevel * _config.CoinsPerSecondMultiplier);
            _economy.SetCoinsPerSecond(_coinsPerSecond);
        }

        public long GetCoinsPerSecond() => _coinsPerSecond;
        public int GetTowerLevel() => _totalTowerLevel;

        public void LoadFromSave(int towerLevel)
        {
            _totalTowerLevel = towerLevel;
            RecalculateCoinsPerSecond();
        }
    }
}
```

- [ ] Ejecutar los 3 tests. Verificar que pasan.

- [ ] Agregar en `GameLifetimeScope.Configure`:

```csharp
builder.Register<TowerProgressionSystem>(Lifetime.Singleton);
```

- [ ] Commit:

```bash
git add Assets/_Project/Scripts/Systems/TowerProgressionSystem.cs Assets/Tests/EditMode/TowerProgressionSystemTests.cs Assets/_Project/Scripts/DI/GameLifetimeScope.cs
git commit -m "feat: add TowerProgressionSystem with coins/sec recalculation"
```

---

### Task 13: MergeGridSystem

**Files:**
- Create: `merge-tower-unity/Assets/_Project/Scripts/Systems/MergeGridSystem.cs`
- Create: `merge-tower-unity/Assets/Tests/EditMode/MergeGridSystemTests.cs`

- [ ] Crear `Assets/Tests/EditMode/MergeGridSystemTests.cs`:

```csharp
using NUnit.Framework;
using UnityEngine;
using MergeTower;

namespace MergeTower.Tests
{
    public class MergeGridSystemTests
    {
        private MergeGridSystem _grid;
        private EconomySystem _economy;
        private TowerProgressionSystem _tower;
        private GameConfig _config;

        [SetUp]
        public void SetUp()
        {
            _config = ScriptableObject.CreateInstance<GameConfig>();
            _config.UpgradeCostBase = 100;
            _config.CostMultiplier = 1.8f;
            _config.CoinsPerSecondMultiplier = 1f;
            _economy = new EconomySystem(null, _config);
            _tower = new TowerProgressionSystem(_economy, _config);
            _grid = new MergeGridSystem(null, _tower, _economy);
        }

        [Test]
        public void ValidateMerge_SameLevel_ReturnsTrue()
        {
            _grid.PlaceElement(0, new ElementData(2, false));
            _grid.PlaceElement(1, new ElementData(2, false));
            Assert.IsTrue(_grid.ValidateMerge(0, 1));
        }

        [Test]
        public void ValidateMerge_DifferentLevel_ReturnsFalse()
        {
            _grid.PlaceElement(0, new ElementData(1, false));
            _grid.PlaceElement(1, new ElementData(2, false));
            Assert.IsFalse(_grid.ValidateMerge(0, 1));
        }

        [Test]
        public void ValidateMerge_EmptyCell_ReturnsFalse()
        {
            _grid.PlaceElement(0, new ElementData(1, false));
            Assert.IsFalse(_grid.ValidateMerge(0, 1));
        }

        [Test]
        public void ValidateMerge_SameCell_ReturnsFalse()
        {
            _grid.PlaceElement(0, new ElementData(1, false));
            Assert.IsFalse(_grid.ValidateMerge(0, 0));
        }

        [Test]
        public void ExecuteMerge_ProducesHigherLevel()
        {
            _grid.PlaceElement(0, new ElementData(2, false));
            _grid.PlaceElement(1, new ElementData(2, false));
            var result = _grid.ExecuteMerge(0, 1);
            Assert.AreEqual(3, result.ResultLevel);
        }

        [Test]
        public void ExecuteMerge_ClearsSourceCell()
        {
            _grid.PlaceElement(0, new ElementData(2, false));
            _grid.PlaceElement(1, new ElementData(2, false));
            _grid.ExecuteMerge(0, 1);
            Assert.IsFalse(_grid.ValidateMerge(0, 1));
        }

        [Test]
        public void ExecuteMerge_RareElement_AppliesBonus()
        {
            _grid.PlaceElement(0, new ElementData(2, true));
            _grid.PlaceElement(1, new ElementData(2, false));
            var result = _grid.ExecuteMerge(0, 1);
            Assert.AreEqual(2L * 3L, result.BonusCoins);
        }

        [Test]
        public void GetMergeResult_ReturnsLevelPlusOne()
            => Assert.AreEqual(4, _grid.GetMergeResult(3));

        [Test]
        public void HasEmptyCell_EmptyGrid_ReturnsTrue()
        {
            Assert.IsTrue(_grid.HasEmptyCell(out int idx));
            Assert.AreEqual(0, idx);
        }

        [Test]
        public void HasEmptyCell_FullGrid_ReturnsFalse()
        {
            for (int i = 0; i < 16; i++)
                _grid.PlaceElement(i, new ElementData(1, false));
            Assert.IsFalse(_grid.HasEmptyCell(out _));
        }
    }
}
```

- [ ] Ejecutar tests. Verificar que fallan.

- [ ] Crear `Assets/_Project/Scripts/Systems/MergeGridSystem.cs`:

```csharp
namespace MergeTower
{
    public sealed class MergeGridSystem
    {
        private readonly ElementData[] _grid = new ElementData[16];
        private readonly MergeCompletedChannel _mergeChannel;
        private readonly TowerProgressionSystem _tower;
        private readonly EconomySystem _economy;

        public MergeGridSystem(
            MergeCompletedChannel mergeChannel,
            TowerProgressionSystem tower,
            EconomySystem economy)
        {
            _mergeChannel = mergeChannel;
            _tower = tower;
            _economy = economy;
        }

        public void PlaceElement(int cellIndex, ElementData data) =>
            _grid[cellIndex] = data;

        public bool ValidateMerge(int cellA, int cellB)
        {
            if (cellA == cellB) return false;
            if (_grid[cellA].Level == 0 || _grid[cellB].Level == 0) return false;
            return _grid[cellA].Level == _grid[cellB].Level;
        }

        public MergeResult ExecuteMerge(int cellA, int cellB)
        {
            bool anyRare = _grid[cellA].IsRare || _grid[cellB].IsRare;
            long bonus = anyRare ? _grid[cellA].Level * 3L : _grid[cellA].Level * 1L;
            var result = new MergeResult(_grid[cellA].Level + 1, bonus);

            _grid[cellB] = new ElementData(result.ResultLevel, false);
            _grid[cellA] = default;

            _economy.AddCoins(result.BonusCoins);
            _tower.AddMergedElement(result.ResultLevel);
            _mergeChannel?.Raise(result);

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

        public ElementData GetCell(int index) => _grid[index];

        public void LoadFromSave(int[] gridState)
        {
            for (int i = 0; i < gridState.Length && i < _grid.Length; i++)
                _grid[i] = new ElementData(gridState[i], false);
        }
    }
}
```

- [ ] Ejecutar los 10 tests. Verificar que pasan.

- [ ] Agregar en `GameLifetimeScope.Configure`:

```csharp
builder.Register<MergeGridSystem>(Lifetime.Singleton);
```

- [ ] Commit:

```bash
git add Assets/_Project/Scripts/Systems/MergeGridSystem.cs Assets/Tests/EditMode/MergeGridSystemTests.cs Assets/_Project/Scripts/DI/GameLifetimeScope.cs
git commit -m "feat: add MergeGridSystem with merge validation, execution and rare bonus"
```

---

### Task 14: SpawnSystem

**Files:**
- Create: `merge-tower-unity/Assets/_Project/Scripts/Systems/SpawnSystem.cs`
- Create: `merge-tower-unity/Assets/Tests/EditMode/SpawnSystemTests.cs`

- [ ] Crear `Assets/Tests/EditMode/SpawnSystemTests.cs`:

```csharp
using NUnit.Framework;
using UnityEngine;
using MergeTower;

namespace MergeTower.Tests
{
    public class SpawnSystemTests
    {
        private SpawnSystem _spawn;
        private MergeGridSystem _grid;
        private GameConfig _config;

        [SetUp]
        public void SetUp()
        {
            _config = ScriptableObject.CreateInstance<GameConfig>();
            _config.SpawnIntervalSeconds = 3f;
            _config.RareSpawnChance = 0f;
            _config.UpgradeCostBase = 100;
            _config.CostMultiplier = 1.8f;
            _config.CoinsPerSecondMultiplier = 1f;
            var economy = new EconomySystem(null, _config);
            var tower = new TowerProgressionSystem(economy, _config);
            _grid = new MergeGridSystem(null, tower, economy);
            _spawn = new SpawnSystem(_grid, null, _config);
        }

        [Test]
        public void Tick_BeforeInterval_DoesNotSpawn()
        {
            _spawn.Tick(2.9f);
            Assert.AreEqual(default(ElementData), _grid.GetCell(0));
        }

        [Test]
        public void Tick_AfterInterval_SpawnsElement()
        {
            _spawn.Tick(3.0f);
            Assert.AreEqual(1, _grid.GetCell(0).Level);
        }

        [Test]
        public void Tick_FullGrid_DoesNotThrow()
        {
            for (int i = 0; i < 16; i++)
                _grid.PlaceElement(i, new ElementData(1, false));
            Assert.DoesNotThrow(() => _spawn.Tick(3.0f));
        }

        [Test]
        public void RareSpawnChance_Zero_NeverSpawnsRare()
        {
            int rareCount = 0;
            for (int i = 0; i < 100; i++)
            {
                for (int j = 0; j < 16; j++) _grid.PlaceElement(j, default);
                _spawn.Tick(3.0f);
                if (_grid.GetCell(0).IsRare) rareCount++;
            }
            Assert.AreEqual(0, rareCount);
        }
    }
}
```

- [ ] Ejecutar tests. Verificar que fallan.

- [ ] Crear `Assets/_Project/Scripts/Systems/SpawnSystem.cs`:

```csharp
namespace MergeTower
{
    public sealed class SpawnSystem : ITickable
    {
        private float _timer;
        private readonly MergeGridSystem _grid;
        private readonly RareElementChannel _rareChannel;
        private readonly GameConfig _config;
        private readonly System.Random _random = new System.Random();

        public SpawnSystem(
            MergeGridSystem grid,
            RareElementChannel rareChannel,
            GameConfig config)
        {
            _grid = grid;
            _rareChannel = rareChannel;
            _config = config;
        }

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
            var data = new ElementData(1, isRare);
            _grid.PlaceElement(cellIndex, data);
            if (isRare) _rareChannel?.Raise(data);
        }
    }
}
```

- [ ] Ejecutar los 4 tests. Verificar que pasan.

- [ ] Agregar en `GameLifetimeScope.Configure`:

```csharp
builder.Register<SpawnSystem>(Lifetime.Singleton);
builder.RegisterEntryPoint<SpawnSystem>();
```

- [ ] Commit:

```bash
git add Assets/_Project/Scripts/Systems/SpawnSystem.cs Assets/Tests/EditMode/SpawnSystemTests.cs Assets/_Project/Scripts/DI/GameLifetimeScope.cs
git commit -m "feat: add SpawnSystem with configurable interval and rare element probability"
```

---

### Task 15: DailyGoalSystem

**Files:**
- Create: `merge-tower-unity/Assets/_Project/Scripts/Systems/DailyGoalSystem.cs`
- Create: `merge-tower-unity/Assets/Tests/EditMode/DailyGoalSystemTests.cs`

- [ ] Crear `Assets/Tests/EditMode/DailyGoalSystemTests.cs`:

```csharp
using NUnit.Framework;
using UnityEngine;
using MergeTower;

namespace MergeTower.Tests
{
    public class DailyGoalSystemTests
    {
        private DailyGoalSystem _goal;
        private GameConfig _config;

        [SetUp]
        public void SetUp()
        {
            _config = ScriptableObject.CreateInstance<GameConfig>();
            _config.DailyGoalTarget = 5;
            _goal = new DailyGoalSystem(null, _config);
        }

        [Test]
        public void RecordMerge_IncrementsProgress()
        {
            var save = new PlayerSaveData
                { LastGoalResetDateUtc = System.DateTime.UtcNow.ToString("yyyy-MM-dd") };
            _goal.Initialize(save);
            _goal.RecordMerge();
            Assert.AreEqual(1, _goal.GetCurrentProgress());
        }

        [Test]
        public void Initialize_SameDay_PreservesProgress()
        {
            string today = System.DateTime.UtcNow.ToString("yyyy-MM-dd");
            var save = new PlayerSaveData { DailyGoalProgress = 3, LastGoalResetDateUtc = today };
            _goal.Initialize(save);
            Assert.AreEqual(3, _goal.GetCurrentProgress());
        }

        [Test]
        public void Initialize_NewDay_ResetsProgress()
        {
            var save = new PlayerSaveData
                { DailyGoalProgress = 3, LastGoalResetDateUtc = "2000-01-01" };
            _goal.Initialize(save);
            Assert.AreEqual(0, _goal.GetCurrentProgress());
        }

        [Test]
        public void RecordMerge_ReachesTarget_IsCompleted()
        {
            var save = new PlayerSaveData
                { LastGoalResetDateUtc = System.DateTime.UtcNow.ToString("yyyy-MM-dd") };
            _goal.Initialize(save);
            for (int i = 0; i < 5; i++) _goal.RecordMerge();
            Assert.IsTrue(_goal.IsGoalCompleted());
        }
    }
}
```

- [ ] Ejecutar tests. Verificar que fallan.

- [ ] Crear `Assets/_Project/Scripts/Systems/DailyGoalSystem.cs`:

```csharp
using System;

namespace MergeTower
{
    public sealed class DailyGoalSystem
    {
        private int _mergeCount;
        private bool _goalCompleted;
        private readonly DailyGoalChannel _goalChannel;
        private readonly GameConfig _config;

        public DailyGoalSystem(DailyGoalChannel goalChannel, GameConfig config)
        {
            _goalChannel = goalChannel;
            _config = config;
        }

        public void Initialize(PlayerSaveData saveData)
        {
            string todayUtc = DateTime.UtcNow.ToString("yyyy-MM-dd");
            if (saveData.LastGoalResetDateUtc != todayUtc)
            {
                saveData.DailyGoalProgress = 0;
                saveData.LastGoalResetDateUtc = todayUtc;
            }
            _mergeCount = saveData.DailyGoalProgress;
            _goalCompleted = _mergeCount >= _config.DailyGoalTarget;
        }

        public void RecordMerge()
        {
            if (_goalCompleted) return;
            _mergeCount++;
            var progress = new GoalProgress(_mergeCount, _config.DailyGoalTarget);
            _goalChannel?.Raise(progress);
            if (_mergeCount >= _config.DailyGoalTarget)
                _goalCompleted = true;
        }

        public int GetCurrentProgress() => _mergeCount;
        public bool IsGoalCompleted() => _goalCompleted;
    }
}
```

- [ ] Ejecutar los 4 tests. Verificar que pasan.

- [ ] Agregar en `GameLifetimeScope.Configure`:

```csharp
builder.Register<DailyGoalSystem>(Lifetime.Singleton);
```

- [ ] Commit:

```bash
git add Assets/_Project/Scripts/Systems/DailyGoalSystem.cs Assets/Tests/EditMode/DailyGoalSystemTests.cs Assets/_Project/Scripts/DI/GameLifetimeScope.cs
git commit -m "feat: add DailyGoalSystem with UTC day reset and completion tracking"
```

---

## Fase 4 — Monetización

### Task 16: AdsService (AppLovin MAX)

**Files:**
- Create: `merge-tower-unity/Assets/_Project/Scripts/Services/AdsService.cs`

> **Prerequisito:** Importar AppLovin MAX SDK. Descargar el `.unitypackage` desde el dashboard de AppLovin (`dash.applovin.com`) e importarlo via `Assets > Import Package > Custom Package`.

- [ ] Importar AppLovin MAX SDK al proyecto Unity. Verificar que compila sin errores.

- [ ] Crear `Assets/_Project/Scripts/Services/AdsService.cs`:

```csharp
using UnityEngine;

namespace MergeTower
{
    public sealed class AdsService : MonoBehaviour
    {
        [SerializeField] private string _sdkKey = "YOUR_APPLOVIN_SDK_KEY";
        [SerializeField] private string _rewardedAdUnitId = "YOUR_REWARDED_AD_UNIT_ID";
        [SerializeField] private string _interstitialAdUnitId = "YOUR_INTERSTITIAL_AD_UNIT_ID";

        private bool _isInitialized;
        private RewardType _pendingRewardType;
        private TenjinService _tenjinService;
        private AdRewardChannel _rewardChannel;

        public void Construct(TenjinService tenjinService, AdRewardChannel rewardChannel)
        {
            _tenjinService = tenjinService;
            _rewardChannel = rewardChannel;
        }

        private void Awake()
        {
            MaxSdkCallbacks.OnSdkInitializedEvent += OnSdkInitialized;
            MaxSdk.SetSdkKey(_sdkKey);
            MaxSdk.InitializeSdk();
        }

        private void OnSdkInitialized(MaxSdkBase.SdkConfiguration config)
        {
            _isInitialized = true;
            RegisterRewardedCallbacks();
            RegisterInterstitialCallbacks();
            MaxSdk.LoadRewardedAd(_rewardedAdUnitId);
            MaxSdk.LoadInterstitial(_interstitialAdUnitId);
        }

        private void RegisterRewardedCallbacks()
        {
            MaxSdkCallbacks.Rewarded.OnAdHiddenEvent += (id, info) =>
                MaxSdk.LoadRewardedAd(_rewardedAdUnitId);
            MaxSdkCallbacks.Rewarded.OnAdReceivedRewardEvent += (id, reward, info) =>
                _rewardChannel?.Raise(_pendingRewardType);
            MaxSdkCallbacks.Rewarded.OnAdRevenuePaidEvent += OnAdRevenuePaid;
        }

        private void RegisterInterstitialCallbacks()
        {
            MaxSdkCallbacks.Interstitial.OnAdHiddenEvent += (id, info) =>
                MaxSdk.LoadInterstitial(_interstitialAdUnitId);
            MaxSdkCallbacks.Interstitial.OnAdRevenuePaidEvent += OnAdRevenuePaid;
        }

        public void ShowRewarded(RewardType rewardType)
        {
            if (!_isInitialized) return;
            if (!MaxSdk.IsRewardedAdReady(_rewardedAdUnitId)) return;
            _pendingRewardType = rewardType;
            MaxSdk.ShowRewardedAd(_rewardedAdUnitId);
        }

        public void ShowInterstitial()
        {
            if (!_isInitialized) return;
            if (!MaxSdk.IsInterstitialReady(_interstitialAdUnitId)) return;
            MaxSdk.ShowInterstitial(_interstitialAdUnitId);
        }

        public bool IsRewardedReady =>
            _isInitialized && MaxSdk.IsRewardedAdReady(_rewardedAdUnitId);

        private void OnAdRevenuePaid(string adUnitId, MaxSdkBase.AdInfo adInfo)
        {
            var payload = new AdRevenuePayload(
                adInfo.Revenue * 1000.0,
                adInfo.NetworkName,
                adUnitId
            );
            _tenjinService?.TrackAdRevenue(payload);
        }
    }
}
```

- [ ] En Unity Editor, crear un GameObject `[AdsService]` en la escena, agregarle el componente `AdsService`. Asignar SDK Key, Rewarded Ad Unit ID e Interstitial Ad Unit ID desde el dashboard de AppLovin.

- [ ] Agregar en `GameLifetimeScope.Configure`:

```csharp
builder.RegisterComponentInHierarchy<AdsService>();
```

- [ ] Verificar que Unity compila sin errores.

- [ ] Commit:

```bash
git add Assets/_Project/Scripts/Services/AdsService.cs Assets/_Project/Scripts/DI/GameLifetimeScope.cs
git commit -m "feat: add AdsService with async AppLovin MAX init, rewarded and interstitial"
```

---

### Task 17: TenjinService

**Files:**
- Create: `merge-tower-unity/Assets/_Project/Scripts/Services/TenjinService.cs`

> **Prerequisito:** Importar Tenjin Unity SDK. Descargar desde `github.com/tenjin/tenjin-unity-sdk` e importar el `.unitypackage`.

- [ ] Importar Tenjin SDK al proyecto Unity. Verificar que compila sin errores.

- [ ] Crear `Assets/_Project/Scripts/Services/TenjinService.cs`:

```csharp
using UnityEngine;

namespace MergeTower
{
    public sealed class TenjinService : MonoBehaviour
    {
        [SerializeField] private string _apiKey = "YOUR_TENJIN_API_KEY";

        private BaseTenjin _instance;

        private void Awake()
        {
            _instance = Tenjin.getInstance(_apiKey);
        }

        private void OnApplicationPause(bool pauseStatus)
        {
            if (!pauseStatus)
                _instance.Connect();
        }

        public void TrackAdRevenue(AdRevenuePayload payload)
        {
            if (_instance == null) return;
            _instance.TrackAdRevenue(
                adPlatform: "applovin_max",
                adSource: payload.NetworkName,
                adUnitName: payload.AdUnitId,
                adFormat: "rewarded",
                value: payload.eCPM / 1000.0,
                currency: "USD"
            );
        }
    }
}
```

- [ ] En Unity Editor, crear un GameObject `[TenjinService]` en la escena, agregarle el componente `TenjinService`. Asignar API Key desde el dashboard de Tenjin.

- [ ] Agregar en `GameLifetimeScope.Configure`:

```csharp
builder.RegisterComponentInHierarchy<TenjinService>();
```

- [ ] Verificar que Unity compila sin errores.

- [ ] Commit:

```bash
git add Assets/_Project/Scripts/Services/TenjinService.cs Assets/_Project/Scripts/DI/GameLifetimeScope.cs
git commit -m "feat: add TenjinService for ad revenue ROAS tracking"
```

---

### Task 18: IAPService (Unity IAP)

**Files:**
- Create: `merge-tower-unity/Assets/_Project/Scripts/Services/IAPService.cs`

> **Prerequisito:** Instalar Unity IAP via Package Manager: `Window > Package Manager > Unity Registry > In App Purchasing`.

- [ ] Instalar Unity IAP desde Package Manager. Verificar que compila sin errores.

- [ ] Crear `Assets/_Project/Scripts/Services/IAPService.cs`:

```csharp
using UnityEngine;
using UnityEngine.Purchasing;
using UnityEngine.Purchasing.Extension;

namespace MergeTower
{
    public sealed class IAPService : MonoBehaviour, IDetailedStoreListener
    {
        public const string ExtraSlotsProductId = "merge_tower_extra_slots";
        public const string PremiumCoins1ProductId = "merge_tower_premium_coins_1";
        public const string RemoveAdsProductId = "merge_tower_remove_ads";

        private IStoreController _storeController;
        private PurchaseChannel _purchaseChannel;

        public void Construct(PurchaseChannel purchaseChannel)
        {
            _purchaseChannel = purchaseChannel;
        }

        private void Start()
        {
            var builder = ConfigurationBuilder.Instance(StandardPurchasingModule.Instance());
            builder.AddProduct(ExtraSlotsProductId, ProductType.NonConsumable);
            builder.AddProduct(PremiumCoins1ProductId, ProductType.Consumable);
            builder.AddProduct(RemoveAdsProductId, ProductType.NonConsumable);
            UnityPurchasing.Initialize(this, builder);
        }

        public void Purchase(string productId)
        {
            if (_storeController == null) return;
            _storeController.InitiatePurchase(productId);
        }

        public void OnInitialized(IStoreController controller, IExtensionProvider extensions)
            => _storeController = controller;

        public void OnInitializeFailed(InitializationFailureReason error)
            => Debug.LogWarning($"[IAP] Init failed: {error}");

        public void OnInitializeFailed(InitializationFailureReason error, string message)
            => Debug.LogWarning($"[IAP] Init failed: {error} — {message}");

        public PurchaseProcessingResult ProcessPurchase(PurchaseEventArgs args)
        {
            _purchaseChannel?.Raise(args.purchasedProduct.definition.id);
            return PurchaseProcessingResult.Complete;
        }

        public void OnPurchaseFailed(Product product, PurchaseFailureDescription failureDescription)
            => Debug.LogWarning($"[IAP] Purchase failed: {product.definition.id} — {failureDescription.reason}");
    }
}
```

- [ ] En Unity Editor, crear un GameObject `[IAPService]` en la escena, agregarle el componente `IAPService`.

- [ ] Agregar en `GameLifetimeScope.Configure`:

```csharp
builder.RegisterComponentInHierarchy<IAPService>();
```

- [ ] Verificar que Unity compila sin errores.

- [ ] Commit:

```bash
git add Assets/_Project/Scripts/Services/IAPService.cs Assets/_Project/Scripts/DI/GameLifetimeScope.cs
git commit -m "feat: add IAPService with Unity IAP for slots, coins and remove ads products"
```

---

## Fase 5 — HUD y Guardado Automático

### Task 19: HudPresenter y AutoSaveController

**Files:**
- Create: `merge-tower-unity/Assets/_Project/Scripts/UI/HudPresenter.cs`
- Create: `merge-tower-unity/Assets/_Project/Scripts/Services/AutoSaveController.cs`

- [ ] Crear `Assets/_Project/Scripts/UI/HudPresenter.cs`:

```csharp
using UnityEngine;
using TMPro;
using UnityEngine.UI;

namespace MergeTower
{
    public sealed class HudPresenter : MonoBehaviour
    {
        [SerializeField] private TextMeshProUGUI _coinsLabel;
        [SerializeField] private TextMeshProUGUI _dailyGoalLabel;
        [SerializeField] private Button _rewardedAdButton;
        [SerializeField] private CoinsChangedChannel _coinsChangedChannel;
        [SerializeField] private DailyGoalChannel _dailyGoalChannel;
        [SerializeField] private AdsService _adsService;

        private void OnEnable()
        {
            _coinsChangedChannel.OnEventRaised += OnCoinsChanged;
            _dailyGoalChannel.OnEventRaised += OnGoalProgress;
        }

        private void OnDisable()
        {
            _coinsChangedChannel.OnEventRaised -= OnCoinsChanged;
            _dailyGoalChannel.OnEventRaised -= OnGoalProgress;
        }

        private void Update()
        {
            if (_rewardedAdButton != null && _adsService != null)
                _rewardedAdButton.interactable = _adsService.IsRewardedReady;
        }

        private void OnCoinsChanged(long coins) =>
            _coinsLabel.text = FormatCoins(coins);

        private void OnGoalProgress(GoalProgress progress) =>
            _dailyGoalLabel.text = $"{progress.Current}/{progress.Target}";

        public void OnRewardedAdButtonPressed() =>
            _adsService?.ShowRewarded(RewardType.ExtraElement);

        private static string FormatCoins(long coins)
        {
            if (coins >= 1_000_000_000L) return $"{coins / 1_000_000_000f:F1}B";
            if (coins >= 1_000_000L) return $"{coins / 1_000_000f:F1}M";
            if (coins >= 1_000L) return $"{coins / 1_000f:F1}K";
            return coins.ToString();
        }
    }
}
```

- [ ] Crear `Assets/_Project/Scripts/Services/AutoSaveController.cs`:

```csharp
using UnityEngine;

namespace MergeTower
{
    public sealed class AutoSaveController : MonoBehaviour, ITickable
    {
        private float _autoSaveTimer;
        private const float AutoSaveInterval = 60f;

        private SaveSystem _saveSystem;
        private PlayerSaveData _saveData;
        private EconomySystem _economy;
        private TowerProgressionSystem _tower;
        private DailyGoalSystem _dailyGoal;

        public void Construct(
            SaveSystem saveSystem,
            PlayerSaveData saveData,
            EconomySystem economy,
            TowerProgressionSystem tower,
            DailyGoalSystem dailyGoal)
        {
            _saveSystem = saveSystem;
            _saveData = saveData;
            _economy = economy;
            _tower = tower;
            _dailyGoal = dailyGoal;
        }

        public void Tick(float deltaTime)
        {
            _autoSaveTimer += deltaTime;
            if (_autoSaveTimer < AutoSaveInterval) return;
            _autoSaveTimer = 0f;
            SaveGame();
        }

        private void OnApplicationPause(bool pauseStatus)
        {
            if (pauseStatus) SaveGame();
        }

        private void OnApplicationFocus(bool hasFocus)
        {
            if (!hasFocus) SaveGame();
        }

        public void SaveGame()
        {
            if (_saveSystem == null) return;
            _saveData.Coins = _economy.GetBalance();
            _saveData.TowerLevel = _tower.GetTowerLevel();
            _saveData.CoinsPerSecond = _tower.GetCoinsPerSecond();
            _saveData.DailyGoalProgress = _dailyGoal.GetCurrentProgress();
            _saveSystem.Save(_saveData);
        }
    }
}
```

- [ ] En Unity Editor, crear el Canvas con el HUD. Agregar `HudPresenter` al Canvas. Asignar referencias de canales de eventos (`CoinsChangedChannel`, `DailyGoalChannel`) y `AdsService` en el inspector.

- [ ] Crear un GameObject `[AutoSave]` con el componente `AutoSaveController`.

- [ ] Verificar que Unity compila sin errores y el HUD muestra coins al arrancar.

- [ ] Commit:

```bash
git add Assets/_Project/Scripts/UI/HudPresenter.cs Assets/_Project/Scripts/Services/AutoSaveController.cs
git commit -m "feat: add HudPresenter with event-driven UI and AutoSaveController"
```

---

## Fase 6 — Testing de Integración y Build

### Task 20: Test de integración del loop principal

**Files:**
- Create: `merge-tower-unity/Assets/Tests/PlayMode/GameLoopIntegrationTests.cs`

- [ ] Crear `Assets/Tests/PlayMode/GameLoopIntegrationTests.cs`:

```csharp
using System.Collections;
using NUnit.Framework;
using UnityEngine;
using UnityEngine.TestTools;
using MergeTower;

namespace MergeTower.Tests
{
    public class GameLoopIntegrationTests
    {
        private EconomySystem _economy;
        private MergeGridSystem _grid;
        private TowerProgressionSystem _tower;
        private GameConfig _config;

        [SetUp]
        public void SetUp()
        {
            _config = ScriptableObject.CreateInstance<GameConfig>();
            _config.UpgradeCostBase = 100;
            _config.CostMultiplier = 1.8f;
            _config.CoinsPerSecondMultiplier = 1f;
            _economy = new EconomySystem(null, _config);
            _tower = new TowerProgressionSystem(_economy, _config);
            _grid = new MergeGridSystem(null, _tower, _economy);
        }

        [UnityTest]
        public IEnumerator MergeLoop_TenMerges_TowerLevelIncreases()
        {
            for (int i = 0; i < 10; i++)
            {
                _grid.PlaceElement(0, new ElementData(1, false));
                _grid.PlaceElement(1, new ElementData(1, false));
                _grid.ExecuteMerge(0, 1);
                yield return null;
            }
            Assert.Greater(_tower.GetTowerLevel(), 0);
        }

        [UnityTest]
        public IEnumerator EconomyTick_ManualOneSecond_AccumulatesCoins()
        {
            _tower.AddMergedElement(5);
            _economy.Tick(1.0f);
            yield return null;
            Assert.AreEqual(5L, _economy.GetBalance());
        }

        [UnityTest]
        public IEnumerator MergeLoop_NoGCAllocation_DuringHundredMerges()
        {
            // Pre-calentar para evitar falsos positivos de JIT
            for (int i = 0; i < 5; i++)
            {
                _grid.PlaceElement(0, new ElementData(1, false));
                _grid.PlaceElement(1, new ElementData(1, false));
                _grid.ExecuteMerge(0, 1);
            }
            yield return null;

            long gcBefore = System.GC.GetTotalMemory(false);

            for (int i = 0; i < 100; i++)
            {
                _grid.PlaceElement(0, new ElementData(1, false));
                _grid.PlaceElement(1, new ElementData(1, false));
                _grid.ExecuteMerge(0, 1);
                yield return null;
            }

            long gcAfter = System.GC.GetTotalMemory(false);
            long allocated = gcAfter - gcBefore;

            Assert.AreEqual(0L, allocated,
                $"GC allocated {allocated} bytes during 100 merges. " +
                "Open Profiler > Deep Profile to find the allocation source.");
        }
    }
}
```

- [ ] Ejecutar en Unity Test Runner > PlayMode. Verificar que los 3 tests pasan.

- [ ] Si `MergeLoop_NoGCAllocation` falla: abrir `Window > Analysis > Profiler`, activar `Allocation Tracking`, reproducir 100 merges y buscar `GC.Alloc` en el frame. Corregir la fuente antes de continuar.

- [ ] Commit:

```bash
git add Assets/Tests/PlayMode/GameLoopIntegrationTests.cs
git commit -m "test: add PlayMode integration tests including zero-GC allocation check"
```

---

### Task 21: Configuración de Build Android (AAB + IL2CPP)

**Files:**
- Modify: `merge-tower-unity/ProjectSettings/ProjectSettings.asset` (via Unity Editor)

- [ ] En Unity Editor, abrir `File > Build Settings`, seleccionar `Android`, click `Switch Platform`.

- [ ] En `Player Settings > Android`:
  - Scripting Backend: `IL2CPP`
  - Target Architectures: activar `ARM64` y `ARMv7`
  - Bundle Identifier: `com.yourcompany.mergetower`
  - Minimum API Level: `Android 6.0 (API 23)`
  - Managed Stripping Level: `Minimal`

- [ ] Crear keystore para firma del AAB:

```bash
keytool -genkey -v -keystore merge-tower.keystore \
  -alias merge-tower -keyalg RSA -keysize 2048 -validity 10000
```

- [ ] Codificar keystore en base64 y agregar como secret en GitHub:

```bash
base64 -i merge-tower.keystore | pbcopy
# Pegar en GitHub > Settings > Secrets > ANDROID_KEYSTORE_BASE64
```

- [ ] Hacer push a `main`. Verificar que el job `build-android` en GitHub Actions genera el AAB como artefacto descargable.

- [ ] Descargar el AAB y verificar que instala en un dispositivo Android físico ARM64.

- [ ] Commit:

```bash
git add ProjectSettings/
git commit -m "build: configure Android IL2CPP AAB build settings"
```

---

### Task 22: Configuración de Build iOS (Xcode Project)

**Files:**
- Modify: `merge-tower-unity/ProjectSettings/ProjectSettings.asset` (via Unity Editor)

- [ ] En Unity Editor, abrir `File > Build Settings`, seleccionar `iOS`, click `Switch Platform`.

- [ ] En `Player Settings > iOS`:
  - Scripting Backend: `IL2CPP` (único disponible)
  - Architecture: `ARM64`
  - Bundle Identifier: `com.yourcompany.mergetower`
  - Target minimum iOS Version: `14.0`
  - Enable Bitcode: `false`

- [ ] Hacer push a `main`. Verificar que el job `build-ios` en GitHub Actions genera el Xcode project como artefacto.

- [ ] Descargar el Xcode project, abrirlo en Xcode en macOS, configurar Team ID y provisioning profile, compilar y subir a TestFlight.

- [ ] Verificar que AppLovin MAX y Unity IAP están correctamente configurados en Xcode (frameworks presentes, `Info.plist` con `SKAdNetworkItems` para MAX).

- [ ] Commit:

```bash
git add ProjectSettings/
git commit -m "build: configure iOS IL2CPP Xcode project build settings"
```

---

## Checklist de Criterios de Aceptación Final

Antes de considerar el MVP completo, verificar que todos estos criterios pasan:

- [ ] Loop de merge funciona sin crashes en 10 minutos de sesión continua
- [ ] Torre crece visualmente con cada merge completado
- [ ] Coins/sec se acumula correctamente en background (idle)
- [ ] Elemento raro aparece con probabilidad configurable y aplica bonus 3x
- [ ] Preview de resultado de merge visible durante drag
- [ ] Objetivo diario se resetea correctamente al cambiar de día UTC
- [ ] Rewarded Ad se muestra y entrega recompensa correcta
- [ ] Intersticial se muestra solo en transiciones (no durante gameplay activo)
- [ ] IAP completa ciclo de compra y aplica beneficio
- [ ] eCPM capturado y enviado a Tenjin en cada ad completado
- [ ] Guardado local persiste entre sesiones sin corrupción
- [ ] Build Android AAB generado por CI/CD sin errores
- [ ] Build iOS Xcode project generado por CI/CD sin errores
- [ ] Cero GC allocations en Profiler durante tick de gameplay (merge, spawn, economy)
