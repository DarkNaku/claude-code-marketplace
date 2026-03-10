---
name: unity-nsubstitute
description: Unity용 모킹 프레임워크, 테스트 더블 생성, 동작 검증, 스텁 설정을 전문으로 하는 NSubstitute 테스팅 전문가. 인터페이스 모킹, Received 검증, Returns 설정, Arg 매칭에 능통. 단위 테스트 작성, 의존성 격리, TDD 개발, 테스트 가능한 아키텍처 구축 시 적극적으로 사용.
---

# Unity NSubstitute - Unity용 모킹 프레임워크

## 개요

NSubstitute는 .NET용 모킹 프레임워크로, 테스트에서 의존성을 대체하는 테스트 더블을 쉽게 생성할 수 있습니다.

**핵심 주제**:
- 인터페이스 및 클래스 모킹
- Returns를 통한 반환값 설정
- Received를 통한 호출 검증
- Arg 매칭 (인자 검증)
- 예외 발생 시뮬레이션
- 부분 대체 (Partial Substitutes)

**학습 경로**: 모킹 기초 → Returns/Received → Arg 매칭 → 고급 패턴

## 빠른 시작

```csharp
using NUnit.Framework;
using NSubstitute;

// 테스트할 인터페이스
public interface IPlayerService
{
    int GetPlayerHealth(int playerId);
    void SavePlayerData(int playerId, string data);
    bool IsPlayerAlive(int playerId);
}

// 테스트할 클래스
public class GameManager
{
    private readonly IPlayerService mPlayerService;

    public GameManager(IPlayerService playerService)
    {
        mPlayerService = playerService;
    }

    public string GetPlayerStatus(int playerId)
    {
        var health = mPlayerService.GetPlayerHealth(playerId);
        return health > 0 ? $"체력: {health}" : "사망";
    }

    public void SaveGame(int playerId)
    {
        mPlayerService.SavePlayerData(playerId, "game_data");
    }
}

// 단위 테스트
[TestFixture]
public class GameManagerTests
{
    [Test]
    public void GetPlayerStatus_WhenHealthPositive_ReturnsHealthMessage()
    {
        // Arrange - 모킹 설정
        var mockPlayerService = Substitute.For<IPlayerService>();
        mockPlayerService.GetPlayerHealth(1).Returns(100);

        var manager = new GameManager(mockPlayerService);

        // Act - 실행
        var status = manager.GetPlayerStatus(1);

        // Assert - 검증
        Assert.AreEqual("체력: 100", status);
    }

    [Test]
    public void SaveGame_CallsSavePlayerData()
    {
        // Arrange
        var mockPlayerService = Substitute.For<IPlayerService>();
        var manager = new GameManager(mockPlayerService);

        // Act
        manager.SaveGame(1);

        // Assert - 메서드 호출 검증
        mockPlayerService.Received(1).SavePlayerData(1, "game_data");
    }
}
```

## 핵심 개념

### 모킹 vs 스텁
- **Stub (스텁)**: 미리 정의된 값을 반환 (Returns 사용)
- **Mock (모킹)**: 호출 여부와 방식을 검증 (Received 사용)

### 반환값 설정
- **Returns**: 단일 반환값
- **ReturnsForAnyArgs**: 모든 인자에 대해 동일 반환
- **Returns with callback**: 동적 반환값 생성

### 호출 검증
- **Received()**: 정확히 한 번 호출
- **Received(n)**: 정확히 n번 호출
- **DidNotReceive()**: 호출되지 않음 검증

### 인자 매칭
- **Arg.Any<T>()**: 모든 값 매칭
- **Arg.Is<T>(predicate)**: 조건 매칭
- **Arg.Do<T>(action)**: 인자로 액션 수행

## 참고 문서

### [NSubstitute 모범 사례](references/nsubstitute-patterns.md)
핵심 모킹 패턴:
- Returns 패턴 (단일, 시퀀스, 콜백)
- Received 검증 패턴
- Arg 매칭 전략
- 예외 및 이벤트 모킹

### [NSubstitute 통합 패턴](references/nsubstitute-integrations.md)
고급 통합:
- VContainer 테스트 통합
- UniTask 비동기 테스트
- MonoBehaviour 테스트
- 통합 테스트 패턴

### [NSubstitute 공식 사이트](https://nsubstitute.github.io/)

## 모범 사례

1. **인터페이스로 추상화**: 모킹은 인터페이스에 대해 수행
2. **Arrange-Act-Assert 패턴**: 명확한 테스트 구조
3. **최소 모킹**: 필요한 것만 모킹
4. **Received는 마지막**: 모든 Act 후에 검증
5. **명확한 테스트 이름**: 무엇을_언제_어떻게 형식
6. **하나의 Assert**: 각 테스트는 한 가지만 검증
7. **SetUp/TearDown 활용**: 공통 설정 재사용
