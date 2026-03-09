---
name: unity-tdd
description: Unity에서 Kent Beck 스타일의 테스트 주도 개발(TDD) 전문가. Red-Green-Refactor 사이클, Tidy First 원칙, 점진적 설계에 능통. 작은 단계로 테스트를 먼저 작성하고, 최소한의 구현으로 통과시킨 후, 리팩토링으로 품질을 향상. 모든 기능 개발, 버그 수정, 리팩토링 작업에서 적극적으로 사용.
---

# Unity TDD - 테스트 주도 개발

## 핵심 철학

"항상 TDD 사이클을 따른다: Red → Green → Refactor"
테스트를 통과시키는 데 필요한 **최소한의 코드만** 구현한다.

## 개발 사이클

1. **Red (빨강)**: 작은 기능 하나에 대한 실패하는 테스트 작성
2. **Green (초록)**: 테스트를 통과시키는 최소한의 코드 구현
3. **Refactor (리팩토링)**: 테스트를 유지하며 구조 개선

## 핵심 원칙

### 테스트 주도 개발
- 동작을 설명하는 의미 있는 테스트 이름 작성
- 테스트 실패 메시지를 명확하고 유익하게 작성
- 테스트를 만족시키는 데 충분한 만큼만 구현

### Tidy First 접근법
변경사항을 두 가지 범주로 분리:
- **구조적 변경**: 동작 변경 없이 코드 재구성 (리팩토링)
- **행동적 변경**: 새로운 기능 또는 수정된 기능
이 두 가지를 **절대** 하나의 커밋에 섞지 않음.

### 품질 기준
- 중복을 무자비하게 제거
- 명확한 이름으로 의도 표현
- 메서드를 작고 집중적으로 유지
- 가장 단순한 작동 솔루션 사용

## 커밋 규율

다음 조건을 **모두** 만족할 때만 커밋:
- 모든 테스트 통과
- 컴파일러/린터 경고 없음
- 하나의 논리적 단위 변경
- 메시지가 구조적 vs 행동적 변경 명확히 구분

## 워크플로우 패턴

테스트 하나 작성 → 실행 → 구조 개선 → 반복
의미 있는 변경 후에는 항상 모든 테스트 실행.

## 빠른 시작

```csharp
// 1. RED - 실패하는 테스트 작성
[Test]
public void NewPlayer_HasFullHealth()
{
    var player = new Player();
    Assert.AreEqual(100, player.Health);
}
// 실행 → 컴파일 에러 (Player 클래스 없음)

// 2. GREEN - 최소한의 구현
public class Player
{
    public int Health => 100; // 가장 단순한 구현
}
// 실행 → 테스트 통과 ✓

// 3. REFACTOR - 다음 테스트로 이동
// (아직 리팩토링할 것이 없음)

// 4. RED - 다음 테스트
[Test]
public void TakeDamage_ReducesHealth()
{
    var player = new Player();
    player.TakeDamage(30);
    Assert.AreEqual(70, player.Health);
}
// 실행 → 실패 (TakeDamage 메서드 없음)

// 5. GREEN - 최소한의 구현
public class Player
{
    public int Health { get; private set; } = 100;

    public void TakeDamage(int damage)
    {
        Health -= damage;
    }
}
// 실행 → 모든 테스트 통과 ✓

// 6. REFACTOR - 구조 개선 (필요 시)
// (현재는 단순하므로 다음 테스트로)
```

## 참고 문서

### [TDD 패턴 가이드](references/tdd-patterns.md)
실전 TDD 패턴:
- 가짜 구현 → 실제 구현
- 명백한 구현
- 삼각측량
- 테스트 목록 작성
- 작은 단계로 진행

### [TDD 워크플로우](references/tdd-workflow.md)
Unity TDD 워크플로우:
- Red-Green-Refactor 상세 가이드
- Tidy First 실천 방법
- 커밋 전략
- Unity 특화 TDD 패턴
- 실전 예제

## 모범 사례

1. **작은 단계**: 한 번에 하나의 작은 기능만 테스트
2. **빠른 피드백**: 테스트 실행 주기를 짧게 유지 (몇 초 이내)
3. **테스트 먼저**: 구현 전에 항상 테스트 먼저 작성
4. **최소 구현**: 테스트를 통과시키는 가장 단순한 코드
5. **리팩토링 자신감**: 테스트가 있으면 두려움 없이 리팩토링
6. **분리된 커밋**: 리팩토링과 기능 추가를 별도 커밋으로
