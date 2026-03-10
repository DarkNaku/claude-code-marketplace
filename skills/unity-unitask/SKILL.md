---
name: unity-unitask
description: Unity용 고성능 비동기 프로그래밍, async/await 패턴, 코루틴 대체, 취소 토큰 관리를 전문으로 하는 UniTask 비동기 처리 전문가. 제로 할당 비동기 작업, 비동기 LINQ, 타이머 및 딜레이 처리에 능통. 비동기 데이터 로드, UI 애니메이션, 네트워크 통신, 복잡한 비동기 흐름 구현 시 적극적으로 사용.
---

# Unity UniTask - Unity용 고성능 비동기 라이브러리

## 개요

UniTask는 Unity용 고성능 비동기 프로그래밍 라이브러리로, 제로 할당 async/await를 제공하며 코루틴을 대체합니다.

**핵심 주제**:
- async/await 패턴
- 코루틴 대체 및 변환
- CancellationToken을 통한 취소 처리
- 비동기 LINQ (Async LINQ)
- PlayerLoop 기반 타이머
- UniTaskCompletionSource

**학습 경로**: async/await 기초 → UniTask 기본 → 취소 토큰 → 고급 패턴

## 빠른 시작

```csharp
using System;
using System.Threading;
using Cysharp.Threading.Tasks;
using UnityEngine;

public class DataLoader : MonoBehaviour
{
    async void Start()
    {
        // 비동기 데이터 로드
        var data = await LoadDataAsync();
        Debug.Log($"데이터 로드 완료: {data}");

        // 딜레이
        await UniTask.Delay(TimeSpan.FromSeconds(1));
        Debug.Log("1초 후 실행");

        // 프레임 대기
        await UniTask.DelayFrame(10);
        Debug.Log("10프레임 후 실행");
    }

    async UniTask<string> LoadDataAsync()
    {
        // 비동기 작업 시뮬레이션
        await UniTask.Delay(1000);
        return "게임 데이터";
    }
}
```

### 취소 토큰 사용

```csharp
public class CancellableOperation : MonoBehaviour
{
    private CancellationTokenSource mCts;

    async void Start()
    {
        mCts = new CancellationTokenSource();

        try
        {
            await LongRunningTaskAsync(mCts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("작업이 취소되었습니다");
        }
    }

    void OnDestroy()
    {
        // GameObject 파괴 시 취소
        mCts?.Cancel();
        mCts?.Dispose();
    }

    async UniTask LongRunningTaskAsync(CancellationToken cancellationToken)
    {
        for (int i = 0; i < 100; i++)
        {
            // 취소 확인
            cancellationToken.ThrowIfCancellationRequested();

            await UniTask.Delay(100, cancellationToken: cancellationToken);
            Debug.Log($"진행: {i}%");
        }
    }
}
```

### 코루틴을 UniTask로 변환

```csharp
// 기존 코루틴
IEnumerator OldCoroutine()
{
    yield return new WaitForSeconds(1f);
    Debug.Log("1초 후");
}

// UniTask로 변환
async UniTaskVoid NewAsyncMethod()
{
    await UniTask.Delay(TimeSpan.FromSeconds(1f));
    Debug.Log("1초 후");
}

// 코루틴을 UniTask로 await
async UniTask ConvertCoroutine()
{
    await OldCoroutine().ToUniTask();
}
```

## 핵심 개념

### UniTask vs Task
- **제로 할당**: GC 압력 없음
- **PlayerLoop 통합**: Unity의 업데이트 루프에서 실행
- **취소 토큰 네이티브 지원**: Unity 생명주기와 통합

### 취소 패턴
- **CancellationTokenSource**: 수동 취소
- **GetCancellationTokenOnDestroy()**: GameObject 파괴 시 자동 취소
- **TimeoutController**: 타임아웃 취소

### 비동기 처리 타입
- **UniTask**: 반환값이 있는 비동기 작업
- **UniTaskVoid**: 반환값이 없는 fire-and-forget
- **UniTask.WaitUntil**: 조건 대기
- **UniTask.WhenAll/WhenAny**: 병렬 처리

## 참고 문서

### [UniTask 모범 사례](references/unitask-patterns.md)
핵심 비동기 패턴:
- 취소 토큰 관리
- 비동기 LINQ와 시퀀스
- 타이머와 딜레이
- 병렬 및 순차 실행

### [UniTask 통합 패턴](references/unitask-integrations.md)
고급 통합:
- VContainer와의 통합
- MessagePipe와의 조합
- UnityWebRequest 비동기 처리
- Addressables 비동기 로딩

### [UniTask 공식 GitHub](https://github.com/Cysharp/UniTask)

## 모범 사례

1. **취소 토큰 항상 전달**: 메모리 누수 방지
2. **GetCancellationTokenOnDestroy() 활용**: MonoBehaviour 생명주기 연동
3. **UniTaskVoid는 최소화**: 예외 처리 어려움
4. **async void 지양**: UniTaskVoid 사용
5. **병렬 처리로 최적화**: WhenAll로 동시 실행
6. **PlayerLoopTiming 이해**: 적절한 타이밍 선택
7. **비동기 LINQ 활용**: 선언적 비동기 처리
