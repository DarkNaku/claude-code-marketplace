---
description: Unity 게임 개발을 위한 경량 Domain Driven Design 방식.
  게임플레이 로직과 Unity 런타임 시스템을 분리하되, 안전한 Unity 데이터
  타입 사용은 허용한다.
name: unity-lightweight-ddd
---

# Unity 경량 DDD 스킬

이 스킬은 Unity 프로젝트에서 사용할 수 있는 **실용적인 Domain Driven
Design 방식**을 적용합니다.

엄격한 엔터프라이즈 DDD 구조를 따르기보다는 다음 목표에 집중합니다.

-   게임플레이 규칙을 분리
-   Unity MonoBehaviour를 최대한 단순하게 유지
-   빠른 개발 반복 유지
-   게임 로직의 단위 테스트 가능

------------------------------------------------------------------------

# 아키텍처 철학

Unity 게임 개발에서는 지나치게 엄격한 아키텍처보다 **유연한 구조가 더
중요합니다.**

따라서 이 구조는 코드를 두 영역으로만 나눕니다.

Domain (게임 규칙)\
Infrastructure (Unity 런타임 연동)

의도적으로 단순한 구조입니다.

------------------------------------------------------------------------

# 레이어 구조

## Domain

게임플레이 규칙과 로직을 포함합니다.

예시

-   전투 계산
-   이동 규칙
-   카드 효과
-   인벤토리 규칙
-   성장 시스템

Domain은 가능한 한 Unity와 독립적이어야 하지만\
**테스트에 문제가 없는 Unity 데이터 타입 사용은 허용됩니다.**

------------------------------------------------------------------------

## Infrastructure

Unity 런타임과 관련된 코드를 포함합니다.

예시

-   MonoBehaviour
-   GameObject 생명주기
-   Scene 관리
-   Animation
-   UI
-   Input
-   Addressables
-   Networking
-   Persistence

Infrastructure는 Domain 로직을 호출하는 역할을 합니다.

------------------------------------------------------------------------

# Domain에서 허용되는 Unity 사용

Domain 레이어에서는 **Unity의 데이터 타입과 정적 유틸리티 함수 사용이
허용됩니다.**

허용 예시

Vector2\
Vector3\
Quaternion\
Bounds\
Mathf\
Random\
Color

예시 코드

    Vector3 direction = (target - position).normalized;
    float distance = Vector3.Distance(a, b);
    float damage = Mathf.Max(1, baseDamage - defense);

이러한 타입들은 사실상 **데이터 구조 또는 수학 유틸리티**입니다.

------------------------------------------------------------------------

# Domain에서 금지되는 Unity 사용

Domain 레이어에서는 **Unity 런타임 객체를 사용하면 안 됩니다.**

금지 예시

MonoBehaviour\
GameObject\
Transform\
ScriptableObject\
Component\
SceneManager\
Addressables\
Physics 콜백\
Coroutine

잘못된 예

    enemyGameObject.transform.position

올바른 예

    Enemy.Move(Vector3 newPosition)

------------------------------------------------------------------------

# 권장 폴더 구조

    /Game
        /Domain
            /Entities
            /ValueObjects
            /Services
            /UseCases

        /Infrastructure
            /Unity
            /Input
            /Persistence
            /Adapters

------------------------------------------------------------------------

# Entity

Entity는 게임 상태와 행동을 표현합니다.

예시

Player\
Enemy\
Card\
Inventory\
GridCell

Entity의 역할

-   상태 보관
-   게임 규칙 적용
-   행동 제공

좋은 예

    enemy.TakeDamage(damage)

나쁜 예

    enemyGameObject.SetActive(false)

------------------------------------------------------------------------

# Value Object

Value Object는 **작고 변경되지 않는 데이터 객체**입니다.

예시

Health\
Damage\
Score\
Range

예시 코드

    public struct Damage
    {
        public int Value;

        public Damage(int value)
        {
            Value = Mathf.Max(0, value);
        }
    }

------------------------------------------------------------------------

# Domain Service

Domain Service는 **여러 Entity가 관련된 로직**을 처리합니다.

예시

CombatResolver\
PathfindingLogic\
CardEffectResolver\
DamageCalculator

예시 코드

    damage = CombatResolver.Calculate(attacker, defender);

------------------------------------------------------------------------

# UseCase

UseCase는 **플레이어 행동 또는 게임 흐름**을 표현합니다.

예시

MoveCharacter\
AttackEnemy\
PlayCard\
EndTurn

예시 코드

    AttackEnemyUseCase.Execute(player, enemy);

------------------------------------------------------------------------

# Infrastructure 역할

Infrastructure는 Unity와 Domain을 연결합니다.

예시

-   MonoBehaviour 어댑터
-   Input 시스템
-   UI 컨트롤러
-   애니메이션
-   사운드
-   VFX

Infrastructure는 Unity 이벤트를 Domain 로직으로 전달합니다.

------------------------------------------------------------------------

# Adapter 패턴

Unity 컴포넌트는 Adapter 역할을 합니다.

    PlayerInput (Unity)
    ↓
    MoveCharacterAdapter
    ↓
    MoveCharacterUseCase
    ↓
    PlayerEntity

------------------------------------------------------------------------

# 데이터 흐름

    Unity Input
    ↓
    Adapter
    ↓
    UseCase
    ↓
    Domain Logic
    ↓
    결과 반환
    ↓
    Unity 화면 반영

------------------------------------------------------------------------

# 동작 예시

플레이어가 공격 버튼을 누른 경우

    Unity Input System
    ↓
    AttackInputAdapter
    ↓
    AttackEnemyUseCase
    ↓
    CombatResolver
    ↓
    Enemy.TakeDamage()
    ↓
    Infrastructure에서 UI / 애니메이션 처리

------------------------------------------------------------------------

# 테스트 전략

Domain 로직은 Unity 없이도 테스트 가능해야 합니다.

테스트에서 다음 사용은 허용됩니다.

Vector3\
Mathf\
Random

예시 테스트

    CombatResolver.CalculateDamage(attacker, defender)

------------------------------------------------------------------------

# 안티 패턴

게임 규칙을 MonoBehaviour 안에 넣지 마십시오.

나쁜 예

MonoBehaviour가 전투 데미지를 계산함

좋은 예

MonoBehaviour가 CombatUseCase를 호출함

------------------------------------------------------------------------

# 언제 단순화해야 하는가

모든 기능을 Domain으로 옮길 필요는 없습니다.

다음은 Infrastructure에 있어도 됩니다.

-   UI 로직
-   시각 효과
-   애니메이션 트리거

Domain에는 **게임 규칙만 포함해야 합니다.**
