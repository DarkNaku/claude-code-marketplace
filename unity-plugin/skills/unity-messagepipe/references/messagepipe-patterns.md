# MessagePipe 모범 사례

Unity에서 고급 MessagePipe 패턴을 위한 심층 참조 문서.

## 글로벌 vs 스코프 메시징

### 글로벌 메시징

애플리케이션 전체에서 메시지를 전달. 게임 전체 이벤트에 사용.

```csharp
using MessagePipe;
using VContainer;
using VContainer.Unity;

// 게임 전체 이벤트
public struct GamePausedMessage { }
public struct PlayerLevelUpMessage
{
    public int PlayerId;
    public int NewLevel;
}

public class RootLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        // 글로벌 메시징 등록
        builder.AddMessagePipe();

        builder.RegisterComponentInHierarchy<PauseManager>();
        builder.RegisterComponentInHierarchy<UIManager>();
    }
}

// Publisher
public class PauseManager : MonoBehaviour
{
    [Inject] private IPublisher<GamePausedMessage> mPausePublisher;

    public void PauseGame()
    {
        Time.timeScale = 0f;
        mPausePublisher.Publish(new GamePausedMessage());
    }
}

// Subscriber
public class UIManager : MonoBehaviour
{
    [Inject] private ISubscriber<GamePausedMessage> mPauseSubscriber;
    private IDisposable mPauseSubscription;

    void Start()
    {
        mPauseSubscription = mPauseSubscriber.Subscribe(_ =>
        {
            ShowPauseMenu();
        });
    }

    void OnDestroy()
    {
        mPauseSubscription?.Dispose();
    }

    private void ShowPauseMenu() { /* UI 표시 */ }
}
```

### 스코프 메시징

특정 생명주기 범위 내에서만 메시지 전달. 씬별/기능별 이벤트에 사용.

```csharp
// 배틀씬 전용 메시지
public struct EnemyDefeatedMessage
{
    public int EnemyId;
    public Vector3 Position;
}

public class BattleSceneLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        // 스코프 메시징 등록 - 이 씬에서만 유효
        builder.AddMessagePipe(options =>
        {
            options.InstanceLifetime = InstanceLifetime.Scoped;
        });

        builder.RegisterComponentInHierarchy<EnemyManager>();
        builder.RegisterComponentInHierarchy<RewardSystem>();
    }
}

// 스코프 Publisher
public class EnemyManager : MonoBehaviour
{
    [Inject] private IPublisher<EnemyDefeatedMessage> mEnemyDefeatedPublisher;

    public void OnEnemyDeath(int enemyId, Vector3 position)
    {
        mEnemyDefeatedPublisher.Publish(new EnemyDefeatedMessage
        {
            EnemyId = enemyId,
            Position = position
        });
    }
}

// 스코프 Subscriber - 배틀씬에서만 작동
public class RewardSystem : MonoBehaviour
{
    [Inject] private ISubscriber<EnemyDefeatedMessage> mEnemyDefeatedSubscriber;
    private IDisposable mSubscription;

    void Start()
    {
        mSubscription = mEnemyDefeatedSubscriber.Subscribe(msg =>
        {
            DropReward(msg.Position);
        });
    }

    void OnDestroy()
    {
        mSubscription?.Dispose();
    }

    private void DropReward(Vector3 position) { /* 보상 드롭 */ }
}
```

## 구독 생명주기 관리

### DisposableBag을 사용한 구독 관리

여러 구독을 한 번에 관리.

```csharp
using System;
using MessagePipe;

public class GameplayUI : MonoBehaviour
{
    [Inject] private ISubscriber<PlayerDamagedMessage> mDamageSubscriber;
    [Inject] private ISubscriber<PlayerHealedMessage> mHealSubscriber;
    [Inject] private ISubscriber<PlayerDiedMessage> mDeathSubscriber;

    private DisposableBag mSubscriptions = new DisposableBag();

    void Start()
    {
        // 여러 구독을 DisposableBag에 추가
        mDamageSubscriber.Subscribe(OnPlayerDamaged).AddTo(mSubscriptions);
        mHealSubscriber.Subscribe(OnPlayerHealed).AddTo(mSubscriptions);
        mDeathSubscriber.Subscribe(OnPlayerDied).AddTo(mSubscriptions);
    }

    void OnDestroy()
    {
        // 한 번에 모든 구독 해제
        mSubscriptions.Dispose();
    }

    private void OnPlayerDamaged(PlayerDamagedMessage msg) { /* 처리 */ }
    private void OnPlayerHealed(PlayerHealedMessage msg) { /* 처리 */ }
    private void OnPlayerDied(PlayerDiedMessage msg) { /* 처리 */ }
}
```

### 조건부 구독

특정 조건에서만 메시지 수신.

```csharp
public class DebugLogger : MonoBehaviour
{
    [Inject] private ISubscriber<GameEventMessage> mEventSubscriber;
    private IDisposable mSubscription;

    [SerializeField] private bool mEnableLogging = true;

    void Start()
    {
        if (mEnableLogging)
        {
            mSubscription = mEventSubscriber.Subscribe(msg =>
            {
                Debug.Log($"이벤트: {msg.EventName}");
            });
        }
    }

    void OnDestroy()
    {
        mSubscription?.Dispose();
    }
}
```

## 비동기 메시징

### 비동기 Publisher/Subscriber

```csharp
using System.Threading;
using Cysharp.Threading.Tasks;
using MessagePipe;

public struct DataLoadMessage
{
    public string DataId;
}

public class DataLoadSystem : MonoBehaviour
{
    [Inject] private IAsyncPublisher<DataLoadMessage> mLoadPublisher;

    public async UniTask LoadDataAsync(string dataId)
    {
        // 비동기 메시지 발행
        await mLoadPublisher.PublishAsync(new DataLoadMessage
        {
            DataId = dataId
        }, CancellationToken.None);
    }
}

public class DataCacheSystem : MonoBehaviour
{
    [Inject] private IAsyncSubscriber<DataLoadMessage> mLoadSubscriber;
    private IDisposable mSubscription;

    void Start()
    {
        mSubscription = mLoadSubscriber.Subscribe(async (msg, ct) =>
        {
            // 비동기 처리
            await LoadAndCacheDataAsync(msg.DataId, ct);
        });
    }

    void OnDestroy()
    {
        mSubscription?.Dispose();
    }

    private async UniTask LoadAndCacheDataAsync(string dataId, CancellationToken ct)
    {
        // 비동기 데이터 로드
        await UniTask.Delay(1000, cancellationToken: ct);
        Debug.Log($"데이터 {dataId} 로드 완료");
    }
}
```

## 필터 체인

### 글로벌 필터

모든 메시지에 적용되는 필터.

```csharp
using MessagePipe;
using UnityEngine;

// 로깅 필터
public class MessageLoggingFilter<T> : MessageHandlerFilter<T>
{
    public override void Handle(T message, Action<T> next)
    {
        Debug.Log($"[MessagePipe] 메시지 발행: {typeof(T).Name}");
        next(message);
        Debug.Log($"[MessagePipe] 메시지 처리 완료: {typeof(T).Name}");
    }
}

// 예외 처리 필터
public class MessageExceptionFilter<T> : MessageHandlerFilter<T>
{
    public override void Handle(T message, Action<T> next)
    {
        try
        {
            next(message);
        }
        catch (Exception ex)
        {
            Debug.LogError($"메시지 처리 중 예외 발생: {ex.Message}");
        }
    }
}

// LifetimeScope에서 필터 등록
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.AddMessagePipe(options =>
        {
            // 글로벌 필터 추가
            options.AddGlobalMessageHandlerFilter(typeof(MessageLoggingFilter<>));
            options.AddGlobalMessageHandlerFilter(typeof(MessageExceptionFilter<>));
        });
    }
}
```

### 메시지별 필터

특정 메시지에만 적용되는 필터.

```csharp
// 특정 메시지용 검증 필터
public class PlayerActionValidationFilter : MessageHandlerFilter<PlayerActionMessage>
{
    private readonly IPlayerAuthService mAuthService;

    public PlayerActionValidationFilter(IPlayerAuthService authService)
    {
        mAuthService = authService;
    }

    public override void Handle(PlayerActionMessage message, Action<PlayerActionMessage> next)
    {
        // 플레이어 권한 검증
        if (mAuthService.IsPlayerAuthorized(message.PlayerId))
        {
            next(message);
        }
        else
        {
            Debug.LogWarning($"플레이어 {message.PlayerId}의 액션이 거부되었습니다");
        }
    }
}

// 등록
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.AddMessagePipe(options =>
        {
            // 특정 메시지에만 필터 적용
            options.AddMessageHandlerFilter<PlayerActionMessage, PlayerActionValidationFilter>();
        });

        builder.Register<IPlayerAuthService, PlayerAuthService>(Lifetime.Singleton);
    }
}
```

## Request/Response 패턴

동기식 요청-응답 패턴.

```csharp
using MessagePipe;

// Request 정의
public struct GetPlayerDataRequest
{
    public int PlayerId;
}

// Response 정의
public struct GetPlayerDataResponse
{
    public string PlayerName;
    public int Level;
    public float Health;
}

// Request Handler
public class PlayerDataRequestHandler : IRequestHandler<GetPlayerDataRequest, GetPlayerDataResponse>
{
    private readonly IPlayerRepository mRepository;

    public PlayerDataRequestHandler(IPlayerRepository repository)
    {
        mRepository = repository;
    }

    public GetPlayerDataResponse Invoke(GetPlayerDataRequest request)
    {
        var data = mRepository.GetPlayerData(request.PlayerId);
        return new GetPlayerDataResponse
        {
            PlayerName = data.Name,
            Level = data.Level,
            Health = data.Health
        };
    }
}

// LifetimeScope 등록
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.AddMessagePipe();

        // Request Handler 등록
        builder.RegisterMessageHandler<GetPlayerDataRequest, GetPlayerDataResponse, PlayerDataRequestHandler>();

        builder.Register<IPlayerRepository, PlayerRepository>(Lifetime.Singleton);
    }
}

// Request 사용
public class UIPlayerInfo : MonoBehaviour
{
    [Inject] private IRequestHandler<GetPlayerDataRequest, GetPlayerDataResponse> mRequestHandler;

    public void ShowPlayerInfo(int playerId)
    {
        // 요청 전송 및 응답 수신
        var response = mRequestHandler.Invoke(new GetPlayerDataRequest
        {
            PlayerId = playerId
        });

        Debug.Log($"플레이어: {response.PlayerName}, 레벨: {response.Level}, 체력: {response.Health}");
        UpdateUI(response);
    }

    private void UpdateUI(GetPlayerDataResponse data) { /* UI 업데이트 */ }
}
```

### 비동기 Request/Response

```csharp
using System.Threading;
using Cysharp.Threading.Tasks;
using MessagePipe;

// 비동기 Request Handler
public class AsyncPlayerDataRequestHandler : IAsyncRequestHandler<GetPlayerDataRequest, GetPlayerDataResponse>
{
    private readonly IPlayerRepository mRepository;

    public AsyncPlayerDataRequestHandler(IPlayerRepository repository)
    {
        mRepository = repository;
    }

    public async UniTask<GetPlayerDataResponse> InvokeAsync(GetPlayerDataRequest request, CancellationToken cancellationToken)
    {
        // 비동기 데이터 로드
        var data = await mRepository.GetPlayerDataAsync(request.PlayerId, cancellationToken);

        return new GetPlayerDataResponse
        {
            PlayerName = data.Name,
            Level = data.Level,
            Health = data.Health
        };
    }
}

// 비동기 Request 사용
public class UIPlayerInfo : MonoBehaviour
{
    [Inject] private IAsyncRequestHandler<GetPlayerDataRequest, GetPlayerDataResponse> mAsyncRequestHandler;

    public async UniTask ShowPlayerInfoAsync(int playerId)
    {
        // 비동기 요청
        var response = await mAsyncRequestHandler.InvokeAsync(
            new GetPlayerDataRequest { PlayerId = playerId },
            this.GetCancellationTokenOnDestroy()
        );

        Debug.Log($"플레이어: {response.PlayerName}");
        UpdateUI(response);
    }

    private void UpdateUI(GetPlayerDataResponse data) { /* UI 업데이트 */ }
}
```

## 버퍼링 메시지

구독하기 전에 발행된 메시지를 버퍼링.

```csharp
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.AddMessagePipe(options =>
        {
            // 버퍼링 활성화 - 최대 10개 메시지 저장
            options.EnableCaptureStackTrace = false;
            options.DefaultInstanceLifetime = InstanceLifetime.Singleton;
        });

        // 버퍼링 Publisher 등록
        builder.RegisterMessageBroker<GameStartedMessage>(options =>
        {
            options.EnableCaptureStackTrace = false;
        });
    }
}

// 버퍼링된 메시지는 늦게 구독해도 수신 가능
public class LateSubscriber : MonoBehaviour
{
    [Inject] private ISubscriber<GameStartedMessage> mGameStartSubscriber;

    async void Start()
    {
        // 5초 대기
        await UniTask.Delay(5000);

        // 늦게 구독해도 버퍼링된 메시지 수신
        mGameStartSubscriber.Subscribe(msg =>
        {
            Debug.Log("게임 시작 메시지를 늦게 받았습니다!");
        });
    }
}
```

## 메시지 설계 모범 사례

### 구조체 사용

클래스 대신 구조체 사용으로 GC 압력 감소.

```csharp
// 좋은 예: 구조체
public struct EnemySpawnedMessage
{
    public int EnemyId;
    public Vector3 Position;
    public EnemyType Type;
}

// 나쁜 예: 클래스 (불필요한 할당)
public class EnemySpawnedMessage
{
    public int EnemyId { get; set; }
    public Vector3 Position { get; set; }
    public EnemyType Type { get; set; }
}
```

### 의미 있는 메시지 이름

```csharp
// 좋은 예: 명확한 의미
public struct PlayerHealthChangedMessage
{
    public int PlayerId;
    public float OldHealth;
    public float NewHealth;
}

// 나쁜 예: 모호한 의미
public struct PlayerMessage
{
    public int Id;
    public float Value1;
    public float Value2;
}
```

### 메시지 그룹화

관련된 메시지를 네임스페이스로 그룹화.

```csharp
namespace Game.Messages.Player
{
    public struct DamagedMessage { /* ... */ }
    public struct HealedMessage { /* ... */ }
    public struct DiedMessage { /* ... */ }
    public struct RespawnedMessage { /* ... */ }
}

namespace Game.Messages.Enemy
{
    public struct SpawnedMessage { /* ... */ }
    public struct DefeatedMessage { /* ... */ }
    public struct AggroChangedMessage { /* ... */ }
}
```
