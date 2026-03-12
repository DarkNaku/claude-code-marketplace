# TDD 패턴 가이드

Unity에서 실전 TDD 패턴을 위한 심층 참조 문서.

## 가짜 구현 (Fake It)

테스트를 통과시키기 위해 먼저 상수를 반환하고, 나중에 실제 구현으로 대체.

### 예제: 인벤토리 시스템

```csharp
// RED - 테스트 작성
[Test]
public void AddItem_EmptyInventory_CountBecomesOne()
{
    var inventory = new Inventory();
    inventory.AddItem(new Item("Sword"));
    Assert.AreEqual(1, inventory.Count);
}

// GREEN - 가짜 구현 (상수 반환)
public class Inventory
{
    public int Count => 1; // 하드코딩!
    public void AddItem(Item item) { }
}
// 테스트 통과 ✓

// RED - 다음 테스트로 일반화 강제
[Test]
public void AddItem_TwoItems_CountBecomesTwo()
{
    var inventory = new Inventory();
    inventory.AddItem(new Item("Sword"));
    inventory.AddItem(new Item("Shield"));
    Assert.AreEqual(2, inventory.Count);
}

// GREEN - 실제 구현으로 대체
public class Inventory
{
    private List<Item> mItems = new();
    public int Count => mItems.Count;

    public void AddItem(Item item)
    {
        mItems.Add(item);
    }
}
// 모든 테스트 통과 ✓
```

**언제 사용**: 구현이 명확하지 않을 때, 작은 단계로 진행하고 싶을 때

## 명백한 구현 (Obvious Implementation)

구현이 명확하면 바로 실제 코드 작성.

### 예제: 데미지 계산

```csharp
// RED - 테스트 작성
[Test]
public void CalculateDamage_BaseDamage10_Returns10()
{
    var calculator = new DamageCalculator();
    var damage = calculator.Calculate(baseDamage: 10);
    Assert.AreEqual(10, damage);
}

// GREEN - 명백한 구현을 바로 작성
public class DamageCalculator
{
    public int Calculate(int baseDamage)
    {
        return baseDamage; // 명백함!
    }
}
// 테스트 통과 ✓

// RED - 다음 테스트
[Test]
public void CalculateDamage_WithCritical_DoublesBaseDamage()
{
    var calculator = new DamageCalculator();
    var damage = calculator.Calculate(baseDamage: 10, isCritical: true);
    Assert.AreEqual(20, damage);
}

// GREEN - 명백한 구현
public int Calculate(int baseDamage, bool isCritical = false)
{
    return isCritical ? baseDamage * 2 : baseDamage;
}
// 모든 테스트 통과 ✓
```

**언제 사용**: 구현이 명확하고 자신감이 있을 때

## 삼각측량 (Triangulation)

두 개 이상의 테스트로 일반화를 강제.

### 예제: 레벨 업 시스템

```csharp
// RED - 첫 번째 테스트
[Test]
public void GetRequiredExp_Level1_Returns100()
{
    var calculator = new LevelCalculator();
    Assert.AreEqual(100, calculator.GetRequiredExp(level: 1));
}

// GREEN - 가짜 구현
public class LevelCalculator
{
    public int GetRequiredExp(int level) => 100;
}

// RED - 두 번째 테스트로 일반화 강제
[Test]
public void GetRequiredExp_Level2_Returns200()
{
    var calculator = new LevelCalculator();
    Assert.AreEqual(200, calculator.GetRequiredExp(level: 2));
}

// GREEN - 일반화된 구현
public int GetRequiredExp(int level)
{
    return level * 100;
}

// RED - 세 번째 테스트로 공식 확인
[Test]
public void GetRequiredExp_Level5_Returns500()
{
    var calculator = new LevelCalculator();
    Assert.AreEqual(500, calculator.GetRequiredExp(level: 5));
}
// 모든 테스트 통과 ✓ - 공식 확정
```

**언제 사용**: 올바른 일반화가 불명확할 때

## 테스트 목록 작성

시작 전에 구현할 기능 목록 작성.

### 예제: 체력 시스템 테스트 목록

```csharp
/*
테스트 목록:
☐ 새 플레이어는 최대 체력으로 시작
☐ 데미지를 받으면 체력 감소
☐ 체력은 0 이하로 내려가지 않음
☐ 체력이 0이 되면 IsAlive는 false
☐ 힐을 받으면 체력 증가
☐ 힐로 최대 체력을 초과하지 않음
☐ 죽은 플레이어는 힐을 받을 수 없음
*/

// ✓ 새 플레이어는 최대 체력으로 시작
[Test]
public void NewPlayer_HasMaxHealth()
{
    var player = new Player(maxHealth: 100);
    Assert.AreEqual(100, player.Health);
    Assert.AreEqual(100, player.MaxHealth);
}

// 구현...
public class Player
{
    public int Health { get; private set; }
    public int MaxHealth { get; }

    public Player(int maxHealth)
    {
        MaxHealth = maxHealth;
        Health = maxHealth;
    }
}

// ✓ 데미지를 받으면 체력 감소
[Test]
public void TakeDamage_ReducesHealth()
{
    var player = new Player(100);
    player.TakeDamage(30);
    Assert.AreEqual(70, player.Health);
}

public void TakeDamage(int damage)
{
    Health -= damage;
}

// 계속 목록의 항목들을 하나씩 체크...
```

**이점**:
- 전체 작업 범위 파악
- 다음에 할 일 명확
- 진행 상황 추적

## 작은 단계로 진행

한 번에 하나의 작은 변경만 수행.

### 예제: 퀘스트 시스템

```csharp
// 큰 기능을 작은 단계로 분해

// 1단계: 퀘스트 생성
[Test]
public void CreateQuest_HasTitle()
{
    var quest = new Quest("드래곤 토벌");
    Assert.AreEqual("드래곤 토벌", quest.Title);
}

public class Quest
{
    public string Title { get; }
    public Quest(string title) => Title = title;
}

// 2단계: 퀘스트 진행 상태
[Test]
public void NewQuest_IsNotCompleted()
{
    var quest = new Quest("드래곤 토벌");
    Assert.IsFalse(quest.IsCompleted);
}

public bool IsCompleted { get; private set; }

// 3단계: 퀘스트 완료
[Test]
public void Complete_SetsIsCompletedTrue()
{
    var quest = new Quest("드래곤 토벌");
    quest.Complete();
    Assert.IsTrue(quest.IsCompleted);
}

public void Complete() => IsCompleted = true;

// 4단계: 완료된 퀘스트는 다시 완료 불가
[Test]
public void Complete_AlreadyCompleted_ThrowsException()
{
    var quest = new Quest("드래곤 토벌");
    quest.Complete();
    Assert.Throws<InvalidOperationException>(() => quest.Complete());
}

public void Complete()
{
    if (IsCompleted)
        throw new InvalidOperationException("이미 완료된 퀘스트");
    IsCompleted = true;
}

// 각 단계마다 테스트 실행, 커밋
```

## 하나에서 여럿으로 (One to Many)

먼저 하나의 경우로 구현하고, 컬렉션으로 일반화.

### 예제: 파티 시스템

```csharp
// RED - 파티원 한 명
[Test]
public void AddMember_OneMember_CountIsOne()
{
    var party = new Party();
    party.AddMember(new Player("전사"));
    Assert.AreEqual(1, party.MemberCount);
}

// GREEN - 간단한 구현
public class Party
{
    private Player mMember; // 하나만!
    public int MemberCount => mMember != null ? 1 : 0;

    public void AddMember(Player player)
    {
        mMember = player;
    }
}

// RED - 두 명으로 일반화 강제
[Test]
public void AddMember_TwoMembers_CountIsTwo()
{
    var party = new Party();
    party.AddMember(new Player("전사"));
    party.AddMember(new Player("마법사"));
    Assert.AreEqual(2, party.MemberCount);
}

// GREEN - 컬렉션으로 리팩토링
public class Party
{
    private List<Player> mMembers = new();
    public int MemberCount => mMembers.Count;

    public void AddMember(Player player)
    {
        mMembers.Add(player);
    }
}
```

## 예외 테스트 패턴

예외도 동작의 일부 - 먼저 테스트.

### 예제: 아이템 사용 제한

```csharp
// RED - 정상 케이스
[Test]
public void UsePotion_HasPotion_RestoresHealth()
{
    var inventory = new Inventory();
    inventory.AddItem(new Potion());
    var player = new Player(health: 50);

    inventory.UsePotion(player);

    Assert.AreEqual(100, player.Health);
}

// GREEN
public class Inventory
{
    private List<Item> mItems = new();

    public void AddItem(Item item) => mItems.Add(item);

    public void UsePotion(Player player)
    {
        var potion = mItems.OfType<Potion>().First();
        player.Heal(50);
        mItems.Remove(potion);
    }
}

// RED - 예외 케이스
[Test]
public void UsePotion_NoPotion_ThrowsException()
{
    var inventory = new Inventory();
    var player = new Player(health: 50);

    var exception = Assert.Throws<InvalidOperationException>(
        () => inventory.UsePotion(player));

    StringAssert.Contains("포션이 없습니다", exception.Message);
}

// GREEN
public void UsePotion(Player player)
{
    var potion = mItems.OfType<Potion>().FirstOrDefault();
    if (potion == null)
        throw new InvalidOperationException("포션이 없습니다");

    player.Heal(50);
    mItems.Remove(potion);
}
```

## 중복 제거

테스트가 통과하면 즉시 중복 제거.

### 예제: 스킬 쿨다운

```csharp
// GREEN - 테스트는 통과하지만 중복이 있음
public class Skill
{
    private float mLastUseTime;
    public float Cooldown { get; }

    public Skill(float cooldown)
    {
        Cooldown = cooldown;
        mLastUseTime = -cooldown; // 즉시 사용 가능
    }

    public bool CanUse()
    {
        return Time.time - mLastUseTime >= Cooldown;
    }

    public void Use()
    {
        if (Time.time - mLastUseTime >= Cooldown) // 중복!
        {
            mLastUseTime = Time.time;
            Execute();
        }
    }

    private void Execute() { }
}

// REFACTOR - 중복 제거
public void Use()
{
    if (CanUse()) // 중복 제거!
    {
        mLastUseTime = Time.time;
        Execute();
    }
}
```

## 테스트 데이터 빌더 패턴

복잡한 객체 생성을 간단하게.

### 예제: 캐릭터 생성

```csharp
// 테스트마다 복잡한 설정이 반복됨
[Test]
public void Warrior_HasHighDefense()
{
    var character = new Character
    {
        Name = "전사",
        Level = 10,
        Class = CharacterClass.Warrior,
        Strength = 20,
        Dexterity = 10,
        Intelligence = 5,
        Equipment = new Equipment
        {
            Weapon = new Sword(),
            Armor = new PlateArmor(),
            Accessory = new Ring()
        }
    };

    Assert.Greater(character.Defense, 50);
}

// 테스트 데이터 빌더 패턴
public class CharacterBuilder
{
    private string mName = "테스트";
    private int mLevel = 1;
    private CharacterClass mClass = CharacterClass.Warrior;
    private int mStrength = 10;
    private int mDexterity = 10;
    private int mIntelligence = 10;

    public CharacterBuilder WithName(string name)
    {
        mName = name;
        return this;
    }

    public CharacterBuilder WithLevel(int level)
    {
        mLevel = level;
        return this;
    }

    public CharacterBuilder AsWarrior()
    {
        mClass = CharacterClass.Warrior;
        mStrength = 20;
        mDexterity = 10;
        mIntelligence = 5;
        return this;
    }

    public CharacterBuilder AsMage()
    {
        mClass = CharacterClass.Mage;
        mStrength = 5;
        mDexterity = 10;
        mIntelligence = 20;
        return this;
    }

    public Character Build()
    {
        return new Character
        {
            Name = mName,
            Level = mLevel,
            Class = mClass,
            Strength = mStrength,
            Dexterity = mDexterity,
            Intelligence = mIntelligence,
            Equipment = new Equipment
            {
                Weapon = GetDefaultWeapon(),
                Armor = GetDefaultArmor()
            }
        };
    }

    private Weapon GetDefaultWeapon() => mClass switch
    {
        CharacterClass.Warrior => new Sword(),
        CharacterClass.Mage => new Staff(),
        _ => new Sword()
    };

    private Armor GetDefaultArmor() => mClass switch
    {
        CharacterClass.Warrior => new PlateArmor(),
        CharacterClass.Mage => new Robe(),
        _ => new LeatherArmor()
    };
}

// 테스트가 훨씬 간단해짐
[Test]
public void Warrior_HasHighDefense()
{
    var character = new CharacterBuilder()
        .AsWarrior()
        .WithLevel(10)
        .Build();

    Assert.Greater(character.Defense, 50);
}

[Test]
public void Mage_HasLowDefense()
{
    var character = new CharacterBuilder()
        .AsMage()
        .WithLevel(10)
        .Build();

    Assert.Less(character.Defense, 30);
}
```

## 단언 우선 (Assertion First)

테스트를 작성할 때 Assert부터 시작.

### 예제

```csharp
// 1. 원하는 결과부터 작성 (Assert First)
[Test]
public void CalculateTotalDamage_()
{
    // Assert - 무엇을 검증하고 싶은가?
    Assert.AreEqual(150, totalDamage);

    // Act - 그러려면 무엇을 해야 하는가?
    var totalDamage = calculator.CalculateTotalDamage(attacks);

    // Arrange - 그러려면 무엇이 필요한가?
    var calculator = new DamageCalculator();
    var attacks = new List<Attack>
    {
        new Attack { Damage = 50 },
        new Attack { Damage = 100 }
    };
}

// 2. 위에서 아래로 정리 (AAA 순서)
[Test]
public void CalculateTotalDamage_TwoAttacks_ReturnsSumOfDamage()
{
    // Arrange
    var calculator = new DamageCalculator();
    var attacks = new List<Attack>
    {
        new Attack { Damage = 50 },
        new Attack { Damage = 100 }
    };

    // Act
    var totalDamage = calculator.CalculateTotalDamage(attacks);

    // Assert
    Assert.AreEqual(150, totalDamage);
}
```

**이점**: 테스트의 목적이 먼저 명확해짐

## 격리된 테스트

각 테스트는 다른 테스트와 독립적.

### 예제: 게임 상태

```csharp
// 나쁜 예 - 테스트 간 의존성
public class GameStateTests
{
    private static GameState sGameState; // 공유 상태!

    [Test]
    public void Test1_StartGame()
    {
        sGameState = new GameState();
        sGameState.Start();
        Assert.IsTrue(sGameState.IsRunning);
    }

    [Test]
    public void Test2_PauseGame() // Test1에 의존!
    {
        sGameState.Pause();
        Assert.IsFalse(sGameState.IsRunning);
    }
}

// 좋은 예 - 각 테스트 독립
public class GameStateTests
{
    private GameState mGameState;

    [SetUp]
    public void SetUp()
    {
        mGameState = new GameState(); // 매번 새로 생성
    }

    [Test]
    public void Start_SetsIsRunningTrue()
    {
        mGameState.Start();
        Assert.IsTrue(mGameState.IsRunning);
    }

    [Test]
    public void Pause_RunningGame_SetsIsRunningFalse()
    {
        mGameState.Start(); // 명시적 전제조건
        mGameState.Pause();
        Assert.IsFalse(mGameState.IsRunning);
    }
}
```

## 테스트 냄새와 해결

### 냄새 1: 긴 테스트

```csharp
// 나쁜 예
[Test]
public void ComplexBattle()
{
    var player = new Player(100);
    var enemy1 = new Enemy(50);
    var enemy2 = new Enemy(50);
    var battle = new Battle(player, new[] { enemy1, enemy2 });

    battle.PlayerAttacks(enemy1);
    Assert.AreEqual(40, enemy1.Health);

    battle.EnemyAttacks(0, player);
    Assert.AreEqual(90, player.Health);

    battle.PlayerAttacks(enemy1);
    // ... 계속 ...
}

// 좋은 예 - 하나의 테스트는 하나의 개념
[Test]
public void PlayerAttack_ReducesEnemyHealth()
{
    var player = new Player(100);
    var enemy = new Enemy(50);
    var battle = new Battle(player, new[] { enemy });

    battle.PlayerAttacks(enemy);

    Assert.AreEqual(40, enemy.Health);
}

[Test]
public void EnemyAttack_ReducesPlayerHealth()
{
    var player = new Player(100);
    var enemy = new Enemy(50);
    var battle = new Battle(player, new[] { enemy });

    battle.EnemyAttacks(0, player);

    Assert.AreEqual(90, player.Health);
}
```

### 냄새 2: 테스트 코드 중복

```csharp
// 나쁜 예
[Test]
public void Warrior_Test1()
{
    var character = new Character("전사", CharacterClass.Warrior);
    character.Strength = 20;
    character.Level = 10;
    // ... 테스트
}

[Test]
public void Warrior_Test2()
{
    var character = new Character("전사", CharacterClass.Warrior);
    character.Strength = 20;
    character.Level = 10;
    // ... 다른 테스트
}

// 좋은 예 - 헬퍼 메서드 또는 빌더 사용
private Character CreateWarrior(int level = 10)
{
    return new Character("전사", CharacterClass.Warrior)
    {
        Strength = 20,
        Level = level
    };
}

[Test]
public void Warrior_Test1()
{
    var character = CreateWarrior();
    // ... 테스트
}

[Test]
public void Warrior_Test2()
{
    var character = CreateWarrior();
    // ... 다른 테스트
}
```
