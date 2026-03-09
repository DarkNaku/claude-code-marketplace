# Feature 분해 패턴

장르별 Feature 분해 예시와 공통 Feature 패턴.

---

## 로그라이크 서바이벌 MVP Feature 목록

Vampire Survivors류 게임 기준.

### 에픽 구조

```
[에픽] 플레이어
  F-001 플레이어 이동
  F-002 플레이어 HP 시스템
  F-003 플레이어 피격 및 무적 시간

[에픽] 레벨업 시스템
  F-004 경험치 시스템 (획득, 임계값, 레벨업 이벤트)
  F-005 레벨업 선택 UI (카드 3장 표시, 선택, 시간 정지)
  F-006 스킬 효과 적용 (공격력, 이동속도 등 능력치 증가)

[에픽] 전투
  F-007 WeaponBase 추상 클래스
  F-008 채찍 무기 구현 (WeaponWhip)
  F-009 오브젝트 풀 구현
  F-010 타겟팅 시스템 (가장 가까운 적)

[에픽] 적
  F-011 EnemyBase 구현 (이동, HP, 피격, 사망)
  F-012 기본 슬라임 적 구현
  F-013 경험치 젬 드롭 및 수집
  F-014 적 스폰 시스템 (시간 기반, 수량 증가)

[에픽] 게임 루프
  F-015 타이머 시스템 (생존 목표, 화면 표시)
  F-016 게임 오버 처리 (판정, 화면 전환)
  F-017 재시작 기능

[에픽] UI
  F-018 HUD (HP바, 경험치바, 레벨, 타이머)
  F-019 레벨업 선택 UI (F-005와 연동)
  F-020 게임 오버 화면
```

### 의존성 순서

```
1순위 (의존 없음):
  F-009 오브젝트 풀
  F-001 플레이어 이동

2순위:
  F-002 플레이어 HP          (F-001 이후)
  F-007 WeaponBase           (F-009 이후)
  F-011 EnemyBase            (F-009 이후)

3순위:
  F-003 무적 시간            (F-002 이후)
  F-010 타겟팅 시스템        (F-007, F-011 이후)
  F-012 기본 슬라임          (F-011 이후)
  F-013 경험치 젬            (F-011 이후)

4순위:
  F-008 채찍 무기            (F-007, F-010 이후)
  F-004 경험치 시스템        (F-013 이후)
  F-014 스폰 시스템          (F-012 이후)

5순위:
  F-005 레벨업 선택 UI       (F-004 이후)
  F-015 타이머               (독립)
  F-016 게임 오버            (F-002 이후)

6순위:
  F-006 스킬 효과 적용       (F-005 이후)
  F-017 재시작               (F-016 이후)
  F-018 HUD                  (F-002, F-004, F-015 이후)
```

---

## 덱빌딩 카드게임 MVP Feature 목록

Slay the Spire류 게임 기준.

```
[에픽] 덱 시스템
  F-001 카드 데이터 구조 (CardData ScriptableObject)
  F-002 덱 관리 (드로우 파일, 버리기 파일, 손패)
  F-003 카드 드로우 로직 (턴 시작 시 N장)
  F-004 손패 UI (카드 배치, 드래그)

[에픽] 전투 시스템
  F-005 에너지 시스템 (턴당 3, 카드 코스트 소모)
  F-006 카드 플레이 처리 (대상 선택, 효과 발동)
  F-007 공격 카드 효과 (데미지)
  F-008 방어 카드 효과 (블럭)
  F-009 턴 종료 처리 (손패 버리기, 적 행동)

[에픽] 적 시스템
  F-010 적 데이터 및 HP
  F-011 적 인텐트 시스템 (다음 행동 예고)
  F-012 적 행동 실행 (공격, 강화)

[에픽] 진행 구조
  F-013 전투 씬 구성 (플레이어 HP, 적 1마리)
  F-014 전투 승리/패배 처리
  F-015 전투 보상 (카드 3장 중 1장 선택)
  F-016 전투 연속 진행 (3회 → 완주)

[에픽] UI
  F-017 전투 HUD (플레이어 HP, 블럭, 에너지)
  F-018 적 HP/인텐트 표시
  F-019 카드 선택 보상 UI
```

---

## 타워 디펜스 MVP Feature 목록

```
[에픽] 맵
  F-001 경로 데이터 (웨이포인트 배열)
  F-002 타워 배치 가능 셀 정의

[에픽] 적
  F-003 EnemyBase (경로 이동, HP, 사망)
  F-004 기본 적 1종
  F-005 웨이브 시스템 (적 순차 스폰)
  F-006 기지 도달 처리 (기지 HP 감소)

[에픽] 타워
  F-007 타워 배치 시스템 (클릭, 골드 소모)
  F-008 TowerBase (사거리 내 타겟, 공격 쿨다운)
  F-009 기본 타워 1종 (단일 타겟, 투사체)
  F-010 투사체 시스템 (오브젝트 풀, 데미지)

[에픽] 경제
  F-011 골드 시스템 (적 처치 획득, UI 표시)

[에픽] 게임 루프
  F-012 웨이브 시작/종료 판정
  F-013 게임 오버 (기지 HP 0)
  F-014 스테이지 클리어 (모든 웨이브 완료)
  F-015 HUD (기지 HP, 골드, 웨이브 번호)
```

---

## 공통 Feature 패턴

장르를 막론하고 거의 모든 게임에서 반복되는 Feature 패턴.

### HP 시스템

```csharp
// 어떤 장르든 재사용 가능한 HP 컴포넌트
public class HealthComponent : MonoBehaviour
{
    [SerializeField] private int mMaxHp;
    public int mCurrentHp { get; private set; }

    public event System.Action<int, int> OnHpChanged;   // (현재, 최대)
    public event System.Action OnDeath;

    private void Awake() => mCurrentHp = mMaxHp;

    public void TakeDamage(int damage)
    {
        if (mCurrentHp <= 0) return;
        mCurrentHp = Mathf.Max(0, mCurrentHp - damage);
        OnHpChanged?.Invoke(mCurrentHp, mMaxHp);
        if (mCurrentHp == 0) OnDeath?.Invoke();
    }

    public void Heal(int amount)
    {
        mCurrentHp = Mathf.Min(mMaxHp, mCurrentHp + amount);
        OnHpChanged?.Invoke(mCurrentHp, mMaxHp);
    }
}
```

**완료 조건**:
- [ ] `TakeDamage()` 호출 시 HP 감소
- [ ] HP가 0이 되면 `OnDeath` 이벤트 발생
- [ ] HP가 MaxHp를 초과하지 않음
- [ ] `OnHpChanged` 이벤트로 UI 연동 가능

---

### 오브젝트 풀

```csharp
// 총알, 이펙트, 적에 범용 사용
public class ObjectPool<T> where T : Component
{
    private readonly T mPrefab;
    private readonly Transform mParent;
    private readonly Queue<T> mPool = new Queue<T>();

    public ObjectPool(T prefab, int initialSize, Transform parent = null)
    {
        mPrefab = prefab;
        mParent = parent;
        for (int i = 0; i < initialSize; i++)
            ReturnToPool(CreateNew());
    }

    public T Get()
    {
        T instance = mPool.Count > 0 ? mPool.Dequeue() : CreateNew();
        instance.gameObject.SetActive(true);
        return instance;
    }

    public void ReturnToPool(T instance)
    {
        instance.gameObject.SetActive(false);
        mPool.Enqueue(instance);
    }

    private T CreateNew()
    {
        T instance = Object.Instantiate(mPrefab, mParent);
        instance.gameObject.SetActive(false);
        return instance;
    }
}
```

**완료 조건**:
- [ ] `Get()` 호출 시 비활성 오브젝트를 반환하거나 새로 생성
- [ ] `ReturnToPool()` 호출 시 오브젝트 비활성화
- [ ] 초기 풀 크기 이상의 요청 시 자동 생성
- [ ] 1000회 Get/Return 반복 후 씬에 오브젝트 잔류 없음 (메모리 누수 없음)

---

### 이벤트 버스 (시스템 간 결합 해소)

```csharp
// 시스템 간 직접 참조 없이 통신
public static class GameEvents
{
    public static event System.Action<int> OnPlayerLevelUp;
    public static event System.Action OnPlayerDeath;
    public static event System.Action<int> OnEnemyKilled;   // 경험치 값

    public static void RaisePlayerLevelUp(int newLevel) =>
        OnPlayerLevelUp?.Invoke(newLevel);

    public static void RaisePlayerDeath() =>
        OnPlayerDeath?.Invoke();

    public static void RaiseEnemyKilled(int expValue) =>
        OnEnemyKilled?.Invoke(expValue);
}

// 사용 예
// 경험치 시스템: GameEvents.OnEnemyKilled += OnEnemyKilled;
// UI 시스템: GameEvents.OnPlayerLevelUp += UpdateLevelDisplay;
```

**완료 조건**:
- [ ] 이벤트 발생 시 구독한 모든 리스너가 호출됨
- [ ] 리스너가 없어도 예외 없음 (null 체크)
- [ ] 씬 전환 시 이벤트 구독 해제 (메모리 누수 방지)

---

### ScriptableObject 데이터 패턴

```csharp
// 수치를 코드와 분리하여 코드 수정 없이 밸런싱 가능
[CreateAssetMenu(fileName = "EnemyData_Slime", menuName = "Game/Enemy/SlimeData")]
public class EnemyData : ScriptableObject
{
    [Header("기본 수치")]
    public int mMaxHp = 30;
    public float mMoveSpeed = 2f;
    public int mContactDamage = 10;

    [Header("보상")]
    public int mExpValue = 5;
    public GameObject mExpGemPrefab;

    [Header("비주얼")]
    public Sprite mSprite;
    public RuntimeAnimatorController mAnimatorController;
}
```

**사용 원칙**:
- 적 종류마다 별도 ScriptableObject 에셋 파일 생성
- 런타임에 수치 변경 금지 (읽기 전용으로 취급)
- 밸런싱은 에셋 파일 직접 수정

---

### 씬 전환 및 GameManager

```csharp
// 씬 간 상태 유지 싱글턴
public class GameManager : MonoBehaviour
{
    public static GameManager Instance { get; private set; }

    public int mCurrentLevel { get; private set; }
    public int mTotalScore { get; private set; }

    private void Awake()
    {
        if (Instance != null)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
        DontDestroyOnLoad(gameObject);
    }

    public void StartGame()
    {
        mCurrentLevel = 1;
        mTotalScore = 0;
        LoadGameScene();
    }

    public void GameOver()
    {
        UnityEngine.SceneManagement.SceneManager.LoadScene("GameOver");
    }

    private void LoadGameScene()
    {
        UnityEngine.SceneManagement.SceneManager.LoadScene("Game");
    }
}
```

**완료 조건**:
- [ ] 씬 전환 후에도 GameManager 인스턴스 유지
- [ ] 씬을 다시 로드해도 인스턴스 중복 생성 없음
- [ ] `GameOver()` 호출 시 GameOver 씬으로 전환

---

### 상태 머신 (플레이어/적 AI)

```csharp
// 상태 인터페이스
public interface IState
{
    void Enter();
    void Update();
    void Exit();
}

// 상태 머신
public class StateMachine
{
    private IState mCurrentState;

    public void ChangeState(IState newState)
    {
        mCurrentState?.Exit();
        mCurrentState = newState;
        mCurrentState.Enter();
    }

    public void Update() => mCurrentState?.Update();
}

// 사용 예 (적 AI)
public class EnemyChaseState : IState
{
    private readonly EnemyController mEnemy;
    private readonly Transform mTarget;

    public EnemyChaseState(EnemyController enemy, Transform target)
    {
        mEnemy = enemy;
        mTarget = target;
    }

    public void Enter() { /* 추적 시작 */ }

    public void Update()
    {
        // 매 프레임 플레이어 방향으로 이동
        Vector2 mDirection = (mTarget.position - mEnemy.transform.position).normalized;
        mEnemy.Move(mDirection);
    }

    public void Exit() { /* 추적 종료 */ }
}
```

**사용 기준**:
- 오브젝트의 상태가 3개 이상이고 전환 조건이 복잡할 때
- 단순 bool 플래그로 관리되는 상태가 if-else 지옥을 만들 때

---

## Feature ID 네이밍 규칙

```
F-[단계][번호] [이름]

단계:
  M  = MVP
  A  = 알파
  B  = 베타

예시:
  FM-001 플레이어 이동        ← MVP의 첫 번째 Feature
  FM-002 플레이어 HP 시스템
  FA-001 대시 기능            ← 알파의 첫 번째 Feature
  FB-001 음악 시스템          ← 베타의 첫 번째 Feature
```

단계 구분 없이 단순 번호만 써도 무방 (F-001, F-002 ...).

---

## Feature 완료 기준 작성 가이드

### 나쁜 완료 조건 (검증 불가)
```
❌ "플레이어 이동이 잘 된다"
❌ "적이 자연스럽게 움직인다"
❌ "UI가 보기 좋다"
```

### 좋은 완료 조건 (검증 가능)
```
✅ "WASD 입력 시 8방향 이동 가능, 대각선 이동 속도 정규화 확인"
✅ "씬에 적 10마리 배치 시 모두 독립적으로 플레이어 추적"
✅ "HP 0 이하 시 OnDeath 이벤트 발생, 중복 발생 없음"
✅ "오브젝트 풀 1000회 Get/Return 후 씬에 잔여 오브젝트 없음"
```

### 완료 조건 작성 체크리스트
```
각 완료 조건에 대해:
□ Play 모드에서 직접 확인 가능한가?
□ "됐다/안 됐다"로 이분법적으로 판단 가능한가?
□ 작성자가 아닌 다른 사람도 동일하게 판단할 수 있는가?
```
