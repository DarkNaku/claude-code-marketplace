---
name: unity-zlinq
description: Unity용 Zero-allocation LINQ 라이브러리 ZLinq 전문가. 가비지 프리 컬렉션 처리, 성능 최적화된 쿼리, Span/Memory 기반 연산에 능통. 일반 LINQ의 allocation 문제를 해결하여 고성능 게임 로직 구현. 대량 데이터 처리, 매 프레임 컬렉션 연산, 메모리 민감한 상황에서 적극적으로 사용.
---

# Unity ZLinq - Unity용 Zero-allocation LINQ

## 개요

ZLinq는 Unity용 Zero-allocation LINQ 라이브러리로, 일반 LINQ의 가비지 생성 문제를 해결합니다. Span\<T\>과 구조체 기반 열거자를 사용하여 allocation 없이 컬렉션 연산을 수행할 수 있습니다.

**핵심 주제**:
- Zero-allocation LINQ 연산자
- Span\<T\>와 Memory\<T\> 기반 처리
- 구조체 열거자 패턴
- Unity 컬렉션 통합 (NativeArray, List 등)
- 성능 최적화 기법
- LINQ vs ZLinq 비교

**학습 경로**: LINQ 기초 → Allocation 문제 이해 → ZLinq 기본 → Span 기반 연산 → 고급 최적화

## 빠른 시작

```csharp
using Cysharp.Threading.Tasks.Linq;
using UnityEngine;
using System.Collections.Generic;

public class EnemyManager : MonoBehaviour
{
    private List<Enemy> mEnemies = new();

    void Update()
    {
        // 일반 LINQ - 매 프레임 가비지 생성
        // var nearbyEnemies = mEnemies.Where(e => e.IsAlive && e.Distance < 10f).ToList();

        // ZLinq - Zero allocation
        foreach (var enemy in mEnemies.AsSpan().Where(e => e.IsAlive && e.Distance < 10f))
        {
            enemy.Update();
        }

        // 필터링 + 변환 + 정렬
        var count = 0;
        foreach (var damage in mEnemies.AsSpan()
            .Where(e => e.IsAlive)
            .Select(e => e.CalculateDamage())
            .OrderByDescending(d => d))
        {
            if (count++ >= 5) break; // 상위 5개만
            ApplyDamage(damage);
        }
    }
}

public class Enemy
{
    public bool IsAlive { get; set; }
    public float Distance { get; set; }
    public int CalculateDamage() => 10;
    public void Update() { }
}
```

## 핵심 개념

### Zero-allocation이란?

일반 LINQ는 `Where`, `Select` 등의 연산자가 `IEnumerable<T>`를 반환하며, 이는 힙 할당을 발생시킵니다. ZLinq는 구조체 열거자를 사용하여 스택에만 할당됩니다.

```csharp
// 일반 LINQ - 힙 할당 발생
var result = list.Where(x => x > 5).Select(x => x * 2);

// ZLinq - 스택만 사용 (zero allocation)
var result = list.AsSpan().Where(x => x > 5).Select(x => x * 2);
```

### Span 기반 연산

`Span<T>`는 연속된 메모리 영역에 대한 타입 안전 포인터입니다.

```csharp
public class DataProcessor : MonoBehaviour
{
    private int[] mData = new int[1000];

    void Process()
    {
        // Span으로 변환하여 zero-allocation 연산
        var span = mData.AsSpan();

        // 필터링
        var filtered = span.Where(x => x > 100);

        // 변환
        var doubled = filtered.Select(x => x * 2);

        // 집계
        var sum = doubled.Sum();

        Debug.Log($"합계: {sum}");
    }
}
```

### Unity 컬렉션 지원

```csharp
using Unity.Collections;

public class NativeArrayExample : MonoBehaviour
{
    private NativeArray<float> mPositions;

    void Start()
    {
        mPositions = new NativeArray<float>(100, Allocator.Persistent);

        // NativeArray를 Span으로 변환
        var span = mPositions.AsSpan();

        // ZLinq 연산
        var avg = span
            .Where(x => x > 0)
            .Average();

        Debug.Log($"평균: {avg}");
    }

    void OnDestroy()
    {
        if (mPositions.IsCreated)
            mPositions.Dispose();
    }
}
```

## 참고 문서

### [ZLinq 패턴 가이드](references/zlinq-patterns.md)
핵심 ZLinq 패턴:
- Span/Memory 기반 연산자
- 필터링, 변환, 정렬 패턴
- 집계 연산 (Sum, Average, Min, Max)
- 성능 측정 및 비교
- 일반 LINQ 대체 가이드

### [ZLinq 통합 패턴](references/zlinq-integrations.md)
고급 통합:
- Unity Collections (NativeArray, NativeList)
- ECS와 함께 사용하기
- Job System 통합
- Burst Compiler 최적화

### [ZLinq GitHub](https://github.com/Cysharp/ZLinq)

## 모범 사례

1. **매 프레임 연산에 사용**: Update/FixedUpdate에서 컬렉션 처리 시 필수
2. **AsSpan() 활용**: List, Array를 Span으로 변환하여 사용
3. **ToList() 지양**: 최종 결과를 컬렉션으로 만들 필요가 없다면 foreach로 순회
4. **구조체 열거 이해**: `var` 대신 명시적 타입으로 구조체 열거자 유지
5. **성능 측정**: Profiler로 allocation 감소 확인
