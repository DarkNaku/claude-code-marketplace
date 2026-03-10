---
name: unity-r3
description: Unity용 반응형 프로그래밍 라이브러리 R3 전문가. Observable 스트림, ReactiveProperty, Unity 통합 패턴, 비동기 이벤트 처리에 능통. UniRx의 현대적 후속작으로 zero-allocation과 성능 최적화에 집중. 이벤트 기반 아키텍처, MVVM 패턴, 비동기 작업 체이닝이 필요할 때 적극적으로 사용.
---

# Unity R3 - Unity용 반응형 프로그래밍

## 개요

R3는 Unity용 고성능 Reactive Extensions 라이브러리로, UniRx의 현대적 후속작입니다. Zero-allocation과 성능 최적화에 집중하며, 이벤트 기반 프로그래밍과 비동기 작업 처리를 간결하게 만듭니다.

**핵심 주제**:
- Observable과 Observer 패턴
- ReactiveProperty와 데이터 바인딩
- Operators (Select, Where, Merge, Throttle 등)
- Unity 통합 (OnDestroyAsync, EveryUpdate, EveryValueChanged)
- UniTask와의 통합
- 메모리 관리와 Dispose 패턴

**학습 경로**: Reactive 기초 → Observable → Operators → ReactiveProperty → Unity 통합 → 고급 패턴

## 빠른 시작

```csharp
using R3;
using UnityEngine;

public class PlayerController : MonoBehaviour
{
    // ReactiveProperty로 상태 관리
    private ReactiveProperty<int> mHealth = new(100);

    void Start()
    {
        // 값 변경 구독
        mHealth.Subscribe(health =>
        {
            Debug.Log($"체력: {health}");
            if (health <= 0) Die();
        }).AddTo(this); // GameObject 파괴 시 자동 해제

        // Unity 이벤트를 Observable로 변환
        Observable.EveryUpdate()
            .Where(_ => Input.GetKeyDown(KeyCode.Space))
            .Subscribe(_ => Attack())
            .AddTo(this);
    }

    public void TakeDamage(int damage)
    {
        mHealth.Value -= damage; // ReactiveProperty 값 변경
    }

    void Attack() => Debug.Log("공격!");
    void Die() => Debug.Log("사망!");
}
```

## 핵심 개념

### Observable과 Observer

Observable은 시간에 따른 값의 스트림입니다. Observer는 이 스트림을 구독하여 값을 받습니다.

```csharp
// Observable 생성
var observable = Observable.Timer(TimeSpan.FromSeconds(1));

// 구독
observable.Subscribe(
    onNext: _ => Debug.Log("1초 경과"),
    onCompleted: () => Debug.Log("완료")
);
```

### ReactiveProperty

변경 가능한 값을 Observable로 만들어주는 컨테이너입니다.

```csharp
public class GameManager : MonoBehaviour
{
    public ReactiveProperty<int> Score { get; } = new(0);
    public ReadOnlyReactiveProperty<bool> IsGameOver { get; }

    public GameManager()
    {
        // Score가 100 이상이면 게임 오버
        IsGameOver = Score
            .Select(score => score >= 100)
            .ToReadOnlyReactiveProperty();
    }
}
```

### Operators

Observable 스트림을 변환, 필터링, 조합하는 메서드들입니다.

- **Select**: 값 변환
- **Where**: 조건 필터링
- **Throttle**: 일정 시간 간격으로 제한
- **Merge**: 여러 스트림 병합
- **CombineLatest**: 최신 값들 조합

### Unity 통합

```csharp
// 매 프레임마다 실행
Observable.EveryUpdate()
    .Subscribe(_ => transform.Rotate(0, 1, 0))
    .AddTo(this);

// 값 변경 감지
Observable.EveryValueChanged(this, x => x.transform.position)
    .Subscribe(pos => Debug.Log($"위치: {pos}"))
    .AddTo(this);

// GameObject 파괴 시 이벤트
this.OnDestroyAsObservable()
    .Subscribe(_ => Debug.Log("오브젝트 파괴됨"));
```

## 참고 문서

### [R3 패턴 가이드](references/r3-patterns.md)
핵심 반응형 패턴:
- Observable 생성 및 구독 패턴
- Operators 체이닝
- Subject와 메시지 버스
- 메모리 관리 및 Dispose 패턴
- MVVM 패턴 구현

### [R3 통합 패턴](references/r3-integrations.md)
고급 통합:
- UniTask와 Observable 통합
- VContainer와 함께 사용하기
- UI 이벤트 바인딩
- 네트워크 요청 처리

### [R3 공식 문서](https://github.com/Cysharp/R3)

## 모범 사례

1. **AddTo로 자동 해제**: Subscribe 후 항상 `.AddTo(gameObject)` 사용
2. **ReactiveProperty 선호**: 변경 가능한 상태는 ReactiveProperty 사용
3. **Hot vs Cold Observable 이해**: 구독 타이밍에 따른 동작 차이 파악
4. **Throttle로 성능 최적화**: 빈번한 이벤트는 Throttle/Sample로 제한
5. **CompositeDisposable 활용**: 여러 구독을 그룹으로 관리
