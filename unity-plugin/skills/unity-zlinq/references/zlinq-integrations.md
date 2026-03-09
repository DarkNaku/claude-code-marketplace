# ZLinq 통합 패턴

ZLinq와 Unity 시스템 및 다른 라이브러리와의 통합 패턴.

## Unity Collections 통합

### NativeArray와 ZLinq

```csharp
using Unity.Collections;
using UnityEngine;

public class NativeArrayIntegration : MonoBehaviour
{
    private NativeArray<Vector3> mPositions;
    private NativeArray<float> mHealthValues;

    void Start()
    {
        mPositions = new NativeArray<Vector3>(1000, Allocator.Persistent);
        mHealthValues = new NativeArray<float>(1000, Allocator.Persistent);

        // NativeArray를 Span으로 변환
        var posSpan = mPositions.AsSpan();
        var healthSpan = mHealthValues.AsSpan();

        // Zero-allocation 연산
        var aliveCount = healthSpan.Count(h => h > 0);
        Debug.Log($"생존 유닛: {aliveCount}");

        // 필터링 + 변환
        for (int i = 0; i < posSpan.Length; i++)
        {
            if (healthSpan[i] > 0)
            {
                posSpan[i] += Vector3.up * Time.deltaTime;
            }
        }

        // 집계 연산
        var avgHealth = healthSpan.Average();
        var maxHealth = healthSpan.Max();
    }

    void OnDestroy()
    {
        if (mPositions.IsCreated) mPositions.Dispose();
        if (mHealthValues.IsCreated) mHealthValues.Dispose();
    }
}
```

### NativeList 활용

```csharp
using Unity.Collections;

public class NativeListExample : MonoBehaviour
{
    private NativeList<int> mScores;

    void Start()
    {
        mScores = new NativeList<int>(Allocator.Persistent);

        // 데이터 추가
        for (int i = 0; i < 100; i++)
            mScores.Add(i * 10);

        // NativeList.AsArray()로 NativeArray 얻기
        var array = mScores.AsArray();
        var span = array.AsSpan();

        // ZLinq 연산
        var highScores = span.Where(s => s > 500).ToArray();

        Debug.Log($"고득점: {highScores.Length}개");
    }

    void OnDestroy()
    {
        if (mScores.IsCreated) mScores.Dispose();
    }
}
```

### NativeHashMap과 병행 사용

```csharp
using Unity.Collections;

public class NativeHashMapIntegration : MonoBehaviour
{
    private NativeHashMap<int, float> mHealthMap;
    private NativeArray<int> mEntityIds;

    void Start()
    {
        mHealthMap = new NativeHashMap<int, float>(100, Allocator.Persistent);
        mEntityIds = new NativeArray<int>(100, Allocator.Persistent);

        // 데이터 초기화
        for (int i = 0; i < 100; i++)
        {
            mEntityIds[i] = i;
            mHealthMap[i] = 100f;
        }

        ProcessEntities();
    }

    void ProcessEntities()
    {
        var idSpan = mEntityIds.AsSpan();

        // 살아있는 엔티티만 처리
        foreach (var id in idSpan.Where(id => mHealthMap.TryGetValue(id, out var health) && health > 0))
        {
            UpdateEntity(id);
        }
    }

    void UpdateEntity(int id)
    {
        if (mHealthMap.TryGetValue(id, out var health))
        {
            mHealthMap[id] = health - 1f;
        }
    }

    void OnDestroy()
    {
        if (mHealthMap.IsCreated) mHealthMap.Dispose();
        if (mEntityIds.IsCreated) mEntityIds.Dispose();
    }
}
```

## ECS (Entity Component System) 통합

### EntityQuery와 ZLinq

```csharp
using Unity.Entities;
using Unity.Collections;
using Unity.Transforms;

public partial class ProcessingSystem : SystemBase
{
    protected override void OnUpdate()
    {
        var entityQuery = GetEntityQuery(typeof(LocalTransform), typeof(HealthComponent));
        var entities = entityQuery.ToEntityArray(Allocator.Temp);

        try
        {
            var span = entities.AsSpan();

            // 엔티티 필터링 및 처리
            foreach (var entity in span.Where(e => EntityManager.HasComponent<HealthComponent>(e)))
            {
                var health = EntityManager.GetComponentData<HealthComponent>(entity);

                if (health.Value > 0)
                {
                    ProcessEntity(entity);
                }
            }
        }
        finally
        {
            entities.Dispose();
        }
    }

    void ProcessEntity(Entity entity) { }
}

public struct HealthComponent : IComponentData
{
    public float Value;
}
```

### DynamicBuffer와 ZLinq

```csharp
using Unity.Entities;

public partial class BufferProcessingSystem : SystemBase
{
    protected override void OnUpdate()
    {
        Entities.ForEach((DynamicBuffer<PathPoint> pathBuffer) =>
        {
            // DynamicBuffer를 Span으로 변환
            var span = pathBuffer.AsNativeArray().AsSpan();

            // ZLinq 연산
            var avgDistance = span
                .Select(p => p.Distance)
                .Average();

            Debug.Log($"평균 거리: {avgDistance}");

            // 특정 조건 찾기
            var hasLongPath = span.Any(p => p.Distance > 100f);

        }).Run();
    }
}

public struct PathPoint : IBufferElementData
{
    public float Distance;
    public Vector3 Position;
}
```

## Job System 통합

### IJob과 Span

```csharp
using Unity.Jobs;
using Unity.Collections;
using Unity.Burst;

[BurstCompile]
public struct ProcessDataJob : IJob
{
    public NativeArray<float> InputData;
    public NativeArray<float> OutputData;

    public void Execute()
    {
        var inputSpan = InputData.AsSpan();
        var outputSpan = OutputData.AsSpan();

        // ZLinq는 Burst와 함께 사용 시 추가 최적화 가능
        int outputIndex = 0;
        foreach (var value in inputSpan.Where(x => x > 0))
        {
            if (outputIndex < outputSpan.Length)
            {
                outputSpan[outputIndex++] = value * 2;
            }
        }
    }
}

public class JobExample : MonoBehaviour
{
    void Start()
    {
        var input = new NativeArray<float>(1000, Allocator.TempJob);
        var output = new NativeArray<float>(1000, Allocator.TempJob);

        // 데이터 초기화
        for (int i = 0; i < 1000; i++)
            input[i] = i - 500;

        var job = new ProcessDataJob
        {
            InputData = input,
            OutputData = output
        };

        var handle = job.Schedule();
        handle.Complete();

        Debug.Log($"처리 완료: {output.Length}");

        input.Dispose();
        output.Dispose();
    }
}
```

### IJobParallelFor와 Span

```csharp
[BurstCompile]
public struct ParallelProcessJob : IJobParallelFor
{
    [ReadOnly] public NativeArray<float> Input;
    [WriteOnly] public NativeArray<float> Output;
    public float Threshold;

    public void Execute(int index)
    {
        // 개별 인덱스 처리
        if (Input[index] > Threshold)
        {
            Output[index] = Input[index] * 2f;
        }
        else
        {
            Output[index] = 0f;
        }
    }
}

public class ParallelJobExample : MonoBehaviour
{
    void Start()
    {
        const int count = 10000;
        var input = new NativeArray<float>(count, Allocator.TempJob);
        var output = new NativeArray<float>(count, Allocator.TempJob);

        // 초기화
        for (int i = 0; i < count; i++)
            input[i] = UnityEngine.Random.value * 100f;

        var job = new ParallelProcessJob
        {
            Input = input,
            Output = output,
            Threshold = 50f
        };

        var handle = job.Schedule(count, 64);
        handle.Complete();

        // 결과 처리 (메인 스레드에서)
        var outputSpan = output.AsSpan();
        var processedCount = outputSpan.Count(x => x > 0);

        Debug.Log($"처리된 항목: {processedCount}");

        input.Dispose();
        output.Dispose();
    }
}
```

## Burst Compiler 최적화

### Burst 호환 코드 작성

```csharp
using Unity.Burst;
using Unity.Collections;
using Unity.Mathematics;

[BurstCompile]
public struct BurstOptimizedJob : IJob
{
    [ReadOnly] public NativeArray<float3> Positions;
    public NativeArray<float> Distances;

    public void Execute()
    {
        var posSpan = Positions.AsSpan();
        var distSpan = Distances.AsSpan();

        // Burst와 ZLinq 조합
        for (int i = 0; i < posSpan.Length; i++)
        {
            distSpan[i] = math.length(posSpan[i]);
        }

        // 수동 필터링 (Burst 최적화)
        float maxDist = 0f;
        for (int i = 0; i < distSpan.Length; i++)
        {
            if (distSpan[i] > maxDist)
                maxDist = distSpan[i];
        }
    }
}
```

### SIMD 최적화 활용

```csharp
using Unity.Mathematics;

[BurstCompile]
public struct SIMDJob : IJobParallelFor
{
    [ReadOnly] public NativeArray<float4> VectorData;
    [WriteOnly] public NativeArray<float> Magnitudes;

    public void Execute(int index)
    {
        // float4는 SIMD 연산 최적화
        var vec = VectorData[index];
        Magnitudes[index] = math.length(vec);
    }
}

public class SIMDExample : MonoBehaviour
{
    void Start()
    {
        const int count = 1000;
        var vectors = new NativeArray<float4>(count, Allocator.TempJob);
        var magnitudes = new NativeArray<float>(count, Allocator.TempJob);

        var job = new SIMDJob
        {
            VectorData = vectors,
            Magnitudes = magnitudes
        };

        var handle = job.Schedule(count, 64);
        handle.Complete();

        // ZLinq로 결과 분석
        var magSpan = magnitudes.AsSpan();
        var avg = magSpan.Average();
        var max = magSpan.Max();

        Debug.Log($"평균: {avg}, 최대: {max}");

        vectors.Dispose();
        magnitudes.Dispose();
    }
}
```

## R3 (Reactive Extensions) 통합

### Observable과 ZLinq 조합

```csharp
using R3;
using System.Collections.Generic;

public class R3Integration : MonoBehaviour
{
    private List<Enemy> mEnemies = new();
    private ReactiveProperty<int> mTargetCount = new(0);

    void Start()
    {
        // ReactiveProperty 업데이트 시 ZLinq로 처리
        Observable.EveryUpdate()
            .Subscribe(_ => UpdateTargets())
            .AddTo(this);

        mTargetCount
            .Subscribe(count => Debug.Log($"타겟 수: {count}"))
            .AddTo(this);
    }

    void UpdateTargets()
    {
        // ZLinq로 타겟 카운트
        var count = mEnemies.AsSpan()
            .Count(e => e.IsAlive && e.Distance < 10f);

        mTargetCount.Value = count;
    }
}

public class Enemy
{
    public bool IsAlive { get; set; }
    public float Distance { get; set; }
}
```

### ReactiveCollection과 Span

```csharp
using R3;

public class ReactiveCollectionIntegration : MonoBehaviour
{
    private ReactiveCollection<Item> mInventory = new();
    private List<Item> mCachedList = new();

    void Start()
    {
        // 컬렉션 변경 감지
        mInventory.ObserveAdd()
            .Subscribe(_ => UpdateCache())
            .AddTo(this);

        mInventory.ObserveRemove()
            .Subscribe(_ => UpdateCache())
            .AddTo(this);
    }

    void UpdateCache()
    {
        mCachedList.Clear();
        mCachedList.AddRange(mInventory);

        // ZLinq로 캐시된 리스트 처리
        var rareItems = mCachedList.AsSpan()
            .Where(item => item.Rarity >= 3)
            .Count();

        Debug.Log($"희귀 아이템: {rareItems}");
    }
}

public class Item
{
    public int Rarity { get; set; }
}
```

## VContainer 통합

### DI와 ZLinq 서비스

```csharp
using VContainer;
using VContainer.Unity;
using System.Collections.Generic;

public interface IEntityService
{
    IEnumerable<Entity> GetActiveEntities();
    int CountEntitiesInRange(float range);
}

public class EntityService : IEntityService
{
    private List<Entity> mEntities = new();

    public IEnumerable<Entity> GetActiveEntities()
    {
        // ZLinq 없이 반환하면 외부에서 사용 가능
        return mEntities;
    }

    public int CountEntitiesInRange(float range)
    {
        // 내부에서 ZLinq 사용
        return mEntities.AsSpan()
            .Count(e => e.IsActive && e.Distance < range);
    }

    public void ProcessEntities()
    {
        // Zero-allocation 처리
        foreach (var entity in mEntities.AsSpan().Where(e => e.IsActive))
        {
            entity.Update();
        }
    }
}

public class Entity
{
    public bool IsActive { get; set; }
    public float Distance { get; set; }
    public void Update() { }
}

public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<IEntityService, EntityService>(Lifetime.Singleton);
    }
}
```

## Transform 계층 구조 처리

### GetComponentsInChildren 대체

```csharp
using System.Collections.Generic;

public class HierarchyTraversal : MonoBehaviour
{
    private List<Renderer> mRenderers = new();

    void Start()
    {
        // GetComponentsInChildren은 allocation 발생
        // var renderers = GetComponentsInChildren<Renderer>();

        // 수동 수집 후 ZLinq 사용
        CollectRenderers(transform);

        // ZLinq로 처리
        foreach (var renderer in mRenderers.AsSpan().Where(r => r.enabled))
        {
            renderer.material.color = Color.red;
        }
    }

    void CollectRenderers(Transform parent)
    {
        if (parent.TryGetComponent<Renderer>(out var renderer))
            mRenderers.Add(renderer);

        foreach (Transform child in parent)
            CollectRenderers(child);
    }
}
```

### 계층 구조 캐싱

```csharp
public class CachedHierarchy : MonoBehaviour
{
    private List<Transform> mChildren = new();
    private bool mCacheDirty = true;

    void Start()
    {
        RebuildCache();
    }

    void RebuildCache()
    {
        mChildren.Clear();
        GetAllChildren(transform);
        mCacheDirty = false;
    }

    void GetAllChildren(Transform parent)
    {
        for (int i = 0; i < parent.childCount; i++)
        {
            var child = parent.GetChild(i);
            mChildren.Add(child);
            GetAllChildren(child);
        }
    }

    public void ProcessActiveChildren()
    {
        if (mCacheDirty) RebuildCache();

        // ZLinq로 활성화된 자식만 처리
        foreach (var child in mChildren.AsSpan().Where(c => c.gameObject.activeSelf))
        {
            Debug.Log(child.name);
        }
    }
}
```

## Physics 통합

### Raycast 결과 처리

```csharp
using Unity.Collections;

public class PhysicsIntegration : MonoBehaviour
{
    void Update()
    {
        // RaycastAll 대신 RaycastNonAlloc 사용
        var hits = new RaycastHit[100];
        var hitCount = Physics.RaycastNonAlloc(
            transform.position,
            transform.forward,
            hits,
            100f
        );

        // Span으로 변환 (실제 hit 개수만큼만)
        var hitSpan = hits.AsSpan(0, hitCount);

        // ZLinq로 가장 가까운 적 찾기
        var closestEnemy = hitSpan
            .Where(hit => hit.collider.CompareTag("Enemy"))
            .OrderBy(hit => hit.distance)
            .FirstOrDefault();

        if (closestEnemy.collider != null)
        {
            Debug.Log($"가장 가까운 적: {closestEnemy.collider.name}");
        }
    }
}
```

### OverlapSphere 최적화

```csharp
public class OverlapOptimization : MonoBehaviour
{
    private Collider[] mColliderBuffer = new Collider[100];

    void Update()
    {
        // OverlapSphere는 매번 allocation 발생
        // var colliders = Physics.OverlapSphere(transform.position, 10f);

        // OverlapSphereNonAlloc 사용
        var count = Physics.OverlapSphereNonAlloc(
            transform.position,
            10f,
            mColliderBuffer
        );

        var span = mColliderBuffer.AsSpan(0, count);

        // ZLinq로 적만 필터링
        var enemyCount = span.Count(c => c.CompareTag("Enemy"));
        Debug.Log($"범위 내 적: {enemyCount}");

        // 가장 가까운 아이템 찾기
        foreach (var collider in span
            .Where(c => c.CompareTag("Item"))
            .OrderBy(c => Vector3.Distance(transform.position, c.transform.position)))
        {
            PickupItem(collider.gameObject);
            break; // 가장 가까운 것만
        }
    }

    void PickupItem(GameObject item) { }
}
```

## Memory\<T\>와 ReadOnlyMemory\<T\>

### Memory 활용

```csharp
using System;
using System.Buffers;

public class MemoryExample : MonoBehaviour
{
    private Memory<byte> mDataMemory;
    private byte[] mBackingArray;

    void Start()
    {
        mBackingArray = new byte[1000];
        mDataMemory = new Memory<byte>(mBackingArray);

        ProcessData(mDataMemory);
    }

    void ProcessData(Memory<byte> data)
    {
        // Memory를 Span으로 변환
        var span = data.Span;

        // ZLinq 연산
        var nonZeroCount = span.Count(b => b != 0);
        Debug.Log($"0이 아닌 바이트: {nonZeroCount}");
    }
}
```

### ReadOnlyMemory와 불변성

```csharp
public class ReadOnlyMemoryExample : MonoBehaviour
{
    void Start()
    {
        var data = new[] { 1, 2, 3, 4, 5 };
        var readOnlyMem = new ReadOnlyMemory<int>(data);

        ProcessReadOnly(readOnlyMem);
    }

    void ProcessReadOnly(ReadOnlyMemory<int> data)
    {
        var span = data.Span;

        // 읽기 전용 연산만 가능
        var sum = span.Sum();
        var avg = span.Average();
        var max = span.Max();

        Debug.Log($"합: {sum}, 평균: {avg}, 최대: {max}");
    }
}
```

## 문자열 처리 최적화

### Span\<char\>로 문자열 처리

```csharp
public class StringOptimization : MonoBehaviour
{
    void Start()
    {
        var text = "Player_12345";

        // Substring 대신 Span 사용 (allocation 없음)
        var span = text.AsSpan();
        var numberPart = span.Slice(7); // "12345"

        // Span으로 파싱
        if (int.TryParse(numberPart, out var playerId))
        {
            Debug.Log($"플레이어 ID: {playerId}");
        }

        // 문자 필터링
        var letterCount = span.Count(c => char.IsLetter(c));
        Debug.Log($"문자 개수: {letterCount}");
    }
}
```

### CSV 파싱 최적화

```csharp
using System;

public class CSVParser : MonoBehaviour
{
    void Start()
    {
        var csv = "100,200,300,400,500";
        ParseCSV(csv);
    }

    void ParseCSV(string csv)
    {
        var span = csv.AsSpan();
        var values = new List<int>();

        int start = 0;
        for (int i = 0; i < span.Length; i++)
        {
            if (span[i] == ',' || i == span.Length - 1)
            {
                var end = (i == span.Length - 1) ? i + 1 : i;
                var valueSpan = span.Slice(start, end - start);

                if (int.TryParse(valueSpan, out var value))
                    values.Add(value);

                start = i + 1;
            }
        }

        // ZLinq로 처리
        var sum = values.AsSpan().Sum();
        Debug.Log($"합계: {sum}");
    }
}
```
