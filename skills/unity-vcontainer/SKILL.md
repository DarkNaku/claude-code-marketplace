---
name: unity-vcontainer
description: IoC 컨테이너 구성, 라이프사이클 관리, Unity 최적화 DI 패턴을 전문으로 하는 VContainer 의존성 주입 전문가. 의존성 해결, 스코프 컨테이너, 테스트 가능한 아키텍처 설계에 능통. VContainer 설정, 서비스 등록, SOLID 원칙 구현 시 적극적으로 사용.
---

# Unity VContainer - Unity용 고성능 DI

## 개요

VContainer는 Unity용 고성능 IoC 컨테이너로, 테스트 가능하고 유지보수 가능한 코드를 위한 의존성 주입 패턴을 제공합니다.

**핵심 주제**:
- 생성자 및 메서드 주입
- 서비스 등록 패턴 (Singleton, Transient, Scoped)
- LifetimeScope 계층 구조
- MonoBehaviour 주입
- DI를 사용한 팩토리 패턴
- Mock을 사용한 테스트

**학습 경로**: DI 기초 → VContainer 기본 → 고급 패턴 → 테스트

## 빠른 시작

```csharp
using VContainer;
using VContainer.Unity;

// 서비스 인터페이스 정의
public interface IPlayerService
{
    void Initialize();
}

// 서비스 구현
public class PlayerService : IPlayerService
{
    public void Initialize() => Debug.Log("플레이어 초기화됨");
}

// LifetimeScope 설정
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<IPlayerService, PlayerService>(Lifetime.Singleton);
        builder.RegisterComponentInHierarchy<PlayerController>();
    }
}

// MonoBehaviour에 주입
public class PlayerController : MonoBehaviour
{
    [Inject] private readonly IPlayerService mPlayerService;

    void Start() => mPlayerService.Initialize();
}
```

## 핵심 개념

### 라이프타임 스코프
- **Singleton**: 컨테이너당 하나의 인스턴스
- **Transient**: 매 해결마다 새 인스턴스
- **Scoped**: 스코프당 하나의 인스턴스

### 주입 타입
- **생성자 주입**: 필수 의존성에 선호됨
- **메서드 주입**: 선택적 의존성에 사용
- **속성/필드 주입**: `[Inject]` 어트리뷰트 사용

## 참고 문서

### [VContainer 모범 사례](references/vcontainer-patterns.md)
핵심 DI 패턴:
- 등록 패턴 및 라이프타임 관리
- LifetimeScope 계층 구조
- Mock 의존성을 사용한 테스트

### [VContainer 통합 패턴](references/vcontainer-integrations.md)
고급 통합:
- 반응형 속성을 사용한 MVVM
- 크로스 프레임워크 통합 패턴

### [VContainer 공식 문서](https://vcontainer.hadashikick.jp/)

## 모범 사례

1. **인터페이스 등록**: 느슨한 결합과 테스트 가능성
2. **생성자 주입 우선**: 명시적 의존성
3. **서비스 로케이터 패턴 지양**: Update 루프에서 resolve 하지 말 것
4. **Mock으로 테스트**: 테스트에서 ContainerBuilder 사용
5. **명확한 계층 구조**: Root → Scene → Local 스코프