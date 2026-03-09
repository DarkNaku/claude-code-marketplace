---
name: unity-test-runner
description: Unity Test Runner 및 NUnit 기반 테스트 전문가. 단위 테스트, 통합 테스트, Play Mode/Edit Mode 테스트 작성에 능통. TDD(테스트 주도 개발), Mock/Stub 패턴, Assertion 전략, 비동기 테스트에 전문. 코드 품질 향상, 리팩토링 안전성 확보, CI/CD 파이프라인 구축 시 적극적으로 사용.
---

# Unity Test Runner - Unity 공식 테스트 프레임워크

## 개요

Unity Test Runner는 NUnit 프레임워크 기반의 Unity 공식 테스트 도구로, Edit Mode와 Play Mode 테스트를 지원합니다. 단위 테스트와 통합 테스트를 통해 코드 품질을 보장하고 리팩토링을 안전하게 수행할 수 있습니다.

**핵심 주제**:
- Edit Mode vs Play Mode 테스트
- NUnit Assertions (Assert, CollectionAssert, StringAssert)
- SetUp/TearDown 생명주기
- UnityTest (코루틴 기반 비동기 테스트)
- Mock과 Stub 패턴
- Scene 기반 테스트
- 테스트 주도 개발 (TDD)

**학습 경로**: NUnit 기초 → Edit Mode 테스트 → Play Mode 테스트 → Mock 패턴 → TDD

## 빠른 시작

```csharp
using NUnit.Framework;
using UnityEngine;

// Edit Mode 테스트 (순수 C# 로직)
public class CalculatorTests
{
    private Calculator mCalculator;

    [SetUp]
    public void SetUp()
    {
        mCalculator = new Calculator();
    }

    [Test]
    public void Add_TwoPositiveNumbers_ReturnsCorrectSum()
    {
        // Arrange
        var a = 5;
        var b = 3;

        // Act
        var result = mCalculator.Add(a, b);

        // Assert
        Assert.AreEqual(8, result);
    }

    [Test]
    public void Divide_ByZero_ThrowsException()
    {
        Assert.Throws<System.DivideByZeroException>(() =>
        {
            mCalculator.Divide(10, 0);
        });
    }
}

public class Calculator
{
    public int Add(int a, int b) => a + b;
    public float Divide(float a, float b)
    {
        if (b == 0) throw new System.DivideByZeroException();
        return a / b;
    }
}
```

```csharp
using NUnit.Framework;
using UnityEngine;
using UnityEngine.TestTools;
using System.Collections;

// Play Mode 테스트 (Unity 런타임 필요)
public class PlayerControllerTests
{
    private GameObject mPlayerObject;
    private PlayerController mPlayer;

    [SetUp]
    public void SetUp()
    {
        mPlayerObject = new GameObject("Player");
        mPlayer = mPlayerObject.AddComponent<PlayerController>();
    }

    [TearDown]
    public void TearDown()
    {
        Object.Destroy(mPlayerObject);
    }

    [UnityTest]
    public IEnumerator Jump_IncreasesYPosition()
    {
        // Arrange
        var initialY = mPlayer.transform.position.y;

        // Act
        mPlayer.Jump();
        yield return new WaitForSeconds(0.5f);

        // Assert
        Assert.Greater(mPlayer.transform.position.y, initialY);
    }
}
```

## 핵심 개념

### Edit Mode vs Play Mode

**Edit Mode 테스트**:
- 에디터에서 즉시 실행
- Unity 런타임 불필요
- 순수 C# 로직 테스트
- 빠른 실행 속도

**Play Mode 테스트**:
- Play Mode에서 실행
- MonoBehaviour, GameObject 사용 가능
- 물리, 렌더링, 코루틴 테스트
- 느린 실행 속도

### Assertions

```csharp
// 기본 Assertion
Assert.AreEqual(expected, actual);
Assert.AreNotEqual(expected, actual);
Assert.IsTrue(condition);
Assert.IsFalse(condition);
Assert.IsNull(obj);
Assert.IsNotNull(obj);

// 숫자 비교
Assert.Greater(actual, expected);
Assert.Less(actual, expected);
Assert.AreEqual(expected, actual, delta: 0.01f); // 부동소수점

// 컬렉션
CollectionAssert.AreEqual(expectedList, actualList);
CollectionAssert.Contains(collection, item);
CollectionAssert.IsEmpty(collection);

// 문자열
StringAssert.Contains("substring", actualString);
StringAssert.StartsWith("prefix", actualString);
```

### 테스트 생명주기

```csharp
[TestFixture]
public class LifecycleExample
{
    [OneTimeSetUp]
    public void OneTimeSetUp()
    {
        // 전체 테스트 시작 전 한 번
    }

    [SetUp]
    public void SetUp()
    {
        // 각 테스트 전에 실행
    }

    [Test]
    public void Test1() { }

    [Test]
    public void Test2() { }

    [TearDown]
    public void TearDown()
    {
        // 각 테스트 후에 실행
    }

    [OneTimeTearDown]
    public void OneTimeTearDown()
    {
        // 전체 테스트 종료 후 한 번
    }
}
```

## 참고 문서

### [Test Runner 패턴 가이드](references/test-runner-patterns.md)
핵심 테스트 패턴:
- AAA 패턴 (Arrange-Act-Assert)
- 테스트 케이스 작성 전략
- 매개변수화된 테스트
- 비동기 및 코루틴 테스트
- Mock과 Stub 패턴
- Scene 기반 테스트

### [Test Runner 통합 패턴](references/test-runner-integrations.md)
고급 통합:
- VContainer DI 테스트
- R3 ReactiveProperty 테스트
- Physics 및 Collision 테스트
- UI 테스트
- Network 테스트
- CI/CD 통합

### [Unity Test Framework 공식 문서](https://docs.unity3d.com/Packages/com.unity.test-framework@latest)

## 모범 사례

1. **테스트 이름은 명확하게**: `MethodName_Scenario_ExpectedResult` 형식 사용
2. **AAA 패턴 준수**: Arrange, Act, Assert 구조로 작성
3. **독립적인 테스트**: 각 테스트는 다른 테스트에 의존하지 않음
4. **빠른 테스트 우선**: Edit Mode 테스트를 먼저 고려
5. **TDD 실천**: 테스트를 먼저 작성하고 구현
