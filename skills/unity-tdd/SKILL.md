---
name: unity-tdd

description: Unity에서 Kent Beck 스타일의 테스트 주도 개발(TDD) 전문가. Red-Green-Refactor 사이클, Tidy First 원칙, 점진적 설계에 능통. 작은 단계로 테스트를 먼저 작성하고, 최소한의 구현으로 통과시킨 후, 리팩토링으로 품질을 향상. 모든 기능 개발, 버그 수정, 리팩토링 작업에서 적극적으로 사용.

agent: darknaku:game-developer

allowed-tools:
    - Read
    - Write
    - Edit
    - Glob
    - Grep
    - Bash

user-invocable: false

---

# Unity TDD - 테스트 주도 개발

## 핵심 철학

Kent Beck의 테스트 주도 개발(TDD)과 Tidy First 원칙을 정확하게 따르며 개발을 안내한다.

## TDD 방법론 안내
- 작은 기능 증분을 정의하는 실패하는 테스트부터 작성하세요
- 의미 있는 테스트 이름을 사용하세요
- 테스트 실패는 명확해야 합니다
- 필요한 최소한의 코드만 작성하세요
- 테스트 통과 후에만 리팩토링하세요
- 결함 수정 시 먼저 실패하는 테스트를 작성하세요

## Tidy First 접근법

### 변경 유형
1. 구조적 변경: 동작 변경 없이 코드 구조 개선
2. 행동 변경: 실제 기능 변경

- 두 변경을 한 커밋에 섞지 마세요
- 구조적 변경을 먼저 수행하세요
- 변경 전후 테스트로 동작 동일성을 검증하세요

## 워크플로우
1. **Red (빨강)**: 작은 기능 하나에 대한 실패하는 테스트 작성
2. **Green (초록)**: 테스트를 통과시키는 최소한의 코드 구현
3. 테스트 통과
4. **Refactor (리팩토링)**: 테스트를 유지하며 구조 개선
5. 커밋
6. 반복

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
// 커밋

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
// 커밋
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