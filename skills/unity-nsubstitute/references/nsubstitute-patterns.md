# NSubstitute 모범 사례

Unity에서 고급 NSubstitute 모킹 패턴을 위한 심층 참조 문서.

## 기본 모킹

### 인터페이스 모킹

```csharp
using NUnit.Framework;
using NSubstitute;

public interface IWeaponService
{
    int GetDamage(string weaponId);
    bool IsWeaponAvailable(string weaponId);
    void EquipWeapon(int playerId, string weaponId);
}

[TestFixture]
public class WeaponSystemTests
{
    private IWeaponService mMockWeaponService;

    [SetUp]
    public void SetUp()
    {
        // 인터페이스 모킹 생성
        mMockWeaponService = Substitute.For<IWeaponService>();
    }

    [Test]
    public void GetDamage_ReturnsConfiguredValue()
    {
        // Arrange - 반환값 설정
        mMockWeaponService.GetDamage("sword").Returns(50);

        // Act
        var damage = mMockWeaponService.GetDamage("sword");

        // Assert
        Assert.AreEqual(50, damage);
    }
}
```

### 추상 클래스 모킹

```csharp
public abstract class EnemyBase
{
    public abstract int GetHealth();
    public abstract void TakeDamage(int damage);

    public bool IsAlive()
    {
        return GetHealth() > 0;
    }
}

[TestFixture]
public class EnemyTests
{
    [Test]
    public void IsAlive_WhenHealthPositive_ReturnsTrue()
    {
        // Arrange - 추상 클래스 모킹
        var mockEnemy = Substitute.For<EnemyBase>();
        mockEnemy.GetHealth().Returns(100);

        // Act
        var isAlive = mockEnemy.IsAlive();

        // Assert
        Assert.IsTrue(isAlive);
    }
}
```

### 여러 인터페이스 동시 모킹

```csharp
public interface IDamageable
{
    void TakeDamage(int damage);
}

public interface IHealable
{
    void Heal(int amount);
}

[TestFixture]
public class MultiInterfaceTests
{
    [Test]
    public void CanMockMultipleInterfaces()
    {
        // Arrange - 여러 인터페이스를 구현하는 모킹
        var mock = Substitute.For<IDamageable, IHealable>();

        // Act & Assert
        mock.TakeDamage(10);
        mock.Received(1).TakeDamage(10);

        ((IHealable)mock).Heal(5);
        ((IHealable)mock).Received(1).Heal(5);
    }
}
```

## Returns 패턴

### 단일 반환값

```csharp
[TestFixture]
public class ReturnsSimpleTests
{
    [Test]
    public void Returns_SingleValue()
    {
        // Arrange
        var mockService = Substitute.For<IPlayerService>();

        // 특정 인자에 대한 반환값 설정
        mockService.GetPlayerHealth(1).Returns(100);
        mockService.GetPlayerHealth(2).Returns(50);

        // Act & Assert
        Assert.AreEqual(100, mockService.GetPlayerHealth(1));
        Assert.AreEqual(50, mockService.GetPlayerHealth(2));
        Assert.AreEqual(0, mockService.GetPlayerHealth(3)); // 기본값
    }
}
```

### 시퀀스 반환값

```csharp
[TestFixture]
public class ReturnsSequenceTests
{
    [Test]
    public void Returns_Sequence()
    {
        // Arrange
        var mockService = Substitute.For<IRandomService>();

        // 순차적으로 다른 값 반환
        mockService.GetRandomNumber().Returns(1, 2, 3, 4);

        // Act & Assert
        Assert.AreEqual(1, mockService.GetRandomNumber());
        Assert.AreEqual(2, mockService.GetRandomNumber());
        Assert.AreEqual(3, mockService.GetRandomNumber());
        Assert.AreEqual(4, mockService.GetRandomNumber());
        Assert.AreEqual(4, mockService.GetRandomNumber()); // 마지막 값 반복
    }
}
```

### 콜백을 사용한 동적 반환값

```csharp
[TestFixture]
public class ReturnsCallbackTests
{
    [Test]
    public void Returns_WithCallback()
    {
        // Arrange
        var mockCalculator = Substitute.For<ICalculator>();

        // 인자를 기반으로 동적 반환값 생성
        mockCalculator.Add(Arg.Any<int>(), Arg.Any<int>())
            .Returns(callInfo =>
            {
                var a = callInfo.ArgAt<int>(0);
                var b = callInfo.ArgAt<int>(1);
                return a + b;
            });

        // Act & Assert
        Assert.AreEqual(5, mockCalculator.Add(2, 3));
        Assert.AreEqual(10, mockCalculator.Add(7, 3));
    }
}
```

### ReturnsForAnyArgs

모든 인자에 대해 동일한 값 반환.

```csharp
[TestFixture]
public class ReturnsForAnyArgsTests
{
    [Test]
    public void ReturnsForAnyArgs_Example()
    {
        // Arrange
        var mockService = Substitute.For<IPlayerService>();

        // 어떤 인자가 와도 동일한 값 반환
        mockService.GetPlayerHealth(Arg.Any<int>()).ReturnsForAnyArgs(100);

        // Act & Assert
        Assert.AreEqual(100, mockService.GetPlayerHealth(1));
        Assert.AreEqual(100, mockService.GetPlayerHealth(999));
        Assert.AreEqual(100, mockService.GetPlayerHealth(-1));
    }
}
```

## Received 검증

### 기본 호출 검증

```csharp
[TestFixture]
public class ReceivedBasicTests
{
    [Test]
    public void Received_ExactlyOnce()
    {
        // Arrange
        var mockService = Substitute.For<IPlayerService>();

        // Act
        mockService.SavePlayerData(1, "data");

        // Assert - 정확히 1번 호출 검증
        mockService.Received(1).SavePlayerData(1, "data");
    }

    [Test]
    public void Received_Multiple()
    {
        // Arrange
        var mockService = Substitute.For<IPlayerService>();

        // Act
        mockService.SavePlayerData(1, "data");
        mockService.SavePlayerData(1, "data");
        mockService.SavePlayerData(1, "data");

        // Assert - 정확히 3번 호출 검증
        mockService.Received(3).SavePlayerData(1, "data");
    }

    [Test]
    public void DidNotReceive_Example()
    {
        // Arrange
        var mockService = Substitute.For<IPlayerService>();

        // Act - 아무것도 호출하지 않음

        // Assert - 호출되지 않았음을 검증
        mockService.DidNotReceive().SavePlayerData(Arg.Any<int>(), Arg.Any<string>());
    }
}
```

### Received 변형

```csharp
[TestFixture]
public class ReceivedVariantsTests
{
    [Test]
    public void Received_WithoutArguments_MeansExactlyOnce()
    {
        // Arrange
        var mockService = Substitute.For<IPlayerService>();

        // Act
        mockService.SavePlayerData(1, "data");

        // Assert - Received()는 Received(1)과 동일
        mockService.Received().SavePlayerData(1, "data");
    }

    [Test]
    public void ReceivedWithAnyArgs()
    {
        // Arrange
        var mockService = Substitute.For<IPlayerService>();

        // Act
        mockService.SavePlayerData(1, "data");

        // Assert - 어떤 인자로든 호출되었는지 검증
        mockService.ReceivedWithAnyArgs().SavePlayerData(default, default);
    }

    [Test]
    public void DidNotReceiveWithAnyArgs()
    {
        // Arrange
        var mockService = Substitute.For<IPlayerService>();

        // Act - 호출하지 않음

        // Assert
        mockService.DidNotReceiveWithAnyArgs().SavePlayerData(default, default);
    }
}
```

### 호출 순서 검증

```csharp
[TestFixture]
public class CallOrderTests
{
    [Test]
    public void Received_InOrder()
    {
        // Arrange
        var mockService = Substitute.For<IGameService>();

        // Act
        mockService.Initialize();
        mockService.Start();
        mockService.Stop();

        // Assert - 호출 순서 검증
        Received.InOrder(() =>
        {
            mockService.Initialize();
            mockService.Start();
            mockService.Stop();
        });
    }
}
```

## Arg 매칭

### Arg.Any - 모든 값 매칭

```csharp
[TestFixture]
public class ArgAnyTests
{
    [Test]
    public void ArgAny_Example()
    {
        // Arrange
        var mockService = Substitute.For<IPlayerService>();
        mockService.GetPlayerHealth(Arg.Any<int>()).Returns(100);

        // Act & Assert - 어떤 int 값이든 100 반환
        Assert.AreEqual(100, mockService.GetPlayerHealth(1));
        Assert.AreEqual(100, mockService.GetPlayerHealth(999));
    }

    [Test]
    public void ArgAny_WithReceived()
    {
        // Arrange
        var mockService = Substitute.For<IPlayerService>();

        // Act
        mockService.SavePlayerData(42, "test_data");

        // Assert - 첫 번째 인자는 상관없이, 두 번째는 "test_data"
        mockService.Received().SavePlayerData(Arg.Any<int>(), "test_data");
    }
}
```

### Arg.Is - 조건 매칭

```csharp
[TestFixture]
public class ArgIsTests
{
    [Test]
    public void ArgIs_WithPredicate()
    {
        // Arrange
        var mockService = Substitute.For<IPlayerService>();

        // 양수 ID에 대해서만 100 반환
        mockService.GetPlayerHealth(Arg.Is<int>(id => id > 0)).Returns(100);

        // Act & Assert
        Assert.AreEqual(100, mockService.GetPlayerHealth(1));
        Assert.AreEqual(100, mockService.GetPlayerHealth(999));
        Assert.AreEqual(0, mockService.GetPlayerHealth(-1)); // 조건 불만족
    }

    [Test]
    public void ArgIs_WithReceived()
    {
        // Arrange
        var mockService = Substitute.For<IPlayerService>();

        // Act
        mockService.SavePlayerData(100, "data");

        // Assert - 100 이상의 ID로 호출되었는지 검증
        mockService.Received().SavePlayerData(
            Arg.Is<int>(id => id >= 100),
            Arg.Any<string>()
        );
    }
}
```

### Arg.Do - 인자로 액션 수행

```csharp
[TestFixture]
public class ArgDoTests
{
    [Test]
    public void ArgDo_CapturesArgument()
    {
        // Arrange
        var mockService = Substitute.For<IPlayerService>();
        var capturedData = string.Empty;

        // Act
        mockService.SavePlayerData(
            Arg.Any<int>(),
            Arg.Do<string>(data => capturedData = data)
        );

        mockService.SavePlayerData(1, "captured_data");

        // Assert - 인자가 캡처되었는지 확인
        Assert.AreEqual("captured_data", capturedData);
    }

    [Test]
    public void ArgDo_WithMultipleCaptures()
    {
        // Arrange
        var mockLogger = Substitute.For<ILogger>();
        var loggedMessages = new List<string>();

        mockLogger.Log(Arg.Do<string>(msg => loggedMessages.Add(msg)));

        // Act
        mockLogger.Log("Message 1");
        mockLogger.Log("Message 2");
        mockLogger.Log("Message 3");

        // Assert
        Assert.AreEqual(3, loggedMessages.Count);
        Assert.Contains("Message 1", loggedMessages);
        Assert.Contains("Message 2", loggedMessages);
        Assert.Contains("Message 3", loggedMessages);
    }
}
```

### Arg.Invoke - 콜백 실행

```csharp
public interface IAsyncService
{
    void LoadDataAsync(System.Action<string> onComplete);
}

[TestFixture]
public class ArgInvokeTests
{
    [Test]
    public void ArgInvoke_ExecutesCallback()
    {
        // Arrange
        var mockService = Substitute.For<IAsyncService>();
        var result = string.Empty;

        // 콜백이 호출되면 "loaded_data" 전달
        mockService.LoadDataAsync(Arg.Invoke("loaded_data"));

        // Act
        mockService.LoadDataAsync(data => result = data);

        // Assert
        Assert.AreEqual("loaded_data", result);
    }
}
```

## 예외 모킹

### Throws를 사용한 예외 발생

```csharp
[TestFixture]
public class ThrowsTests
{
    [Test]
    public void Throws_Exception()
    {
        // Arrange
        var mockService = Substitute.For<IPlayerService>();

        // 특정 조건에서 예외 발생 설정
        mockService
            .When(x => x.SavePlayerData(-1, Arg.Any<string>()))
            .Do(x => throw new System.ArgumentException("Invalid player ID"));

        // Act & Assert
        Assert.Throws<System.ArgumentException>(() =>
        {
            mockService.SavePlayerData(-1, "data");
        });
    }

    [Test]
    public void Throws_ForAnyCall()
    {
        // Arrange
        var mockService = Substitute.For<INetworkService>();

        // 모든 호출에서 예외 발생
        mockService.Connect().Returns(x => throw new System.Exception("Network error"));

        // Act & Assert
        Assert.Throws<System.Exception>(() =>
        {
            mockService.Connect();
        });
    }
}
```

## 프로퍼티 모킹

### 프로퍼티 읽기/쓰기

```csharp
public interface IGameConfig
{
    int MaxPlayers { get; set; }
    string GameMode { get; set; }
}

[TestFixture]
public class PropertyTests
{
    [Test]
    public void Property_GetAndSet()
    {
        // Arrange
        var mockConfig = Substitute.For<IGameConfig>();

        // Act - 프로퍼티는 자동으로 동작
        mockConfig.MaxPlayers = 10;
        var maxPlayers = mockConfig.MaxPlayers;

        // Assert
        Assert.AreEqual(10, maxPlayers);
    }

    [Test]
    public void Property_WithReturns()
    {
        // Arrange
        var mockConfig = Substitute.For<IGameConfig>();

        // 특정 값 반환 설정
        mockConfig.MaxPlayers.Returns(100);

        // Act & Assert
        Assert.AreEqual(100, mockConfig.MaxPlayers);
    }

    [Test]
    public void Property_SetterReceived()
    {
        // Arrange
        var mockConfig = Substitute.For<IGameConfig>();

        // Act
        mockConfig.MaxPlayers = 20;

        // Assert - setter 호출 검증
        mockConfig.Received().MaxPlayers = 20;
    }
}
```

## 이벤트 모킹

### 이벤트 발생 시뮬레이션

```csharp
public interface IGameEventSystem
{
    event System.Action<int> OnPlayerJoined;
    event System.Action<string> OnGameStateChanged;
}

[TestFixture]
public class EventTests
{
    [Test]
    public void Event_CanBeRaised()
    {
        // Arrange
        var mockEventSystem = Substitute.For<IGameEventSystem>();
        var playerIdReceived = 0;

        mockEventSystem.OnPlayerJoined += playerId => playerIdReceived = playerId;

        // Act - 이벤트 발생
        mockEventSystem.OnPlayerJoined += Raise.Event<System.Action<int>>(42);

        // Assert
        Assert.AreEqual(42, playerIdReceived);
    }

    [Test]
    public void Event_MultipleSubscribers()
    {
        // Arrange
        var mockEventSystem = Substitute.For<IGameEventSystem>();
        var callCount = 0;

        mockEventSystem.OnGameStateChanged += _ => callCount++;
        mockEventSystem.OnGameStateChanged += _ => callCount++;

        // Act
        mockEventSystem.OnGameStateChanged += Raise.Event<System.Action<string>>("Playing");

        // Assert
        Assert.AreEqual(2, callCount);
    }
}
```

## When/Do 패턴

### When을 사용한 부작용

```csharp
[TestFixture]
public class WhenDoTests
{
    [Test]
    public void When_ExecutesAction()
    {
        // Arrange
        var mockService = Substitute.For<IPlayerService>();
        var saveCount = 0;

        // SavePlayerData가 호출되면 카운터 증가
        mockService
            .When(x => x.SavePlayerData(Arg.Any<int>(), Arg.Any<string>()))
            .Do(x => saveCount++);

        // Act
        mockService.SavePlayerData(1, "data1");
        mockService.SavePlayerData(2, "data2");

        // Assert
        Assert.AreEqual(2, saveCount);
    }

    [Test]
    public void When_WithMultipleActions()
    {
        // Arrange
        var mockLogger = Substitute.For<ILogger>();
        var logs = new List<string>();

        mockLogger
            .When(x => x.Log(Arg.Any<string>()))
            .Do(x =>
            {
                var message = x.Arg<string>();
                logs.Add(message);
                System.Console.WriteLine($"Logged: {message}");
            });

        // Act
        mockLogger.Log("Test 1");
        mockLogger.Log("Test 2");

        // Assert
        Assert.AreEqual(2, logs.Count);
    }
}
```

## 부분 대체 (Partial Substitutes)

실제 구현의 일부만 모킹.

```csharp
public class GameCalculator
{
    public virtual int Add(int a, int b)
    {
        return a + b;
    }

    public virtual int Multiply(int a, int b)
    {
        return a * b;
    }
}

[TestFixture]
public class PartialSubstituteTests
{
    [Test]
    public void PartialSubstitute_OverrideSomeMembers()
    {
        // Arrange - 부분 대체 생성
        var partialCalc = Substitute.ForPartsOf<GameCalculator>();

        // Multiply만 모킹, Add는 실제 구현 사용
        partialCalc.Multiply(Arg.Any<int>(), Arg.Any<int>()).Returns(100);

        // Act & Assert
        Assert.AreEqual(5, partialCalc.Add(2, 3)); // 실제 구현
        Assert.AreEqual(100, partialCalc.Multiply(2, 3)); // 모킹된 구현
    }
}
```

## 복잡한 시나리오

### 메서드 체이닝 모킹

```csharp
public interface IQueryBuilder
{
    IQueryBuilder Where(string condition);
    IQueryBuilder OrderBy(string field);
    string Build();
}

[TestFixture]
public class MethodChainingTests
{
    [Test]
    public void MethodChaining_Mock()
    {
        // Arrange
        var mockBuilder = Substitute.For<IQueryBuilder>();

        // 체이닝 설정
        mockBuilder.Where(Arg.Any<string>()).Returns(mockBuilder);
        mockBuilder.OrderBy(Arg.Any<string>()).Returns(mockBuilder);
        mockBuilder.Build().Returns("SELECT * FROM players WHERE active=1 ORDER BY name");

        // Act
        var query = mockBuilder
            .Where("active=1")
            .OrderBy("name")
            .Build();

        // Assert
        Assert.AreEqual("SELECT * FROM players WHERE active=1 ORDER BY name", query);
    }
}
```

### 제네릭 메서드 모킹

```csharp
public interface IRepository
{
    T Get<T>(int id) where T : class;
    void Save<T>(T entity) where T : class;
}

[TestFixture]
public class GenericMethodTests
{
    [Test]
    public void GenericMethod_Mock()
    {
        // Arrange
        var mockRepo = Substitute.For<IRepository>();

        var player = new Player { Id = 1, Name = "Player1" };
        mockRepo.Get<Player>(1).Returns(player);

        // Act
        var result = mockRepo.Get<Player>(1);

        // Assert
        Assert.AreEqual(player, result);
    }
}

public class Player
{
    public int Id { get; set; }
    public string Name { get; set; }
}
```
