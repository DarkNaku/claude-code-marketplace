# UniTask 통합 패턴

UniTask를 Unity 기능 및 다른 라이브러리와 통합하는 고급 패턴.

## UnityWebRequest 비동기 처리

### 기본 GET 요청

```csharp
using Cysharp.Threading.Tasks;
using UnityEngine;
using UnityEngine.Networking;

public class WebRequestExample : MonoBehaviour
{
    async void Start()
    {
        var result = await FetchTextAsync("https://api.example.com/data");
        Debug.Log($"응답: {result}");
    }

    async UniTask<string> FetchTextAsync(string url)
    {
        using var request = UnityWebRequest.Get(url);

        // UnityWebRequest를 비동기로 실행
        await request.SendWebRequest();

        if (request.result == UnityWebRequest.Result.Success)
        {
            return request.downloadHandler.text;
        }
        else
        {
            throw new System.Exception($"요청 실패: {request.error}");
        }
    }
}
```

### 진행 상황 추적

```csharp
public class DownloadProgressExample : MonoBehaviour
{
    [SerializeField] private UnityEngine.UI.Slider mProgressBar;

    async void Start()
    {
        await DownloadFileWithProgressAsync("https://example.com/file.zip");
    }

    async UniTask DownloadFileWithProgressAsync(string url)
    {
        using var request = UnityWebRequest.Get(url);

        // Progress를 사용하여 진행 상황 추적
        var progress = Progress.Create<float>(p =>
        {
            mProgressBar.value = p;
            Debug.Log($"다운로드 진행: {p * 100:F1}%");
        });

        await request.SendWebRequest().ToUniTask(progress: progress);

        if (request.result == UnityWebRequest.Result.Success)
        {
            Debug.Log("다운로드 완료!");
        }
    }
}
```

### POST 요청과 JSON 처리

```csharp
using System.Text;

public class PostRequestExample : MonoBehaviour
{
    [System.Serializable]
    public class LoginRequest
    {
        public string username;
        public string password;
    }

    [System.Serializable]
    public class LoginResponse
    {
        public string token;
        public string userId;
    }

    async void Start()
    {
        var request = new LoginRequest
        {
            username = "player1",
            password = "password123"
        };

        var response = await PostJsonAsync<LoginResponse>(
            "https://api.example.com/login",
            request
        );

        Debug.Log($"로그인 토큰: {response.token}");
    }

    async UniTask<TResponse> PostJsonAsync<TResponse>(string url, object data)
    {
        var json = JsonUtility.ToJson(data);
        var bytes = Encoding.UTF8.GetBytes(json);

        using var request = new UnityWebRequest(url, "POST");
        request.uploadHandler = new UploadHandlerRaw(bytes);
        request.downloadHandler = new DownloadHandlerBuffer();
        request.SetRequestHeader("Content-Type", "application/json");

        await request.SendWebRequest();

        if (request.result == UnityWebRequest.Result.Success)
        {
            return JsonUtility.FromJson<TResponse>(request.downloadHandler.text);
        }
        else
        {
            throw new System.Exception($"요청 실패: {request.error}");
        }
    }
}
```

### 재시도 로직

```csharp
public class RetryExample : MonoBehaviour
{
    async void Start()
    {
        var data = await FetchWithRetryAsync("https://api.example.com/data", maxRetries: 3);
        Debug.Log($"데이터: {data}");
    }

    async UniTask<string> FetchWithRetryAsync(string url, int maxRetries)
    {
        for (int i = 0; i < maxRetries; i++)
        {
            try
            {
                using var request = UnityWebRequest.Get(url);
                await request.SendWebRequest();

                if (request.result == UnityWebRequest.Result.Success)
                {
                    return request.downloadHandler.text;
                }
            }
            catch (Exception ex)
            {
                Debug.LogWarning($"시도 {i + 1}/{maxRetries} 실패: {ex.Message}");

                if (i < maxRetries - 1)
                {
                    // 지수 백오프
                    await UniTask.Delay((int)Mathf.Pow(2, i) * 1000);
                }
            }
        }

        throw new Exception($"{maxRetries}번 재시도 후 실패");
    }
}
```

## Addressables 비동기 로딩

### 기본 에셋 로드

```csharp
using Cysharp.Threading.Tasks;
using UnityEngine;
using UnityEngine.AddressableAssets;

public class AddressablesExample : MonoBehaviour
{
    async void Start()
    {
        var prefab = await LoadAssetAsync<GameObject>("Player");
        Instantiate(prefab);
    }

    async UniTask<T> LoadAssetAsync<T>(string key) where T : Object
    {
        var handle = Addressables.LoadAssetAsync<T>(key);
        return await handle.ToUniTask();
    }
}
```

### 여러 에셋 병렬 로드

```csharp
public class AddressablesParallelExample : MonoBehaviour
{
    async void Start()
    {
        var (player, enemy, item) = await UniTask.WhenAll(
            LoadAssetAsync<GameObject>("Player"),
            LoadAssetAsync<GameObject>("Enemy"),
            LoadAssetAsync<GameObject>("Item")
        );

        Instantiate(player);
        Instantiate(enemy);
        Instantiate(item);
    }

    async UniTask<T> LoadAssetAsync<T>(string key) where T : Object
    {
        var handle = Addressables.LoadAssetAsync<T>(key);
        return await handle.ToUniTask();
    }
}
```

### 진행 상황과 함께 씬 로드

```csharp
using UnityEngine.ResourceManagement.AsyncOperations;
using UnityEngine.ResourceManagement.ResourceProviders;
using UnityEngine.SceneManagement;

public class AddressablesSceneExample : MonoBehaviour
{
    [SerializeField] private UnityEngine.UI.Slider mProgressBar;
    [SerializeField] private UnityEngine.UI.Text mProgressText;

    async void Start()
    {
        await LoadSceneWithProgressAsync("GameLevel1");
    }

    async UniTask LoadSceneWithProgressAsync(string sceneKey)
    {
        var handle = Addressables.LoadSceneAsync(sceneKey, LoadSceneMode.Additive);

        while (!handle.IsDone)
        {
            var progress = handle.PercentComplete;
            mProgressBar.value = progress;
            mProgressText.text = $"로딩... {progress * 100:F0}%";

            await UniTask.Yield();
        }

        if (handle.Status == AsyncOperationStatus.Succeeded)
        {
            Debug.Log("씬 로드 완료!");
        }
    }
}
```

## VContainer 통합

### 비동기 초기화

```csharp
using Cysharp.Threading.Tasks;
using VContainer;
using VContainer.Unity;
using UnityEngine;

// 비동기 초기화가 필요한 서비스
public interface IGameDataService
{
    UniTask InitializeAsync();
    string GetPlayerName();
}

public class GameDataService : IGameDataService
{
    private string mPlayerName;

    public async UniTask InitializeAsync()
    {
        Debug.Log("게임 데이터 로드 중...");
        await UniTask.Delay(2000);
        mPlayerName = "플레이어1";
        Debug.Log("게임 데이터 로드 완료");
    }

    public string GetPlayerName() => mPlayerName;
}

// 비동기 EntryPoint
public class GameInitializer : IAsyncStartable
{
    private readonly IGameDataService mDataService;

    public GameInitializer(IGameDataService dataService)
    {
        mDataService = dataService;
    }

    public async UniTask StartAsync(CancellationToken cancellation)
    {
        Debug.Log("게임 초기화 시작");
        await mDataService.InitializeAsync();
        Debug.Log("게임 초기화 완료");
    }
}

// LifetimeScope 등록
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<IGameDataService, GameDataService>(Lifetime.Singleton);
        builder.RegisterEntryPoint<GameInitializer>();
    }
}
```

### 비동기 팩토리 패턴

```csharp
public interface IEnemyFactory
{
    UniTask<GameObject> CreateEnemyAsync(string enemyType);
}

public class EnemyFactory : IEnemyFactory
{
    private readonly IObjectResolver mResolver;

    public EnemyFactory(IObjectResolver resolver)
    {
        mResolver = resolver;
    }

    public async UniTask<GameObject> CreateEnemyAsync(string enemyType)
    {
        // Addressables에서 비동기 로드
        var handle = Addressables.LoadAssetAsync<GameObject>($"Enemy_{enemyType}");
        var prefab = await handle.ToUniTask();

        var enemy = Object.Instantiate(prefab);

        // DI를 통한 의존성 주입
        mResolver.Inject(enemy);

        return enemy;
    }
}

// 사용 예
public class EnemySpawner : MonoBehaviour
{
    [Inject] private IEnemyFactory mEnemyFactory;

    async void Start()
    {
        var enemy = await mEnemyFactory.CreateEnemyAsync("Orc");
        Debug.Log("적 생성 완료");
    }
}
```

## MessagePipe 통합

### 비동기 메시지 핸들러

```csharp
using MessagePipe;
using Cysharp.Threading.Tasks;
using System.Threading;

public struct SaveGameMessage
{
    public string SaveSlot;
}

public class AsyncSaveHandler : MonoBehaviour
{
    [Inject] private IAsyncSubscriber<SaveGameMessage> mSaveSubscriber;
    private IDisposable mSubscription;

    void Start()
    {
        mSubscription = mSaveSubscriber.Subscribe(async (msg, ct) =>
        {
            await HandleSaveAsync(msg, ct);
        });
    }

    void OnDestroy()
    {
        mSubscription?.Dispose();
    }

    async UniTask HandleSaveAsync(SaveGameMessage msg, CancellationToken ct)
    {
        Debug.Log($"슬롯 {msg.SaveSlot}에 저장 중...");

        // 비동기 저장 작업
        await UniTask.Delay(1000, cancellationToken: ct);

        Debug.Log("저장 완료!");
    }
}
```

### 비동기 Request/Response

```csharp
public struct LoadPlayerDataRequest
{
    public int PlayerId;
}

public struct LoadPlayerDataResponse
{
    public string PlayerName;
    public int Level;
}

public class AsyncPlayerDataHandler : IAsyncRequestHandler<LoadPlayerDataRequest, LoadPlayerDataResponse>
{
    public async UniTask<LoadPlayerDataResponse> InvokeAsync(
        LoadPlayerDataRequest request,
        CancellationToken cancellationToken)
    {
        // 비동기 데이터베이스 쿼리
        await UniTask.Delay(500, cancellationToken: cancellationToken);

        return new LoadPlayerDataResponse
        {
            PlayerName = $"Player{request.PlayerId}",
            Level = 10
        };
    }
}

// 사용 예
public class PlayerInfoUI : MonoBehaviour
{
    [Inject] private IAsyncRequestHandler<LoadPlayerDataRequest, LoadPlayerDataResponse> mRequestHandler;

    async void Start()
    {
        var response = await mRequestHandler.InvokeAsync(
            new LoadPlayerDataRequest { PlayerId = 1 },
            this.GetCancellationTokenOnDestroy()
        );

        Debug.Log($"플레이어: {response.PlayerName}, 레벨: {response.Level}");
    }
}
```

## DOTween 통합

### 애니메이션 대기

```csharp
using DG.Tweening;
using Cysharp.Threading.Tasks;
using UnityEngine;

public class DOTweenExample : MonoBehaviour
{
    async void Start()
    {
        // DOTween 애니메이션을 UniTask로 대기
        await transform.DOMoveX(5f, 1f).ToUniTask();
        Debug.Log("이동 완료");

        await transform.DORotate(new Vector3(0, 180, 0), 1f).ToUniTask();
        Debug.Log("회전 완료");
    }
}
```

### 순차 애니메이션

```csharp
public class SequentialAnimationExample : MonoBehaviour
{
    async void Start()
    {
        await PlaySequentialAnimationAsync();
    }

    async UniTask PlaySequentialAnimationAsync()
    {
        // 1. 위로 이동
        await transform.DOMoveY(transform.position.y + 2f, 0.5f).ToUniTask();

        // 2. 회전
        await transform.DORotate(new Vector3(0, 360, 0), 1f).ToUniTask();

        // 3. 스케일
        await transform.DOScale(Vector3.one * 2f, 0.5f).ToUniTask();

        Debug.Log("모든 애니메이션 완료");
    }
}
```

### 병렬 애니메이션

```csharp
public class ParallelAnimationExample : MonoBehaviour
{
    async void Start()
    {
        // 여러 애니메이션 동시 실행
        await UniTask.WhenAll(
            transform.DOMoveX(5f, 1f).ToUniTask(),
            transform.DORotate(new Vector3(0, 180, 0), 1f).ToUniTask(),
            transform.DOScale(Vector3.one * 2f, 1f).ToUniTask()
        );

        Debug.Log("모든 애니메이션 동시 완료");
    }
}
```

## UI 이벤트 처리

### 버튼 클릭 대기

```csharp
using Cysharp.Threading.Tasks;
using UnityEngine;
using UnityEngine.UI;

public class ButtonAwaitExample : MonoBehaviour
{
    [SerializeField] private Button mStartButton;

    async void Start()
    {
        Debug.Log("시작 버튼 클릭 대기 중...");

        // 버튼 클릭을 비동기로 대기
        await mStartButton.OnClickAsync();

        Debug.Log("게임 시작!");
        await StartGameAsync();
    }

    async UniTask StartGameAsync()
    {
        await UniTask.Delay(1000);
        Debug.Log("게임 로드 완료");
    }
}
```

### 복잡한 UI 플로우

```csharp
public class DialogFlowExample : MonoBehaviour
{
    [SerializeField] private Button mYesButton;
    [SerializeField] private Button mNoButton;
    [SerializeField] private GameObject mDialogPanel;

    async void Start()
    {
        var userConfirmed = await ShowConfirmDialogAsync("게임을 시작하시겠습니까?");

        if (userConfirmed)
        {
            await StartGameAsync();
        }
        else
        {
            Debug.Log("게임 시작 취소");
        }
    }

    async UniTask<bool> ShowConfirmDialogAsync(string message)
    {
        mDialogPanel.SetActive(true);

        // 사용자가 버튼을 클릭할 때까지 대기
        var (winIndex, _) = await UniTask.WhenAny(
            mYesButton.OnClickAsync(),
            mNoButton.OnClickAsync()
        );

        mDialogPanel.SetActive(false);

        return winIndex == 0; // 0이면 Yes, 1이면 No
    }

    async UniTask StartGameAsync()
    {
        await UniTask.Delay(1000);
        Debug.Log("게임 시작!");
    }
}
```

## AsyncLazy - 지연 초기화

한 번만 실행되는 비동기 초기화.

```csharp
public class LazyInitExample : MonoBehaviour
{
    private AsyncLazy<Texture2D> mLazyTexture;

    void Awake()
    {
        // 처음 사용될 때 로드됨
        mLazyTexture = UniTask.Lazy(async () =>
        {
            Debug.Log("텍스처 로드 시작");
            await UniTask.Delay(2000);
            var texture = await LoadTextureAsync("player_avatar");
            Debug.Log("텍스처 로드 완료");
            return texture;
        });
    }

    async void Start()
    {
        // 첫 번째 호출: 로드 시작
        var texture1 = await mLazyTexture;
        Debug.Log("첫 번째 접근");

        // 두 번째 호출: 캐시된 결과 반환
        var texture2 = await mLazyTexture;
        Debug.Log("두 번째 접근 (캐시)");

        // texture1과 texture2는 동일한 인스턴스
    }

    async UniTask<Texture2D> LoadTextureAsync(string name)
    {
        await UniTask.Delay(1000);
        return new Texture2D(256, 256);
    }
}
```

## Channel - 프로듀서/컨슈머 패턴

```csharp
using Cysharp.Threading.Tasks;
using Cysharp.Threading.Tasks.Linq;
using System.Threading.Channels;

public class ChannelExample : MonoBehaviour
{
    async void Start()
    {
        var channel = Channel.CreateUnbounded<int>();

        // 프로듀서 시작
        ProduceNumbersAsync(channel.Writer, this.GetCancellationTokenOnDestroy()).Forget();

        // 컨슈머 시작
        await ConsumeNumbersAsync(channel.Reader, this.GetCancellationTokenOnDestroy());
    }

    async UniTaskVoid ProduceNumbersAsync(ChannelWriter<int> writer, CancellationToken ct)
    {
        try
        {
            for (int i = 0; i < 10; i++)
            {
                await UniTask.Delay(500, cancellationToken: ct);
                await writer.WriteAsync(i, ct);
                Debug.Log($"생성: {i}");
            }
        }
        finally
        {
            writer.Complete();
        }
    }

    async UniTask ConsumeNumbersAsync(ChannelReader<int> reader, CancellationToken ct)
    {
        await foreach (var number in reader.ReadAllAsync(ct))
        {
            Debug.Log($"소비: {number}");
            await ProcessNumberAsync(number);
        }
    }

    async UniTask ProcessNumberAsync(int number)
    {
        await UniTask.Delay(200);
        // 처리 로직
    }
}
```

## AsyncReactiveProperty - 반응형 값

값 변경을 비동기로 관찰.

```csharp
using Cysharp.Threading.Tasks;
using Cysharp.Threading.Tasks.Linq;

public class AsyncReactivePropertyExample : MonoBehaviour
{
    private AsyncReactiveProperty<int> mHealth = new AsyncReactiveProperty<int>(100);

    async void Start()
    {
        // 값 변경 관찰
        ObserveHealthChangesAsync(this.GetCancellationTokenOnDestroy()).Forget();

        // 값 변경
        await UniTask.Delay(1000);
        mHealth.Value = 80;

        await UniTask.Delay(1000);
        mHealth.Value = 50;

        await UniTask.Delay(1000);
        mHealth.Value = 0;
    }

    async UniTaskVoid ObserveHealthChangesAsync(CancellationToken ct)
    {
        await foreach (var health in mHealth.WithoutCurrent().WithCancellation(ct))
        {
            Debug.Log($"체력 변경: {health}");

            if (health <= 0)
            {
                Debug.Log("플레이어 사망!");
                break;
            }
            else if (health < 30)
            {
                Debug.LogWarning("체력이 낮습니다!");
            }
        }
    }
}
```

## 씬 전환과 비동기 처리

### 씬 로드 및 초기화

```csharp
using UnityEngine.SceneManagement;

public class SceneTransitionExample : MonoBehaviour
{
    [SerializeField] private UnityEngine.UI.Slider mProgressBar;

    async void Start()
    {
        await LoadSceneWithProgressAsync("GameLevel1");
    }

    async UniTask LoadSceneWithProgressAsync(string sceneName)
    {
        // 씬 로드 시작
        var operation = SceneManager.LoadSceneAsync(sceneName);
        operation.allowSceneActivation = false;

        // 90%까지 로드 진행
        while (operation.progress < 0.9f)
        {
            mProgressBar.value = operation.progress;
            await UniTask.Yield();
        }

        // 나머지 리소스 로드
        await LoadAdditionalResourcesAsync();

        mProgressBar.value = 1f;

        // 씬 활성화
        operation.allowSceneActivation = true;
        await operation;

        Debug.Log("씬 로드 및 초기화 완료");
    }

    async UniTask LoadAdditionalResourcesAsync()
    {
        await UniTask.Delay(1000);
        // 추가 리소스 로드
    }
}
```

### 멀티씬 동시 로드

```csharp
public class MultiSceneLoadExample : MonoBehaviour
{
    async void Start()
    {
        await LoadMultipleScenesAsync();
    }

    async UniTask LoadMultipleScenesAsync()
    {
        // 여러 씬 동시 로드
        await UniTask.WhenAll(
            LoadSceneAsync("UI", LoadSceneMode.Additive),
            LoadSceneAsync("Gameplay", LoadSceneMode.Additive),
            LoadSceneAsync("Audio", LoadSceneMode.Additive)
        );

        Debug.Log("모든 씬 로드 완료");
    }

    async UniTask LoadSceneAsync(string sceneName, LoadSceneMode mode)
    {
        var operation = SceneManager.LoadSceneAsync(sceneName, mode);
        await operation;
        Debug.Log($"{sceneName} 로드 완료");
    }
}
```
