# Unity 구현 템플릿

Feature 작성 시 자주 사용하는 Unity 코드 구조 모음.

---

## 폴더 구조 템플릿

### MVP 단계 폴더 구조

```
Assets/
├── Scripts/
│   ├── Core/                  ← 게임 규칙, 도메인 로직
│   │   ├── GameManager.cs
│   │   └── GameEvents.cs
│   ├── Player/
│   │   ├── PlayerMovement.cs
│   │   ├── PlayerCombat.cs
│   │   └── PlayerLevelSystem.cs
│   ├── Enemy/
│   │   ├── EnemyBase.cs
│   │   └── EnemySlime.cs
│   ├── Weapon/
│   │   ├── WeaponBase.cs
│   │   └── WeaponWhip.cs
│   ├── Systems/               ← 게임 시스템
│   │   ├── SpawnSystem.cs
│   │   ├── ExperienceSystem.cs
│   │   └── ObjectPool.cs
│   ├── Data/                  ← ScriptableObject 정의
│   │   ├── PlayerStatData.cs
│   │   ├── EnemyData.cs
│   │   └── WeaponData.cs
│   └── UI/
│       ├── HUDController.cs
│       └── LevelUpUI.cs
├── ScriptableObjects/         ← ScriptableObject 에셋 파일
│   ├── Player/
│   │   └── PlayerStatData.asset
│   ├── Enemies/
│   │   └── EnemyData_Slime.asset
│   └── Weapons/
│       └── WeaponData_Whip.asset
├── Prefabs/
│   ├── Player.prefab
│   ├── Enemies/
│   │   └── Slime.prefab
│   └── Weapons/
│       └── WhipProjectile.prefab
└── Scenes/
    ├── Boot.unity             ← GameManager 초기화
    └── Game.unity             ← 게임 씬
```

---

## 자주 쓰는 코드 패턴

### 싱글턴 매니저

```csharp
// MonoBehaviour 기반 싱글턴 (씬 간 유지 필요 시)
public class SoundManager : MonoBehaviour
{
    public static SoundManager Instance { get; private set; }

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
}

// 순수 C# 싱글턴 (MonoBehaviour 불필요 시)
public class DataRepository
{
    private static DataRepository mInstance;
    public static DataRepository Instance => mInstance ??= new DataRepository();

    private DataRepository() { }
}
```

---

### 오브젝트 풀 (MonoBehaviour 기반)

```csharp
// 풀 매니저 (씬에 배치)
public class ProjectilePool : MonoBehaviour
{
    [SerializeField] private GameObject mPrefab;
    [SerializeField] private int mInitialSize = 20;

    private Queue<GameObject> mPool = new Queue<GameObject>();

    private void Awake()
    {
        for (int i = 0; i < mInitialSize; i++)
            ReturnToPool(CreateNew());
    }

    public GameObject Get(Vector3 position, Quaternion rotation)
    {
        GameObject obj = mPool.Count > 0 ? mPool.Dequeue() : CreateNew();
        obj.transform.SetPositionAndRotation(position, rotation);
        obj.SetActive(true);
        return obj;
    }

    public void ReturnToPool(GameObject obj)
    {
        obj.SetActive(false);
        mPool.Enqueue(obj);
    }

    private GameObject CreateNew()
    {
        GameObject obj = Instantiate(mPrefab, transform);
        obj.SetActive(false);
        return obj;
    }
}

// 투사체 컴포넌트에서 풀로 반환
public class Projectile : MonoBehaviour
{
    [SerializeField] private float mLifetime = 3f;
    private ProjectilePool mPool;

    public void Initialize(ProjectilePool pool)
    {
        mPool = pool;
        StartCoroutine(ReturnAfterDelay());
    }

    private IEnumerator ReturnAfterDelay()
    {
        yield return new WaitForSeconds(mLifetime);
        mPool.ReturnToPool(gameObject);
    }

    private void OnTriggerEnter2D(Collider2D other)
    {
        if (other.TryGetComponent<HealthComponent>(out var health))
        {
            health.TakeDamage(10); // 실제로는 무기 데이터에서 가져옴
            mPool.ReturnToPool(gameObject);
        }
    }
}
```

---

### HP 컴포넌트 + UI 연동

```csharp
// HP 로직
public class HealthComponent : MonoBehaviour
{
    [SerializeField] private int mMaxHp = 100;
    public int mCurrentHp { get; private set; }
    public int mMaxHpValue => mMaxHp;

    public event System.Action<int, int> OnHpChanged;
    public event System.Action OnDeath;

    private bool mIsDead;

    private void Awake() => mCurrentHp = mMaxHp;

    public void TakeDamage(int damage)
    {
        if (mIsDead) return;
        mCurrentHp = Mathf.Max(0, mCurrentHp - damage);
        OnHpChanged?.Invoke(mCurrentHp, mMaxHp);
        if (mCurrentHp == 0)
        {
            mIsDead = true;
            OnDeath?.Invoke();
        }
    }

    public void Heal(int amount)
    {
        if (mIsDead) return;
        mCurrentHp = Mathf.Min(mMaxHp, mCurrentHp + amount);
        OnHpChanged?.Invoke(mCurrentHp, mMaxHp);
    }
}

// HP UI 연동 (HUD에서 사용)
public class HpBarUI : MonoBehaviour
{
    [SerializeField] private Slider mSlider;
    [SerializeField] private HealthComponent mTarget;

    private void OnEnable()
    {
        mTarget.OnHpChanged += UpdateBar;
        UpdateBar(mTarget.mCurrentHp, mTarget.mMaxHpValue);
    }

    private void OnDisable() => mTarget.OnHpChanged -= UpdateBar;

    private void UpdateBar(int current, int max)
    {
        mSlider.value = (float)current / max;
    }
}
```

---

### 이동 + 스프라이트 반전 (2D)

```csharp
[RequireComponent(typeof(Rigidbody2D))]
public class MovementController2D : MonoBehaviour
{
    [SerializeField] private float mMoveSpeed = 5f;
    [SerializeField] private SpriteRenderer mSpriteRenderer;

    private Rigidbody2D mRigidbody;

    private void Awake()
    {
        mRigidbody = GetComponent<Rigidbody2D>();
        if (mSpriteRenderer == null)
            mSpriteRenderer = GetComponentInChildren<SpriteRenderer>();
    }

    // 외부에서 방향 벡터를 넘겨 이동 (입력 처리와 분리)
    public void Move(Vector2 direction)
    {
        mRigidbody.linearVelocity = direction.normalized * mMoveSpeed;

        if (direction.x != 0)
            mSpriteRenderer.flipX = direction.x < 0;
    }

    public void SetSpeed(float speed) => mMoveSpeed = speed;
}
```

---

### 쿨다운 유틸리티

```csharp
// 쿨다운 상태를 관리하는 경량 구조체
public struct Cooldown
{
    private float mCooldownTime;
    private float mNextReadyTime;

    public Cooldown(float cooldownTime)
    {
        mCooldownTime = cooldownTime;
        mNextReadyTime = 0f;
    }

    public bool IsReady => Time.time >= mNextReadyTime;

    public void Use() => mNextReadyTime = Time.time + mCooldownTime;

    public bool TryUse()
    {
        if (!IsReady) return false;
        Use();
        return true;
    }

    public float GetProgress() =>
        Mathf.Clamp01(1f - (mNextReadyTime - Time.time) / mCooldownTime);
}

// 사용 예
public class AutoAttack : MonoBehaviour
{
    private Cooldown mAttackCooldown;

    private void Start()
    {
        mAttackCooldown = new Cooldown(0.5f); // 0.5초마다 공격
    }

    private void Update()
    {
        if (mAttackCooldown.TryUse())
            Fire();
    }

    private void Fire() { /* 공격 로직 */ }
}
```

---

### 타이머 컴포넌트

```csharp
public class GameTimer : MonoBehaviour
{
    [SerializeField] private float mTargetSeconds = 900f; // 15분
    public float mElapsedSeconds { get; private set; }
    public bool mIsRunning { get; private set; }

    public event System.Action<float> OnTimerUpdated;  // 매 초
    public event System.Action OnTimerCompleted;

    private float mLastWholeSecond;

    public void StartTimer()
    {
        mElapsedSeconds = 0f;
        mLastWholeSecond = 0f;
        mIsRunning = true;
    }

    public void StopTimer() => mIsRunning = false;

    private void Update()
    {
        if (!mIsRunning) return;

        mElapsedSeconds += Time.deltaTime;

        // 1초마다 이벤트
        if (mElapsedSeconds - mLastWholeSecond >= 1f)
        {
            mLastWholeSecond = Mathf.Floor(mElapsedSeconds);
            OnTimerUpdated?.Invoke(mElapsedSeconds);
        }

        if (mElapsedSeconds >= mTargetSeconds)
        {
            mIsRunning = false;
            OnTimerCompleted?.Invoke();
        }
    }

    // 표시용 문자열 (MM:SS)
    public string GetFormattedTime()
    {
        int mMinutes = (int)mElapsedSeconds / 60;
        int mSeconds = (int)mElapsedSeconds % 60;
        return $"{mMinutes:00}:{mSeconds:00}";
    }
}
```

---

### 씬 전환 로딩 화면

```csharp
public class SceneLoader : MonoBehaviour
{
    public static SceneLoader Instance { get; private set; }

    [SerializeField] private GameObject mLoadingScreen;

    private void Awake()
    {
        if (Instance != null) { Destroy(gameObject); return; }
        Instance = this;
        DontDestroyOnLoad(gameObject);
    }

    public void LoadScene(string sceneName)
    {
        StartCoroutine(LoadSceneAsync(sceneName));
    }

    private IEnumerator LoadSceneAsync(string sceneName)
    {
        mLoadingScreen.SetActive(true);
        AsyncOperation mOperation = UnityEngine.SceneManagement.SceneManager.LoadSceneAsync(sceneName);

        while (!mOperation.isDone)
        {
            // 필요 시 progress 활용
            yield return null;
        }

        mLoadingScreen.SetActive(false);
    }
}
```

---

### 무적 시간 (피격 후 깜빡임)

```csharp
public class InvincibilityHandler : MonoBehaviour
{
    [SerializeField] private float mInvincibleDuration = 1f;
    [SerializeField] private float mBlinkInterval = 0.1f;
    [SerializeField] private SpriteRenderer mSpriteRenderer;

    public bool mIsInvincible { get; private set; }

    public void StartInvincibility()
    {
        if (mIsInvincible) return;
        StartCoroutine(InvincibilityCoroutine());
    }

    private IEnumerator InvincibilityCoroutine()
    {
        mIsInvincible = true;
        float mTimer = 0f;

        while (mTimer < mInvincibleDuration)
        {
            mSpriteRenderer.enabled = !mSpriteRenderer.enabled;
            yield return new WaitForSeconds(mBlinkInterval);
            mTimer += mBlinkInterval;
        }

        mSpriteRenderer.enabled = true;
        mIsInvincible = false;
    }
}

// 피격 처리와 연동
public class PlayerDamageHandler : MonoBehaviour
{
    private HealthComponent mHealth;
    private InvincibilityHandler mInvincibility;

    private void Awake()
    {
        mHealth = GetComponent<HealthComponent>();
        mInvincibility = GetComponent<InvincibilityHandler>();
    }

    private void OnTriggerEnter2D(Collider2D other)
    {
        if (mInvincibility.mIsInvincible) return;
        if (other.TryGetComponent<EnemyBase>(out var enemy))
        {
            mHealth.TakeDamage(enemy.mContactDamage);
            mInvincibility.StartInvincibility();
        }
    }
}
```

---

### 카메라 팔로우

```csharp
public class CameraFollow : MonoBehaviour
{
    [SerializeField] private Transform mTarget;
    [SerializeField] private float mSmoothTime = 0.2f;
    [SerializeField] private Vector3 mOffset = new Vector3(0, 0, -10);

    private Vector3 mVelocity;

    private void LateUpdate()
    {
        if (mTarget == null) return;
        Vector3 mTargetPos = mTarget.position + mOffset;
        transform.position = Vector3.SmoothDamp(
            transform.position, mTargetPos, ref mVelocity, mSmoothTime);
    }
}
```

---

### 간단한 UI 팝업 관리

```csharp
// UI 레이어 관리 (팝업 열기/닫기)
public class UIManager : MonoBehaviour
{
    public static UIManager Instance { get; private set; }

    [SerializeField] private GameObject mLevelUpPanel;
    [SerializeField] private GameObject mGameOverPanel;
    [SerializeField] private GameObject mHUDPanel;

    private void Awake()
    {
        if (Instance != null) { Destroy(gameObject); return; }
        Instance = this;
    }

    public void ShowLevelUpPanel()
    {
        Time.timeScale = 0f;         // 시간 정지
        mLevelUpPanel.SetActive(true);
    }

    public void HideLevelUpPanel()
    {
        mLevelUpPanel.SetActive(false);
        Time.timeScale = 1f;         // 시간 재개
    }

    public void ShowGameOver()
    {
        mHUDPanel.SetActive(false);
        mGameOverPanel.SetActive(true);
    }
}
```

---

## Unity 컴포넌트 설정 치트시트

### Rigidbody2D 서바이벌/액션 게임 설정

```
Body Type: Dynamic
Gravity Scale: 0          ← 탑뷰 게임은 중력 없음
Collision Detection: Continuous  ← 빠른 오브젝트 관통 방지
Freeze Rotation Z: ✓      ← 회전 고정
```

### Collider2D 레이어 설정 권장

```
Layer 설정:
  Player    (Layer 6)
  Enemy     (Layer 7)
  Projectile (Layer 8)
  PickUp    (Layer 9)

Physics2D Matrix (충돌 허용):
  Player    ↔ Enemy      ✓
  Player    ↔ PickUp     ✓
  Projectile ↔ Enemy     ✓
  Enemy     ↔ Enemy      ✗ (적끼리 충돌 없음, 성능 절약)
```

### 카메라 설정 (2D 탑뷰)

```
Projection: Orthographic
Size: 5~8 (해상도에 따라)
Background: 단색 또는 스카이박스 없음
```
