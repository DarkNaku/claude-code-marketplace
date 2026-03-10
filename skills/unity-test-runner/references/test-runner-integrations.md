# Test Runner 통합 패턴

Unity Test Runner와 다른 시스템 및 라이브러리와의 통합 패턴.

## VContainer DI 테스트

### 테스트용 Container 구성

```csharp
using VContainer;
using VContainer.Unity;
using NUnit.Framework;

public class VContainerTests
{
    private IObjectResolver mContainer;

    [SetUp]
    public void SetUp()
    {
        // 테스트용 컨테이너 빌드
        var builder = new ContainerBuilder();
        ConfigureContainer(builder);
        mContainer = builder.Build();
    }

    [TearDown]
    public void TearDown()
    {
        mContainer?.Dispose();
    }

    void ConfigureContainer(IContainerBuilder builder)
    {
        // 실제 서비스 대신 Mock 등록
        builder.Register<IPlayerService, MockPlayerService>(Lifetime.Singleton);
        builder.Register<IScoreService, ScoreService>(Lifetime.Singleton);
        builder.Register<GameManager>(Lifetime.Singleton);
    }

    [Test]
    public void GameManager_ResolvesWithDependencies()
    {
        // Act
        var gameManager = mContainer.Resolve<GameManager>();

        // Assert
        Assert.IsNotNull(gameManager);
    }

    [Test]
    public void ScoreService_UsesInjectedPlayerService()
    {
        // Arrange
        var scoreService = mContainer.Resolve<IScoreService>();

        // Act
        scoreService.AddScore(100);

        // Assert
        var playerService = mContainer.Resolve<IPlayerService>() as MockPlayerService;
        Assert.IsNotNull(playerService);
        Assert.AreEqual(1, playerService.UpdateScoreCallCount);
    }
}

public interface IPlayerService
{
    void UpdateScore(int score);
}

public class MockPlayerService : IPlayerService
{
    public int UpdateScoreCallCount { get; private set; }
    public void UpdateScore(int score) => UpdateScoreCallCount++;
}

public interface IScoreService
{
    void AddScore(int points);
}

public class ScoreService : IScoreService
{
    private readonly IPlayerService mPlayerService;

    public ScoreService(IPlayerService playerService)
    {
        mPlayerService = playerService;
    }

    public void AddScore(int points)
    {
        mPlayerService.UpdateScore(points);
    }
}

public class GameManager
{
    private readonly IPlayerService mPlayerService;
    private readonly IScoreService mScoreService;

    public GameManager(IPlayerService playerService, IScoreService scoreService)
    {
        mPlayerService = playerService;
        mScoreService = scoreService;
    }
}
```

### LifetimeScope 테스트

```csharp
using UnityEngine;
using UnityEngine.TestTools;
using System.Collections;

public class LifetimeScopeTests
{
    [UnityTest]
    public IEnumerator LifetimeScope_InjectsIntoMonoBehaviour()
    {
        // Arrange
        var scopeObject = new GameObject("LifetimeScope");
        var scope = scopeObject.AddComponent<TestLifetimeScope>();

        var playerObject = new GameObject("Player");
        var player = playerObject.AddComponent<PlayerController>();

        // Act
        yield return null; // LifetimeScope 초기화 대기

        // Assert
        Assert.IsNotNull(player.PlayerService);

        // Cleanup
        Object.Destroy(scopeObject);
        Object.Destroy(playerObject);
    }
}

public class TestLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<IPlayerService, MockPlayerService>(Lifetime.Singleton);
        builder.RegisterComponentInHierarchy<PlayerController>();
    }
}

public class PlayerController : MonoBehaviour
{
    [Inject]
    public IPlayerService PlayerService { get; private set; }
}
```

## R3 ReactiveProperty 테스트

### ReactiveProperty 값 변경 테스트

```csharp
using R3;
using NUnit.Framework;

public class ReactivePropertyTests
{
    [Test]
    public void ReactiveProperty_ValueChanged_TriggersSubscription()
    {
        // Arrange
        var property = new ReactiveProperty<int>(0);
        var receivedValue = 0;
        var callCount = 0;

        property.Subscribe(value =>
        {
            receivedValue = value;
            callCount++;
        });

        // Act
        property.Value = 42;

        // Assert
        Assert.AreEqual(42, receivedValue);
        Assert.AreEqual(2, callCount); // 초기값 + 변경된 값
    }

    [Test]
    public void ReactiveProperty_SameValue_DoesNotTrigger()
    {
        // Arrange
        var property = new ReactiveProperty<int>(10);
        var callCount = 0;

        property.Skip(1).Subscribe(_ => callCount++); // 초기값 스킵

        // Act
        property.Value = 10; // 같은 값

        // Assert
        Assert.AreEqual(0, callCount);
    }

    [Test]
    public void ReadOnlyReactiveProperty_DerivedFromSource()
    {
        // Arrange
        var source = new ReactiveProperty<int>(5);
        var derived = source.Select(x => x * 2).ToReadOnlyReactiveProperty();

        // Assert - 초기값
        Assert.AreEqual(10, derived.CurrentValue);

        // Act
        source.Value = 10;

        // Assert - 변경된 값
        Assert.AreEqual(20, derived.CurrentValue);
    }
}
```

### Observable 스트림 테스트

```csharp
using R3;
using System.Collections.Generic;

public class ObservableTests
{
    [Test]
    public void Observable_Range_EmitsCorrectValues()
    {
        // Arrange
        var receivedValues = new List<int>();

        // Act
        Observable.Range(1, 5).Subscribe(x => receivedValues.Add(x));

        // Assert
        CollectionAssert.AreEqual(new[] { 1, 2, 3, 4, 5 }, receivedValues);
    }

    [Test]
    public void Observable_Where_FiltersCorrectly()
    {
        // Arrange
        var source = new Subject<int>();
        var filteredValues = new List<int>();

        source.Where(x => x > 5).Subscribe(x => filteredValues.Add(x));

        // Act
        source.OnNext(3);
        source.OnNext(7);
        source.OnNext(4);
        source.OnNext(10);

        // Assert
        CollectionAssert.AreEqual(new[] { 7, 10 }, filteredValues);
    }

    [Test]
    public void Observable_CombineLatest_CombinesValues()
    {
        // Arrange
        var source1 = new ReactiveProperty<int>(0);
        var source2 = new ReactiveProperty<int>(0);
        var combinedValues = new List<(int, int)>();

        source1.CombineLatest(source2, (a, b) => (a, b))
            .Subscribe(tuple => combinedValues.Add(tuple));

        // Act
        source1.Value = 1;
        source2.Value = 2;
        source1.Value = 3;

        // Assert
        Assert.AreEqual((3, 2), combinedValues[combinedValues.Count - 1]);
    }
}
```

### UnityTest와 Observable 통합

```csharp
using UnityEngine;
using UnityEngine.TestTools;
using System.Collections;

public class ObservableUnityTests
{
    [UnityTest]
    public IEnumerator Observable_EveryUpdate_EmitsFrames()
    {
        // Arrange
        var frameCount = 0;
        var disposable = Observable.EveryUpdate()
            .Subscribe(_ => frameCount++);

        // Act
        yield return null;
        yield return null;
        yield return null;

        // Assert
        Assert.GreaterOrEqual(frameCount, 3);

        // Cleanup
        disposable.Dispose();
    }

    [UnityTest]
    public IEnumerator Observable_Timer_CompletesAfterDelay()
    {
        // Arrange
        var completed = false;
        Observable.Timer(System.TimeSpan.FromSeconds(0.5f))
            .Subscribe(_ => { }, () => completed = true);

        // Assert - 아직 완료 안됨
        Assert.IsFalse(completed);

        // Act
        yield return new WaitForSeconds(0.6f);

        // Assert - 완료됨
        Assert.IsTrue(completed);
    }
}
```

## Physics 및 Collision 테스트

### Rigidbody 테스트

```csharp
using UnityEngine;
using UnityEngine.TestTools;
using System.Collections;
using NUnit.Framework;

public class PhysicsTests
{
    [UnityTest]
    public IEnumerator Rigidbody_ApplyForce_ChangesVelocity()
    {
        // Arrange
        var go = GameObject.CreatePrimitive(PrimitiveType.Sphere);
        var rb = go.AddComponent<Rigidbody>();
        rb.useGravity = false;

        // Act
        rb.AddForce(Vector3.up * 100f);
        yield return new WaitForFixedUpdate();

        // Assert
        Assert.Greater(rb.velocity.magnitude, 0f);

        Object.Destroy(go);
    }

    [UnityTest]
    public IEnumerator Collision_BetweenObjects_TriggersEvent()
    {
        // Arrange
        var obj1 = GameObject.CreatePrimitive(PrimitiveType.Cube);
        obj1.transform.position = Vector3.zero;
        var rb1 = obj1.AddComponent<Rigidbody>();

        var obj2 = GameObject.CreatePrimitive(PrimitiveType.Cube);
        obj2.transform.position = Vector3.up * 3f;
        var rb2 = obj2.AddComponent<Rigidbody>();

        var detector = obj1.AddComponent<CollisionCounter>();

        // Act
        rb2.velocity = Vector3.down * 10f;

        // 충돌까지 대기
        float timeout = 2f;
        while (detector.CollisionCount == 0 && timeout > 0)
        {
            yield return new WaitForFixedUpdate();
            timeout -= Time.fixedDeltaTime;
        }

        // Assert
        Assert.Greater(detector.CollisionCount, 0);

        Object.Destroy(obj1);
        Object.Destroy(obj2);
    }
}

public class CollisionCounter : MonoBehaviour
{
    public int CollisionCount { get; private set; }

    void OnCollisionEnter(Collision collision)
    {
        CollisionCount++;
    }
}
```

### Raycast 테스트

```csharp
public class RaycastTests
{
    [UnityTest]
    public IEnumerator Raycast_HitsObject_ReturnsTrue()
    {
        // Arrange
        var target = GameObject.CreatePrimitive(PrimitiveType.Cube);
        target.transform.position = Vector3.forward * 5f;

        yield return new WaitForFixedUpdate();

        // Act
        var hit = Physics.Raycast(Vector3.zero, Vector3.forward, out var hitInfo, 10f);

        // Assert
        Assert.IsTrue(hit);
        Assert.AreEqual(target, hitInfo.collider.gameObject);

        Object.Destroy(target);
    }

    [UnityTest]
    public IEnumerator OverlapSphere_FindsObjectsInRange()
    {
        // Arrange
        var obj1 = GameObject.CreatePrimitive(PrimitiveType.Sphere);
        obj1.transform.position = Vector3.right * 2f;

        var obj2 = GameObject.CreatePrimitive(PrimitiveType.Sphere);
        obj2.transform.position = Vector3.right * 10f;

        yield return new WaitForFixedUpdate();

        // Act
        var colliders = Physics.OverlapSphere(Vector3.zero, 5f);

        // Assert
        Assert.AreEqual(1, colliders.Length);
        Assert.AreEqual(obj1, colliders[0].gameObject);

        Object.Destroy(obj1);
        Object.Destroy(obj2);
    }
}
```

## UI 테스트

### Button 클릭 테스트

```csharp
using UnityEngine;
using UnityEngine.UI;
using UnityEngine.TestTools;
using System.Collections;
using NUnit.Framework;

public class UITests
{
    [UnityTest]
    public IEnumerator Button_Click_InvokesCallback()
    {
        // Arrange
        var canvas = CreateTestCanvas();
        var buttonGO = new GameObject("Button");
        buttonGO.transform.SetParent(canvas.transform);
        var button = buttonGO.AddComponent<Button>();

        var clicked = false;
        button.onClick.AddListener(() => clicked = true);

        yield return null;

        // Act
        button.onClick.Invoke();

        // Assert
        Assert.IsTrue(clicked);

        Object.Destroy(canvas.gameObject);
    }

    [UnityTest]
    public IEnumerator InputField_TextChanged_UpdatesValue()
    {
        // Arrange
        var canvas = CreateTestCanvas();
        var inputGO = new GameObject("InputField");
        inputGO.transform.SetParent(canvas.transform);
        var inputField = inputGO.AddComponent<InputField>();

        var textGO = new GameObject("Text");
        textGO.transform.SetParent(inputGO.transform);
        var text = textGO.AddComponent<Text>();
        inputField.textComponent = text;

        yield return null;

        // Act
        inputField.text = "Test Input";

        // Assert
        Assert.AreEqual("Test Input", inputField.text);

        Object.Destroy(canvas.gameObject);
    }

    private Canvas CreateTestCanvas()
    {
        var canvasGO = new GameObject("Canvas");
        var canvas = canvasGO.AddComponent<Canvas>();
        canvas.renderMode = RenderMode.ScreenSpaceOverlay;
        return canvas;
    }
}
```

### Slider 및 Toggle 테스트

```csharp
public class UIControlTests
{
    [UnityTest]
    public IEnumerator Slider_ValueChanged_TriggersEvent()
    {
        // Arrange
        var sliderGO = new GameObject("Slider");
        var slider = sliderGO.AddComponent<Slider>();

        var changedValue = 0f;
        slider.onValueChanged.AddListener(value => changedValue = value);

        yield return null;

        // Act
        slider.value = 0.75f;

        // Assert
        Assert.AreEqual(0.75f, changedValue, 0.01f);

        Object.Destroy(sliderGO);
    }

    [UnityTest]
    public IEnumerator Toggle_IsOn_ChangesState()
    {
        // Arrange
        var toggleGO = new GameObject("Toggle");
        var toggle = toggleGO.AddComponent<Toggle>();

        var stateChanged = false;
        toggle.onValueChanged.AddListener(isOn => stateChanged = isOn);

        yield return null;

        // Act
        toggle.isOn = true;

        // Assert
        Assert.IsTrue(stateChanged);
        Assert.IsTrue(toggle.isOn);

        Object.Destroy(toggleGO);
    }
}
```

## Animation 테스트

### Animator 상태 테스트

```csharp
using UnityEngine;
using UnityEngine.TestTools;
using System.Collections;

public class AnimationTests
{
    [UnityTest]
    public IEnumerator Animator_SetTrigger_ChangesState()
    {
        // Arrange
        var go = new GameObject("Character");
        var animator = go.AddComponent<Animator>();
        // 실제로는 AnimatorController를 할당해야 함

        yield return null;

        // Act
        animator.SetTrigger("Attack");
        yield return new WaitForSeconds(0.1f);

        // Assert
        // var currentState = animator.GetCurrentAnimatorStateInfo(0);
        // Assert.IsTrue(currentState.IsName("AttackState"));

        Object.Destroy(go);
    }

    [UnityTest]
    public IEnumerator Animation_PlaysToCompletion()
    {
        // Arrange
        var go = new GameObject("Object");
        var animation = go.AddComponent<Animation>();
        var clip = new AnimationClip();
        clip.legacy = true;
        animation.AddClip(clip, "TestClip");

        // Act
        animation.Play("TestClip");
        yield return new WaitForSeconds(clip.length + 0.1f);

        // Assert
        Assert.IsFalse(animation.isPlaying);

        Object.Destroy(go);
    }
}
```

## Audio 테스트

### AudioSource 테스트

```csharp
public class AudioTests
{
    [UnityTest]
    public IEnumerator AudioSource_Play_StartsPlaying()
    {
        // Arrange
        var go = new GameObject("AudioObject");
        var audioSource = go.AddComponent<AudioSource>();

        // AudioClip 생성 (실제로는 리소스 로드)
        var clip = AudioClip.Create("TestClip", 44100, 1, 44100, false);
        audioSource.clip = clip;

        yield return null;

        // Act
        audioSource.Play();

        // Assert
        Assert.IsTrue(audioSource.isPlaying);

        Object.Destroy(go);
    }

    [UnityTest]
    public IEnumerator AudioSource_Volume_Changes()
    {
        // Arrange
        var go = new GameObject("AudioObject");
        var audioSource = go.AddComponent<AudioSource>();

        yield return null;

        // Act
        audioSource.volume = 0.5f;

        // Assert
        Assert.AreEqual(0.5f, audioSource.volume, 0.01f);

        Object.Destroy(go);
    }
}
```

## Coroutine 테스트

### 복잡한 코루틴 테스트

```csharp
public class CoroutineTests
{
    [UnityTest]
    public IEnumerator Coroutine_MultipleSteps_CompletesInOrder()
    {
        // Arrange
        var go = new GameObject("Controller");
        var controller = go.AddComponent<SequenceController>();
        var steps = new List<string>();

        // Act
        controller.StartSequence(steps);

        // 각 단계 확인
        yield return new WaitForSeconds(0.5f);
        Assert.AreEqual(1, steps.Count);
        Assert.AreEqual("Step1", steps[0]);

        yield return new WaitForSeconds(0.5f);
        Assert.AreEqual(2, steps.Count);
        Assert.AreEqual("Step2", steps[1]);

        yield return new WaitForSeconds(0.5f);
        Assert.AreEqual(3, steps.Count);
        Assert.AreEqual("Step3", steps[2]);

        Object.Destroy(go);
    }
}

public class SequenceController : MonoBehaviour
{
    public void StartSequence(List<string> steps)
    {
        StartCoroutine(SequenceCoroutine(steps));
    }

    private IEnumerator SequenceCoroutine(List<string> steps)
    {
        steps.Add("Step1");
        yield return new WaitForSeconds(0.5f);

        steps.Add("Step2");
        yield return new WaitForSeconds(0.5f);

        steps.Add("Step3");
    }
}
```

## Custom Assertion 작성

### 커스텀 Assert 메서드

```csharp
public static class CustomAssert
{
    public static void AreApproximatelyEqual(Vector3 expected, Vector3 actual, float tolerance = 0.01f)
    {
        var distance = Vector3.Distance(expected, actual);
        Assert.LessOrEqual(distance, tolerance,
            $"Vector3 not approximately equal. Expected: {expected}, Actual: {actual}, Distance: {distance}");
    }

    public static void IsInRange(float value, float min, float max)
    {
        Assert.GreaterOrEqual(value, min, $"Value {value} is less than min {min}");
        Assert.LessOrEqual(value, max, $"Value {value} is greater than max {max}");
    }

    public static void HasComponent<T>(GameObject go) where T : Component
    {
        var component = go.GetComponent<T>();
        Assert.IsNotNull(component, $"GameObject {go.name} does not have component {typeof(T).Name}");
    }
}

public class CustomAssertTests
{
    [Test]
    public void CustomAssert_Vector3_ApproximatelyEqual()
    {
        var vec1 = new Vector3(1.0f, 2.0f, 3.0f);
        var vec2 = new Vector3(1.001f, 2.001f, 3.001f);

        CustomAssert.AreApproximatelyEqual(vec1, vec2, tolerance: 0.01f);
    }

    [Test]
    public void CustomAssert_Value_InRange()
    {
        CustomAssert.IsInRange(5f, 0f, 10f);
    }

    [UnityTest]
    public IEnumerator CustomAssert_GameObject_HasComponent()
    {
        var go = new GameObject();
        go.AddComponent<Rigidbody>();

        yield return null;

        CustomAssert.HasComponent<Rigidbody>(go);

        Object.Destroy(go);
    }
}
```

## CI/CD 통합

### Command Line에서 테스트 실행

```bash
# Unity Test Runner를 Command Line에서 실행
Unity -runTests -batchmode -projectPath /path/to/project -testResults /path/to/results.xml -testPlatform EditMode

# Play Mode 테스트
Unity -runTests -batchmode -projectPath /path/to/project -testResults /path/to/results.xml -testPlatform PlayMode
```

### GitHub Actions 예제

```yaml
# .github/workflows/test.yml
name: Unity Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - uses: game-ci/unity-test-runner@v2
        env:
          UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}
        with:
          projectPath: .
          testMode: all

      - uses: actions/upload-artifact@v2
        if: always()
        with:
          name: Test results
          path: artifacts
```

### 테스트 결과 분석 스크립트

```csharp
#if UNITY_EDITOR
using UnityEditor;
using UnityEditor.TestTools.TestRunner.Api;
using UnityEngine;

public class TestResultLogger : ScriptableObject, ICallbacks
{
    public void RunStarted(ITestAdaptor testsToRun)
    {
        Debug.Log($"테스트 시작: {testsToRun.TestCaseCount}개");
    }

    public void RunFinished(ITestResultAdaptor result)
    {
        Debug.Log($"테스트 완료: {result.PassCount}개 성공, {result.FailCount}개 실패");

        if (result.FailCount > 0)
        {
            EditorApplication.Exit(1); // CI/CD에서 실패 표시
        }
    }

    public void TestStarted(ITestAdaptor test)
    {
        Debug.Log($"테스트 실행: {test.Name}");
    }

    public void TestFinished(ITestResultAdaptor result)
    {
        if (result.TestStatus == TestStatus.Failed)
        {
            Debug.LogError($"테스트 실패: {result.Name}\n{result.Message}");
        }
    }
}
#endif
```

## 테스트 커버리지

### Code Coverage 패키지 사용

```csharp
// Unity Code Coverage 패키지 설치 후

#if UNITY_EDITOR
using UnityEditor.TestTools.CodeCoverage;

[MenuItem("Tests/Generate Coverage Report")]
public static void GenerateCoverageReport()
{
    var settings = new CoverageSettings
    {
        resultsPathOptions = ResultsPathOptions.ProjectFolder,
        generateHTMLReport = true,
        generateBadgeReport = true
    };

    Coverage.GenerateReport(settings);
}
#endif
```

### 테스트 커버리지 목표

```csharp
// 핵심 로직은 80% 이상 커버리지 목표
[TestFixture]
public class CriticalSystemTests
{
    // 모든 분기를 테스트하여 높은 커버리지 달성
    [TestCase(0, ExpectedResult = "Zero")]
    [TestCase(1, ExpectedResult = "Positive")]
    [TestCase(-1, ExpectedResult = "Negative")]
    public string GetNumberType_AllBranches_Covered(int number)
    {
        return NumberHelper.GetType(number);
    }
}

public static class NumberHelper
{
    public static string GetType(int number)
    {
        if (number == 0) return "Zero";
        if (number > 0) return "Positive";
        return "Negative";
    }
}
```
