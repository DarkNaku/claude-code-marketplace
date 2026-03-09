# NSubstitute 통합 패턴

NSubstitute를 Unity 및 다른 라이브러리와 통합하는 고급 패턴.

## VContainer 테스트 통합

### 기본 DI 컨테이너 테스트

```csharp
using NUnit.Framework;
using NSubstitute;
using VContainer;
using VContainer.Unity;

public interface IPlayerService
{
    int GetPlayerScore(int playerId);
}

public class ScoreManager
{
    private readonly IPlayerService mPlayerService;

    public ScoreManager(IPlayerService playerService)
    {
        mPlayerService = playerService;
    }

    public int GetTotalScore(int[] playerIds)
    {
        var total = 0;
        foreach (var id in playerIds)
        {
            total += mPlayerService.GetPlayerScore(id);
        }
        return total;
    }
}

[TestFixture]
public class VContainerTests
{
    private IObjectResolver mContainer;

    [SetUp]
    public void SetUp()
    {
        var builder = new ContainerBuilder();

        // Mock 서비스 등록
        var mockPlayerService = Substitute.For<IPlayerService>();
        mockPlayerService.GetPlayerScore(Arg.Any<int>()).Returns(100);

        builder.RegisterInstance(mockPlayerService).As<IPlayerService>();
        builder.Register<ScoreManager>(Lifetime.Transient);

        mContainer = builder.Build();
    }

    [TearDown]
    public void TearDown()
    {
        mContainer?.Dispose();
    }

    [Test]
    public void ScoreManager_CalculatesTotalScore()
    {
        // Arrange
        var manager = mContainer.Resolve<ScoreManager>();

        // Act
        var total = manager.GetTotalScore(new[] { 1, 2, 3 });

        // Assert
        Assert.AreEqual(300, total);
    }
}
```

### 복잡한 의존성 그래프 테스트

```csharp
public interface IDataRepository
{
    string LoadData(string key);
}

public interface ICacheService
{
    bool TryGetValue(string key, out string value);
    void SetValue(string key, string value);
}

public class DataManager
{
    private readonly IDataRepository mRepository;
    private readonly ICacheService mCache;

    public DataManager(IDataRepository repository, ICacheService cache)
    {
        mRepository = repository;
        mCache = cache;
    }

    public string GetData(string key)
    {
        // 캐시 확인
        if (mCache.TryGetValue(key, out var cachedValue))
        {
            return cachedValue;
        }

        // 캐시 미스: 저장소에서 로드
        var value = mRepository.LoadData(key);
        mCache.SetValue(key, value);
        return value;
    }
}

[TestFixture]
public class DataManagerTests
{
    private IObjectResolver mContainer;
    private IDataRepository mMockRepository;
    private ICacheService mMockCache;

    [SetUp]
    public void SetUp()
    {
        var builder = new ContainerBuilder();

        // Mock 생성
        mMockRepository = Substitute.For<IDataRepository>();
        mMockCache = Substitute.For<ICacheService>();

        // 컨테이너에 등록
        builder.RegisterInstance(mMockRepository).As<IDataRepository>();
        builder.RegisterInstance(mMockCache).As<ICacheService>();
        builder.Register<DataManager>(Lifetime.Transient);

        mContainer = builder.Build();
    }

    [TearDown]
    public void TearDown()
    {
        mContainer?.Dispose();
    }

    [Test]
    public void GetData_WhenCacheHit_DoesNotAccessRepository()
    {
        // Arrange
        var manager = mContainer.Resolve<DataManager>();

        string outValue = "cached_data";
        mMockCache.TryGetValue("key1", out outValue).Returns(x =>
        {
            x[1] = "cached_data";
            return true;
        });

        // Act
        var result = manager.GetData("key1");

        // Assert
        Assert.AreEqual("cached_data", result);
        mMockRepository.DidNotReceive().LoadData(Arg.Any<string>());
    }

    [Test]
    public void GetData_WhenCacheMiss_LoadsFromRepository()
    {
        // Arrange
        var manager = mContainer.Resolve<DataManager>();

        string outValue;
        mMockCache.TryGetValue("key1", out outValue).Returns(false);
        mMockRepository.LoadData("key1").Returns("repository_data");

        // Act
        var result = manager.GetData("key1");

        // Assert
        Assert.AreEqual("repository_data", result);
        mMockRepository.Received(1).LoadData("key1");
        mMockCache.Received(1).SetValue("key1", "repository_data");
    }
}
```

## UniTask 비동기 테스트

### 기본 비동기 메서드 테스트

```csharp
using Cysharp.Threading.Tasks;
using System.Threading;

public interface IAsyncDataService
{
    UniTask<string> LoadDataAsync(string key, CancellationToken ct);
    UniTask SaveDataAsync(string key, string value, CancellationToken ct);
}

public class AsyncDataManager
{
    private readonly IAsyncDataService mDataService;

    public AsyncDataManager(IAsyncDataService dataService)
    {
        mDataService = dataService;
    }

    public async UniTask<string> ProcessDataAsync(string key, CancellationToken ct)
    {
        var data = await mDataService.LoadDataAsync(key, ct);
        return data.ToUpper();
    }
}

[TestFixture]
public class AsyncDataManagerTests
{
    [Test]
    public async Task ProcessDataAsync_TransformsData()
    {
        // Arrange
        var mockService = Substitute.For<IAsyncDataService>();
        mockService.LoadDataAsync("key1", Arg.Any<CancellationToken>())
            .Returns(UniTask.FromResult("test data"));

        var manager = new AsyncDataManager(mockService);

        // Act
        var result = await manager.ProcessDataAsync("key1", CancellationToken.None);

        // Assert
        Assert.AreEqual("TEST DATA", result);
    }
}
```

### 비동기 예외 테스트

```csharp
[TestFixture]
public class AsyncExceptionTests
{
    [Test]
    public void AsyncMethod_ThrowsException()
    {
        // Arrange
        var mockService = Substitute.For<IAsyncDataService>();
        mockService.LoadDataAsync(Arg.Any<string>(), Arg.Any<CancellationToken>())
            .Returns<UniTask<string>>(x => throw new System.Exception("Load failed"));

        var manager = new AsyncDataManager(mockService);

        // Act & Assert
        Assert.ThrowsAsync<System.Exception>(async () =>
        {
            await manager.ProcessDataAsync("key1", CancellationToken.None);
        });
    }
}
```

### 취소 토큰 동작 테스트

```csharp
[TestFixture]
public class CancellationTests
{
    [Test]
    public async Task AsyncMethod_RespectsCancellation()
    {
        // Arrange
        var mockService = Substitute.For<IAsyncDataService>();
        var cts = new CancellationTokenSource();

        mockService.LoadDataAsync(Arg.Any<string>(), Arg.Any<CancellationToken>())
            .Returns(async callInfo =>
            {
                var ct = callInfo.Arg<CancellationToken>();
                await UniTask.Delay(100, cancellationToken: ct);
                return "data";
            });

        var manager = new AsyncDataManager(mockService);

        // Act
        cts.Cancel();

        // Assert
        Assert.ThrowsAsync<OperationCanceledException>(async () =>
        {
            await manager.ProcessDataAsync("key1", cts.Token);
        });
    }
}
```

## MonoBehaviour 테스트

### MonoBehaviour 의존성 주입 테스트

```csharp
using UnityEngine;

public class PlayerController : MonoBehaviour
{
    private IPlayerService mPlayerService;
    private IInputService mInputService;

    public void Initialize(IPlayerService playerService, IInputService inputService)
    {
        mPlayerService = playerService;
        mInputService = inputService;
    }

    public void ProcessInput()
    {
        if (mInputService.IsJumpPressed())
        {
            mPlayerService.Jump();
        }
    }
}

[TestFixture]
public class PlayerControllerTests
{
    [Test]
    public void ProcessInput_WhenJumpPressed_CallsJump()
    {
        // Arrange
        var mockPlayerService = Substitute.For<IPlayerService>();
        var mockInputService = Substitute.For<IInputService>();

        mockInputService.IsJumpPressed().Returns(true);

        var gameObject = new GameObject();
        var controller = gameObject.AddComponent<PlayerController>();
        controller.Initialize(mockPlayerService, mockInputService);

        // Act
        controller.ProcessInput();

        // Assert
        mockPlayerService.Received(1).Jump();

        // Cleanup
        Object.DestroyImmediate(gameObject);
    }
}
```

### UnityEvent 테스트

```csharp
using UnityEngine.Events;

public interface IEventService
{
    UnityEvent<int> OnScoreChanged { get; }
}

public class ScoreDisplay : MonoBehaviour
{
    private IEventService mEventService;
    private int mCurrentScore;

    public void Initialize(IEventService eventService)
    {
        mEventService = eventService;
        mEventService.OnScoreChanged.AddListener(OnScoreChanged);
    }

    private void OnScoreChanged(int newScore)
    {
        mCurrentScore = newScore;
    }

    public int GetCurrentScore() => mCurrentScore;
}

[TestFixture]
public class ScoreDisplayTests
{
    [Test]
    public void OnScoreChanged_UpdatesDisplay()
    {
        // Arrange
        var mockEventService = Substitute.For<IEventService>();
        var scoreEvent = new UnityEvent<int>();
        mockEventService.OnScoreChanged.Returns(scoreEvent);

        var gameObject = new GameObject();
        var display = gameObject.AddComponent<ScoreDisplay>();
        display.Initialize(mockEventService);

        // Act
        scoreEvent.Invoke(100);

        // Assert
        Assert.AreEqual(100, display.GetCurrentScore());

        // Cleanup
        Object.DestroyImmediate(gameObject);
    }
}
```

## MessagePipe 통합 테스트

### Publisher/Subscriber 모킹

```csharp
using MessagePipe;

public struct PlayerDamagedMessage
{
    public int PlayerId;
    public float Damage;
}

public class DamageHandler
{
    private readonly IPublisher<PlayerDamagedMessage> mPublisher;

    public DamageHandler(IPublisher<PlayerDamagedMessage> publisher)
    {
        mPublisher = publisher;
    }

    public void ApplyDamage(int playerId, float damage)
    {
        mPublisher.Publish(new PlayerDamagedMessage
        {
            PlayerId = playerId,
            Damage = damage
        });
    }
}

[TestFixture]
public class MessagePipeTests
{
    [Test]
    public void ApplyDamage_PublishesMessage()
    {
        // Arrange
        var mockPublisher = Substitute.For<IPublisher<PlayerDamagedMessage>>();
        var handler = new DamageHandler(mockPublisher);

        // Act
        handler.ApplyDamage(1, 50f);

        // Assert
        mockPublisher.Received(1).Publish(Arg.Is<PlayerDamagedMessage>(msg =>
            msg.PlayerId == 1 && msg.Damage == 50f
        ));
    }
}
```

### Subscriber 동작 테스트

```csharp
public class HealthDisplay
{
    private readonly ISubscriber<PlayerDamagedMessage> mSubscriber;
    private IDisposable mSubscription;
    private float mDisplayedHealth = 100f;

    public HealthDisplay(ISubscriber<PlayerDamagedMessage> subscriber)
    {
        mSubscriber = subscriber;
    }

    public void Start()
    {
        mSubscription = mSubscriber.Subscribe(msg =>
        {
            mDisplayedHealth -= msg.Damage;
        });
    }

    public void Stop()
    {
        mSubscription?.Dispose();
    }

    public float GetDisplayedHealth() => mDisplayedHealth;
}

[TestFixture]
public class HealthDisplayTests
{
    [Test]
    public void Subscribe_UpdatesHealthOnMessage()
    {
        // Arrange
        var mockSubscriber = Substitute.For<ISubscriber<PlayerDamagedMessage>>();
        System.Action<PlayerDamagedMessage> capturedCallback = null;

        mockSubscriber.Subscribe(Arg.Do<System.Action<PlayerDamagedMessage>>(
            callback => capturedCallback = callback
        ));

        var display = new HealthDisplay(mockSubscriber);
        display.Start();

        // Act - 수동으로 콜백 실행
        capturedCallback?.Invoke(new PlayerDamagedMessage
        {
            PlayerId = 1,
            Damage = 30f
        });

        // Assert
        Assert.AreEqual(70f, display.GetDisplayedHealth());

        // Cleanup
        display.Stop();
    }
}
```

## 통합 테스트 패턴

### 멀티레이어 통합 테스트

```csharp
// Domain Layer
public class Player
{
    public int Id { get; set; }
    public string Name { get; set; }
    public int Health { get; set; }
}

// Repository Layer
public interface IPlayerRepository
{
    Player GetPlayer(int id);
    void SavePlayer(Player player);
}

// Service Layer
public class PlayerService
{
    private readonly IPlayerRepository mRepository;

    public PlayerService(IPlayerRepository repository)
    {
        mRepository = repository;
    }

    public void HealPlayer(int playerId, int amount)
    {
        var player = mRepository.GetPlayer(playerId);
        player.Health += amount;
        mRepository.SavePlayer(player);
    }
}

[TestFixture]
public class IntegrationTests
{
    [Test]
    public void HealPlayer_LoadsAndSavesCorrectly()
    {
        // Arrange
        var mockRepository = Substitute.For<IPlayerRepository>();

        var player = new Player { Id = 1, Name = "TestPlayer", Health = 50 };
        mockRepository.GetPlayer(1).Returns(player);

        var service = new PlayerService(mockRepository);

        // Act
        service.HealPlayer(1, 30);

        // Assert
        mockRepository.Received(1).GetPlayer(1);
        mockRepository.Received(1).SavePlayer(Arg.Is<Player>(p =>
            p.Id == 1 && p.Health == 80
        ));
    }
}
```

### 시나리오 기반 테스트

```csharp
public interface IGameStateManager
{
    void StartGame();
    void PauseGame();
    void ResumeGame();
    void EndGame();
    string GetCurrentState();
}

public class GameController
{
    private readonly IGameStateManager mStateManager;

    public GameController(IGameStateManager stateManager)
    {
        mStateManager = stateManager;
    }

    public void PlaySession()
    {
        mStateManager.StartGame();
        // 게임 플레이
        mStateManager.PauseGame();
        // 일시정지
        mStateManager.ResumeGame();
        // 재개
        mStateManager.EndGame();
    }
}

[TestFixture]
public class GameControllerScenarioTests
{
    [Test]
    public void PlaySession_ExecutesCorrectSequence()
    {
        // Arrange
        var mockStateManager = Substitute.For<IGameStateManager>();
        var controller = new GameController(mockStateManager);

        // Act
        controller.PlaySession();

        // Assert - 올바른 순서로 호출되었는지 검증
        Received.InOrder(() =>
        {
            mockStateManager.StartGame();
            mockStateManager.PauseGame();
            mockStateManager.ResumeGame();
            mockStateManager.EndGame();
        });
    }
}
```

## 테스트 헬퍼 및 유틸리티

### 커스텀 Arg 매처

```csharp
public static class CustomArgMatchers
{
    public static Player IsPlayerWithHealth(int minHealth)
    {
        return Arg.Is<Player>(p => p.Health >= minHealth);
    }

    public static string ContainsSubstring(string substring)
    {
        return Arg.Is<string>(s => s.Contains(substring));
    }
}

[TestFixture]
public class CustomMatcherTests
{
    [Test]
    public void CustomMatcher_Example()
    {
        // Arrange
        var mockService = Substitute.For<IPlayerService>();

        // Act
        mockService.SavePlayer(new Player { Id = 1, Health = 80 });

        // Assert
        mockService.Received().SavePlayer(CustomArgMatchers.IsPlayerWithHealth(50));
    }
}
```

### 테스트 데이터 빌더

```csharp
public class PlayerBuilder
{
    private int mId = 1;
    private string mName = "TestPlayer";
    private int mHealth = 100;

    public PlayerBuilder WithId(int id)
    {
        mId = id;
        return this;
    }

    public PlayerBuilder WithName(string name)
    {
        mName = name;
        return this;
    }

    public PlayerBuilder WithHealth(int health)
    {
        mHealth = health;
        return this;
    }

    public Player Build()
    {
        return new Player
        {
            Id = mId,
            Name = mName,
            Health = mHealth
        };
    }
}

[TestFixture]
public class BuilderPatternTests
{
    [Test]
    public void Builder_CreatesTestData()
    {
        // Arrange
        var player = new PlayerBuilder()
            .WithId(42)
            .WithName("Hero")
            .WithHealth(150)
            .Build();

        var mockService = Substitute.For<IPlayerService>();

        // Act
        mockService.SavePlayer(player);

        // Assert
        mockService.Received().SavePlayer(Arg.Is<Player>(p =>
            p.Id == 42 && p.Name == "Hero" && p.Health == 150
        ));
    }
}
```

### Mock 팩토리

```csharp
public static class MockFactory
{
    public static IPlayerService CreatePlayerService(int defaultHealth = 100)
    {
        var mock = Substitute.For<IPlayerService>();
        mock.GetPlayerHealth(Arg.Any<int>()).Returns(defaultHealth);
        return mock;
    }

    public static IPlayerRepository CreatePlayerRepository()
    {
        var mock = Substitute.For<IPlayerRepository>();
        mock.GetPlayer(Arg.Any<int>()).Returns(callInfo =>
        {
            var id = callInfo.Arg<int>();
            return new Player { Id = id, Name = $"Player{id}", Health = 100 };
        });
        return mock;
    }
}

[TestFixture]
public class MockFactoryTests
{
    [Test]
    public void MockFactory_CreatesConfiguredMocks()
    {
        // Arrange
        var mockService = MockFactory.CreatePlayerService(150);

        // Act
        var health = mockService.GetPlayerHealth(1);

        // Assert
        Assert.AreEqual(150, health);
    }
}
```

## 성능 테스트

### 반복 호출 테스트

```csharp
[TestFixture]
public class PerformanceTests
{
    [Test]
    public void Service_HandlesMultipleCalls()
    {
        // Arrange
        var mockService = Substitute.For<IPlayerService>();
        mockService.GetPlayerHealth(Arg.Any<int>()).Returns(100);

        // Act
        for (int i = 0; i < 1000; i++)
        {
            mockService.GetPlayerHealth(i);
        }

        // Assert
        mockService.Received(1000).GetPlayerHealth(Arg.Any<int>());
    }
}
```

### 메모리 누수 테스트

```csharp
[TestFixture]
public class MemoryTests
{
    [Test]
    public void Subscriptions_AreDisposedCorrectly()
    {
        // Arrange
        var mockSubscriber = Substitute.For<ISubscriber<PlayerDamagedMessage>>();
        var disposable = Substitute.For<IDisposable>();
        mockSubscriber.Subscribe(Arg.Any<System.Action<PlayerDamagedMessage>>())
            .Returns(disposable);

        var display = new HealthDisplay(mockSubscriber);

        // Act
        display.Start();
        display.Stop();

        // Assert
        disposable.Received(1).Dispose();
    }
}
```

## 에러 케이스 테스트

### Null 처리 테스트

```csharp
[TestFixture]
public class NullHandlingTests
{
    [Test]
    public void Service_HandlesNullArguments()
    {
        // Arrange
        var mockService = Substitute.For<IPlayerService>();

        // Act & Assert
        Assert.Throws<System.ArgumentNullException>(() =>
        {
            mockService.SavePlayerData(1, null);
        });
    }
}
```

### 경계값 테스트

```csharp
[TestFixture]
public class BoundaryTests
{
    [Test]
    public void Service_HandlesMaxValue()
    {
        // Arrange
        var mockService = Substitute.For<IPlayerService>();
        mockService.GetPlayerHealth(Arg.Any<int>()).Returns(int.MaxValue);

        // Act
        var health = mockService.GetPlayerHealth(1);

        // Assert
        Assert.AreEqual(int.MaxValue, health);
    }

    [Test]
    public void Service_HandlesMinValue()
    {
        // Arrange
        var mockService = Substitute.For<IPlayerService>();
        mockService.GetPlayerHealth(Arg.Any<int>()).Returns(int.MinValue);

        // Act
        var health = mockService.GetPlayerHealth(1);

        // Assert
        Assert.AreEqual(int.MinValue, health);
    }
}
```
