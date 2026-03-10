# MessagePipe 통합 패턴

MessagePipe를 다른 라이브러리 및 프레임워크와 통합하는 고급 패턴.

## VContainer 통합

### 기본 통합

VContainer의 DI를 통한 MessagePipe 설정.

```csharp
using MessagePipe;
using VContainer;
using VContainer.Unity;

public class RootLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        // MessagePipe 기본 설정
        builder.AddMessagePipe();

        // 서비스 등록
        builder.Register<IPlayerService, PlayerService>(Lifetime.Singleton);
        builder.Register<IEnemyService, EnemyService>(Lifetime.Singleton);

        // MonoBehaviour 등록
        builder.RegisterComponentInHierarchy<GameController>();
    }
}
```

### 고급 옵션 설정

```csharp
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.AddMessagePipe(options =>
        {
            // 인스턴스 생명주기 설정
            options.InstanceLifetime = InstanceLifetime.Singleton;

            // 스택 트레이스 캡처 (디버깅용, 성능 영향 있음)
            options.EnableCaptureStackTrace = false;

            // 글로벌 필터 추가
            options.AddGlobalMessageHandlerFilter(typeof(MessageLoggingFilter<>));
            options.AddGlobalAsyncMessageHandlerFilter(typeof(AsyncMessageLoggingFilter<>));

            // 예외 처리 필터
            options.AddGlobalMessageHandlerFilter(typeof(MessageExceptionFilter<>));
        });
    }
}
```

### 씬별 스코프 메시징

```csharp
// Root 스코프: 글로벌 메시징
public class RootLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        // 글로벌 메시지 - Singleton
        builder.AddMessagePipe(options =>
        {
            options.InstanceLifetime = InstanceLifetime.Singleton;
        });

        builder.Register<IAnalyticsService, AnalyticsService>(Lifetime.Singleton);
    }
}

// Scene 스코프: 씬 전용 메시징
public class BattleSceneLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        // 스코프 메시지 - 이 씬에서만 유효
        builder.AddMessagePipe(options =>
        {
            options.InstanceLifetime = InstanceLifetime.Scoped;
        });

        builder.Register<IBattleManager, BattleManager>(Lifetime.Scoped);
        builder.RegisterComponentInHierarchy<BattleUI>();
    }
}
```

## R3 (Reactive Extensions) 통합

MessagePipe와 R3를 조합하여 반응형 프로그래밍.

### MessagePipe를 Observable로 변환

```csharp
using System;
using MessagePipe;
using R3;
using UnityEngine;

public static class MessagePipeR3Extensions
{
    // ISubscriber를 Observable로 변환
    public static Observable<T> AsObservable<T>(this ISubscriber<T> subscriber)
    {
        return Observable.Create<T>(observer =>
        {
            var subscription = subscriber.Subscribe(msg => observer.OnNext(msg));
            return Disposable.Create(() => subscription.Dispose());
        });
    }
}

// 사용 예
public class PlayerHealthUI : MonoBehaviour
{
    [Inject] private ISubscriber<PlayerDamagedMessage> mDamageSubscriber;
    private IDisposable mSubscription;

    void Start()
    {
        // MessagePipe를 Observable로 변환하고 R3 연산자 사용
        mSubscription = mDamageSubscriber.AsObservable()
            .Where(msg => msg.Damage > 10f)  // 10 이상의 데미지만
            .Throttle(TimeSpan.FromSeconds(0.5f))  // 0.5초마다 하나씩
            .Subscribe(msg =>
            {
                ShowDamageEffect(msg.Damage);
            });
    }

    void OnDestroy()
    {
        mSubscription?.Dispose();
    }

    private void ShowDamageEffect(float damage) { /* 이펙트 표시 */ }
}
```

### 복잡한 이벤트 조합

```csharp
public class ComboSystem : MonoBehaviour
{
    [Inject] private ISubscriber<PlayerAttackMessage> mAttackSubscriber;
    [Inject] private IPublisher<ComboAchievedMessage> mComboPublisher;

    private IDisposable mComboSubscription;

    void Start()
    {
        // 1초 내에 3번 공격하면 콤보
        mComboSubscription = mAttackSubscriber.AsObservable()
            .Buffer(TimeSpan.FromSeconds(1))  // 1초 단위로 버퍼링
            .Where(attacks => attacks.Count >= 3)  // 3번 이상 공격
            .Subscribe(attacks =>
            {
                mComboPublisher.Publish(new ComboAchievedMessage
                {
                    ComboCount = attacks.Count
                });
            });
    }

    void OnDestroy()
    {
        mComboSubscription?.Dispose();
    }
}
```

### 여러 메시지 조합

```csharp
public class GameStateManager : MonoBehaviour
{
    [Inject] private ISubscriber<PlayerReadyMessage> mPlayerReadySubscriber;
    [Inject] private ISubscriber<EnemyReadyMessage> mEnemyReadySubscriber;
    [Inject] private IPublisher<BattleStartMessage> mBattleStartPublisher;

    private IDisposable mStateSubscription;

    void Start()
    {
        var playerReady = mPlayerReadySubscriber.AsObservable();
        var enemyReady = mEnemyReadySubscriber.AsObservable();

        // 플레이어와 적이 모두 준비되면 전투 시작
        mStateSubscription = Observable.CombineLatest(playerReady, enemyReady)
            .First()  // 첫 번째 조합만
            .Subscribe(_ =>
            {
                mBattleStartPublisher.Publish(new BattleStartMessage());
            });
    }

    void OnDestroy()
    {
        mStateSubscription?.Dispose();
    }
}
```

## 멀티씬 메시징 아키텍처

### 글로벌-로컬 하이브리드 패턴

```csharp
// 글로벌 메시지 (전체 게임)
namespace Game.Messages.Global
{
    public struct GamePausedMessage { }
    public struct GameResumedMessage { }
    public struct PlayerLevelUpMessage
    {
        public int PlayerId;
        public int NewLevel;
    }
}

// 로컬 메시지 (씬 전용)
namespace Game.Messages.Battle
{
    public struct WaveStartedMessage
    {
        public int WaveNumber;
    }

    public struct EnemyDefeatedMessage
    {
        public int EnemyId;
    }
}

// Root 스코프: 글로벌 메시지
public class RootLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.AddMessagePipe(options =>
        {
            options.InstanceLifetime = InstanceLifetime.Singleton;
        });

        // 글로벌 시스템
        builder.Register<IGameStateManager, GameStateManager>(Lifetime.Singleton);
        builder.Register<IPlayerProgressSystem, PlayerProgressSystem>(Lifetime.Singleton);
    }
}

// Battle 씬 스코프: 로컬 메시지
public class BattleSceneLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        // 로컬 MessagePipe 인스턴스
        builder.AddMessagePipe(options =>
        {
            options.InstanceLifetime = InstanceLifetime.Scoped;
        });

        // 배틀 전용 시스템
        builder.Register<IWaveManager, WaveManager>(Lifetime.Scoped);
        builder.Register<IEnemySpawner, EnemySpawner>(Lifetime.Scoped);
    }
}

// 글로벌 메시지 사용
public class GameStateManager : IGameStateManager
{
    private readonly IPublisher<GamePausedMessage> mPausePublisher;

    public GameStateManager(IPublisher<GamePausedMessage> pausePublisher)
    {
        mPausePublisher = pausePublisher;
    }

    public void PauseGame()
    {
        // 모든 씬에서 수신 가능
        mPausePublisher.Publish(new GamePausedMessage());
    }
}

// 로컬 메시지 사용
public class WaveManager : IWaveManager
{
    private readonly IPublisher<WaveStartedMessage> mWaveStartPublisher;

    public WaveManager(IPublisher<WaveStartedMessage> waveStartPublisher)
    {
        mWaveStartPublisher = waveStartPublisher;
    }

    public void StartWave(int waveNumber)
    {
        // 현재 씬에서만 수신 가능
        mWaveStartPublisher.Publish(new WaveStartedMessage
        {
            WaveNumber = waveNumber
        });
    }
}
```

### 씬 간 메시지 브릿지

씬 전환 시에도 메시지를 유지.

```csharp
// 씬 전환 메시지 브릿지
public interface ISceneMessageBridge
{
    void PublishToNextScene<T>(T message);
    IObservable<T> ReceiveFromPreviousScene<T>();
}

public class SceneMessageBridge : ISceneMessageBridge, IDisposable
{
    private readonly Dictionary<Type, object> mPendingMessages = new();

    public void PublishToNextScene<T>(T message)
    {
        mPendingMessages[typeof(T)] = message;
    }

    public IObservable<T> ReceiveFromPreviousScene<T>()
    {
        if (mPendingMessages.TryGetValue(typeof(T), out var message))
        {
            mPendingMessages.Remove(typeof(T));
            return Observable.Return((T)message);
        }

        return Observable.Empty<T>();
    }

    public void Dispose()
    {
        mPendingMessages.Clear();
    }
}

// Root 스코프 등록
public class RootLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.AddMessagePipe();
        builder.Register<ISceneMessageBridge, SceneMessageBridge>(Lifetime.Singleton);
    }
}

// 씬 A: 메시지 전송
public class LevelCompleteHandler : MonoBehaviour
{
    [Inject] private ISceneMessageBridge mBridge;

    public void OnLevelComplete()
    {
        mBridge.PublishToNextScene(new LevelResultMessage
        {
            Score = 1000,
            Stars = 3
        });

        // 다음 씬으로 전환
        SceneManager.LoadScene("ResultScene");
    }
}

// 씬 B: 메시지 수신
public class ResultScreenController : MonoBehaviour
{
    [Inject] private ISceneMessageBridge mBridge;

    void Start()
    {
        mBridge.ReceiveFromPreviousScene<LevelResultMessage>()
            .Subscribe(result =>
            {
                DisplayResults(result.Score, result.Stars);
            })
            .AddTo(this);
    }

    private void DisplayResults(int score, int stars) { /* UI 표시 */ }
}
```

## MVVM 패턴과 통합

MessagePipe를 사용한 View-ViewModel 통신.

```csharp
// Model
public class PlayerModel
{
    public int Health { get; set; }
    public int MaxHealth { get; set; }
}

// ViewModel
public class PlayerViewModel : IDisposable
{
    private readonly PlayerModel mModel;
    private readonly IPublisher<PlayerHealthChangedMessage> mHealthChangedPublisher;
    private readonly ISubscriber<PlayerDamagedMessage> mDamageSubscriber;
    private readonly DisposableBag mDisposables = new();

    public PlayerViewModel(
        PlayerModel model,
        IPublisher<PlayerHealthChangedMessage> healthChangedPublisher,
        ISubscriber<PlayerDamagedMessage> damageSubscriber)
    {
        mModel = model;
        mHealthChangedPublisher = healthChangedPublisher;
        mDamageSubscriber = damageSubscriber;

        // 데미지 메시지 구독
        mDamageSubscriber.Subscribe(OnDamaged).AddTo(mDisposables);
    }

    private void OnDamaged(PlayerDamagedMessage msg)
    {
        var oldHealth = mModel.Health;
        mModel.Health = Mathf.Max(0, mModel.Health - (int)msg.Damage);

        // View에 변경 알림
        mHealthChangedPublisher.Publish(new PlayerHealthChangedMessage
        {
            OldHealth = oldHealth,
            NewHealth = mModel.Health,
            MaxHealth = mModel.MaxHealth
        });
    }

    public void Dispose()
    {
        mDisposables.Dispose();
    }
}

// View
public class PlayerHealthView : MonoBehaviour
{
    [Inject] private ISubscriber<PlayerHealthChangedMessage> mHealthChangedSubscriber;
    private IDisposable mSubscription;

    [SerializeField] private UnityEngine.UI.Slider mHealthBar;

    void Start()
    {
        mSubscription = mHealthChangedSubscriber.Subscribe(msg =>
        {
            UpdateHealthBar(msg.NewHealth, msg.MaxHealth);
        });
    }

    void OnDestroy()
    {
        mSubscription?.Dispose();
    }

    private void UpdateHealthBar(int health, int maxHealth)
    {
        mHealthBar.value = (float)health / maxHealth;
    }
}
```

## 테스트 패턴

### Mock Publisher/Subscriber

```csharp
using NUnit.Framework;
using MessagePipe;
using VContainer;

[TestFixture]
public class PlayerHealthSystemTests
{
    private IObjectResolver mContainer;
    private TestPublisher<PlayerDamagedMessage> mTestPublisher;
    private TestSubscriber<PlayerHealthChangedMessage> mTestSubscriber;

    [SetUp]
    public void SetUp()
    {
        var builder = new ContainerBuilder();

        // MessagePipe 등록
        builder.AddMessagePipe();

        // 시스템 등록
        builder.Register<PlayerHealthSystem>(Lifetime.Transient);

        mContainer = builder.Build();

        // 테스트용 Publisher/Subscriber
        mTestPublisher = new TestPublisher<PlayerDamagedMessage>();
        mTestSubscriber = new TestSubscriber<PlayerHealthChangedMessage>();
    }

    [TearDown]
    public void TearDown()
    {
        mContainer.Dispose();
    }

    [Test]
    public void TakeDamage_ReducesHealth()
    {
        var system = mContainer.Resolve<PlayerHealthSystem>();

        // 메시지 발행
        mTestPublisher.Publish(new PlayerDamagedMessage
        {
            PlayerId = 1,
            Damage = 10f
        });

        // 응답 메시지 확인
        Assert.AreEqual(1, mTestSubscriber.ReceivedMessages.Count);
        Assert.AreEqual(90, mTestSubscriber.ReceivedMessages[0].NewHealth);
    }
}

// 테스트용 Publisher
public class TestPublisher<T> : IPublisher<T>
{
    private readonly List<Action<T>> mSubscribers = new();

    public void Publish(T message)
    {
        foreach (var subscriber in mSubscribers)
        {
            subscriber(message);
        }
    }

    public void Subscribe(Action<T> handler)
    {
        mSubscribers.Add(handler);
    }
}

// 테스트용 Subscriber
public class TestSubscriber<T> : ISubscriber<T>
{
    public List<T> ReceivedMessages { get; } = new();

    public IDisposable Subscribe(Action<T> handler)
    {
        return Subscribe(handler, Array.Empty<MessageHandlerFilter<T>>());
    }

    public IDisposable Subscribe(Action<T> handler, MessageHandlerFilter<T>[] filters)
    {
        return Subscribe(handler, filters, CancellationToken.None);
    }

    public IDisposable Subscribe(Action<T> handler, MessageHandlerFilter<T>[] filters, CancellationToken cancellationToken)
    {
        return new DisposableAction(() =>
        {
            handler += msg => ReceivedMessages.Add(msg);
        });
    }
}
```

## 성능 최적화

### 메시지 풀링

자주 발행되는 메시지의 할당 최소화.

```csharp
// 메시지 풀
public class MessagePool<T> where T : struct, new()
{
    private readonly Stack<T> mPool = new();
    private readonly int mMaxSize;

    public MessagePool(int maxSize = 100)
    {
        mMaxSize = maxSize;
    }

    public T Rent()
    {
        return mPool.Count > 0 ? mPool.Pop() : new T();
    }

    public void Return(T message)
    {
        if (mPool.Count < mMaxSize)
        {
            mPool.Push(message);
        }
    }
}

// 사용 예
public class OptimizedMessageSystem : MonoBehaviour
{
    private readonly MessagePool<EnemyPositionUpdateMessage> mMessagePool = new(200);
    [Inject] private IPublisher<EnemyPositionUpdateMessage> mPositionPublisher;

    public void UpdateEnemyPosition(int enemyId, Vector3 position)
    {
        // 풀에서 메시지 대여
        var msg = mMessagePool.Rent();
        msg.EnemyId = enemyId;
        msg.Position = position;

        mPositionPublisher.Publish(msg);

        // 풀에 반환
        mMessagePool.Return(msg);
    }
}
```

### 배치 메시지 발행

여러 메시지를 한 번에 처리.

```csharp
public struct BatchEnemyUpdateMessage
{
    public List<EnemyUpdate> Updates;
}

public struct EnemyUpdate
{
    public int EnemyId;
    public Vector3 Position;
    public float Health;
}

public class EnemyBatchUpdateSystem : MonoBehaviour
{
    [Inject] private IPublisher<BatchEnemyUpdateMessage> mBatchPublisher;

    private readonly List<EnemyUpdate> mPendingUpdates = new(100);

    void Update()
    {
        // 모든 적 업데이트 수집
        CollectEnemyUpdates(mPendingUpdates);

        if (mPendingUpdates.Count > 0)
        {
            // 한 번에 발행
            mBatchPublisher.Publish(new BatchEnemyUpdateMessage
            {
                Updates = new List<EnemyUpdate>(mPendingUpdates)
            });

            mPendingUpdates.Clear();
        }
    }

    private void CollectEnemyUpdates(List<EnemyUpdate> updates)
    {
        // 적 업데이트 수집 로직
    }
}
```
