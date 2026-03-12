# 분석 체크리스트 — project-context

## manifest.json 파싱 포인트

### 경로
```
Glob: Packages/manifest.json
```

### 파싱 항목
- `dependencies` 객체: 패키지명 → 버전/출처 맵
- 버전 형식 분류:
  - `"1.2.3"` → Registry 패키지
  - `"https://github.com/..."` → git URL 패키지
  - `"file:../../local-packages/..."` → 로컬 경로 패키지
- `scopedRegistries` 존재 시 → 커스텀 레지스트리 기록

### 패키지 카테고리 분류 기준

| 카테고리 | 패키지명 키워드 |
|----------|----------------|
| DI | vcontainer, zenject, extenject |
| Messaging | messagepipe, unirx, signalbus |
| Async | unitask, dotween.async |
| Animation | primetween, dotween, animancer |
| Reactive | r3, unirx, observablecollections |
| LINQ | zlinq, linq2db |
| Graphics | render-pipelines, urp, hdrp |
| Input | inputsystem |
| Asset Management | addressables |
| UI Foundation | 프로젝트 커스텀 패키지 (darknaku 등) |
| Testing | nunit, nsubstitute, moq |

---

## .asmdef 레이어 판별 기준

### 탐색
```
Glob: **/*.asmdef
```

### 레이어 판별 휴리스틱

| .asmdef 이름 패턴 | 추정 레이어 |
|-------------------|-------------|
| `*.Core` | Core / Domain 레이어 |
| `*.Presentation` | Presentation / View 레이어 |
| `*.Infrastructure` | Infrastructure 레이어 |
| `*.Tests.EditMode` | EditMode 테스트 |
| `*.Tests.PlayMode` | PlayMode 테스트 |
| `*.Editor` | 에디터 전용 |

### 의존 방향 확인
각 .asmdef의 `references` 배열을 비교:
- Core가 Presentation을 참조하면 → **역방향 경고**
- Presentation이 Core를 참조하면 → 정상

---

## 코딩 스타일 판별 휴리스틱

### 중괄호 스타일
```
Grep: "^    \{$"     → Allman (줄 시작 중괄호)
Grep: "\) \{$"      → K&R (같은 줄 중괄호)
```

### 필드 접두사
```
Grep: "private.*_[a-zA-Z]"   → 언더스코어 접두사 (_field)
Grep: "private.*m[A-Z]"      → m 접두사 (mField)
Grep: "private.*_m[A-Z]"     → _m 접두사
```

### 네이밍 케이스
```
Grep: "void [a-z][a-zA-Z]*\(" → camelCase 메서드 (비정상)
Grep: "void [A-Z][a-zA-Z]*\(" → PascalCase 메서드 (정상)
```

### 테스트 언어
```
Grep: "public void [가-힣_]"  → 한국어 테스트 메서드명
Grep: "class [가-힣_]"        → 한국어 테스트 클래스명
```

---

## 패턴 감지 Grep 쿼리 모음

### VContainer
```
Grep: "LifetimeScope"
Grep: "IContainerBuilder"
Grep: "\.Register<"
Grep: "RegisterEntryPoint"
Grep: "\[Inject\]"
```

### MessagePipe
```
Grep: "IPublisher<\|ISubscriber<\|IMessageBroker"
Grep: "RegisterMessagePipe\|RegisterMessageBroker"
Grep: "\.Publish(\|\.Subscribe("
Grep: "IMessageService"
```

### UniTask
```
Grep: "UniTask\b"
Grep: "async UniTask"
Grep: "UniTaskVoid"
Grep: "\.ToCoroutine()"
Grep: "CancellationToken"
```

### R3
```
Grep: "Observable\."
Grep: "ReactiveProperty<"
Grep: "\.Subscribe("
Grep: "ObservableCollections\|ObservableList"
```

### PrimeTween
```
Grep: "Tween\."
Grep: "Sequence\.Create"
Grep: "\.Chain(\|\.Group("
Grep: "ShakePosition\|ShakeRotation"
```

### Addressables
```
Grep: "Addressables\."
Grep: "AssetReference"
Grep: "LoadAssetAsync"
Grep: "AssetReferenceT<"
```

### Input System
```
Grep: "InputActionAsset\|InputAction\b"
Grep: "\.Enable()\|\.Disable()"
Grep: "\.performed \+="
```

### Pool 패턴
```
Grep: "Pool<\|ObjectPool<"
Grep: "\.Spawn(\|\.Despawn("
```

### Presenter / View
```
Grep: "UIView\|UIPresenter"
Grep: "IStartable\b"
Grep: "protected override void OnEnter\|OnExit"
```

### 테스트
```
Grep: "\[TestFixture\]"
Grep: "\[Test\]\b"
Grep: "\[UnityTest\]"
Grep: "NSubstitute\|Substitute\.For\|Moq\|Mock<"
```

### 로깅 유틸리티
```
Grep: "Log\.(Info|Warning|Error|Check)"
Grep: "Debug\.Log\b"
```

---

## ScriptableObject 계층 감지

```
Grep: ": ScriptableObject"
Grep: "AssetReferenceT<"
Grep: "CollectionSO\|PackageSO\|LevelSO"  (프로젝트명 변형 포함)
```

- 계층 구조를 발견하면 Addressables 키와 함께 도식화

---

## 출력 전 최종 체크

- [ ] `references/` 디렉토리 존재 여부 확인 (없으면 생성)
- [ ] 모든 패턴 섹션에 실제 코드 기반 예제 포함
- [ ] 사용하지 않는 패키지의 섹션 제거
- [ ] 패키지 버전이 manifest.json 원문과 일치
- [ ] 레이어 도식에 의존 방향 화살표 포함
- [ ] 파일 저장 경로: `references/project-context.md`
