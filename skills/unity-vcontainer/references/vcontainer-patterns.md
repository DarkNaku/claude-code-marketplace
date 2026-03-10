# VContainer 모범 사례

Unity에서 고급 VContainer 패턴을 위한 L3 심층 참조 문서.

## LifetimeScope 계층 구조

자식 스코프는 부모 등록을 상속받고 재정의할 수 있습니다. 부모를 Dispose하면 모든 자식도 함께 Dispose됩니다.

### Root, Scene, Local 스코프

```csharp
// Root 스코프: 애플리케이션 전체 싱글톤, 전체 앱 라이프타임 동안 유지
public class RootLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<IAnalyticsService, AnalyticsService>(Lifetime.Singleton);
        builder.Register<ISaveSystem, SaveSystem>(Lifetime.Singleton);
    }
}

// Scene 스코프: Inspector에서 Parent 설정하거나 Parent 속성 재정의
public class BattleSceneLifetimeScope : LifetimeScope
{
    [SerializeField] private BattleConfig mBattleConfig;

    protected override void Configure(IContainerBuilder builder)
    {
        builder.RegisterInstance(mBattleConfig);
        builder.Register<IBattleManager, BattleManager>(Lifetime.Scoped);
        builder.RegisterComponentInHierarchy<BattleUI>();
    }
}

// Local 스코프: 엔티티별 컨테이너를 위한 런타임 자식
public class EnemySpawner : MonoBehaviour
{
    [Inject] private LifetimeScope mParentScope;

    public void SpawnEnemy(EnemyData data)
    {
        var childScope = mParentScope.CreateChild(builder =>
        {
            builder.RegisterInstance(data);
            builder.Register<IEnemyAI, EnemyAI>(Lifetime.Scoped);
            builder.RegisterComponentOnNewGameObject<EnemyController>(
                Lifetime.Scoped, "Enemy");
        });
    }
}
```

### 부모-자식 연결

```csharp
// 1. Inspector: 부모 LifetimeScope를 "Parent" 필드에 드래그

// 2. Parent 속성 재정의
public class GameplayLifetimeScope : LifetimeScope
{
    protected override LifetimeScope Parent => LifetimeScope.Find<RootLifetimeScope>();
}

// 3. 런타임 생성 스코프에 EnqueueParent 사용
using (LifetimeScope.EnqueueParent(parentScope))
{
    // 이 프레임에서 다음 LifetimeScope.Awake()가 parentScope를 사용
}
```

## 주입 타입

### 생성자 주입 (권장)

순수 C# 클래스의 모든 필수 의존성에 사용.

```csharp
public class WeaponSystem : IWeaponSystem
{
    private readonly IInventoryService mInventory;
    private readonly IDamageCalculator mDamageCalc;

    // VContainer는 해결 가능한 매개변수가 가장 많은 생성자를 선택
    public WeaponSystem(IInventoryService inventory, IDamageCalculator damageCalc)
    {
        mInventory = inventory;
        mDamageCalc = damageCalc;
    }
}

builder.Register<IWeaponSystem, WeaponSystem>(Lifetime.Singleton);
```

### 메서드 주입

매개변수가 있는 생성자를 가질 수 없는 MonoBehaviour에 사용.

```csharp
public class HUDController : MonoBehaviour
{
    private IScoreService mScoreService;

    [Inject]
    public void Construct(IScoreService scoreService)
    {
        mScoreService = scoreService;
    }
}

builder.RegisterComponentInHierarchy<HUDController>();
```

### 속성 / 필드 주입

최후의 수단. 생성자/메서드 주입보다 의존성이 덜 명시적입니다.

```csharp
public class DebugOverlay : MonoBehaviour
{
    [Inject] private readonly IDebugService mDebugService;
}
```

### 선택 가이드

| 타입 | 대상 | 사용 시점 |
|------|--------|----------|
| 생성자 | 순수 C# 클래스 | 서비스의 기본 선택 |
| 메서드 `[Inject]` | MonoBehaviours | DI가 필요한 MonoBehaviour |
| 필드/속성 `[Inject]` | 모두 | 레거시 코드; 다른 옵션 없을 때 |

## Mock 등록으로 테스트하기

### 기본 테스트 설정

`ContainerBuilder`를 직접 생성. 씬이나 `LifetimeScope` 불필요.

```csharp
using NUnit.Framework;
using NSubstitute;
using VContainer;

[TestFixture]
public class WeaponSystemTests
{
    private IObjectResolver mContainer;
    private IInventoryService mMockInventory;

    [SetUp]
    public void SetUp()
    {
        mMockInventory = Substitute.For<IInventoryService>();
        var builder = new ContainerBuilder();
        builder.RegisterInstance(mMockInventory).As<IInventoryService>();
        builder.Register<IDamageCalculator, DamageCalculator>(Lifetime.Transient);
        builder.Register<IWeaponSystem, WeaponSystem>(Lifetime.Transient);
        mContainer = builder.Build();
    }

    [TearDown]
    public void TearDown() => mContainer.Dispose();

    [Test]
    public void Attack_ConsumesAmmo()
    {
        var system = mContainer.Resolve<IWeaponSystem>();
        system.Attack(new WeaponData());
        mMockInventory.Received(1).ConsumeAmmo(Arg.Any<int>());
    }
}
```

### 단일 서비스 대체하기

기본 빌더를 공유하고 특정 테스트에 필요한 것만 재정의.

```csharp
[Test]
public void BossMultiplier_IsApplied()
{
    // CreateBaseBuilder()는 ILogger, IConfigProvider 등을 등록
    var builder = TestContainerHelper.CreateBaseBuilder();

    // IConfigProvider만 mock으로 재정의
    var mockConfig = Substitute.For<IConfigProvider>();
    mockConfig.BossDamageMultiplier.Returns(3.0f);
    builder.RegisterInstance(mockConfig).As<IConfigProvider>();
    builder.Register<IDamageCalculator, DamageCalculator>(Lifetime.Transient);

    using var container = builder.Build();
    var calc = container.Resolve<IDamageCalculator>();
    Assert.AreEqual(30f, calc.Calculate(new WeaponData { BaseDamage = 10 }));
}
```

### EntryPoint 테스트하기

`IStartable.Start()`는 `Build()` 중에 호출되므로, 단언문이 즉시 작동합니다.

```csharp
[Test]
public void GameInitializer_LoadsSaveOnStart()
{
    var mockSave = Substitute.For<ISaveSystem>();
    var builder = new ContainerBuilder();
    builder.RegisterInstance(mockSave).As<ISaveSystem>();
    builder.RegisterEntryPoint<GameInitializer>();

    using var container = builder.Build();
    mockSave.Received(1).LoadGame();
}
```

## 고급 등록 패턴

### 팩토리 등록

객체 생성이 런타임 매개변수에 의존할 때 사용.

```csharp
public class CombatLifetimeScope : LifetimeScope
{
    [SerializeField] private Projectile mPrefab;

    protected override void Configure(IContainerBuilder builder)
    {
        // 런타임 매개변수를 가진 Func<> 팩토리
        builder.RegisterFactory<Vector3, Vector3, Projectile>(container =>
        {
            var pool = container.Resolve<IObjectPool>();
            return (pos, dir) =>
            {
                var proj = pool.Get(mPrefab);
                proj.transform.position = pos;
                proj.Launch(dir);
                return proj;
            };
        }, Lifetime.Scoped);
    }
}
```

복잡한 생성 로직의 경우, `builder.Register<IProjectileFactory, ProjectileFactory>(Lifetime.Scoped)`로 등록된 커스텀 팩토리 클래스를 추출하고, `IObjectResolver.Inject()`를 사용하여 인스턴스화된 MonoBehaviour에 의존성을 주입합니다.

### 개방형 제네릭 등록

VContainer는 개방형 제네릭 자동 해결을 지원하지 않습니다. 각 닫힌 제네릭을 명시적으로 등록하세요.

```csharp
// 주어진 것: IRepository<T> 인터페이스와 Repository<T> 구현
builder.Register<IRepository<Player>, Repository<Player>>(Lifetime.Singleton);
builder.Register<IRepository<Enemy>, Repository<Enemy>>(Lifetime.Singleton);
builder.Register<IRepository<Item>, Repository<Item>>(Lifetime.Singleton);
```

### 조건부 등록

```csharp
public class PlatformLifetimeScope : LifetimeScope
{
    [SerializeField] private GameConfig mConfig;

    protected override void Configure(IContainerBuilder builder)
    {
        // 컴파일 타임 플랫폼 선택
#if UNITY_ANDROID || UNITY_IOS
        builder.Register<IFileSystem, MobileFileSystem>(Lifetime.Singleton);
#else
        builder.Register<IFileSystem, DesktopFileSystem>(Lifetime.Singleton);
#endif

        // 런타임 기능 플래그
        if (mConfig.UseNewMatchmaking)
            builder.Register<IMatchmaker, EloMatchmaker>(Lifetime.Singleton);
        else
            builder.Register<IMatchmaker, RandomMatchmaker>(Lifetime.Singleton);
    }
}
```

### EntryPoint 패턴

EntryPoint는 매니저 MonoBehaviour를 대체합니다. Unity의 PlayerLoopSystem에 연결됩니다.

```csharp
public class GameLoopManager : IStartable, ITickable, IDisposable
{
    private readonly IInputSystem mInput;
    private readonly IPhysicsSystem mPhysics;

    public GameLoopManager(IInputSystem input, IPhysicsSystem physics)
    {
        mInput = input;
        mPhysics = physics;
    }

    public void Start() => mInput.Enable();               // Build() 후 한 번 호출
    public void Tick() => mPhysics.Step(Time.deltaTime);   // 매 프레임 호출
    public void Dispose() => mInput.Disable();
}

// 등록
builder.RegisterEntryPoint<GameLoopManager>();
builder.RegisterEntryPoint<UIUpdateManager>();

// 인터페이스로도 해결 가능한 EntryPoint
builder.RegisterEntryPoint<ScoreTracker>().As<IScoreTracker>();
```

사용 가능한 인터페이스: `IStartable`, `ITickable`, `IFixedTickable`, `ILateTickable`, `IDisposable`.
