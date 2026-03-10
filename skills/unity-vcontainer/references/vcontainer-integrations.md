# VContainer 통합 패턴

## DI를 통해 메세지 발행

```csharp
public interface IMessageService
{
    void Publish<T>(T message);
    IDisposable Subscribe<T>(Action<T> handler);
}

// 등록
builder.Register<IMessageService, MessageService>(Lifetime.Singleton);

// 주입된 서비스에서 사용
public class ScoreService
{
    [Inject] private readonly IMessageService _messageService;

    public void AddScore(int points)
    {
        score += points;
        events.Publish(new ScoreChanged { NewScore = score });
    }
}
```

## 멀티 씬 아키텍처

### Root 스코프 (영구적)

```csharp
public class RootLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<INetworkService, NetworkService>(Lifetime.Singleton);
        builder.Register<IAuthService, AuthService>(Lifetime.Singleton);
        builder.Register<IAudioManager, AudioManager>(Lifetime.Singleton);
    }
}
```

### 씬별 스코프

```csharp
public class BattleSceneScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<IBattleService, BattleService>(Lifetime.Scoped);
        builder.Register<IEnemySpawner, EnemySpawner>(Lifetime.Scoped);
        builder.RegisterComponentInHierarchy<BattleUIController>();
    }
}
```

## 소스 제너레이터 지원

VContainer는 리플렉션 오버헤드를 줄이기 위한 컴파일 타임 코드 생성을 지원합니다.

```csharp
// VContainer.SourceGenerator 패키지 추가

[VContainerGenerate] // 특정 타입에 대해 옵트인
public partial class ServiceA
{
    [Inject] private readonly IDependency dependency;
}
```

## 순환 의존성 감지

VContainer는 컨테이너 빌드 시점에 순환 의존성을 자동으로 감지하고 명확한 의존성 체인과 함께 예외를 발생시킵니다.

```csharp
// 이는 런타임이 아닌 Build() 시점에 감지됨
public class ServiceA
{
    public ServiceA(ServiceB b) { }
}

public class ServiceB
{
    public ServiceB(ServiceA a) { } // 순환 의존성 감지됨
}
```

## 성능 프로파일링

```csharp
#if UNITY_EDITOR
[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]
static void EnableDiagnostics()
{
    VContainerSettings.Instance.EnableDiagnostics = true;
}
#endif
```

## 등록 모범 사례

- 컴파일 타임 안전성을 위해 타입 기반보다 제네릭 등록(`Register<IService, Service>()`) 선호
- 복잡한 컨테이너 후처리 초기화에는 `RegisterBuildCallback` 사용
- Update 루프에서 resolve 피하기; 해결된 인스턴스 캐싱
- 적절한 라이프타임 사용: 앱 전체에는 Singleton, 씬별에는 Scoped, 단기에는 Transient