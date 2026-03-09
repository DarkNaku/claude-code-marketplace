# UniTask 모범 사례

Unity에서 고급 UniTask 패턴을 위한 심층 참조 문서.

## 취소 토큰 관리

### GetCancellationTokenOnDestroy()

GameObject 파괴 시 자동으로 비동기 작업 취소.

```csharp
using System.Threading;
using Cysharp.Threading.Tasks;
using UnityEngine;

public class AutoCancelExample : MonoBehaviour
{
    async void Start()
    {
        // GameObject가 파괴되면 자동으로 취소
        await LongTaskAsync(this.GetCancellationTokenOnDestroy());
    }

    async UniTask LongTaskAsync(CancellationToken ct)
    {
        try
        {
            for (int i = 0; i < 100; i++)
            {
                await UniTask.Delay(1000, cancellationToken: ct);
                Debug.Log($"진행: {i}%");
            }
        }
        catch (OperationCanceledException)
        {
            Debug.Log("작업이 취소되었습니다");
        }
    }
}
```

### CancellationTokenSource 수동 관리

명시적 취소 제어가 필요한 경우.

```csharp
public class ManualCancelExample : MonoBehaviour
{
    private CancellationTokenSource mCts;

    async void Start()
    {
        mCts = new CancellationTokenSource();

        try
        {
            await DownloadDataAsync(mCts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("다운로드가 취소되었습니다");
        }
    }

    public void CancelDownload()
    {
        // 수동 취소
        mCts?.Cancel();
    }

    void OnDestroy()
    {
        mCts?.Cancel();
        mCts?.Dispose();
    }

    async UniTask DownloadDataAsync(CancellationToken ct)
    {
        await UniTask.Delay(5000, cancellationToken: ct);
        Debug.Log("다운로드 완료");
    }
}
```

### 타임아웃 처리

```csharp
public class TimeoutExample : MonoBehaviour
{
    async void Start()
    {
        try
        {
            // 3초 타임아웃 설정
            var cts = new CancellationTokenSource();
            cts.CancelAfterSlim(TimeSpan.FromSeconds(3));

            await LongOperationAsync(cts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.LogError("작업 타임아웃!");
        }
    }

    async UniTask LongOperationAsync(CancellationToken ct)
    {
        // 5초 걸리는 작업 (타임아웃 발생)
        await UniTask.Delay(5000, cancellationToken: ct);
    }
}
```

### 링크된 토큰

여러 취소 조건 결합.

```csharp
public class LinkedTokenExample : MonoBehaviour
{
    private CancellationTokenSource mUserCts;

    async void Start()
    {
        mUserCts = new CancellationTokenSource();

        // GameObject 파괴 또는 사용자 취소 시 취소
        var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            mUserCts.Token,
            this.GetCancellationTokenOnDestroy()
        );

        try
        {
            await LoadResourceAsync(linkedCts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("리소스 로드 취소");
        }
        finally
        {
            linkedCts?.Dispose();
        }
    }

    public void CancelByUser()
    {
        mUserCts?.Cancel();
    }

    async UniTask LoadResourceAsync(CancellationToken ct)
    {
        await UniTask.Delay(2000, cancellationToken: ct);
        Debug.Log("리소스 로드 완료");
    }
}
```

## 비동기 LINQ

### Select, Where를 사용한 스트림 처리

```csharp
using System.Linq;
using Cysharp.Threading.Tasks;
using Cysharp.Threading.Tasks.Linq;
using UnityEngine;

public class AsyncLinqExample : MonoBehaviour
{
    async void Start()
    {
        // 비동기 시퀀스 생성 및 처리
        await UniTaskAsyncEnumerable.Range(1, 10)
            .Select(x => x * 2)
            .Where(x => x > 10)
            .ForEachAsync(x =>
            {
                Debug.Log($"값: {x}");
            });
    }
}
```

### 비동기 스트림 생성

```csharp
public class AsyncStreamExample : MonoBehaviour
{
    async void Start()
    {
        await foreach (var value in GenerateNumbersAsync())
        {
            Debug.Log($"생성된 값: {value}");
        }
    }

    async IUniTaskAsyncEnumerable<int> GenerateNumbersAsync()
    {
        for (int i = 0; i < 10; i++)
        {
            await UniTask.Delay(500);
            yield return i;
        }
    }
}
```

### 비동기 이벤트 스트림

```csharp
public class AsyncEventStreamExample : MonoBehaviour
{
    async void Start()
    {
        var ct = this.GetCancellationTokenOnDestroy();

        // 버튼 클릭을 비동기 스트림으로 변환
        await foreach (var _ in GetButtonClickAsyncEnumerable(ct).WithCancellation(ct))
        {
            Debug.Log("버튼 클릭됨!");
            await HandleButtonClickAsync();
        }
    }

    async IUniTaskAsyncEnumerable<AsyncUnit> GetButtonClickAsyncEnumerable(
        [EnumeratorCancellation] CancellationToken ct)
    {
        var button = GetComponent<UnityEngine.UI.Button>();

        while (!ct.IsCancellationRequested)
        {
            await button.OnClickAsync(ct);
            yield return AsyncUnit.Default;
        }
    }

    async UniTask HandleButtonClickAsync()
    {
        await UniTask.Delay(100);
        // 클릭 처리 로직
    }
}
```

## 타이머와 딜레이

### 다양한 딜레이 타입

```csharp
public class DelayExample : MonoBehaviour
{
    async void Start()
    {
        // 실시간 딜레이
        await UniTask.Delay(1000);
        Debug.Log("1초 후 (실시간)");

        // 프레임 딜레이
        await UniTask.DelayFrame(60);
        Debug.Log("60프레임 후");

        // 게임 시간 딜레이 (Time.timeScale 영향 받음)
        await UniTask.Delay(
            TimeSpan.FromSeconds(1),
            delayTiming: PlayerLoopTiming.Update
        );
        Debug.Log("1초 후 (게임 시간)");

        // 특정 프레임 타이밍
        await UniTask.Yield(PlayerLoopTiming.FixedUpdate);
        Debug.Log("다음 FixedUpdate");
    }
}
```

### 반복 타이머

```csharp
public class TimerExample : MonoBehaviour
{
    async void Start()
    {
        var ct = this.GetCancellationTokenOnDestroy();

        // 1초마다 실행
        await RepeatTimerAsync(1000, ct);
    }

    async UniTask RepeatTimerAsync(int intervalMs, CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            await UniTask.Delay(intervalMs, cancellationToken: ct);

            Debug.Log($"타이머 틱: {Time.time}");
            UpdateGameState();
        }
    }

    void UpdateGameState()
    {
        // 주기적 업데이트 로직
    }
}
```

### 프레임 기반 카운트다운

```csharp
public class CountdownExample : MonoBehaviour
{
    [SerializeField] private UnityEngine.UI.Text mCountdownText;

    async void Start()
    {
        await CountdownAsync(10);
        Debug.Log("카운트다운 완료!");
    }

    async UniTask CountdownAsync(int seconds)
    {
        for (int i = seconds; i > 0; i--)
        {
            mCountdownText.text = i.ToString();
            await UniTask.Delay(1000);
        }

        mCountdownText.text = "시작!";
    }
}
```

## 병렬 및 순차 실행

### WhenAll - 병렬 실행

모든 작업이 완료될 때까지 대기.

```csharp
public class ParallelExample : MonoBehaviour
{
    async void Start()
    {
        // 3개 작업 동시 실행
        await UniTask.WhenAll(
            LoadTextureAsync("texture1"),
            LoadAudioAsync("audio1"),
            LoadDataAsync("data1")
        );

        Debug.Log("모든 리소스 로드 완료");
    }

    async UniTask LoadTextureAsync(string name)
    {
        await UniTask.Delay(1000);
        Debug.Log($"텍스처 로드: {name}");
    }

    async UniTask LoadAudioAsync(string name)
    {
        await UniTask.Delay(1500);
        Debug.Log($"오디오 로드: {name}");
    }

    async UniTask LoadDataAsync(string name)
    {
        await UniTask.Delay(500);
        Debug.Log($"데이터 로드: {name}");
    }
}
```

### WhenAll - 결과 수집

```csharp
public class ParallelResultsExample : MonoBehaviour
{
    async void Start()
    {
        // 여러 작업의 결과를 동시에 수집
        var (health, mana, stamina) = await UniTask.WhenAll(
            FetchPlayerHealthAsync(),
            FetchPlayerManaAsync(),
            FetchPlayerStaminaAsync()
        );

        Debug.Log($"체력: {health}, 마나: {mana}, 스태미나: {stamina}");
    }

    async UniTask<int> FetchPlayerHealthAsync()
    {
        await UniTask.Delay(500);
        return 100;
    }

    async UniTask<int> FetchPlayerManaAsync()
    {
        await UniTask.Delay(300);
        return 50;
    }

    async UniTask<int> FetchPlayerStaminaAsync()
    {
        await UniTask.Delay(700);
        return 75;
    }
}
```

### WhenAny - 가장 빠른 작업 대기

```csharp
public class WhenAnyExample : MonoBehaviour
{
    async void Start()
    {
        // 가장 먼저 완료되는 작업만 대기
        var (winIndex, result) = await UniTask.WhenAny(
            FetchFromServer1Async(),
            FetchFromServer2Async(),
            FetchFromServer3Async()
        );

        Debug.Log($"서버 {winIndex + 1}이 가장 빨랐습니다: {result}");
    }

    async UniTask<string> FetchFromServer1Async()
    {
        await UniTask.Delay(1000);
        return "서버1 데이터";
    }

    async UniTask<string> FetchFromServer2Async()
    {
        await UniTask.Delay(500);
        return "서버2 데이터";
    }

    async UniTask<string> FetchFromServer3Async()
    {
        await UniTask.Delay(1500);
        return "서버3 데이터";
    }
}
```

### 순차 실행

```csharp
public class SequentialExample : MonoBehaviour
{
    async void Start()
    {
        // 순차적으로 실행
        await LoadLevelSequentiallyAsync();
    }

    async UniTask LoadLevelSequentiallyAsync()
    {
        Debug.Log("1. 기본 리소스 로드 시작");
        await LoadBasicResourcesAsync();

        Debug.Log("2. 레벨 데이터 로드 시작");
        await LoadLevelDataAsync();

        Debug.Log("3. 적 스폰 시작");
        await SpawnEnemiesAsync();

        Debug.Log("모든 단계 완료");
    }

    async UniTask LoadBasicResourcesAsync()
    {
        await UniTask.Delay(1000);
    }

    async UniTask LoadLevelDataAsync()
    {
        await UniTask.Delay(1500);
    }

    async UniTask SpawnEnemiesAsync()
    {
        await UniTask.Delay(500);
    }
}
```

## UniTaskCompletionSource

콜백을 UniTask로 변환.

### 기본 사용법

```csharp
public class CompletionSourceExample : MonoBehaviour
{
    async void Start()
    {
        var result = await ConvertCallbackToUniTaskAsync();
        Debug.Log($"결과: {result}");
    }

    UniTask<string> ConvertCallbackToUniTaskAsync()
    {
        var tcs = new UniTaskCompletionSource<string>();

        // 콜백 기반 API 호출
        StartLegacyCallbackOperation(result =>
        {
            // 콜백에서 결과 설정
            tcs.TrySetResult(result);
        });

        return tcs.Task;
    }

    void StartLegacyCallbackOperation(System.Action<string> callback)
    {
        // 레거시 콜백 API 시뮬레이션
        StartCoroutine(DelayedCallback(callback));
    }

    System.Collections.IEnumerator DelayedCallback(System.Action<string> callback)
    {
        yield return new WaitForSeconds(1f);
        callback?.Invoke("작업 완료");
    }
}
```

### 타임아웃과 함께 사용

```csharp
public class CompletionSourceTimeoutExample : MonoBehaviour
{
    async void Start()
    {
        try
        {
            var result = await WaitForEventWithTimeoutAsync(3000);
            Debug.Log($"이벤트 수신: {result}");
        }
        catch (TimeoutException)
        {
            Debug.LogError("이벤트 타임아웃!");
        }
    }

    async UniTask<string> WaitForEventWithTimeoutAsync(int timeoutMs)
    {
        var tcs = new UniTaskCompletionSource<string>();

        // 타임아웃 설정
        var cts = new CancellationTokenSource();
        cts.CancelAfterSlim(TimeSpan.FromMilliseconds(timeoutMs));

        // 이벤트 리스너 등록
        RegisterEventListener(result => tcs.TrySetResult(result));

        // 타임아웃 처리
        cts.Token.Register(() => tcs.TrySetException(new TimeoutException()));

        return await tcs.Task;
    }

    void RegisterEventListener(System.Action<string> callback)
    {
        // 이벤트 리스너 등록 로직
    }
}
```

## PlayerLoopTiming

Unity의 업데이트 루프 타이밍 제어.

### 다양한 타이밍 사용

```csharp
public class PlayerLoopTimingExample : MonoBehaviour
{
    async void Start()
    {
        var ct = this.GetCancellationTokenOnDestroy();

        // Update 타이밍
        await UniTask.Yield(PlayerLoopTiming.Update, ct);
        Debug.Log("Update 타이밍");

        // FixedUpdate 타이밍
        await UniTask.Yield(PlayerLoopTiming.FixedUpdate, ct);
        Debug.Log("FixedUpdate 타이밍");

        // LateUpdate 타이밍
        await UniTask.Yield(PlayerLoopTiming.LateUpdate, ct);
        Debug.Log("LateUpdate 타이밍");

        // PreUpdate 타이밍
        await UniTask.Yield(PlayerLoopTiming.PreUpdate, ct);
        Debug.Log("PreUpdate 타이밍");

        // PostLateUpdate 타이밍
        await UniTask.Yield(PlayerLoopTiming.PostLateUpdate, ct);
        Debug.Log("PostLateUpdate 타이밍");
    }
}
```

### 물리 시뮬레이션과 동기화

```csharp
public class PhysicsSyncExample : MonoBehaviour
{
    async void Start()
    {
        await SimulatePhysicsAsync();
    }

    async UniTask SimulatePhysicsAsync()
    {
        for (int i = 0; i < 100; i++)
        {
            // FixedUpdate 타이밍에서 실행
            await UniTask.Yield(PlayerLoopTiming.FixedUpdate);

            // 물리 계산
            ApplyForce();
        }
    }

    void ApplyForce()
    {
        // 물리 힘 적용
    }
}
```

## 조건 대기

### WaitUntil

```csharp
public class WaitUntilExample : MonoBehaviour
{
    private bool mIsReady = false;

    async void Start()
    {
        Debug.Log("준비 대기 중...");

        // 조건이 true가 될 때까지 대기
        await UniTask.WaitUntil(() => mIsReady);

        Debug.Log("준비 완료!");
    }

    void Update()
    {
        if (Input.GetKeyDown(KeyCode.Space))
        {
            mIsReady = true;
        }
    }
}
```

### WaitWhile

```csharp
public class WaitWhileExample : MonoBehaviour
{
    private bool mIsLoading = true;

    async void Start()
    {
        StartLoadingAsync();

        Debug.Log("로딩 중...");

        // 조건이 false가 될 때까지 대기
        await UniTask.WaitWhile(() => mIsLoading);

        Debug.Log("로딩 완료!");
    }

    async UniTaskVoid StartLoadingAsync()
    {
        await UniTask.Delay(3000);
        mIsLoading = false;
    }
}
```

### WaitUntilValueChanged

값이 변경될 때까지 대기.

```csharp
public class WaitUntilValueChangedExample : MonoBehaviour
{
    [SerializeField] private UnityEngine.UI.Slider mHealthSlider;

    async void Start()
    {
        Debug.Log("체력 변경 대기 중...");

        // 슬라이더 값이 변경될 때까지 대기
        await mHealthSlider.OnValueChangedAsync();

        Debug.Log($"체력 변경됨: {mHealthSlider.value}");
    }
}
```

## 예외 처리

### 기본 예외 처리

```csharp
public class ExceptionHandlingExample : MonoBehaviour
{
    async void Start()
    {
        try
        {
            await RiskyOperationAsync();
        }
        catch (InvalidOperationException ex)
        {
            Debug.LogError($"작업 오류: {ex.Message}");
        }
        catch (OperationCanceledException)
        {
            Debug.Log("작업 취소됨");
        }
        catch (Exception ex)
        {
            Debug.LogError($"예기치 않은 오류: {ex.Message}");
        }
    }

    async UniTask RiskyOperationAsync()
    {
        await UniTask.Delay(1000);
        throw new InvalidOperationException("작업 실패");
    }
}
```

### 예외 전파

```csharp
public class ExceptionPropagationExample : MonoBehaviour
{
    async void Start()
    {
        try
        {
            await HighLevelOperationAsync();
        }
        catch (Exception ex)
        {
            Debug.LogError($"상위 레벨 오류: {ex.Message}");
        }
    }

    async UniTask HighLevelOperationAsync()
    {
        // 예외는 상위로 전파됨
        await LowLevelOperationAsync();
    }

    async UniTask LowLevelOperationAsync()
    {
        await UniTask.Delay(500);
        throw new Exception("하위 레벨 오류");
    }
}
```

### SuppressCancellationThrow

취소 예외를 결과로 변환.

```csharp
public class SuppressCancellationExample : MonoBehaviour
{
    async void Start()
    {
        var cts = new CancellationTokenSource();
        cts.CancelAfterSlim(TimeSpan.FromSeconds(1));

        // 취소 예외를 던지지 않고 bool로 반환
        var (success, result) = await LoadDataAsync(cts.Token)
            .SuppressCancellationThrow();

        if (success)
        {
            Debug.Log($"데이터 로드 성공: {result}");
        }
        else
        {
            Debug.Log("데이터 로드 취소됨");
        }
    }

    async UniTask<string> LoadDataAsync(CancellationToken ct)
    {
        await UniTask.Delay(2000, cancellationToken: ct);
        return "게임 데이터";
    }
}
```
