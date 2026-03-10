---
name: unity-lightweight-ddd

description: Unity 게임 개발을 위한 경량 Domain Driven Design 방식.
  게임플레이 로직과 Unity 런타임 시스템을 분리하되, 안전한 Unity 데이터
  타입 사용은 허용한다.

agent: darknaku:unity-developer

allowed-tools:
    - Read
    - Write
    - Edit
    - Glob
    - Grep
    - Bash

user-invocable: false
---

# Unity 경량 DDD 스킬

이 스킬은 Unity 프로젝트에서 사용할 수 있는 **실용적인 Domain Driven
Design 방식**을 적용합니다.

엄격한 엔터프라이즈 DDD 구조를 따르기보다는 다음 목표에 집중합니다.

- 게임플레이 규칙을 분리
- 규칙을 칙여 EditMode에서 테스트 가능하게 유지
- EditMode와 PlayMode
- 빠른 개발 반복 유지
- 게임 로직의 단위 테스트 가능

------------------------------------------------------------------------

# 아키텍처 철학

Unity 게임 개발에서는 지나치게 엄격한 아키텍처보다 **유연한 구조가 더
중요합니다.**

따라서 이 구조는 코드를 두 영역으로만 나눕니다.

Core (게임 규칙)\
Presentation (Unity 런타임 연동)

의도적으로 단순한 구조입니다.

------------------------------------------------------------------------

# 레이어 구조

## Shared

Core와 Presentation 모두에서 사용할 수 있는 공통 코드입니다.

예시
-   유틸리티 함수
-   공용 데이터 타입
-   상수

## Core 

게임플레이 규칙과 로직을 포함하며 DDD에서 Domain과 Application 레이어의 역할을 모두 포함합니다.

예시

-   전투 계산
-   이동 규칙
-   카드 효과
-   인벤토리 규칙
-   성장 시스템

Core는 가능한 한 Unity와 독립적이어야 하지만 **테스트에 문제가 없는 Unity 데이터 타입과 정적 유틸리티 함수 사용은 허용됩니다.**

------------------------------------------------------------------------

## Presentation

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

Presentation은 Core 로직을 호출하는 역할을 합니다.

------------------------------------------------------------------------

# Core에서 허용되는 Unity 사용

Core 레이어에서는 Shared 레이어와 **Unity의 데이터 타입과 정적 유틸리티 함수 사용이 허용됩니다.**

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

# Core에서 금지되는 Unity 사용

Core 레이어에서는 **Unity 런타임 객체를 사용하면 안 됩니다.**

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

    /Scripts
        /Shared
        /Core
            /Data
            /Logic
            /UseCases

        /Presentation
            /LifetimeScopes
            /Services
            /Managers
            /ScriptableObjects
            /SceneHandlers
            /Presenters
            /Views

------------------------------------------------------------------------

# Data

Data는 **작고 변경되지 않는 데이터 객체**입니다.

예시

Health\
Damage\
Score\
Range

예시 코드

    public struct DamageData
    {
        public int Value;

        public DamageData(int value)
        {
            Value = Mathf.Max(0, value);
        }
    }

------------------------------------------------------------------------

# Logic

Logic은 게임 상태와 행동을 표현합니다.

예시

Player\
Enemy\
Card\
Inventory\
GridCell

Logic의 역할

-   상태 보관
-   게임 규칙 적용
-   행동 제공

좋은 예

    enemy.TakeDamage(damage)

나쁜 예

    enemyGameObject.SetActive(false)

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

# Presentation 역할

Presentation은 Unity와 Core을 연결합니다.

예시

-   MonoBehaviour 어댑터
-   Input 시스템
-   UI 컨트롤러
-   애니메이션
-   사운드
-   VFX

Presentation은 Unity 이벤트를 Core 로직으로 전달합니다.

------------------------------------------------------------------------

# 데이터 흐름

    Unity Input
    ↓
    Presenter
    ↓
    UseCase
    ↓
    Core
    ↓
    Presenter
    ↓
    View

------------------------------------------------------------------------

# 동작 예시

플레이어가 공격 버튼을 누른 경우

    Unity Input System
    ↓
    PlayUIPresenter
    ↓
    AttackEnemyUseCase
    ↓
    Enemy.TakeDamage() : 체력 변경 이벤트 발생
    ↓
    EnemyUIPresenter : 체력 변경 이벤트 수신
    ↓
    EnemyUIView : 체력 바 업데이트

------------------------------------------------------------------------

# 테스트 전략

Core 로직은 Presentation 없이도 테스트 가능해야 합니다.

테스트에서 다음 사용은 허용됩니다.

Vector3\
Mathf\
Random

예시 테스트

    CombatResolver.CalculateDamage(attacker, defender)

------------------------------------------------------------------------

# 안티 패턴

Core에서 Uinty 런타임 객체 사용하면 안되며, Presentation에서 로직을 구현하면 안됩니다.

나쁜 예

MonoBehaviour가 전투 데미지를 계산함

좋은 예

MonoBehaviour가 CombatUseCase를 호출함

------------------------------------------------------------------------

# 언제 단순화해야 하는가

모든 기능을 Core으로 옮길 필요는 없습니다.

다음은 Presentation에 있어도 됩니다.

-   UI 로직
-   시각 효과
-   애니메이션 트리거

Core에는 **게임 규칙만 포함해야 합니다.**