---
name: unity-messagepipe
description: 고성능 Pub/Sub 메시징, 글로벌/스코프 이벤트 전달, 비동기 메시지 처리를 전문으로 하는 MessagePipe 이벤트 버스 전문가. 타입 안전한 메시지 전달, 필터 체인, VContainer 통합에 능통. 이벤트 기반 아키텍처, 컴포넌트 간 결합도 감소, 반응형 시스템 구축 시 적극적으로 사용.
---

# Unity MessagePipe - Unity용 고성능 메시징 라이브러리

## 개요

MessagePipe는 Unity용 고성능 메시징 라이브러리로, Pub/Sub 패턴을 통해 타입 안전한 이벤트 기반 아키텍처를 제공합니다.

**핵심 주제**:
- Publisher/Subscriber 패턴
- 글로벌 및 스코프 메시징
- 동기/비동기 메시지 전달
- 필터 체인 (로깅, 예외 처리)
- VContainer DI 통합
- Request/Response 패턴

**학습 경로**: 메시징 기초 → MessagePipe 기본 → 필터 및 스코프 → 고급 패턴

## 빠른 시작

```csharp
using MessagePipe;
using VContainer;
using VContainer.Unity;

// 메시지 정의
public struct PlayerDamagedMessage
{
    public int PlayerId;
    public float Damage;
}

// LifetimeScope에서 MessagePipe 등록
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        // MessagePipe 기본 설정
        builder.AddMessagePipe();

        // 컴포넌트 등록
        builder.RegisterComponentInHierarchy<PlayerHealthSystem>();
        builder.RegisterComponentInHierarchy<UIHealthDisplay>();
    }
}

// Publisher 사용
public class PlayerHealthSystem : MonoBehaviour
{
    [Inject] private IPublisher<PlayerDamagedMessage> mDamagePublisher;

    public void TakeDamage(int playerId, float damage)
    {
        mDamagePublisher.Publish(new PlayerDamagedMessage
        {
            PlayerId = playerId,
            Damage = damage
        });
    }
}

// Subscriber 사용
public class UIHealthDisplay : MonoBehaviour
{
    [Inject] private ISubscriber<PlayerDamagedMessage> mDamageSubscriber;
    private IDisposable mSubscription;

    void Start()
    {
        // 메시지 구독
        mSubscription = mDamageSubscriber.Subscribe(msg =>
        {
            Debug.Log($"플레이어 {msg.PlayerId}가 {msg.Damage} 데미지를 받았습니다");
            UpdateHealthUI(msg.PlayerId, msg.Damage);
        });
    }

    void OnDestroy()
    {
        // 구독 해제
        mSubscription?.Dispose();
    }

    private void UpdateHealthUI(int playerId, float damage)
    {
        /* UI 업데이트 로직 */
    }
}
```

## 핵심 개념

### 메시징 스코프
- **글로벌 메시징**: 애플리케이션 전체에서 메시지 전달
- **스코프 메시징**: 특정 생명주기 범위 내에서만 메시지 전달

### 메시지 전달 타입
- **동기 메시지**: `IPublisher<T>` / `ISubscriber<T>`
- **비동기 메시지**: `IAsyncPublisher<T>` / `IAsyncSubscriber<T>`
- **Request/Response**: `IRequestHandler<TRequest, TResponse>`

### 필터 체인
- 메시지 전달 전후에 로깅, 예외 처리, 검증 등 수행
- 글로벌 필터 및 메시지별 필터 설정 가능

## 참고 문서

### [MessagePipe 모범 사례](references/messagepipe-patterns.md)
핵심 메시징 패턴:
- 글로벌 vs 스코프 메시징
- 구독 생명주기 관리
- 필터 체인 구성
- Request/Response 패턴

### [MessagePipe 통합 패턴](references/messagepipe-integrations.md)
고급 통합:
- VContainer와의 통합
- UniRx/R3와의 조합
- 멀티씬 메시징 아키텍처

### [MessagePipe 공식 GitHub](https://github.com/Cysharp/MessagePipe)

## 모범 사례

1. **구조체 메시지 사용**: 불필요한 할당 방지
2. **구독 해제**: `IDisposable`을 통한 메모리 누수 방지
3. **스코프 메시징 활용**: 씬/기능별 격리
4. **필터로 크로스커팅 로직**: 로깅, 예외 처리를 중앙화
5. **타입 안전성 유지**: 명시적인 메시지 타입 정의
6. **과도한 메시징 지양**: 직접 호출이 더 나은 경우 고려
