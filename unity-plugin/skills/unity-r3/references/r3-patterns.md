# R3 패턴 가이드

Unity에서 고급 R3 반응형 프로그래밍 패턴을 위한 심층 참조 문서.

## Observable 생성 패턴

### Factory 메서드

```csharp
using R3;
using System;
using UnityEngine;

// 단일 값 방출
var single = Observable.Return(42);

// 시퀀스 방출
var sequence = Observable.Range(1, 5); // 1, 2, 3, 4, 5

// 타이머
var timer = Observable.Timer(TimeSpan.FromSeconds(2)); // 2초 후 방출

// 주기적 실행
var interval = Observable.Interval(TimeSpan.FromSeconds(1)); // 매초마다

// 커스텀 Observable
var custom = Observable.Create<int>(observer =>
{
    observer.OnNext(1);
    observer.OnNext(2);
    observer.OnCompleted();
    return Disposable.Empty;
});
```

### Unity 이벤트를 Observable로 변환

```csharp
public class InputController : MonoBehaviour
{
    void Start()
    {
        // 매 프레임 업데이트
        Observable.EveryUpdate()
            .Subscribe(_ => CheckInput())
            .AddTo(this);

        // FixedUpdate
        Observable.EveryFixedUpdate()
            .Subscribe(_ => PhysicsUpdate())
            .AddTo(this);

        // LateUpdate
        Observable.EveryLateUpdate()
            .Subscribe(_ => CameraFollow())
            .AddTo(this);

        // 값 변경 감지
        Observable.EveryValueChanged(this, x => x.transform.position)
            .Where(pos => pos.y < 0)
            .Subscribe(_ => Die())
            .AddTo(this);
    }

    void CheckInput() { }
    void PhysicsUpdate() { }
    void CameraFollow() { }
    void Die() { }
}
```

### UnityEvent를 Observable로 변환

```csharp
using UnityEngine.UI;

public class ButtonController : MonoBehaviour
{
    [SerializeField] private Button mButton;

    void Start()
    {
        // Button onClick를 Observable로
        mButton.OnClickAsObservable()
            .Subscribe(_ => OnButtonClicked())
            .AddTo(this);
    }

    void OnButtonClicked() => Debug.Log("버튼 클릭!");
}
```

## Operators 체이닝

### 변환과 필터링

```csharp
public class ScoreSystem : MonoBehaviour
{
    private ReactiveProperty<int> mScore = new(0);

    void Start()
    {
        // Select: 값 변환
        mScore
            .Select(score => score * 10)
            .Subscribe(points => Debug.Log($"포인트: {points}"))
            .AddTo(this);

        // Where: 조건 필터링
        mScore
            .Where(score => score > 100)
            .Subscribe(_ => ShowHighScoreEffect())
            .AddTo(this);

        // DistinctUntilChanged: 값이 실제로 변경될 때만
        mScore
            .DistinctUntilChanged()
            .Subscribe(score => UpdateUI(score))
            .AddTo(this);
    }

    void ShowHighScoreEffect() { }
    void UpdateUI(int score) { }
}
```

### 시간 제어

```csharp
public class SearchBox : MonoBehaviour
{
    private Subject<string> mSearchQuery = new();

    void Start()
    {
        // Throttle: 마지막 입력 후 0.5초 대기
        mSearchQuery
            .Throttle(TimeSpan.FromSeconds(0.5f))
            .Subscribe(query => PerformSearch(query))
            .AddTo(this);

        // Sample: 1초마다 최신 값만
        Observable.EveryUpdate()
            .Sample(TimeSpan.FromSeconds(1))
            .Subscribe(_ => UpdateFPS())
            .AddTo(this);

        // Delay: 2초 지연
        mSearchQuery
            .Delay(TimeSpan.FromSeconds(2))
            .Subscribe(query => Debug.Log($"지연된 검색: {query}"))
            .AddTo(this);
    }

    public void OnSearchInputChanged(string query)
    {
        mSearchQuery.OnNext(query);
    }

    void PerformSearch(string query) { }
    void UpdateFPS() { }
}
```

### 스트림 조합

```csharp
public class CombatSystem : MonoBehaviour
{
    private ReactiveProperty<bool> mIsAttacking = new(false);
    private ReactiveProperty<bool> mHasAmmo = new(true);
    private ReactiveProperty<int> mComboCount = new(0);

    void Start()
    {
        // CombineLatest: 두 값 조합
        mIsAttacking.CombineLatest(mHasAmmo, (attacking, hasAmmo) => attacking && hasAmmo)
            .Where(canShoot => canShoot)
            .Subscribe(_ => Shoot())
            .AddTo(this);

        // Merge: 여러 스트림 병합
        var leftClick = Observable.EveryUpdate().Where(_ => Input.GetMouseButtonDown(0));
        var rightClick = Observable.EveryUpdate().Where(_ => Input.GetMouseButtonDown(1));

        Observable.Merge(leftClick, rightClick)
            .Subscribe(_ => OnAnyClick())
            .AddTo(this);

        // Zip: 순서대로 쌍 만들기
        var attacks = new Subject<Unit>();
        var hits = new Subject<Unit>();

        attacks.Zip(hits, (a, h) => Unit.Default)
            .Subscribe(_ => mComboCount.Value++)
            .AddTo(this);
    }

    void Shoot() { }
    void OnAnyClick() { }
}
```

## Subject와 ReactiveProperty

### Subject 타입

```csharp
public class MessageBus : MonoBehaviour
{
    // Subject: Hot Observable, 구독 후 값 방출
    private Subject<string> mMessages = new();

    // ReplaySubject: 최근 N개 값을 새 구독자에게 재생
    private ReplaySubject<int> mScoreHistory = new(5);

    // BehaviorSubject: 항상 최신 값 유지 (초기값 필요)
    private BehaviorSubject<GameState> mGameState = new(GameState.Menu);

    void Start()
    {
        // Subject는 Observer이자 Observable
        mMessages.Subscribe(msg => Debug.Log($"메시지: {msg}")).AddTo(this);
        mMessages.OnNext("첫 메시지");

        // ReplaySubject는 버퍼 크기만큼 이전 값도 받음
        mScoreHistory.OnNext(100);
        mScoreHistory.OnNext(200);
        mScoreHistory.Subscribe(score => Debug.Log($"점수: {score}")).AddTo(this); // 100, 200 받음

        // BehaviorSubject는 항상 최신 값 제공
        mGameState.Subscribe(state => Debug.Log($"상태: {state}")).AddTo(this); // 즉시 Menu 받음
    }

    public void PublishMessage(string message) => mMessages.OnNext(message);
}

public enum GameState { Menu, Playing, Paused }
```

### ReactiveProperty 패턴

```csharp
public class Player : MonoBehaviour
{
    // 기본 ReactiveProperty
    public ReactiveProperty<int> Health { get; } = new(100);

    // ReadOnly로 외부 노출
    public ReadOnlyReactiveProperty<bool> IsDead { get; }

    // ReactiveCollection
    public ReactiveCollection<Item> Inventory { get; } = new();

    public Player()
    {
        // 파생 속성
        IsDead = Health
            .Select(h => h <= 0)
            .ToReadOnlyReactiveProperty();

        // ReactiveCollection 변경 감지
        Inventory.ObserveAdd()
            .Subscribe(e => Debug.Log($"아이템 추가: {e.Value}"))
            .AddTo(this);

        Inventory.ObserveRemove()
            .Subscribe(e => Debug.Log($"아이템 제거: {e.Value}"))
            .AddTo(this);
    }

    void Start()
    {
        // 값 변경 구독
        Health.Subscribe(h => UpdateHealthUI(h)).AddTo(this);

        // 이전 값과 현재 값 모두 필요할 때
        Health.Pairwise()
            .Subscribe(pair => Debug.Log($"{pair.Previous} -> {pair.Current}"))
            .AddTo(this);
    }

    void UpdateHealthUI(int health) { }
}

public class Item { }
```

## 메모리 관리 및 Dispose 패턴

### AddTo를 사용한 자동 해제

```csharp
public class Enemy : MonoBehaviour
{
    void Start()
    {
        // GameObject가 파괴되면 자동 해제
        Observable.Interval(TimeSpan.FromSeconds(1))
            .Subscribe(_ => Attack())
            .AddTo(this);

        // 여러 구독을 하나의 GameObject에 연결
        var disposables = new CompositeDisposable();

        Observable.EveryUpdate()
            .Subscribe(_ => Move())
            .AddTo(disposables);

        Observable.EveryUpdate()
            .Subscribe(_ => CheckPlayer())
            .AddTo(disposables);

        // disposables를 GameObject에 연결
        disposables.AddTo(this);
    }

    void Attack() { }
    void Move() { }
    void CheckPlayer() { }
}
```

### 수동 Dispose

```csharp
public class BossController : MonoBehaviour
{
    private IDisposable mPhaseSubscription;
    private CompositeDisposable mDisposables = new();

    void StartPhase1()
    {
        // 이전 구독 해제
        mPhaseSubscription?.Dispose();

        // 새 구독
        mPhaseSubscription = Observable.Interval(TimeSpan.FromSeconds(2))
            .Subscribe(_ => Phase1Attack())
            .AddTo(mDisposables);
    }

    void StartPhase2()
    {
        mPhaseSubscription?.Dispose();
        mPhaseSubscription = Observable.Interval(TimeSpan.FromSeconds(1))
            .Subscribe(_ => Phase2Attack())
            .AddTo(mDisposables);
    }

    void OnDestroy()
    {
        mDisposables?.Dispose();
    }

    void Phase1Attack() { }
    void Phase2Attack() { }
}
```

### 조건부 구독 해제

```csharp
public class QuestSystem : MonoBehaviour
{
    void Start()
    {
        // TakeUntil: 특정 이벤트까지만 구독
        var questComplete = new Subject<Unit>();

        Observable.EveryUpdate()
            .TakeUntil(questComplete)
            .Subscribe(_ => UpdateQuest())
            .AddTo(this);

        // 5초 후 퀘스트 완료
        Observable.Timer(TimeSpan.FromSeconds(5))
            .Subscribe(_ => questComplete.OnNext(Unit.Default))
            .AddTo(this);

        // Take: N개까지만
        Observable.Interval(TimeSpan.FromSeconds(1))
            .Take(10)
            .Subscribe(x => Debug.Log($"카운트: {x}"))
            .AddTo(this);

        // TakeWhile: 조건이 참일 때까지
        var health = new ReactiveProperty<int>(100);
        Observable.EveryUpdate()
            .TakeWhile(_ => health.Value > 0)
            .Subscribe(_ => RegenerateHealth())
            .AddTo(this);
    }

    void UpdateQuest() { }
    void RegenerateHealth() { }
}
```

## MVVM 패턴 구현

### Model

```csharp
public class PlayerModel
{
    public ReactiveProperty<string> Name { get; } = new("플레이어");
    public ReactiveProperty<int> Level { get; } = new(1);
    public ReactiveProperty<int> Exp { get; } = new(0);
    public ReactiveProperty<int> Gold { get; } = new(0);

    public ReadOnlyReactiveProperty<int> ExpToNextLevel { get; }
    public ReadOnlyReactiveProperty<float> ExpProgress { get; }

    public PlayerModel()
    {
        ExpToNextLevel = Level
            .Select(level => level * 100)
            .ToReadOnlyReactiveProperty();

        ExpProgress = Exp.CombineLatest(ExpToNextLevel, (exp, required) => (float)exp / required)
            .ToReadOnlyReactiveProperty();
    }

    public void AddExp(int amount)
    {
        Exp.Value += amount;
        if (Exp.Value >= ExpToNextLevel.Value)
        {
            LevelUp();
        }
    }

    private void LevelUp()
    {
        Level.Value++;
        Exp.Value = 0;
    }
}
```

### ViewModel

```csharp
public class PlayerViewModel
{
    private PlayerModel mModel;

    public ReadOnlyReactiveProperty<string> DisplayName { get; }
    public ReadOnlyReactiveProperty<string> LevelText { get; }
    public ReadOnlyReactiveProperty<string> GoldText { get; }
    public ReadOnlyReactiveProperty<float> ExpBarFill { get; }

    public PlayerViewModel(PlayerModel model)
    {
        mModel = model;

        DisplayName = model.Name
            .Select(name => $"이름: {name}")
            .ToReadOnlyReactiveProperty();

        LevelText = model.Level
            .Select(level => $"레벨 {level}")
            .ToReadOnlyReactiveProperty();

        GoldText = model.Gold
            .Select(gold => $"{gold:N0} G")
            .ToReadOnlyReactiveProperty();

        ExpBarFill = model.ExpProgress;
    }

    public void OnKillEnemy()
    {
        mModel.AddExp(50);
        mModel.Gold.Value += 10;
    }
}
```

### View

```csharp
using UnityEngine.UI;
using TMPro;

public class PlayerView : MonoBehaviour
{
    [SerializeField] private TextMeshProUGUI mNameText;
    [SerializeField] private TextMeshProUGUI mLevelText;
    [SerializeField] private TextMeshProUGUI mGoldText;
    [SerializeField] private Image mExpBar;
    [SerializeField] private Button mKillEnemyButton;

    private PlayerViewModel mViewModel;
    private CompositeDisposable mDisposables = new();

    void Start()
    {
        var model = new PlayerModel();
        mViewModel = new PlayerViewModel(model);

        // ViewModel -> View 바인딩
        mViewModel.DisplayName
            .Subscribe(text => mNameText.text = text)
            .AddTo(mDisposables);

        mViewModel.LevelText
            .Subscribe(text => mLevelText.text = text)
            .AddTo(mDisposables);

        mViewModel.GoldText
            .Subscribe(text => mGoldText.text = text)
            .AddTo(mDisposables);

        mViewModel.ExpBarFill
            .Subscribe(fill => mExpBar.fillAmount = fill)
            .AddTo(mDisposables);

        // View -> ViewModel 이벤트
        mKillEnemyButton.OnClickAsObservable()
            .Subscribe(_ => mViewModel.OnKillEnemy())
            .AddTo(mDisposables);
    }

    void OnDestroy()
    {
        mDisposables?.Dispose();
    }
}
```

## 에러 처리

### Catch와 Retry

```csharp
public class NetworkManager : MonoBehaviour
{
    void Start()
    {
        // Catch: 에러 발생 시 대체 스트림
        FetchDataObservable()
            .Catch((Exception ex) =>
            {
                Debug.LogError($"에러 발생: {ex.Message}");
                return Observable.Return("기본값");
            })
            .Subscribe(data => ProcessData(data))
            .AddTo(this);

        // Retry: 에러 발생 시 재시도
        FetchDataObservable()
            .Retry(3) // 최대 3번 재시도
            .Subscribe(
                onNext: data => ProcessData(data),
                onError: ex => Debug.LogError($"재시도 실패: {ex.Message}")
            )
            .AddTo(this);

        // OnErrorRetry: 지수 백오프와 함께 재시도
        FetchDataObservable()
            .OnErrorRetry((Exception ex, int retryCount) =>
            {
                Debug.LogWarning($"재시도 {retryCount}: {ex.Message}");
                return TimeSpan.FromSeconds(Math.Pow(2, retryCount)); // 2, 4, 8초...
            })
            .Subscribe(data => ProcessData(data))
            .AddTo(this);
    }

    Observable<string> FetchDataObservable()
    {
        return Observable.Create<string>(observer =>
        {
            // 네트워크 요청 시뮬레이션
            if (UnityEngine.Random.value < 0.3f)
            {
                observer.OnError(new Exception("네트워크 에러"));
            }
            else
            {
                observer.OnNext("데이터");
                observer.OnCompleted();
            }
            return Disposable.Empty;
        });
    }

    void ProcessData(string data) { }
}
```

## 성능 최적화 패턴

### Share와 Publish

```csharp
public class GameManager : MonoBehaviour
{
    void Start()
    {
        // Cold Observable: 각 구독자마다 독립적으로 실행
        var cold = Observable.Interval(TimeSpan.FromSeconds(1));

        // Hot Observable: Share로 하나의 스트림 공유
        var hot = cold.Share();

        hot.Subscribe(x => Debug.Log($"구독자1: {x}")).AddTo(this);
        hot.Subscribe(x => Debug.Log($"구독자2: {x}")).AddTo(this);
        // 두 구독자가 같은 값을 받음

        // Publish: 수동으로 연결 제어
        var published = cold.Publish();
        published.Subscribe(x => Debug.Log($"A: {x}")).AddTo(this);
        published.Subscribe(x => Debug.Log($"B: {x}")).AddTo(this);
        published.Connect(); // 여기서 시작
    }
}
```

### Buffer와 Window

```csharp
public class AnalyticsSystem : MonoBehaviour
{
    private Subject<string> mEvents = new();

    void Start()
    {
        // Buffer: N개씩 모아서 처리
        mEvents
            .Buffer(10)
            .Subscribe(events => SendAnalyticsBatch(events))
            .AddTo(this);

        // 시간 기반 Buffer
        mEvents
            .Buffer(TimeSpan.FromSeconds(5))
            .Subscribe(events => Debug.Log($"{events.Count}개 이벤트 수집됨"))
            .AddTo(this);

        // Window: Observable의 Observable
        Observable.Interval(TimeSpan.FromSeconds(1))
            .Window(TimeSpan.FromSeconds(5))
            .Subscribe(window =>
            {
                window.Subscribe(x => Debug.Log($"윈도우 내 값: {x}")).AddTo(this);
            })
            .AddTo(this);
    }

    void SendAnalyticsBatch(System.Collections.Generic.IList<string> events) { }
}
```
