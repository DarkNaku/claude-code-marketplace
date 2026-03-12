# IDE 작업 패턴 레퍼런스

Unity Editor에서 자주 수행하는 수동 작업의 단계별 작성 패턴 모음.

---

## 패키지 설치 (Package Manager)

```markdown
## 작업 N. [패키지명] 패키지 설치

**목적**: [기능명]에 필요한 [패키지명] 라이브러리를 설치합니다.

1. 상단 메뉴 `Window > Package Manager` 클릭
2. 좌측 상단 드롭다운을 `Unity Registry` 또는 `My Registries`로 변경
3. 검색창에 `[패키지명]` 입력
4. 목록에서 `[패키지 표시명]` 선택
5. 우측 하단 `Install` 버튼 클릭
6. 설치 완료 후 `In Project` 배지 확인

**확인**: Package Manager 목록에서 `[패키지명]` 항목이 `✓` 표시로 바뀌어야 합니다.
```

### git URL 패키지 설치 (openupm / GitHub)

```markdown
1. `Window > Package Manager` 클릭
2. 좌측 상단 `+` 버튼 클릭 → `Add package from git URL...` 선택
3. 입력창에 `[git URL]` 붙여넣기
4. `Add` 버튼 클릭
5. 하단 진행 바가 사라질 때까지 대기

**확인**: Package Manager의 `In Project` 탭에 패키지가 나타나야 합니다.
```

---

## Assembly Definition 생성 및 참조 추가

```markdown
## 작업 N. [어셈블리명] Assembly Definition 생성

**목적**: [레이어명] 레이어를 독립 어셈블리로 분리하여 의존성을 명시합니다.

1. Project 창에서 `Assets/[폴더 경로]` 디렉토리 우클릭
2. `Create > Assembly Definition` 선택
3. 파일명을 `[어셈블리명]`으로 변경 후 Enter
4. 생성된 .asmdef 파일 선택
5. Inspector에서 `References` 섹션의 `+` 버튼 클릭
6. 목록에서 `[참조할 어셈블리명]` 선택
7. 필요한 모든 참조 추가 후 `Apply` 클릭

**확인**: Inspector의 `References` 목록에 추가한 어셈블리가 표시되어야 합니다.
```

### 테스트 어셈블리 생성

```markdown
## 작업 N. [테스트 어셈블리명] 테스트 Assembly Definition 생성

1. Project 창에서 `Assets/Tests/[EditMode 또는 PlayMode]` 폴더 우클릭
2. `Create > Assembly Definition` 선택
3. 파일명을 `[프로젝트명].Tests.[EditMode|PlayMode]`로 변경
4. Inspector에서 다음 항목 설정:
   - `Any Platform` 체크 해제 → `Editor` 플랫폼만 체크 (EditMode의 경우)
   - `Test Assemblies` 체크
5. `References`에 다음 추가:
   - `[테스트 대상 어셈블리명]`
   - `UnityEngine.TestRunner`
   - `UnityEditor.TestRunner` (EditMode만)
6. `Apply` 클릭

**확인**: `Window > General > Test Runner`에서 해당 어셈블리의 테스트가 목록에 나타나야 합니다.
```

---

## Project Settings

### Tags & Layers 추가

```markdown
## 작업 N. Tag / Layer 추가

**목적**: [용도 설명]에 사용할 Tag/Layer를 등록합니다.

1. 상단 메뉴 `Edit > Project Settings` 클릭
2. 좌측 목록에서 `Tags and Layers` 선택
3. **Tag 추가**:
   - `Tags` 섹션의 `+` 버튼 클릭
   - `New Tag Name`에 `[태그명]` 입력
   - `Save` 클릭
4. **Layer 추가**:
   - `Layers` 섹션에서 빈 슬롯(User Layer 8 이후)을 찾아 이름 입력
   - `[레이어명]` 입력 후 Enter

**확인**: Inspector의 Tag/Layer 드롭다운에 추가한 항목이 보여야 합니다.
```

### Scripting Define Symbols 추가

```markdown
## 작업 N. Scripting Define Symbol 추가

1. `Edit > Project Settings > Player` 클릭
2. 대상 플랫폼 탭 선택 (PC/Android/iOS 등)
3. `Other Settings` 섹션 펼치기
4. `Scripting Define Symbols` 입력창에 `;[심볼명]` 추가 (기존 값 뒤에 세미콜론으로 구분)
5. `Apply` 클릭

**확인**: 스크립트에서 `#if [심볼명]` 블록이 활성화되어야 합니다.
```

---

## ScriptableObject 에셋 생성

```markdown
## 작업 N. [SO명] 에셋 생성

**목적**: [용도 설명]에 사용할 [SO명] 설정 에셋을 생성합니다.

1. Project 창에서 `Assets/[저장 경로]` 디렉토리 우클릭
2. `Create > [Create 메뉴 경로]` 선택
3. 파일명을 `[에셋명]`으로 변경 후 Enter
4. 생성된 에셋 선택 후 Inspector에서 다음 필드 설정:
   - `[필드명]`: `[값]`
   - `[필드명]`: `[값]`

**확인**: Inspector에 [SO명] 컴포넌트가 표시되고 필드값이 입력된 상태여야 합니다.
```

---

## Prefab 생성 및 컴포넌트 연결

```markdown
## 작업 N. [Prefab명] Prefab 생성

**목적**: [용도 설명]

**Prefab 생성**:
1. Hierarchy에서 빈 GameObject 생성: `우클릭 > Create Empty`
2. 이름을 `[오브젝트명]`으로 변경
3. Inspector에서 필요한 컴포넌트 추가: `Add Component > [컴포넌트명]`
4. Hierarchy의 오브젝트를 `Assets/[Prefabs 경로]`로 드래그 → Prefab 생성됨
5. Hierarchy에서 원본 오브젝트 삭제

**Inspector 필드 연결** (Prefab Edit 모드):
1. Project 창에서 생성된 Prefab 더블클릭 → Edit 모드 진입
2. Inspector에서 다음 필드에 에셋 연결:
   - `[컴포넌트명] > [필드명]`: `[연결할 에셋명]` 드래그 또는 오브젝트 피커 사용
3. 상단 `<` 버튼 클릭하거나 `Ctrl+S` (Win) / `Cmd+S` (Mac)로 Prefab 저장

**확인**: Project 창에서 Prefab 아이콘이 파란색으로 표시되어야 합니다.
```

---

## Addressables 설정

```markdown
## 작업 N. Addressables 초기 설정

**목적**: Addressables 시스템을 초기화하고 에셋 그룹을 구성합니다.

**초기화** (최초 1회):
1. `Window > Asset Management > Addressables > Groups` 클릭
2. `Create Addressables Settings` 버튼 클릭

**그룹 생성**:
1. Addressables Groups 창에서 `New Group > Packed Assets` 선택
2. 그룹명을 `[그룹명]`으로 변경

**에셋 주소 등록**:
1. Project 창에서 에셋 선택
2. Inspector에서 `Addressable` 체크박스 활성화
3. 주소(Address)를 `[Addressables 키값]`으로 수정
4. 필요시 그룹 드롭다운에서 `[그룹명]`으로 변경

**빌드** (에디터 Play 전 필요한 경우):
1. Addressables Groups 창 상단 `Build > New Build > Default Build Script` 클릭

**확인**: Addressables Groups 창에서 에셋이 등록된 그룹 아래에 표시되어야 합니다.
```

---

## Input System 설정

```markdown
## 작업 N. Input Action Asset 설정

**목적**: [기능명]에 사용할 Input Action을 정의합니다.

**Input Action Asset 생성** (없는 경우):
1. Project 창에서 `Assets/[경로]` 우클릭
2. `Create > Input Actions` 선택
3. 파일명을 `[액션셋명]`으로 변경

**Action Map / Action 추가**:
1. 생성된 .inputactions 파일 더블클릭 → Input Actions 에디터 열기
2. `Action Maps` 패널에서 `+` 클릭 → 이름을 `[맵명]`으로 설정
3. `Actions` 패널에서 `+` 클릭 → 이름을 `[액션명]`으로 설정
4. 오른쪽 패널에서 Action Type 설정: `[Button / Value / Pass Through]`
5. Binding `+` 클릭 → `Add Binding` 선택 → Path에서 입력 키 지정
6. 우측 상단 `Save Asset` 클릭

**Player Input 컴포넌트 연결**:
1. 대상 GameObject 선택 → `Add Component > Player Input`
2. `Actions` 필드에 생성한 .inputactions 에셋 드래그
3. `Default Map`을 `[맵명]`으로 설정

**확인**: Play 모드에서 지정한 키 입력 시 이벤트가 발생해야 합니다.
```

---

## Test Runner 설정

```markdown
## 작업 N. Test Runner 확인 및 테스트 실행

**목적**: 작성한 테스트가 Test Runner에 인식되는지 확인합니다.

1. `Window > General > Test Runner` 클릭
2. `EditMode` 또는 `PlayMode` 탭 선택
3. 작성한 테스트 클래스가 목록에 표시되는지 확인
   - 표시되지 않으면: Assembly Definition의 `Test Assemblies` 체크 여부와 참조 확인
4. 테스트 클래스 또는 메서드 선택 후 `Run Selected` 클릭

**확인**: 테스트 항목이 ✓ (통과) 또는 ✗ (실패) 아이콘으로 표시되어야 합니다.
```

---

## URP / Renderer 설정

```markdown
## 작업 N. URP Renderer Feature 추가

**목적**: [기능명]에 필요한 [Feature명] Renderer Feature를 추가합니다.

1. Project 창에서 URP Renderer 에셋 선택 (기본 경로: `Assets/Settings/[프로젝트명]Renderer.asset`)
2. Inspector에서 `Add Renderer Feature` 클릭
3. 목록에서 `[Feature명]` 선택
4. 표시된 설정 항목 구성:
   - `[설정명]`: `[값]`

**확인**: Inspector의 Renderer Features 목록에 추가된 Feature가 표시되어야 합니다.
```
