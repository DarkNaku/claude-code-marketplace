# TDD 워크플로우

Unity에서 Kent Beck 스타일의 TDD 워크플로우 상세 가이드.

## Red-Green-Refactor 사이클

### Red: 실패하는 테스트 작성

**목표**: 다음 작은 기능을 명확히 정의

```csharp
// 1. 테스트 이름으로 의도를 표현
[Test]
public void Weapon_Attack_ReducesEnemyHealth()
{
    // 2. Arrange: 필요한 객체 준비
    var weapon = new Sword(damage: 10);
    var enemy = new Enemy(health: 100);

    // 3. Act: 테스트할 동작 실행
    weapon.Attack(enemy);

    // 4. Assert: 예상 결과 검증
    Assert.AreEqual(90, enemy.Health);
}
```

**실행 → 빨간불 (실패)**
- 컴파일 에러: `Sword` 클래스가 없음
- 또는 테스트 실패: 기대한 동작이 아직 구현되지 않음

**Red 단계의 체크리스트**:
- [ ] 테스트 이름이 동작을 명확히 설명하는가?
- [ ] 한 가지 동작만 테스트하는가?
- [ ] 실패 메시지가 명확한가?

### Green: 테스트를 통과시키는 최소 코드

**목표**: 가능한 가장 빠르게 테스트 통과

```csharp
// 첫 번째 시도: 가장 단순한 구현
public class Sword
{
    private int mDamage;

    public Sword(int damage)
    {
        mDamage = damage;
    }

    public void Attack(Enemy enemy)
    {
        enemy.TakeDamage(mDamage);
    }
}

public class Enemy
{
    public int Health { get; private set; }

    public Enemy(int health)
    {
        Health = health;
    }

    public void TakeDamage(int damage)
    {
        Health -= damage;
    }
}
```

**실행 → 초록불 (통과) ✓**

**Green 단계의 체크리스트**:
- [ ] 모든 테스트가 통과하는가?
- [ ] 코드가 테스트를 통과시키는 최소한인가?
- [ ] 불필요한 기능을 추가하지 않았는가?

**중요**: Green 단계에서는 코드 품질보다 **동작하는 코드**가 우선!

### Refactor: 중복 제거 및 구조 개선

**목표**: 코드 품질 향상 (동작 변경 없음)

```csharp
// 리팩토링 전 (중복 있음)
public class Sword
{
    private int mDamage;

    public Sword(int damage)
    {
        mDamage = damage;
    }

    public void Attack(Enemy enemy)
    {
        enemy.TakeDamage(mDamage);
    }
}

public class Bow
{
    private int mDamage;

    public Bow(int damage)
    {
        mDamage = damage;
    }

    public void Attack(Enemy enemy) // 중복!
    {
        enemy.TakeDamage(mDamage);
    }
}

// 리팩토링 후 (중복 제거)
public interface IWeapon
{
    void Attack(Enemy enemy);
}

public abstract class Weapon : IWeapon
{
    protected int Damage { get; }

    protected Weapon(int damage)
    {
        Damage = damage;
    }

    public virtual void Attack(Enemy enemy)
    {
        enemy.TakeDamage(Damage);
    }
}

public class Sword : Weapon
{
    public Sword(int damage) : base(damage) { }
}

public class Bow : Weapon
{
    public Bow(int damage) : base(damage) { }
}
```

**각 리팩토링 후 테스트 실행 ✓**

**Refactor 단계의 체크리스트**:
- [ ] 모든 테스트가 여전히 통과하는가?
- [ ] 중복이 제거되었는가?
- [ ] 코드가 더 읽기 쉬워졌는가?
- [ ] 각 작은 리팩토링마다 테스트를 실행했는가?

## Tidy First: 구조적 vs 행동적 변경 분리

Kent Beck의 핵심 원칙: **절대 두 가지를 동시에 하지 않는다**

### 구조적 변경 (Tidy)

**정의**: 동작을 변경하지 않고 코드 구조만 개선

**예제**:

```csharp
// BEFORE: 긴 메서드
public class Player
{
    public void Update()
    {
        // 입력 처리
        float horizontal = Input.GetAxis("Horizontal");
        float vertical = Input.GetAxis("Vertical");
        Vector3 direction = new Vector3(horizontal, 0, vertical);

        // 이동
        transform.position += direction * mSpeed * Time.deltaTime;

        // 점프
        if (Input.GetButtonDown("Jump") && mIsGrounded)
        {
            mRigidbody.AddForce(Vector3.up * mJumpForce);
        }

        // 공격
        if (Input.GetButtonDown("Fire"))
        {
            var enemy = FindNearestEnemy();
            if (enemy != null)
            {
                enemy.TakeDamage(mAttackDamage);
            }
        }
    }
}

// 커밋 1: 구조적 변경 (리팩토링)
// 커밋 메시지: "refactor: Extract methods from Player.Update"
public class Player
{
    public void Update()
    {
        HandleMovement();
        HandleJump();
        HandleAttack();
    }

    private void HandleMovement()
    {
        float horizontal = Input.GetAxis("Horizontal");
        float vertical = Input.GetAxis("Vertical");
        Vector3 direction = new Vector3(horizontal, 0, vertical);
        transform.position += direction * mSpeed * Time.deltaTime;
    }

    private void HandleJump()
    {
        if (Input.GetButtonDown("Jump") && mIsGrounded)
        {
            mRigidbody.AddForce(Vector3.up * mJumpForce);
        }
    }

    private void HandleAttack()
    {
        if (Input.GetButtonDown("Fire"))
        {
            var enemy = FindNearestEnemy();
            if (enemy != null)
            {
                enemy.TakeDamage(mAttackDamage);
            }
        }
    }
}

// 테스트: 모두 통과 ✓
// 커밋! (동작 변경 없음)
```

### 행동적 변경 (Feature)

**정의**: 새로운 기능 추가 또는 동작 변경

**예제**:

```csharp
// 커밋 2: 행동적 변경 (새 기능)
// 커밋 메시지: "feat: Add combo attack system"

// RED: 테스트 작성
[Test]
public void Attack_ThreeTimesInTwoSeconds_TriggersCombo()
{
    var player = new Player();
    var enemy = new Enemy(100);

    player.Attack(enemy); // 1회
    player.Attack(enemy); // 2회
    player.Attack(enemy); // 3회 - 콤보 발동!

    Assert.AreEqual(70, enemy.Health); // 10 + 10 + 20 (콤보)
}

// GREEN: 구현
public class Player
{
    private List<float> mAttackTimes = new();
    private const float ComboWindow = 2f;

    public void Attack(Enemy enemy)
    {
        CleanOldAttackTimes();
        mAttackTimes.Add(Time.time);

        int damage = CalculateDamage();
        enemy.TakeDamage(damage);
    }

    private void CleanOldAttackTimes()
    {
        mAttackTimes.RemoveAll(t => Time.time - t > ComboWindow);
    }

    private int CalculateDamage()
    {
        return mAttackTimes.Count >= 3 ? 20 : 10; // 콤보 데미지
    }
}

// 테스트: 통과 ✓
// 커밋! (새 기능 추가)
```

### Tidy First 워크플로우

```
기능 추가를 원함
    ↓
코드가 지저분한가?
    ↓ YES
커밋 1: 구조 정리 (Tidy)
    ↓
커밋 2: 기능 추가 (Feature)
    ↓
코드가 지저분한가?
    ↓ YES
커밋 3: 구조 정리 (Tidy)
```

**중요 규칙**:
1. 구조적 커밋에는 테스트 변경 없음
2. 행동적 커밋에는 새 테스트 추가
3. 절대 두 가지를 한 커밋에 섞지 않음

## 커밋 전략

### 커밋 체크리스트

**다음을 모두 만족할 때만 커밋**:

```bash
# 1. 모든 테스트 통과
Unity -runTests -testPlatform EditMode
# 결과: All tests passed ✓

# 2. 컴파일 경고 없음
# Unity Console: 0 warnings

# 3. 커밋 메시지 작성
git add .
git commit -m "타입: 설명"
```

### 커밋 메시지 규칙

```bash
# 구조적 변경
refactor: Extract CreatePlayer method
refactor: Rename confusing variable name
refactor: Remove duplicate code in DamageCalculator

# 행동적 변경
feat: Add shield blocking mechanic
fix: Player can now jump while moving
test: Add tests for inventory system
```

### 커밋 크기

**작은 커밋 선호**:

```bash
# 좋은 예: 작은 단위 커밋
git log --oneline
a1b2c3d feat: Add Player.TakeDamage method
b2c3d4e test: Add test for TakeDamage
c3d4e5f refactor: Extract health constant
d4e5f6g feat: Add death when health reaches zero
e5f6g7h test: Add test for death

# 나쁜 예: 큰 단위 커밋
git log --oneline
a1b2c3d feat: Complete player combat system
# (모든 기능이 한 커밋에...)
```

## Unity 특화 TDD 패턴

### MonoBehaviour 테스트

```csharp
// RED: MonoBehaviour 테스트
[UnityTest]
public IEnumerator Player_Jump_IncreasesYPosition()
{
    // Arrange
    var playerGO = new GameObject("Player");
    var player = playerGO.AddComponent<PlayerController>();
    var initialY = player.transform.position.y;

    // Act
    player.Jump();
    yield return new WaitForSeconds(0.5f);

    // Assert
    Assert.Greater(player.transform.position.y, initialY);

    // Cleanup
    Object.Destroy(playerGO);
}

// GREEN: 최소 구현
public class PlayerController : MonoBehaviour
{
    private Rigidbody mRigidbody;

    void Awake()
    {
        mRigidbody = GetComponent<Rigidbody>();
        if (mRigidbody == null)
            mRigidbody = gameObject.AddComponent<Rigidbody>();
    }

    public void Jump()
    {
        mRigidbody.AddForce(Vector3.up * 5f, ForceMode.Impulse);
    }
}
```

### 로직과 Unity 분리

```csharp
// 나쁜 예: 로직과 Unity가 섞임
public class Enemy : MonoBehaviour
{
    private int mHealth = 100;

    public void TakeDamage(int damage)
    {
        mHealth -= damage;
        if (mHealth <= 0)
        {
            Destroy(gameObject); // 테스트하기 어려움!
        }
    }
}

// 좋은 예: 로직 분리
public class EnemyHealth // 순수 C# (Edit Mode 테스트 가능)
{
    private int mHealth;
    public bool IsDead => mHealth <= 0;

    public EnemyHealth(int initialHealth)
    {
        mHealth = initialHealth;
    }

    public void TakeDamage(int damage)
    {
        mHealth = Mathf.Max(0, mHealth - damage);
    }
}

public class Enemy : MonoBehaviour // Unity 통합만
{
    private EnemyHealth mHealth;

    void Awake()
    {
        mHealth = new EnemyHealth(100);
    }

    public void TakeDamage(int damage)
    {
        mHealth.TakeDamage(damage);
        if (mHealth.IsDead)
        {
            Destroy(gameObject);
        }
    }
}

// Edit Mode 테스트 (빠름!)
[Test]
public void TakeDamage_LethalDamage_BecomesDeadq()
{
    var health = new EnemyHealth(100);
    health.TakeDamage(100);
    Assert.IsTrue(health.IsDead);
}
```

## 실전 예제: 인벤토리 시스템

### 전체 TDD 과정

```csharp
// === 사이클 1: RED ===
[Test]
public void NewInventory_IsEmpty()
{
    var inventory = new Inventory();
    Assert.AreEqual(0, inventory.Count);
}
// 실행 → 실패 (Inventory 없음)

// === 사이클 1: GREEN ===
public class Inventory
{
    public int Count => 0;
}
// 실행 → 통과 ✓
// 커밋: "feat: Add empty Inventory"

// === 사이클 2: RED ===
[Test]
public void AddItem_IncreasesCount()
{
    var inventory = new Inventory();
    inventory.AddItem(new Item("Sword"));
    Assert.AreEqual(1, inventory.Count);
}
// 실행 → 실패 (AddItem 없음)

// === 사이클 2: GREEN ===
public class Inventory
{
    private int mCount;
    public int Count => mCount;

    public void AddItem(Item item)
    {
        mCount++;
    }
}
// 실행 → 모든 테스트 통과 ✓

// === 사이클 2: REFACTOR ===
// 가짜 mCount 제거하고 실제 리스트로
public class Inventory
{
    private List<Item> mItems = new();
    public int Count => mItems.Count;

    public void AddItem(Item item)
    {
        mItems.Add(item);
    }
}
// 실행 → 모든 테스트 통과 ✓
// 커밋: "feat: Add Inventory.AddItem method"

// === 사이클 3: RED ===
[Test]
public void RemoveItem_DecreasesCount()
{
    var inventory = new Inventory();
    var sword = new Item("Sword");
    inventory.AddItem(sword);
    inventory.RemoveItem(sword);
    Assert.AreEqual(0, inventory.Count);
}
// 실행 → 실패 (RemoveItem 없음)

// === 사이클 3: GREEN ===
public void RemoveItem(Item item)
{
    mItems.Remove(item);
}
// 실행 → 모든 테스트 통과 ✓
// 커밋: "feat: Add Inventory.RemoveItem method"

// === 사이클 4: RED ===
[Test]
public void AddItem_ExceedsCapacity_ThrowsException()
{
    var inventory = new Inventory(capacity: 2);
    inventory.AddItem(new Item("Sword"));
    inventory.AddItem(new Item("Shield"));

    Assert.Throws<InvalidOperationException>(() =>
        inventory.AddItem(new Item("Potion")));
}
// 실행 → 실패 (용량 체크 없음)

// === 사이클 4: GREEN ===
public class Inventory
{
    private List<Item> mItems = new();
    private int mCapacity;

    public int Count => mItems.Count;

    public Inventory(int capacity = int.MaxValue)
    {
        mCapacity = capacity;
    }

    public void AddItem(Item item)
    {
        if (mItems.Count >= mCapacity)
            throw new InvalidOperationException("인벤토리가 가득 참");
        mItems.Add(item);
    }

    public void RemoveItem(Item item)
    {
        mItems.Remove(item);
    }
}
// 실행 → 모든 테스트 통과 ✓
// 커밋: "feat: Add inventory capacity limit"

// === 리팩토링: TIDY ===
// 매직 넘버 제거
public class Inventory
{
    private const int DefaultCapacity = int.MaxValue;
    private List<Item> mItems = new();
    private int mCapacity;

    public int Count => mItems.Count;

    public Inventory(int capacity = DefaultCapacity)
    {
        mCapacity = capacity;
    }

    // ... rest of the code
}
// 실행 → 모든 테스트 통과 ✓
// 커밋: "refactor: Extract default capacity constant"
```

## 빠른 피드백 유지

### 테스트 실행 시간 최소화

```csharp
// 느린 테스트
[UnityTest]
public IEnumerator SlowTest_WaitsForAnimation()
{
    var player = CreatePlayer();
    player.PlayDeathAnimation();
    yield return new WaitForSeconds(5f); // 5초 대기!
    Assert.IsTrue(player.IsDead);
}

// 빠른 테스트
[Test]
public void FastTest_ChecksStateDirectly()
{
    var player = CreatePlayer();
    player.Die(); // 애니메이션 없이 상태만 변경
    Assert.IsTrue(player.IsDead);
}
```

### 테스트 피라미드

```
        /\
       /  \  E2E Tests (느림, 적음)
      /____\
     /      \ Integration Tests (중간)
    /________\
   /          \ Unit Tests (빠름, 많음)
  /__________\
```

**대부분의 테스트는 빠른 Unit Test로!**

## 레거시 코드에 TDD 도입

### 1. 특성화 테스트 (Characterization Test)

```csharp
// 기존 코드의 현재 동작을 테스트로 기록
[Test]
public void LegacyCalculation_CurrentBehavior()
{
    var result = LegacySystem.ComplexCalculation(10, 20);
    Assert.AreEqual(42, result); // 현재 반환값이 무엇이든 기록
}
```

### 2. 커버리지 확보

```csharp
// 변경할 부분 주변에 테스트 추가
[Test]
public void AreaToChange_CurrentBehavior()
{
    // 변경 전 동작 캡처
}
```

### 3. 안전하게 리팩토링

```csharp
// 테스트가 있으므로 안전하게 리팩토링 가능
```

### 4. 새 기능은 TDD로

```csharp
// 새로운 기능부터는 TDD 적용
[Test]
public void NewFeature_Test() { }
```
