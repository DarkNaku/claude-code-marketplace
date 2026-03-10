# ZLinq 패턴 가이드

Unity에서 고급 ZLinq Zero-allocation 패턴을 위한 심층 참조 문서.

## Span 기반 연산자

### 기본 변환

```csharp
using System;
using System.Collections.Generic;
using UnityEngine;

public class SpanBasics : MonoBehaviour
{
    private List<int> mNumbers = new() { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
    private int[] mArray = { 1, 2, 3, 4, 5 };

    void Start()
    {
        // List를 Span으로 변환
        Span<int> listSpan = CollectionsMarshal.AsSpan(mNumbers);

        // Array를 Span으로 변환
        Span<int> arraySpan = mArray.AsSpan();

        // Span 슬라이싱 (zero allocation)
        Span<int> slice = arraySpan.Slice(1, 3); // [2, 3, 4]

        // ReadOnlySpan (읽기 전용)
        ReadOnlySpan<int> readOnly = mArray.AsSpan();
    }
}
```

### Where - 필터링

```csharp
using Cysharp.Threading.Tasks.Linq;

public class FilteringExample : MonoBehaviour
{
    private List<Enemy> mEnemies = new();

    void Update()
    {
        // 기본 필터링
        foreach (var enemy in mEnemies.AsSpan().Where(e => e.Health > 0))
        {
            enemy.Update();
        }

        // 여러 조건
        foreach (var enemy in mEnemies.AsSpan()
            .Where(e => e.Health > 0 && e.Distance < 10f && !e.IsStunned))
        {
            enemy.Attack();
        }

        // 인덱스 기반 필터링
        var span = mEnemies.AsSpan();
        for (int i = 0; i < span.Length; i++)
        {
            if (span[i].Health > 50)
            {
                span[i].Heal(1);
            }
        }
    }
}

public class Enemy
{
    public int Health { get; set; }
    public float Distance { get; set; }
    public bool IsStunned { get; set; }
    public void Update() { }
    public void Attack() { }
    public void Heal(int amount) => Health += amount;
}
```

### Select - 변환

```csharp
public class TransformExample : MonoBehaviour
{
    private List<Player> mPlayers = new();

    void Update()
    {
        // 단순 변환
        foreach (var score in mPlayers.AsSpan().Select(p => p.Score))
        {
            Debug.Log($"점수: {score}");
        }

        // 복잡한 변환
        foreach (var damage in mPlayers.AsSpan()
            .Select(p => p.CalculateDamage(p.Attack, p.Defense)))
        {
            ApplyDamage(damage);
        }

        // 여러 필드 조합
        foreach (var power in mPlayers.AsSpan()
            .Select(p => p.Level * p.Attack + p.Defense))
        {
            Debug.Log($"전투력: {power}");
        }
    }

    void ApplyDamage(int damage) { }
}

public class Player
{
    public int Score { get; set; }
    public int Attack { get; set; }
    public int Defense { get; set; }
    public int Level { get; set; }

    public int CalculateDamage(int atk, int def) => atk - def;
}
```

### SelectMany - 평탄화

```csharp
public class FlattenExample : MonoBehaviour
{
    private List<Squad> mSquads = new();

    void Update()
    {
        // 중첩 컬렉션 평탄화
        // 주의: SelectMany는 allocation이 발생할 수 있으므로 직접 순회 권장
        foreach (var squad in mSquads.AsSpan())
        {
            foreach (var member in squad.Members.AsSpan())
            {
                member.Update();
            }
        }

        // 조건부 평탄화
        foreach (var squad in mSquads.AsSpan().Where(s => s.IsActive))
        {
            foreach (var member in squad.Members.AsSpan().Where(m => m.IsAlive))
            {
                member.Attack();
            }
        }
    }
}

public class Squad
{
    public bool IsActive { get; set; }
    public List<Unit> Members { get; set; } = new();
}

public class Unit
{
    public bool IsAlive { get; set; }
    public void Update() { }
    public void Attack() { }
}
```

## 정렬 연산

### OrderBy / OrderByDescending

```csharp
public class SortingExample : MonoBehaviour
{
    private List<Enemy> mEnemies = new();

    void Update()
    {
        // 오름차순 정렬
        foreach (var enemy in mEnemies.AsSpan().OrderBy(e => e.Health))
        {
            Debug.Log($"체력: {enemy.Health}");
        }

        // 내림차순 정렬
        foreach (var enemy in mEnemies.AsSpan().OrderByDescending(e => e.Threat))
        {
            enemy.Target();
        }

        // 2차 정렬 (ThenBy)
        foreach (var enemy in mEnemies.AsSpan()
            .OrderBy(e => e.Priority)
            .ThenByDescending(e => e.Health))
        {
            ProcessEnemy(enemy);
        }
    }

    void ProcessEnemy(Enemy enemy) { }
}

public class Enemy
{
    public int Health { get; set; }
    public int Threat { get; set; }
    public int Priority { get; set; }
    public void Target() { }
}
```

### 커스텀 정렬

```csharp
using System.Collections.Generic;

public class CustomSortExample : MonoBehaviour
{
    private List<Item> mItems = new();

    void Update()
    {
        // IComparer를 사용한 정렬
        var comparer = new ItemRarityComparer();

        var span = mItems.AsSpan();
        span.Sort(comparer);

        foreach (var item in span)
        {
            Debug.Log($"{item.Name}: {item.Rarity}");
        }
    }
}

public class Item
{
    public string Name { get; set; }
    public int Rarity { get; set; }
}

public class ItemRarityComparer : IComparer<Item>
{
    public int Compare(Item x, Item y)
    {
        return y.Rarity.CompareTo(x.Rarity); // 내림차순
    }
}
```

## 집계 연산

### Sum, Average, Min, Max

```csharp
public class AggregationExample : MonoBehaviour
{
    private List<int> mScores = new();
    private List<float> mDamages = new();
    private List<Player> mPlayers = new();

    void Update()
    {
        // Sum
        var totalScore = mScores.AsSpan().Sum();
        Debug.Log($"총 점수: {totalScore}");

        // Average
        var avgDamage = mDamages.AsSpan().Average();
        Debug.Log($"평균 데미지: {avgDamage}");

        // Min / Max
        var minScore = mScores.AsSpan().Min();
        var maxScore = mScores.AsSpan().Max();
        Debug.Log($"최소/최대: {minScore}/{maxScore}");

        // Select + 집계
        var totalHealth = mPlayers.AsSpan()
            .Select(p => p.Health)
            .Sum();

        var avgLevel = mPlayers.AsSpan()
            .Select(p => p.Level)
            .Average();
    }
}

public class Player
{
    public int Health { get; set; }
    public int Level { get; set; }
}
```

### Count, Any, All

```csharp
public class PredicateExample : MonoBehaviour
{
    private List<Enemy> mEnemies = new();

    void Update()
    {
        // Count - 조건에 맞는 개수
        var aliveCount = mEnemies.AsSpan().Count(e => e.IsAlive);
        Debug.Log($"생존 적: {aliveCount}");

        // Any - 하나라도 있는지
        var hasBoss = mEnemies.AsSpan().Any(e => e.IsBoss);
        if (hasBoss)
            Debug.Log("보스 존재!");

        // All - 모두 만족하는지
        var allDead = mEnemies.AsSpan().All(e => !e.IsAlive);
        if (allDead)
            Debug.Log("모든 적 제거!");

        // First / FirstOrDefault
        var firstBoss = mEnemies.AsSpan().FirstOrDefault(e => e.IsBoss);
        if (firstBoss != null)
            firstBoss.Target();
    }
}

public class Enemy
{
    public bool IsAlive { get; set; }
    public bool IsBoss { get; set; }
    public void Target() { }
}
```

## 성능 최적화 패턴

### 체이닝 최적화

```csharp
public class ChainingOptimization : MonoBehaviour
{
    private List<Enemy> mEnemies = new();

    void Update()
    {
        // 나쁜 예: 중간 ToList() 호출 (allocation 발생)
        // var result = mEnemies.Where(e => e.IsAlive).ToList()
        //                      .Select(e => e.Health).ToList();

        // 좋은 예: 체이닝만 사용 (zero allocation)
        foreach (var health in mEnemies.AsSpan()
            .Where(e => e.IsAlive)
            .Select(e => e.Health))
        {
            ProcessHealth(health);
        }

        // 최적화: 조기 종료
        var count = 0;
        foreach (var enemy in mEnemies.AsSpan()
            .Where(e => e.IsAlive)
            .OrderByDescending(e => e.Threat))
        {
            enemy.Attack();
            if (++count >= 5) break; // 상위 5개만 처리
        }
    }

    void ProcessHealth(int health) { }
}

public class Enemy
{
    public bool IsAlive { get; set; }
    public int Health { get; set; }
    public int Threat { get; set; }
    public void Attack() { }
}
```

### 재사용 가능한 Span

```csharp
public class SpanReuse : MonoBehaviour
{
    private List<float> mPositions = new();
    private Span<float> mCachedSpan;

    void Start()
    {
        // Span은 ref struct이므로 필드로 저장 불가
        // 메서드 내에서만 사용
    }

    void Update()
    {
        // 매번 Span 생성 (하지만 allocation은 없음)
        var span = mPositions.AsSpan();

        ProcessPositions(span);
    }

    void ProcessPositions(Span<float> positions)
    {
        // Span을 매개변수로 전달 가능
        foreach (var pos in positions.Where(p => p > 0))
        {
            Debug.Log(pos);
        }
    }
}
```

### 배열 풀링과 Span

```csharp
using System.Buffers;

public class ArrayPoolExample : MonoBehaviour
{
    private ArrayPool<int> mPool = ArrayPool<int>.Shared;

    void ProcessData(int size)
    {
        // 배열 풀에서 임대
        var array = mPool.Rent(size);

        try
        {
            // Span으로 변환하여 사용
            var span = array.AsSpan(0, size);

            // Zero-allocation 연산
            var sum = span.Where(x => x > 0).Sum();
            Debug.Log($"합계: {sum}");
        }
        finally
        {
            // 풀에 반환
            mPool.Return(array);
        }
    }
}
```

## 일반 LINQ 대체 가이드

### 변환 매핑표

```csharp
public class LinqToZLinq : MonoBehaviour
{
    private List<int> mNumbers = new() { 1, 2, 3, 4, 5 };

    void Examples()
    {
        // LINQ: Where + ToList (allocation)
        // var filtered = mNumbers.Where(x => x > 3).ToList();

        // ZLinq: Where + foreach (zero allocation)
        foreach (var num in mNumbers.AsSpan().Where(x => x > 3))
        {
            Debug.Log(num);
        }

        // LINQ: Select + Sum (allocation)
        // var sum = mNumbers.Select(x => x * 2).Sum();

        // ZLinq: Select + Sum (zero allocation)
        var sum = mNumbers.AsSpan().Select(x => x * 2).Sum();

        // LINQ: OrderBy + First (allocation)
        // var first = mNumbers.OrderBy(x => x).First();

        // ZLinq: OrderBy + First (zero allocation)
        var first = mNumbers.AsSpan().OrderBy(x => x).First();

        // LINQ: Count (조건) (allocation)
        // var count = mNumbers.Count(x => x > 3);

        // ZLinq: Count (조건) (zero allocation)
        var count = mNumbers.AsSpan().Count(x => x > 3);
    }
}
```

### 복잡한 쿼리 변환

```csharp
public class ComplexQueryConversion : MonoBehaviour
{
    private List<Player> mPlayers = new();
    private List<Enemy> mEnemies = new();

    void Update()
    {
        // LINQ 버전 (allocation 多)
        // var topPlayers = mPlayers
        //     .Where(p => p.Score > 100)
        //     .OrderByDescending(p => p.Score)
        //     .Take(10)
        //     .Select(p => p.Name)
        //     .ToList();

        // ZLinq 버전 (zero allocation)
        var topPlayerNames = new List<string>(10); // 최종 결과만 리스트
        var count = 0;

        foreach (var name in mPlayers.AsSpan()
            .Where(p => p.Score > 100)
            .OrderByDescending(p => p.Score)
            .Select(p => p.Name))
        {
            topPlayerNames.Add(name);
            if (++count >= 10) break;
        }

        // 조인 연산 대체
        // LINQ: var result = mPlayers.Join(mEnemies, ...);

        // ZLinq: 중첩 루프 사용
        foreach (var player in mPlayers.AsSpan())
        {
            foreach (var enemy in mEnemies.AsSpan()
                .Where(e => e.TargetId == player.Id))
            {
                ProcessCombat(player, enemy);
            }
        }
    }

    void ProcessCombat(Player player, Enemy enemy) { }
}

public class Player
{
    public int Id { get; set; }
    public string Name { get; set; }
    public int Score { get; set; }
}

public class Enemy
{
    public int TargetId { get; set; }
}
```

## 타입별 최적화 패턴

### 값 타입 (Value Type)

```csharp
public struct Projectile
{
    public Vector3 Position;
    public Vector3 Velocity;
    public float Damage;
    public bool IsActive;
}

public class ProjectileManager : MonoBehaviour
{
    private List<Projectile> mProjectiles = new(1000);

    void Update()
    {
        // 구조체는 복사 비용이 있으므로 ref 사용 고려
        var span = mProjectiles.AsSpan();

        for (int i = 0; i < span.Length; i++)
        {
            ref var projectile = ref span[i];

            if (!projectile.IsActive) continue;

            // ref를 통해 직접 수정 (복사 없음)
            projectile.Position += projectile.Velocity * Time.deltaTime;

            if (projectile.Position.y < 0)
                projectile.IsActive = false;
        }

        // ZLinq로 활성화된 발사체만 처리
        foreach (ref var proj in span.Where(p => p.IsActive))
        {
            CheckCollision(ref proj);
        }
    }

    void CheckCollision(ref Projectile proj) { }
}
```

### 참조 타입 (Reference Type)

```csharp
public class Monster
{
    public int Health { get; set; }
    public Vector3 Position { get; set; }
    public bool IsAlive => Health > 0;

    public void TakeDamage(int damage)
    {
        Health -= damage;
    }
}

public class MonsterManager : MonoBehaviour
{
    private List<Monster> mMonsters = new();

    void Update()
    {
        // 참조 타입은 ref 불필요
        foreach (var monster in mMonsters.AsSpan().Where(m => m.IsAlive))
        {
            monster.TakeDamage(1);
            UpdateMonster(monster);
        }

        // 죽은 몬스터 제거 (역순 순회)
        var span = mMonsters.AsSpan();
        for (int i = span.Length - 1; i >= 0; i--)
        {
            if (!span[i].IsAlive)
                mMonsters.RemoveAt(i);
        }
    }

    void UpdateMonster(Monster monster) { }
}
```

## 성능 측정

### Profiler 활용

```csharp
using UnityEngine.Profiling;
using System.Diagnostics;

public class PerformanceComparison : MonoBehaviour
{
    private List<int> mData = new();

    void Start()
    {
        // 테스트 데이터 생성
        for (int i = 0; i < 10000; i++)
            mData.Add(i);

        MeasureLinq();
        MeasureZLinq();
    }

    void MeasureLinq()
    {
        Profiler.BeginSample("Standard LINQ");
        var sw = Stopwatch.StartNew();

        var result = mData.Where(x => x > 5000).Select(x => x * 2).ToList();
        var sum = result.Sum();

        sw.Stop();
        Profiler.EndSample();

        Debug.Log($"LINQ - 시간: {sw.ElapsedTicks} ticks, GC: {GC.GetTotalMemory(false)}");
    }

    void MeasureZLinq()
    {
        Profiler.BeginSample("ZLinq");
        var sw = Stopwatch.StartNew();

        var sum = mData.AsSpan().Where(x => x > 5000).Select(x => x * 2).Sum();

        sw.Stop();
        Profiler.EndSample();

        Debug.Log($"ZLinq - 시간: {sw.ElapsedTicks} ticks, GC: {GC.GetTotalMemory(false)}");
    }
}
```

### 벤치마크 패턴

```csharp
public class BenchmarkHelper : MonoBehaviour
{
    [ContextMenu("Run Benchmark")]
    void RunBenchmark()
    {
        const int iterations = 1000;
        var data = new List<float>(1000);
        for (int i = 0; i < 1000; i++)
            data.Add(i);

        // Warmup
        for (int i = 0; i < 10; i++)
        {
            _ = data.AsSpan().Where(x => x > 500).Sum();
        }

        // 실제 측정
        var sw = Stopwatch.StartNew();
        for (int i = 0; i < iterations; i++)
        {
            _ = data.AsSpan().Where(x => x > 500).Sum();
        }
        sw.Stop();

        Debug.Log($"ZLinq 평균: {sw.ElapsedMilliseconds / (float)iterations:F4}ms");
    }
}
```
