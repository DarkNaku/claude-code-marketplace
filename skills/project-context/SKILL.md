---
name: project-context

description: Unity 프로젝트를 직접 탐색하여 패키지·라이브러리, 아키텍처 레이어, 코딩 스타일·네이밍 규칙, 핵심 패턴을 추출하고 project-context.md 참조 문서로 저장하는 전문가. manifest.json, .asmdef, 소스 코드(.cs)를 Glob·Grep·Read 툴로 분석하여 신규 기능 개발 시 참조할 수 있는 코드 가이드를 생성. 프로젝트 초기 온보딩, 코딩 규칙 문서화, AI 코드 생성 컨텍스트 준비 시 사용. project-context.md 파일로 저장.

agent: darknaku:developer

allowed-tools:
    - Read
    - Write
    - Edit
    - Glob
    - Grep
    - Bash

user-invocable: true

---

# project-context — 프로젝트 컨텍스트 분석

## 목적

Unity 프로젝트의 **실제 파일을 직접 읽어** 다음을 문서화합니다:

- 사용 중인 패키지와 라이브러리
- 아키텍처 레이어 구조
- 코딩 스타일 및 네이밍 규칙
- 프로젝트 고유의 핵심 패턴 (DI, 비동기, UI, 테스트 등)

출력 파일: **`references/project-context.md`** (프로젝트 루트 기준)

---

## 분석 워크플로우

사용자가 프로젝트 경로를 제공하면 **반드시 다음 순서**로 분석합니다.

### 1단계: 패키지 수집

```
Glob: Packages/manifest.json
Read: manifest.json → dependencies 섹션
Glob: Packages/*/package.json  (로컬 패키지)
Glob: **/*.asmdef              (어셈블리 정의 → 레이어 식별)
```

- `manifest.json`의 `dependencies`에서 모든 패키지 이름·버전 추출
- 로컬 패키지(`"file:..."`)는 별도 표기
- git 패키지(`"https://..."`)는 출처 URL 기록

### 2단계: 아키텍처 레이어 파악

```
Glob: **/*.asmdef
Read: 각 .asmdef → name, references 필드
Glob: Assets/**/*.cs (샘플링 — 최대 20개)
```

- `.asmdef` 이름과 참조 관계로 레이어 계층 도식화
- 폴더 구조(Assets/Scripts/…) 보조 활용
- 레이어 간 의존 방향 명시 (단방향 금지 규칙 포함)

### 3단계: 코딩 스타일 분석

```
Glob: Assets/**/*.cs
Read: 무작위 10~20개 소스 파일
```

분석 항목:
- 중괄호 스타일 (K&R vs Allman)
- 클래스·메서드·프로퍼티 네이밍 (PascalCase 등)
- 로컬 변수·파라미터 네이밍 (camelCase 등)
- private 필드 접두사 (`_`, `m`, `mStr` 등)
- 테스트 클래스·메서드명 언어 (한국어/영어)
- 기타 규칙 (region, partial, record 사용 여부 등)

### 4단계: 핵심 패턴 추출

발견된 패키지에 따라 해당 패턴만 추출합니다.

| 패키지 감지 | 추출할 패턴 |
|-------------|-------------|
| VContainer | LifetimeScope, Register, EntryPoint, Inject |
| MessagePipe | Publish, Subscribe, IMessageService 래퍼 |
| UniTask | async/await, UniTaskVoid, ToCoroutine |
| R3 | Observable, ReactiveProperty, Subscribe |
| PrimeTween | Tween, Sequence, ShakePosition |
| Addressables | LoadAssetAsync, AssetReference, CollectionSO 계층 |
| InputSystem | InputActionAsset, OnEnable/Disable 구독 |
| Pool<T> (커스텀) | Spawn, Despawn, Clear |
| Presenter/View (커스텀) | UIView, UIPresenter<T>, IStartable 바인딩 |
| NSubstitute / Moq | 테스트 mock 패턴 |

각 패턴에 대해 **실제 프로젝트 코드에서 추출한 예제**를 사용합니다.
사전 지식으로 예제를 채우지 말고, Grep으로 실제 사용 코드를 찾아 인용합니다.

```
Grep: "LifetimeScope"     → VContainer 사용 패턴
Grep: "IPublisher\|ISubscriber\|MessageBroker"  → MessagePipe
Grep: "UniTask\|async "  → 비동기 패턴
Grep: "Tween\.\|Sequence\.Create"  → PrimeTween
Grep: "Addressables\."   → Addressables
Grep: "\[Test\]\|\[UnityTest\]"    → 테스트 패턴
Grep: "Log\.\(Info\|Warning\|Error\|Check\)"   → 로깅 유틸
```

### 5단계: 문서 작성 및 저장

수집된 정보를 아래 출력 구조에 따라 `references/project-context.md`로 저장합니다.

---

## 출력 구조

```markdown
# Project Context — [프로젝트명]

> 설계·코딩 참고용 컨텍스트 문서. 패키지, 아키텍처, 코드 패턴을 한 곳에 정리.

---

## 1. 기술 스택 & 패키지

| 카테고리 | 패키지 | 버전/출처 |
|----------|--------|----------|
| ...      | ...    | ...      |

---

## 2. 아키텍처 레이어

(ASCII 도식 + 레이어별 책임 설명)

---

## 3. 코딩 스타일

(스타일 규칙 목록)

---

## 4. [패턴명] 패턴

(실제 코드에서 추출한 예제)

...패턴 수만큼 반복...

---

## N. 로깅 유틸리티

(로그 유틸이 있는 경우)
```

---

## 패턴 예제 품질 기준

- 실제 프로젝트 코드에서 직접 발췌하거나 해당 파일의 실사용 방식을 반영
- 예제는 `// 한국어 주석`으로 의도 설명
- 30줄 이하의 간결한 스니펫 유지
- 프로젝트에서 사용하지 않는 기능은 예제에 포함하지 않음

---

## 참고 문서

### [분석 체크리스트](references/analysis-checklist.md)
각 분석 단계별 확인 항목:
- manifest.json 파싱 포인트
- .asmdef 레이어 판별 기준
- 코딩 스타일 판별 휴리스틱
- 패턴 감지 Grep 쿼리 모음

---

## 모범 사례

1. **실제 코드 우선**: 패턴 예제는 반드시 Grep으로 실제 코드 확인 후 작성
2. **발견된 것만 기록**: 프로젝트에 없는 패키지나 패턴은 섹션에 포함하지 않음
3. **레이어 의존 방향 명시**: 레이어 도식에 화살표와 금지 규칙 함께 표기
4. **버전 정보 보존**: 패키지 버전은 manifest.json 원문 그대로 기록
5. **네이밍 근거 제시**: 코딩 규칙은 실제 발견된 파일명·변수명으로 근거 제시
6. **분량 조절**: 패턴 섹션이 10개를 넘으면 빈도 높은 패턴 우선 정리
7. **저장 경로 확인**: `references/` 디렉토리가 없으면 생성 후 저장
