# R3 통합 패턴

R3와 다른 Unity 시스템 및 라이브러리와의 통합 패턴.

## UniTask와 Observable 통합

### Observable을 UniTask로 변환

```csharp
using R3;
using Cysharp.Threading.Tasks;
using UnityEngine;

public class DataLoader : MonoBehaviour
{
    async UniTaskVoid Start()
    {
        // Observable의 첫 번째 값을 await
        var firstValue = await Observable.Timer(TimeSpan.FromSeconds(1))
            .ToUniTask(useFirstValue: true);

        Debug.Log($"첫 값: {firstValue}");

        // Observable의 마지막 값을 await
        var lastValue = await Observable.Range(1, 5)
            .ToUniTask(useFirstValue: false);

        Debug.Log($"마지막 값: {lastValue}");

        // 여러 값을 리스트로
        var allValues = await Observable.Range(1, 5)
            .ToArrayAsync();

        Debug.Log($"모든 값: {string.Join(", ", allValues)}");
    }
}
```

### UniTask를 Observable로 변환

```csharp
public class AsyncOperations : MonoBehaviour
{
    void Start()
    {
        // UniTask를 Observable로 변환
        LoadDataAsync()
            .ToObservable()
            .Subscribe(
                onNext: data => Debug.Log($"데이터 로드됨: {data}"),
                onError: ex => Debug.LogError($"에러: {ex.Message}")
            )
            .AddTo(this);

        // 여러 UniTask를 Observable로 병합
        Observable.Merge(
            LoadPlayerDataAsync().ToObservable(),
            LoadInventoryDataAsync().ToObservable(),
            LoadQuestDataAsync().ToObservable()
        )
        .Subscribe(
            onNext: _ => Debug.Log("데이터 로드 완료"),
            onCompleted: () => Debug.Log("모든 데이터 로드 완료")
        )
        .AddTo(this);
    }

    async UniTask<string> LoadDataAsync()
    {
        await UniTask.Delay(TimeSpan.FromSeconds(1));
        return "데이터";
    }

    async UniTask LoadPlayerDataAsync() => await UniTask.Delay(100);
    async UniTask LoadInventoryDataAsync() => await UniTask.Delay(200);
    async UniTask LoadQuestDataAsync() => await UniTask.Delay(150);
}
```

### 반응형 비동기 파이프라인

```csharp
public class NetworkClient : MonoBehaviour
{
    private Subject<string> mRequestQueue = new();

    void Start()
    {
        // 요청을 큐에 쌓고 순차적으로 처리
        mRequestQueue
            .SelectAwait(async (url, cancellationToken) =>
            {
                return await FetchAsync(url, cancellationToken);
            })
            .Subscribe(
                onNext: response => ProcessResponse(response),
                onError: ex => Debug.LogError($"네트워크 에러: {ex.Message}")
            )
            .AddTo(this);
    }

    public void QueueRequest(string url)
    {
        mRequestQueue.OnNext(url);
    }

    async UniTask<string> FetchAsync(string url, CancellationToken cancellationToken)
    {
        await UniTask.Delay(TimeSpan.FromSeconds(1), cancellationToken: cancellationToken);
        return $"응답: {url}";
    }

    void ProcessResponse(string response) => Debug.Log(response);
}
```

## VContainer와 함께 사용하기

### ReactiveProperty를 서비스에 주입

```csharp
using VContainer;
using VContainer.Unity;
using R3;
using UnityEngine;

// Model을 서비스로 등록
public class GameStateModel
{
    public ReactiveProperty<int> Score { get; } = new(0);
    public ReactiveProperty<GamePhase> CurrentPhase { get; } = new(GamePhase.Menu);
    public ReadOnlyReactiveProperty<bool> IsPlaying { get; }

    public GameStateModel()
    {
        IsPlaying = CurrentPhase
            .Select(phase => phase == GamePhase.Playing)
            .ToReadOnlyReactiveProperty();
    }
}

public enum GamePhase { Menu, Playing, Paused, GameOver }

// VContainer 설정
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        // Model을 싱글톤으로 등록
        builder.Register<GameStateModel>(Lifetime.Singleton);
        builder.Register<IScoreService, ScoreService>(Lifetime.Singleton);

        // View 등록
        builder.RegisterComponentInHierarchy<GameUI>();
        builder.RegisterComponentInHierarchy<ScoreDisplay>();
    }
}

// 서비스에서 Model 사용
public interface IScoreService
{
    void AddScore(int points);
    IReadOnlyReactiveProperty<int> Score { get; }
}

public class ScoreService : IScoreService
{
    private readonly GameStateModel mGameState;

    public IReadOnlyReactiveProperty<int> Score => mGameState.Score;

    public ScoreService(GameStateModel gameState)
    {
        mGameState = gameState;
    }

    public void AddScore(int points)
    {
        if (mGameState.IsPlaying.CurrentValue)
        {
            mGameState.Score.Value += points;
        }
    }
}

// View에서 구독
public class ScoreDisplay : MonoBehaviour
{
    [SerializeField] private TMPro.TextMeshProUGUI mScoreText;

    private IScoreService mScoreService;
    private CompositeDisposable mDisposables = new();

    [Inject]
    public void Construct(IScoreService scoreService)
    {
        mScoreService = scoreService;
    }

    void Start()
    {
        mScoreService.Score
            .Subscribe(score => mScoreText.text = $"점수: {score:N0}")
            .AddTo(mDisposables);
    }

    void OnDestroy()
    {
        mDisposables?.Dispose();
    }
}
```

### EntryPoint에서 Observable 사용

```csharp
public class GameLoopManager : IStartable, IDisposable
{
    private readonly GameStateModel mGameState;
    private readonly IScoreService mScoreService;
    private readonly CompositeDisposable mDisposables = new();

    public GameLoopManager(GameStateModel gameState, IScoreService scoreService)
    {
        mGameState = gameState;
        mScoreService = scoreService;
    }

    public void Start()
    {
        // 게임 상태 변경 감지
        mGameState.CurrentPhase
            .Subscribe(phase => OnPhaseChanged(phase))
            .AddTo(mDisposables);

        // 플레이 중일 때만 자동 점수 증가
        Observable.Interval(TimeSpan.FromSeconds(1))
            .Where(_ => mGameState.IsPlaying.CurrentValue)
            .Subscribe(_ => mScoreService.AddScore(1))
            .AddTo(mDisposables);
    }

    public void Dispose()
    {
        mDisposables?.Dispose();
    }

    void OnPhaseChanged(GamePhase phase)
    {
        Debug.Log($"게임 페이즈: {phase}");
    }
}

// LifetimeScope에 등록
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<GameStateModel>(Lifetime.Singleton);
        builder.Register<IScoreService, ScoreService>(Lifetime.Singleton);
        builder.RegisterEntryPoint<GameLoopManager>();
    }
}
```

## Unity UI 이벤트 바인딩

### Button 이벤트

```csharp
using UnityEngine.UI;
using TMPro;

public class UIController : MonoBehaviour
{
    [SerializeField] private Button mStartButton;
    [SerializeField] private Button mPauseButton;
    [SerializeField] private Button mResumeButton;

    private CompositeDisposable mDisposables = new();

    void Start()
    {
        // 버튼 클릭 이벤트
        mStartButton.OnClickAsObservable()
            .Subscribe(_ => OnStartGame())
            .AddTo(mDisposables);

        // 버튼 클릭 제한 (1초에 한 번만)
        mPauseButton.OnClickAsObservable()
            .Throttle(TimeSpan.FromSeconds(1))
            .Subscribe(_ => OnPauseGame())
            .AddTo(mDisposables);

        // 더블 클릭 감지
        mResumeButton.OnClickAsObservable()
            .Buffer(mResumeButton.OnClickAsObservable().Throttle(TimeSpan.FromMilliseconds(300)))
            .Where(clicks => clicks.Count >= 2)
            .Subscribe(_ => OnDoubleClick())
            .AddTo(mDisposables);
    }

    void OnDestroy() => mDisposables?.Dispose();

    void OnStartGame() => Debug.Log("게임 시작");
    void OnPauseGame() => Debug.Log("일시정지");
    void OnDoubleClick() => Debug.Log("더블 클릭!");
}
```

### InputField 이벤트

```csharp
public class LoginForm : MonoBehaviour
{
    [SerializeField] private TMP_InputField mUsernameInput;
    [SerializeField] private TMP_InputField mPasswordInput;
    [SerializeField] private Button mLoginButton;

    private ReactiveProperty<string> mUsername = new("");
    private ReactiveProperty<string> mPassword = new("");
    private CompositeDisposable mDisposables = new();

    void Start()
    {
        // InputField 값 변경 감지
        mUsernameInput.OnValueChangedAsObservable()
            .Subscribe(value => mUsername.Value = value)
            .AddTo(mDisposables);

        mPasswordInput.OnValueChangedAsObservable()
            .Subscribe(value => mPassword.Value = value)
            .AddTo(mDisposables);

        // 입력 검증 및 버튼 활성화
        var isValid = mUsername.CombineLatest(mPassword, (user, pass) =>
            !string.IsNullOrEmpty(user) && pass.Length >= 6
        );

        isValid
            .Subscribe(valid => mLoginButton.interactable = valid)
            .AddTo(mDisposables);

        // 로그인 버튼
        mLoginButton.OnClickAsObservable()
            .Subscribe(_ => Login(mUsername.Value, mPassword.Value))
            .AddTo(mDisposables);

        // 실시간 검증 메시지 (0.5초 지연)
        mPasswordInput.OnValueChangedAsObservable()
            .Throttle(TimeSpan.FromSeconds(0.5f))
            .Subscribe(pass => ValidatePassword(pass))
            .AddTo(mDisposables);
    }

    void OnDestroy() => mDisposables?.Dispose();

    void Login(string username, string password)
    {
        Debug.Log($"로그인: {username}");
    }

    void ValidatePassword(string password)
    {
        if (password.Length > 0 && password.Length < 6)
            Debug.Log("비밀번호는 6자 이상이어야 합니다");
    }
}
```

### Slider와 Toggle

```csharp
public class SettingsPanel : MonoBehaviour
{
    [SerializeField] private Slider mVolumeSlider;
    [SerializeField] private Toggle mMuteToggle;
    [SerializeField] private TextMeshProUGUI mVolumeText;

    private ReactiveProperty<float> mVolume = new(1f);
    private ReactiveProperty<bool> mIsMuted = new(false);
    private CompositeDisposable mDisposables = new();

    void Start()
    {
        // Slider 값 변경
        mVolumeSlider.OnValueChangedAsObservable()
            .Subscribe(value => mVolume.Value = value)
            .AddTo(mDisposables);

        // Toggle 상태 변경
        mMuteToggle.OnValueChangedAsObservable()
            .Subscribe(value => mIsMuted.Value = value)
            .AddTo(mDisposables);

        // 실제 볼륨 계산 (뮤트 고려)
        var effectiveVolume = mVolume.CombineLatest(mIsMuted, (vol, muted) =>
            muted ? 0f : vol
        );

        effectiveVolume
            .Subscribe(vol => SetVolume(vol))
            .AddTo(mDisposables);

        // UI 텍스트 업데이트
        mVolume
            .Subscribe(vol => mVolumeText.text = $"볼륨: {(int)(vol * 100)}%")
            .AddTo(mDisposables);
    }

    void OnDestroy() => mDisposables?.Dispose();

    void SetVolume(float volume)
    {
        AudioListener.volume = volume;
    }
}
```

## Input System 통합

### New Input System과 R3

```csharp
using UnityEngine.InputSystem;

public class PlayerInput : MonoBehaviour
{
    [SerializeField] private PlayerInputActions mInputActions;

    private Subject<Vector2> mMoveInput = new();
    private Subject<Unit> mJumpInput = new();
    private CompositeDisposable mDisposables = new();

    void Awake()
    {
        mInputActions = new PlayerInputActions();
    }

    void OnEnable()
    {
        mInputActions.Enable();

        // Move 입력을 Observable로
        mInputActions.Player.Move.performed += ctx => mMoveInput.OnNext(ctx.ReadValue<Vector2>());
        mInputActions.Player.Move.canceled += ctx => mMoveInput.OnNext(Vector2.zero);

        // Jump 입력을 Observable로
        mInputActions.Player.Jump.performed += ctx => mJumpInput.OnNext(Unit.Default);
    }

    void OnDisable()
    {
        mInputActions.Disable();
    }

    void Start()
    {
        // 이동 입력 처리 (중복 제거)
        mMoveInput
            .DistinctUntilChanged()
            .Subscribe(direction => Move(direction))
            .AddTo(mDisposables);

        // 점프 입력 제한 (0.5초 쿨다운)
        mJumpInput
            .Throttle(TimeSpan.FromSeconds(0.5f))
            .Subscribe(_ => Jump())
            .AddTo(mDisposables);

        // 연속 입력 감지 (콤보)
        mJumpInput
            .Buffer(TimeSpan.FromSeconds(1))
            .Where(jumps => jumps.Count >= 2)
            .Subscribe(_ => DoubleJump())
            .AddTo(mDisposables);
    }

    void OnDestroy() => mDisposables?.Dispose();

    void Move(Vector2 direction) => Debug.Log($"이동: {direction}");
    void Jump() => Debug.Log("점프");
    void DoubleJump() => Debug.Log("더블 점프!");
}
```

### 입력 조합 감지

```csharp
public class ComboSystem : MonoBehaviour
{
    private Subject<string> mInputSequence = new();
    private CompositeDisposable mDisposables = new();

    void Start()
    {
        // 1초 내 입력 시퀀스 수집
        mInputSequence
            .Buffer(TimeSpan.FromSeconds(1))
            .Select(inputs => string.Join("", inputs))
            .Where(combo => !string.IsNullOrEmpty(combo))
            .Subscribe(combo => CheckCombo(combo))
            .AddTo(mDisposables);
    }

    void Update()
    {
        // 간단한 입력 예시
        if (Input.GetKeyDown(KeyCode.W)) mInputSequence.OnNext("W");
        if (Input.GetKeyDown(KeyCode.A)) mInputSequence.OnNext("A");
        if (Input.GetKeyDown(KeyCode.S)) mInputSequence.OnNext("S");
        if (Input.GetKeyDown(KeyCode.D)) mInputSequence.OnNext("D");
    }

    void OnDestroy() => mDisposables?.Dispose();

    void CheckCombo(string combo)
    {
        switch (combo)
        {
            case "WWS": Debug.Log("콤보: 상상하"); break;
            case "WASD": Debug.Log("콤보: 회전"); break;
            case "DD": Debug.Log("콤보: 대시"); break;
            default: Debug.Log($"입력: {combo}"); break;
        }
    }
}
```

## 네트워크 요청 처리

### HTTP 요청을 Observable로

```csharp
using UnityEngine.Networking;

public class ApiClient : MonoBehaviour
{
    void Start()
    {
        // 단일 요청
        GetRequest("https://api.example.com/data")
            .Subscribe(
                onNext: response => Debug.Log($"응답: {response}"),
                onError: ex => Debug.LogError($"에러: {ex.Message}")
            )
            .AddTo(this);

        // 재시도가 있는 요청
        GetRequest("https://api.example.com/user")
            .Retry(3)
            .Subscribe(response => ProcessUser(response))
            .AddTo(this);

        // 여러 요청 병렬 처리
        Observable.Merge(
            GetRequest("https://api.example.com/player"),
            GetRequest("https://api.example.com/inventory"),
            GetRequest("https://api.example.com/quests")
        )
        .Subscribe(
            onNext: response => Debug.Log($"응답 받음: {response.Substring(0, 20)}..."),
            onCompleted: () => Debug.Log("모든 데이터 로드 완료")
        )
        .AddTo(this);
    }

    Observable<string> GetRequest(string url)
    {
        return Observable.Create<string>(async (observer, cancellationToken) =>
        {
            using var request = UnityWebRequest.Get(url);

            try
            {
                await request.SendWebRequest().ToUniTask(cancellationToken: cancellationToken);

                if (request.result == UnityWebRequest.Result.Success)
                {
                    observer.OnNext(request.downloadHandler.text);
                    observer.OnCompleted();
                }
                else
                {
                    observer.OnError(new Exception(request.error));
                }
            }
            catch (Exception ex)
            {
                observer.OnError(ex);
            }
        });
    }

    void ProcessUser(string response) { }
}
```

### 폴링과 실시간 업데이트

```csharp
public class LiveDataMonitor : MonoBehaviour
{
    void Start()
    {
        // 5초마다 서버 폴링
        Observable.Interval(TimeSpan.FromSeconds(5))
            .SelectAwait(async (_, ct) => await FetchServerDataAsync(ct))
            .Subscribe(data => UpdateUI(data))
            .AddTo(this);

        // 값이 변경될 때만 처리
        Observable.Interval(TimeSpan.FromSeconds(1))
            .SelectAwait(async (_, ct) => await GetPlayerCountAsync(ct))
            .DistinctUntilChanged()
            .Subscribe(count => Debug.Log($"플레이어 수 변경: {count}"))
            .AddTo(this);
    }

    async UniTask<string> FetchServerDataAsync(CancellationToken ct)
    {
        await UniTask.Delay(100, cancellationToken: ct);
        return "서버 데이터";
    }

    async UniTask<int> GetPlayerCountAsync(CancellationToken ct)
    {
        await UniTask.Delay(50, cancellationToken: ct);
        return UnityEngine.Random.Range(1, 100);
    }

    void UpdateUI(string data) { }
}
```

## DOTween 통합

### Tween을 Observable로

```csharp
using DG.Tweening;

public class AnimationController : MonoBehaviour
{
    void Start()
    {
        // Tween 완료를 Observable로
        transform.DOMove(Vector3.up * 5, 1f)
            .ToObservable()
            .Subscribe(_ => Debug.Log("이동 완료"))
            .AddTo(this);

        // 여러 Tween 순차 실행
        Observable.Concat(
            transform.DOScale(2f, 0.5f).ToObservable(),
            transform.DORotate(new Vector3(0, 180, 0), 0.5f).ToObservable(),
            transform.DOScale(1f, 0.5f).ToObservable()
        )
        .Subscribe(
            onNext: _ => { },
            onCompleted: () => Debug.Log("애니메이션 시퀀스 완료")
        )
        .AddTo(this);
    }
}
```

### 반응형 애니메이션 체인

```csharp
public class UIAnimator : MonoBehaviour
{
    [SerializeField] private CanvasGroup mPanel;
    private Subject<bool> mShowPanel = new();

    void Start()
    {
        mShowPanel
            .SelectAwait(async (show, ct) =>
            {
                if (show)
                {
                    mPanel.gameObject.SetActive(true);
                    await mPanel.DOFade(1f, 0.3f).SetEase(Ease.OutQuad).ToUniTask(cancellationToken: ct);
                }
                else
                {
                    await mPanel.DOFade(0f, 0.3f).SetEase(Ease.InQuad).ToUniTask(cancellationToken: ct);
                    mPanel.gameObject.SetActive(false);
                }
                return show;
            })
            .Subscribe(showed => Debug.Log($"패널 표시: {showed}"))
            .AddTo(this);
    }

    public void Show() => mShowPanel.OnNext(true);
    public void Hide() => mShowPanel.OnNext(false);
}
```

## 씬 전환과 생명주기

### 씬 로드를 Observable로

```csharp
using UnityEngine.SceneManagement;

public class SceneLoader : MonoBehaviour
{
    void Start()
    {
        LoadSceneAsync("GameScene")
            .Subscribe(
                onNext: progress => UpdateLoadingBar(progress),
                onCompleted: () => Debug.Log("씬 로드 완료")
            )
            .AddTo(this);
    }

    Observable<float> LoadSceneAsync(string sceneName)
    {
        return Observable.Create<float>(async (observer, cancellationToken) =>
        {
            var asyncOp = SceneManager.LoadSceneAsync(sceneName);
            asyncOp.allowSceneActivation = false;

            while (!asyncOp.isDone && !cancellationToken.IsCancellationRequested)
            {
                observer.OnNext(asyncOp.progress);

                if (asyncOp.progress >= 0.9f)
                {
                    asyncOp.allowSceneActivation = true;
                }

                await UniTask.Yield(cancellationToken);
            }

            observer.OnNext(1f);
            observer.OnCompleted();
        });
    }

    void UpdateLoadingBar(float progress)
    {
        Debug.Log($"로딩: {progress * 100}%");
    }
}
```
