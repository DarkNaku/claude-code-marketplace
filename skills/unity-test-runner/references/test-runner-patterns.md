# Test Runner 패턴 가이드

Unity Test Runner에서 고급 테스트 패턴을 위한 심층 참조 문서.

## AAA 패턴 (Arrange-Act-Assert)

### 기본 구조

```csharp
using NUnit.Framework;

public class WeaponSystemTests
{
    [Test]
    public void Attack_WithValidTarget_DealsDamage()
    {
        // Arrange (준비): 테스트에 필요한 객체와 상태 설정
        var weapon = new Sword { Damage = 10 };
        var enemy = new Enemy { Health = 100 };

        // Act (실행): 테스트할 동작 실행
        weapon.Attack(enemy);

        // Assert (검증): 예상 결과 확인
        Assert.AreEqual(90, enemy.Health);
    }
}

public class Sword
{
    public int Damage { get; set; }
    public void Attack(Enemy target) => target.TakeDamage(Damage);
}

public class Enemy
{
    public int Health { get; set; }
    public void TakeDamage(int damage) => Health -= damage;
}
```

### 복잡한 Arrange

```csharp
public class InventoryTests
{
    private Inventory mInventory;
    private List<Item> mTestItems;

    [SetUp]
    public void SetUp()
    {
        // 공통 Arrange를 SetUp으로 추출
        mInventory = new Inventory(capacity: 10);
        mTestItems = new List<Item>
        {
            new Item { Id = 1, Name = "Sword", Weight = 5 },
            new Item { Id = 2, Name = "Shield", Weight = 8 },
            new Item { Id = 3, Name = "Potion", Weight = 1 }
        };
    }

    [Test]
    public void AddItem_WhenSpaceAvailable_ReturnsTrue()
    {
        // Arrange
        var item = mTestItems[0];

        // Act
        var result = mInventory.AddItem(item);

        // Assert
        Assert.IsTrue(result);
        Assert.AreEqual(1, mInventory.ItemCount);
    }

    [Test]
    public void AddItem_WhenOverCapacity_ReturnsFalse()
    {
        // Arrange
        foreach (var item in mTestItems)
        {
            mInventory.AddItem(item);
        }
        var extraItem = new Item { Id = 4, Name = "Extra", Weight = 5 };

        // Act
        var result = mInventory.AddItem(extraItem);

        // Assert
        Assert.IsFalse(result);
    }
}

public class Inventory
{
    private List<Item> mItems = new();
    private int mCapacity;

    public int ItemCount => mItems.Count;

    public Inventory(int capacity) => mCapacity = capacity;

    public bool AddItem(Item item)
    {
        var totalWeight = mItems.Sum(i => i.Weight) + item.Weight;
        if (totalWeight > mCapacity) return false;
        mItems.Add(item);
        return true;
    }
}

public class Item
{
    public int Id { get; set; }
    public string Name { get; set; }
    public int Weight { get; set; }
}
```

## 매개변수화된 테스트

### TestCase 사용

```csharp
public class MathTests
{
    [TestCase(2, 3, 5)]
    [TestCase(0, 0, 0)]
    [TestCase(-1, 1, 0)]
    [TestCase(100, 200, 300)]
    public void Add_VariousInputs_ReturnsCorrectSum(int a, int b, int expected)
    {
        // Arrange
        var calculator = new Calculator();

        // Act
        var result = calculator.Add(a, b);

        // Assert
        Assert.AreEqual(expected, result);
    }

    [TestCase(10, 2, 5f)]
    [TestCase(10, 4, 2.5f)]
    [TestCase(-10, 2, -5f)]
    public void Divide_VariousInputs_ReturnsCorrectQuotient(float a, float b, float expected)
    {
        var calculator = new Calculator();
        var result = calculator.Divide(a, b);
        Assert.AreEqual(expected, result, 0.001f);
    }
}

public class Calculator
{
    public int Add(int a, int b) => a + b;
    public float Divide(float a, float b) => a / b;
}
```

### TestCaseSource 사용

```csharp
public class WeaponDamageTests
{
    [TestCaseSource(nameof(DamageTestCases))]
    public void CalculateDamage_WithDifferentWeapons_ReturnsExpectedDamage(
        WeaponType weaponType, int baseAttack, int expectedDamage)
    {
        // Arrange
        var damageCalculator = new DamageCalculator();

        // Act
        var damage = damageCalculator.Calculate(weaponType, baseAttack);

        // Assert
        Assert.AreEqual(expectedDamage, damage);
    }

    private static object[] DamageTestCases =
    {
        new object[] { WeaponType.Sword, 10, 10 },
        new object[] { WeaponType.Axe, 10, 15 },    // 1.5배
        new object[] { WeaponType.Dagger, 10, 7 },  // 0.7배
        new object[] { WeaponType.Bow, 10, 12 }     // 1.2배
    };
}

public enum WeaponType { Sword, Axe, Dagger, Bow }

public class DamageCalculator
{
    public int Calculate(WeaponType type, int baseAttack)
    {
        var multiplier = type switch
        {
            WeaponType.Sword => 1.0f,
            WeaponType.Axe => 1.5f,
            WeaponType.Dagger => 0.7f,
            WeaponType.Bow => 1.2f,
            _ => 1.0f
        };
        return (int)(baseAttack * multiplier);
    }
}
```

### ValueSource 사용

```csharp
public class RangeTests
{
    private static readonly int[] ValidLevels = { 1, 5, 10, 50, 100 };
    private static readonly int[] InvalidLevels = { -1, 0, 101, 1000 };

    [Test]
    public void SetLevel_WithValidLevel_Succeeds([ValueSource(nameof(ValidLevels))] int level)
    {
        var player = new Player();
        player.SetLevel(level);
        Assert.AreEqual(level, player.Level);
    }

    [Test]
    public void SetLevel_WithInvalidLevel_ThrowsException(
        [ValueSource(nameof(InvalidLevels))] int level)
    {
        var player = new Player();
        Assert.Throws<System.ArgumentOutOfRangeException>(() => player.SetLevel(level));
    }
}

public class Player
{
    public int Level { get; private set; } = 1;

    public void SetLevel(int level)
    {
        if (level < 1 || level > 100)
            throw new System.ArgumentOutOfRangeException(nameof(level));
        Level = level;
    }
}
```

## 비동기 및 코루틴 테스트

### UnityTest 코루틴

```csharp
using UnityEngine;
using UnityEngine.TestTools;
using System.Collections;
using NUnit.Framework;

public class AnimationTests
{
    [UnityTest]
    public IEnumerator FadeIn_OverTime_IncreasesAlpha()
    {
        // Arrange
        var go = new GameObject();
        var fader = go.AddComponent<Fader>();
        fader.Duration = 1f;

        // Act
        fader.FadeIn();

        // Assert - 시작 시점
        Assert.AreEqual(0f, fader.Alpha, 0.01f);

        // 중간 시점 확인
        yield return new WaitForSeconds(0.5f);
        Assert.Greater(fader.Alpha, 0.3f);
        Assert.Less(fader.Alpha, 0.7f);

        // 완료 시점 확인
        yield return new WaitForSeconds(0.6f);
        Assert.AreEqual(1f, fader.Alpha, 0.01f);

        Object.Destroy(go);
    }

    [UnityTest]
    public IEnumerator Projectile_MovesOverTime()
    {
        // Arrange
        var projectile = new GameObject().AddComponent<Projectile>();
        projectile.Speed = 10f;
        var startPos = projectile.transform.position;

        // Act
        yield return new WaitForSeconds(1f);

        // Assert
        var distance = Vector3.Distance(startPos, projectile.transform.position);
        Assert.AreEqual(10f, distance, 0.5f);

        Object.Destroy(projectile.gameObject);
    }
}

public class Fader : MonoBehaviour
{
    public float Duration { get; set; } = 1f;
    public float Alpha { get; private set; } = 0f;

    public void FadeIn() => StartCoroutine(FadeInCoroutine());

    private IEnumerator FadeInCoroutine()
    {
        float elapsed = 0f;
        while (elapsed < Duration)
        {
            elapsed += Time.deltaTime;
            Alpha = Mathf.Clamp01(elapsed / Duration);
            yield return null;
        }
        Alpha = 1f;
    }
}

public class Projectile : MonoBehaviour
{
    public float Speed { get; set; } = 5f;

    void Update()
    {
        transform.position += Vector3.forward * Speed * Time.deltaTime;
    }
}
```

### 프레임 기반 대기

```csharp
public class PhysicsTests
{
    [UnityTest]
    public IEnumerator Rigidbody_FallsUnderGravity()
    {
        // Arrange
        var go = new GameObject();
        var rb = go.AddComponent<Rigidbody>();
        var startY = go.transform.position.y;

        // Act - 물리 시뮬레이션을 위해 여러 프레임 대기
        yield return new WaitForFixedUpdate();
        yield return new WaitForFixedUpdate();
        yield return new WaitForFixedUpdate();

        // Assert
        Assert.Less(go.transform.position.y, startY);

        Object.Destroy(go);
    }

    [UnityTest]
    public IEnumerator CollisionDetection_TriggersEvent()
    {
        // Arrange
        var collisionDetected = false;
        var obj1 = CreateCollisionObject(Vector3.zero);
        var obj2 = CreateCollisionObject(Vector3.up * 5);

        var detector = obj1.AddComponent<CollisionDetector>();
        detector.OnCollisionDetected += () => collisionDetected = true;

        // Act - 오브젝트를 서로를 향해 이동
        obj2.GetComponent<Rigidbody>().velocity = Vector3.down * 10f;

        // 충돌까지 대기 (최대 2초)
        float timeout = 2f;
        while (!collisionDetected && timeout > 0)
        {
            yield return new WaitForFixedUpdate();
            timeout -= Time.fixedDeltaTime;
        }

        // Assert
        Assert.IsTrue(collisionDetected, "충돌이 감지되지 않음");

        Object.Destroy(obj1);
        Object.Destroy(obj2);
    }

    private GameObject CreateCollisionObject(Vector3 position)
    {
        var go = GameObject.CreatePrimitive(PrimitiveType.Sphere);
        go.transform.position = position;
        go.AddComponent<Rigidbody>();
        return go;
    }
}

public class CollisionDetector : MonoBehaviour
{
    public event System.Action OnCollisionDetected;

    void OnCollisionEnter(Collision collision)
    {
        OnCollisionDetected?.Invoke();
    }
}
```

## Mock과 Stub 패턴

### 인터페이스 기반 Mock

```csharp
public interface IDataService
{
    string LoadData(string key);
    void SaveData(string key, string value);
}

// Mock 구현
public class MockDataService : IDataService
{
    private Dictionary<string, string> mData = new();
    public int LoadCallCount { get; private set; }
    public int SaveCallCount { get; private set; }

    public string LoadData(string key)
    {
        LoadCallCount++;
        return mData.TryGetValue(key, out var value) ? value : null;
    }

    public void SaveData(string key, string value)
    {
        SaveCallCount++;
        mData[key] = value;
    }
}

public class SaveSystemTests
{
    [Test]
    public void SavePlayerData_CallsSaveDataService()
    {
        // Arrange
        var mockService = new MockDataService();
        var saveSystem = new SaveSystem(mockService);
        var player = new Player { Name = "Hero", Level = 10 };

        // Act
        saveSystem.SavePlayer(player);

        // Assert
        Assert.AreEqual(1, mockService.SaveCallCount);
    }

    [Test]
    public void LoadPlayerData_ReturnsCorrectData()
    {
        // Arrange
        var mockService = new MockDataService();
        mockService.SaveData("player", "Hero:10");
        var saveSystem = new SaveSystem(mockService);

        // Act
        var player = saveSystem.LoadPlayer();

        // Assert
        Assert.AreEqual("Hero", player.Name);
        Assert.AreEqual(10, player.Level);
        Assert.AreEqual(1, mockService.LoadCallCount);
    }
}

public class SaveSystem
{
    private IDataService mDataService;

    public SaveSystem(IDataService dataService)
    {
        mDataService = dataService;
    }

    public void SavePlayer(Player player)
    {
        var data = $"{player.Name}:{player.Level}";
        mDataService.SaveData("player", data);
    }

    public Player LoadPlayer()
    {
        var data = mDataService.LoadData("player");
        if (string.IsNullOrEmpty(data)) return null;

        var parts = data.Split(':');
        return new Player
        {
            Name = parts[0],
            Level = int.Parse(parts[1])
        };
    }
}

public class Player
{
    public string Name { get; set; }
    public int Level { get; set; }
}
```

### Stub으로 외부 의존성 제거

```csharp
public interface INetworkClient
{
    bool IsConnected { get; }
    string SendRequest(string endpoint);
}

// Stub 구현 - 항상 성공하는 네트워크
public class NetworkStubSuccess : INetworkClient
{
    public bool IsConnected => true;
    public string SendRequest(string endpoint) => "{\"status\":\"ok\"}";
}

// Stub 구현 - 항상 실패하는 네트워크
public class NetworkStubFailure : INetworkClient
{
    public bool IsConnected => false;
    public string SendRequest(string endpoint) => null;
}

public class OnlineServiceTests
{
    [Test]
    public void GetServerStatus_WhenConnected_ReturnsOk()
    {
        // Arrange
        var networkStub = new NetworkStubSuccess();
        var service = new OnlineService(networkStub);

        // Act
        var status = service.GetServerStatus();

        // Assert
        Assert.AreEqual("ok", status);
    }

    [Test]
    public void GetServerStatus_WhenDisconnected_ReturnsNull()
    {
        // Arrange
        var networkStub = new NetworkStubFailure();
        var service = new OnlineService(networkStub);

        // Act
        var status = service.GetServerStatus();

        // Assert
        Assert.IsNull(status);
    }
}

public class OnlineService
{
    private INetworkClient mClient;

    public OnlineService(INetworkClient client)
    {
        mClient = client;
    }

    public string GetServerStatus()
    {
        if (!mClient.IsConnected) return null;
        var response = mClient.SendRequest("/status");
        // 간단한 JSON 파싱
        return response?.Contains("ok") == true ? "ok" : "error";
    }
}
```

### 스파이 패턴

```csharp
public interface ILogger
{
    void Log(string message);
    void LogError(string message);
}

// Spy - Mock의 확장으로 호출 기록을 저장
public class LoggerSpy : ILogger
{
    public List<string> LogMessages { get; } = new();
    public List<string> ErrorMessages { get; } = new();

    public void Log(string message) => LogMessages.Add(message);
    public void LogError(string message) => ErrorMessages.Add(message);
}

public class GameManagerTests
{
    [Test]
    public void StartGame_LogsStartMessage()
    {
        // Arrange
        var loggerSpy = new LoggerSpy();
        var gameManager = new GameManager(loggerSpy);

        // Act
        gameManager.StartGame();

        // Assert
        Assert.AreEqual(1, loggerSpy.LogMessages.Count);
        StringAssert.Contains("게임 시작", loggerSpy.LogMessages[0]);
    }

    [Test]
    public void GameOver_LogsErrorWhenPlayerDead()
    {
        // Arrange
        var loggerSpy = new LoggerSpy();
        var gameManager = new GameManager(loggerSpy);

        // Act
        gameManager.GameOver(reason: "플레이어 사망");

        // Assert
        Assert.AreEqual(1, loggerSpy.ErrorMessages.Count);
        StringAssert.Contains("플레이어 사망", loggerSpy.ErrorMessages[0]);
    }
}

public class GameManager
{
    private ILogger mLogger;

    public GameManager(ILogger logger)
    {
        mLogger = logger;
    }

    public void StartGame()
    {
        mLogger.Log("게임 시작");
    }

    public void GameOver(string reason)
    {
        mLogger.LogError($"게임 오버: {reason}");
    }
}
```

## Scene 기반 테스트

### Scene 로드 및 테스트

```csharp
using UnityEngine;
using UnityEngine.SceneManagement;
using UnityEngine.TestTools;
using System.Collections;
using NUnit.Framework;

public class SceneTests
{
    [UnityTest]
    public IEnumerator LoadGameScene_ContainsPlayer()
    {
        // Arrange & Act
        yield return SceneManager.LoadSceneAsync("GameScene", LoadSceneMode.Single);

        // Assert
        var player = GameObject.FindWithTag("Player");
        Assert.IsNotNull(player, "GameScene에 Player가 없음");
    }

    [UnityTest]
    public IEnumerator SceneTransition_PreservesPlayerData()
    {
        // Arrange
        yield return SceneManager.LoadSceneAsync("MenuScene");
        var dataManager = new GameObject().AddComponent<DataManager>();
        Object.DontDestroyOnLoad(dataManager.gameObject);
        dataManager.PlayerName = "TestPlayer";

        // Act
        yield return SceneManager.LoadSceneAsync("GameScene");

        // Assert
        var persistedManager = Object.FindObjectOfType<DataManager>();
        Assert.IsNotNull(persistedManager);
        Assert.AreEqual("TestPlayer", persistedManager.PlayerName);

        // Cleanup
        Object.Destroy(persistedManager.gameObject);
    }
}

public class DataManager : MonoBehaviour
{
    public string PlayerName { get; set; }
}
```

### 임시 Scene 생성

```csharp
public class IsolatedSceneTests
{
    private Scene mTestScene;

    [SetUp]
    public void SetUp()
    {
        // 격리된 테스트 씬 생성
        mTestScene = SceneManager.CreateScene("TestScene");
        SceneManager.SetActiveScene(mTestScene);
    }

    [TearDown]
    public void TearDown()
    {
        // 테스트 씬 제거
        SceneManager.UnloadSceneAsync(mTestScene);
    }

    [Test]
    public void CreateEnemy_InTestScene_ExistsInScene()
    {
        // Arrange & Act
        var enemy = new GameObject("Enemy");
        SceneManager.MoveGameObjectToScene(enemy, mTestScene);

        // Assert
        var foundEnemy = GameObject.Find("Enemy");
        Assert.IsNotNull(foundEnemy);
        Assert.AreEqual(mTestScene, foundEnemy.scene);
    }
}
```

## 예외 및 에러 처리 테스트

### 예외 테스트

```csharp
public class ExceptionTests
{
    [Test]
    public void Divide_ByZero_ThrowsDivideByZeroException()
    {
        var calculator = new Calculator();

        Assert.Throws<System.DivideByZeroException>(() =>
        {
            calculator.Divide(10, 0);
        });
    }

    [Test]
    public void GetItem_InvalidIndex_ThrowsArgumentOutOfRangeException()
    {
        var inventory = new Inventory(5);

        var exception = Assert.Throws<System.ArgumentOutOfRangeException>(() =>
        {
            inventory.GetItem(10);
        });

        StringAssert.Contains("index", exception.Message.ToLower());
    }

    [Test]
    public void ParseConfig_InvalidJson_ThrowsException()
    {
        var parser = new ConfigParser();
        var invalidJson = "{ invalid json }";

        Assert.Throws<System.Exception>(() => parser.Parse(invalidJson));
    }
}

public class ConfigParser
{
    public object Parse(string json)
    {
        if (!json.StartsWith("{")) throw new System.Exception("Invalid JSON");
        return new object();
    }
}
```

### LogAssert 사용

```csharp
using UnityEngine.TestTools;

public class LogTests
{
    [Test]
    public void Warning_LogsWarningMessage()
    {
        // Arrange
        var system = new WarningSystem();

        // Assert & Act
        LogAssert.Expect(LogType.Warning, "경고: 체력 부족");
        system.CheckHealth(10);
    }

    [Test]
    public void Error_LogsErrorMessage()
    {
        var system = new ErrorSystem();

        LogAssert.Expect(LogType.Error, "에러: 파일 없음");
        system.LoadFile("invalid.txt");
    }
}

public class WarningSystem
{
    public void CheckHealth(int health)
    {
        if (health < 20)
            Debug.LogWarning("경고: 체력 부족");
    }
}

public class ErrorSystem
{
    public void LoadFile(string filename)
    {
        if (!System.IO.File.Exists(filename))
            Debug.LogError("에러: 파일 없음");
    }
}
```

## 성능 테스트

### Performance 측정

```csharp
using Unity.PerformanceTesting;

public class PerformanceTests
{
    [Test, Performance]
    public void Sorting_LargeArray_MeasuresTime()
    {
        var data = new int[10000];
        for (int i = 0; i < data.Length; i++)
            data[i] = UnityEngine.Random.Range(0, 1000);

        Measure.Method(() =>
        {
            System.Array.Sort(data);
        })
        .WarmupCount(5)
        .MeasurementCount(10)
        .Run();
    }

    [Test, Performance]
    public void Instantiate_MeasuresMemory()
    {
        Measure.Method(() =>
        {
            var go = new GameObject();
            Object.Destroy(go);
        })
        .SampleGroup("Memory")
        .MeasurementCount(100)
        .Run();
    }
}
```
