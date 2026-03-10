---
name: feature-implementation
description: game-planner가 수립한 개발 계획의 기능 항목을 의미 있는 최소 단위(Feature)로 분해하고, 각 Feature마다 구현 목표·완료 조건·Unity 코드 구조·테스트 방법을 포함한 세부 개발 항목을 작성하는 전문가. 하나의 Feature는 독립적으로 구현·검증·병합 가능한 크기로 정의됨. 개발 착수 직전 작업 항목 상세화, 스프린트 계획, 역할 분담 시 적극적으로 사용.
---

# Feature Implementation - 세부 개발 항목 작성

## 목적

개발 계획의 기능 목록을 **독립적으로 구현하고 검증 가능한 최소 Feature 단위**로 분해하여, 개발자가 바로 착수할 수 있는 세부 항목을 작성합니다.

## Feature 정의 원칙

### 좋은 Feature의 기준 (INVEST)

| 기준 | 설명 | 판단 질문 |
|---|---|---|
| **Independent** | 다른 Feature와 독립적으로 구현 가능 | 이 Feature 혼자 PR을 낼 수 있는가? |
| **Negotiable** | 구현 방식이 협의 가능 | 방법은 바꿔도 목적은 같은가? |
| **Valuable** | 게임에 가치를 더함 | 이것이 없으면 무언가 부족한가? |
| **Estimable** | 구현 기간 추정 가능 | 얼마나 걸릴지 대략 알 수 있는가? |
| **Small** | 1~3일 이내 구현 가능한 크기 | 하루 이상 멈추지 않고 진행 가능한가? |
| **Testable** | 완료 여부를 확인할 수 있음 | 완료 기준이 명확한가? |

### Feature 크기 가이드

```
너무 큰 단위 (에픽, Epic):
  "전투 시스템 구현" → 분해 필요

적절한 단위 (Feature):
  "플레이어 기본 공격 구현 (쿨다운, 데미지, 히트박스)"
  "적 HP 시스템 구현 (피격 판정, 사망 처리)"
  "경험치 젬 드롭 및 수집 구현"

너무 작은 단위 (Task):
  "Animator에 공격 트리거 추가" → Feature 내 세부 작업으로 포함
```

---

## 입력 형태

다음 중 하나를 입력받습니다:

- **game-planner 결과**: 단계별 기능 목록이 담긴 개발 계획
- **기능 목록**: 구현할 기능을 나열한 텍스트
- **단일 기능 설명**: "자동 공격 시스템을 만들고 싶어" 같은 구체적 요청

입력된 기능 목록을 Feature 단위로 분해하고, 각 Feature의 세부 항목을 작성합니다.

---

## Feature 분해 워크플로우

### 1단계: 에픽 → Feature 분해

기능 목록의 각 항목을 INVEST 기준으로 Feature 단위까지 분해합니다.

**분해 예시**:
```
[에픽] 플레이어 시스템
  ├── [Feature] 8방향 이동 구현
  ├── [Feature] HP 시스템 구현 (피격, 사망, UI 연동)
  ├── [Feature] 레벨업 시스템 구현 (경험치 임계값, 레벨업 이벤트)
  └── [Feature] 무적 시간 구현 (피격 후 깜빡임)

[에픽] 전투 시스템
  ├── [Feature] 자동 공격 무기 기반 클래스 구현
  ├── [Feature] 채찍 무기 구현 (WeaponWhip : WeaponBase)
  ├── [Feature] 타겟팅 시스템 구현 (가장 가까운 적 탐색)
  └── [Feature] 오브젝트 풀 구현 (총알/이펙트 재사용)
```

### 2단계: 의존성 순서 정렬

Feature 간 의존성을 파악하여 구현 순서를 결정합니다.

```
의존성 그래프 예시:

ObjectPool          (의존 없음, 먼저 구현)
     ↓
WeaponBase          (ObjectPool 활용)
     ↓
WeaponWhip          (WeaponBase 상속)
     ↓
TargetingSystem     (WeaponWhip이 사용)
```

### 3단계: Feature별 세부 항목 작성

각 Feature마다 아래 형식으로 세부 항목을 작성합니다.

---

## Feature 세부 항목 형식

```markdown
## [Feature ID] Feature 이름

**단계**: MVP / 알파 / 베타
**예상 기간**: [시간 또는 일 수]
**의존 Feature**: [선행 Feature ID 목록]

### 구현 목표
[이 Feature가 완성됐을 때 무엇이 가능해지는가. 1~3문장]

### 완료 조건 (Acceptance Criteria)
- [ ] [검증 가능한 완료 조건 1]
- [ ] [검증 가능한 완료 조건 2]
- [ ] [검증 가능한 완료 조건 3]

### 구현 범위
**포함**:
- [구현할 것 1]
- [구현할 것 2]

**제외** (다음 Feature 또는 이후 단계):
- [이 Feature에서 하지 않을 것]

### Unity 구현 구조

[스크립트 파일 목록 및 역할]

\`\`\`csharp
// 핵심 클래스 골격 코드
\`\`\`

### 세부 작업 목록 (Tasks)
- [ ] [Task 1]
- [ ] [Task 2]
- [ ] [Task 3]

### 테스트 방법
[Unity Editor에서 이 Feature를 어떻게 검증하는가]
```

---

## 출력 예시

아래는 로그라이크 서바이벌 MVP를 Feature로 분해한 예시입니다.

---

### F-001 플레이어 이동

**단계**: MVP | **예상 기간**: 2~4시간 | **의존 Feature**: 없음

#### 구현 목표
플레이어가 WASD 또는 화살표 키로 8방향 이동하며, 이동속도를 외부에서 설정할 수 있다.

#### 완료 조건
- [ ] WASD 입력으로 8방향 이동 가능
- [ ] 이동 중 이동 방향으로 캐릭터 스프라이트가 좌우 반전
- [ ] `PlayerStatData`의 `mMoveSpeed` 값이 실제 이동속도에 반영
- [ ] 화면 경계 밖으로 나가지 않음 (선택: 카메라 팔로우)

#### 구현 범위
**포함**: 입력 처리, Rigidbody2D 이동, 좌우 반전, 이동속도 설정값 연동
**제외**: 대시, 이동속도 업그레이드, 애니메이션 (이후 Feature)

#### Unity 구현 구조

```
Scripts/Player/
  PlayerMovement.cs      ← 이동 로직
  PlayerStatData.cs      ← ScriptableObject (이동속도 등 수치)
```

```csharp
[RequireComponent(typeof(Rigidbody2D))]
public class PlayerMovement : MonoBehaviour
{
    [SerializeField] private PlayerStatData mStatData;

    private Rigidbody2D mRigidbody;
    private Vector2 mMoveInput;
    private SpriteRenderer mSpriteRenderer;

    private void Awake()
    {
        mRigidbody = GetComponent<Rigidbody2D>();
        mSpriteRenderer = GetComponentInChildren<SpriteRenderer>();
    }

    private void Update()
    {
        mMoveInput = new Vector2(
            Input.GetAxisRaw("Horizontal"),
            Input.GetAxisRaw("Vertical")
        ).normalized;

        // 좌우 반전
        if (mMoveInput.x != 0)
            mSpriteRenderer.flipX = mMoveInput.x < 0;
    }

    private void FixedUpdate()
    {
        mRigidbody.linearVelocity = mMoveInput * mStatData.mMoveSpeed;
    }
}

[CreateAssetMenu(fileName = "PlayerStatData", menuName = "Game/PlayerStatData")]
public class PlayerStatData : ScriptableObject
{
    public float mMoveSpeed = 5f;
    public int mMaxHp = 100;
}
```

#### 세부 작업 목록
- [ ] `PlayerStatData` ScriptableObject 생성 및 에셋 파일 추가
- [ ] `PlayerMovement` 스크립트 작성
- [ ] Player 프리팹에 `Rigidbody2D` (Gravity Scale=0), `PlayerMovement` 부착
- [ ] Input System 또는 Input Manager 설정 확인
- [ ] 씬에 Player 배치 후 이동 확인

#### 테스트 방법
Play 모드에서 WASD 입력 시 Player 오브젝트가 상하좌우 및 대각선으로 이동하는지 확인. `PlayerStatData`의 `mMoveSpeed`를 1 / 5 / 10으로 바꾸며 속도 차이 확인.

---

### F-002 적 기본 AI (플레이어 추적)

**단계**: MVP | **예상 기간**: 3~5시간 | **의존 Feature**: F-001

#### 구현 목표
적이 플레이어 방향으로 일정 속도로 이동하며, EnemyData ScriptableObject로 수치를 관리한다.

#### 완료 조건
- [ ] 적이 매 프레임 플레이어 방향을 계산하여 이동
- [ ] 이동 방향에 따라 스프라이트 좌우 반전
- [ ] `EnemyData`의 `mMoveSpeed`가 실제 이동속도에 반영
- [ ] 여러 적이 동시에 독립적으로 플레이어를 추적

#### 구현 범위
**포함**: 플레이어 방향 계산, 이동, 스프라이트 반전, ScriptableObject 수치
**제외**: 적 HP/사망 (F-004), 스폰 시스템 (F-006), 패턴 AI (알파)

#### Unity 구현 구조

```
Scripts/Enemy/
  EnemyMovement.cs
  EnemyData.cs           ← ScriptableObject
```

```csharp
public class EnemyMovement : MonoBehaviour
{
    [SerializeField] private EnemyData mEnemyData;

    private Transform mPlayerTransform;
    private Rigidbody2D mRigidbody;
    private SpriteRenderer mSpriteRenderer;

    private void Awake()
    {
        mRigidbody = GetComponent<Rigidbody2D>();
        mSpriteRenderer = GetComponentInChildren<SpriteRenderer>();
    }

    private void Start()
    {
        // 씬에서 Player 탐색 (MVP에서는 단순화)
        mPlayerTransform = GameObject.FindWithTag("Player").transform;
    }

    private void FixedUpdate()
    {
        if (mPlayerTransform == null) return;

        Vector2 mDirection = (mPlayerTransform.position - transform.position).normalized;
        mRigidbody.linearVelocity = mDirection * mEnemyData.mMoveSpeed;

        if (mDirection.x != 0)
            mSpriteRenderer.flipX = mDirection.x < 0;
    }
}
```

#### 세부 작업 목록
- [ ] `EnemyData` ScriptableObject 생성 (mMoveSpeed, mMaxHp, mContactDamage)
- [ ] `EnemyMovement` 스크립트 작성
- [ ] Enemy 프리팹 구성 (Rigidbody2D, Collider2D, EnemyMovement)
- [ ] Player 태그 설정 확인
- [ ] 씬에 적 3~5마리 배치 후 동시 추적 확인

#### 테스트 방법
Play 모드에서 Player를 이동할 때 모든 적이 Player 방향으로 이동하는지 확인. `EnemyData.mMoveSpeed`를 2 / 4 / 8로 바꾸며 속도 차이 확인.

---

## 참고 문서

### [Feature 분해 패턴](references/feature-breakdown-patterns.md)
장르별 Feature 분해 예시:
- 로그라이크 서바이벌 전체 Feature 목록
- 덱빌딩 카드게임 전체 Feature 목록
- 공통 Feature 패턴 (HP 시스템, 오브젝트 풀, 씬 전환 등)

### [Unity 구현 템플릿](references/unity-implementation-templates.md)
자주 쓰는 Unity 코드 구조:
- ScriptableObject 데이터 패턴
- 오브젝트 풀 패턴
- 이벤트 시스템 패턴
- 상태 머신 패턴
- 싱글턴 매니저 패턴

## 모범 사례

1. **하루 단위 쪼개기**: Feature 하나는 하루 안에 완료 가능한 크기가 이상적
2. **완료 조건 먼저**: 코드 작성 전에 완료 조건을 작성하면 범위 명확화
3. **의존성 최소화**: Feature 간 의존성이 많을수록 병렬 작업이 어려워짐
4. **제외 목록 명시**: "이번엔 하지 않는 것"을 적으면 범위 크리프 방지
5. **골격 코드 우선**: 전체 Feature 골격을 먼저 작성하고 살을 붙이는 방식
6. **테스트 가능한 단위**: 씬에 배치하고 Play 모드에서 바로 확인 가능한 단위로 설계
7. **번호 부여**: F-001, F-002 등 번호를 붙여 의존성과 순서를 명확히 관리
