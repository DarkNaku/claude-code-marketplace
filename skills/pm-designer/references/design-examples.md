# 설계서 예제 모음

설계서 작성 시 참고할 수 있는 실제 예제입니다. 유형별로 분류되어 있습니다.

---

## 유형 A: 신기능 설계

### 예제 1 — LinkDots 초기 구현 (전체 아키텍처)

```markdown
# 설계서: LinkDots (Flow Free 클론)

## 1. 시스템 아키텍처
본 게임은 Phaser 3 프레임워크를 사용하여 개발됩니다. 단일 Scene(`MainScene`) 구조로 시작하며, 필요 시 레벨 선택 Scene을 추가합니다.

### 1.1 Phaser 설정
- **타입**: Phaser.AUTO
- **크기**: 600x800 (세로형 모바일 레이아웃 최적화)
- **배경색**: #000000 (검은색)
- **부모 컨테이너**: `game-container`

## 2. 데이터 구조 설계

### 2.1 레벨 데이터 (Level Data)
```json
{
  "level": 1,
  "size": 5,
  "dots": [
    { "color": "0xFF0000", "positions": [[0,0], [4,4]] },
    { "color": "0x00FF00", "positions": [[0,4], [4,0]] }
  ]
}
```

### 2.2 그리드 상태 (Grid State)
- `Grid` 클래스는 `Cell` 객체의 2차원 배열을 관리합니다.
- `Cell` 속성:
    - `x, y`: 그리드 좌표.
    - `worldX, worldY`: 실제 렌더링 좌표.
    - `dotColor`: 해당 칸에 점이 있는 경우 색상 (null 가능).
    - `pathColor`: 해당 칸을 지나는 선의 색상 (null 가능).
    - `isOccupied`: 선이 점유하고 있는지 여부.

## 3. 클래스 및 메서드 설계

### 3.1 `MainScene` (Phaser.Scene)
- `preload()`: 필요한 에셋 로드.
- `create()`: 그리드 생성, 레벨 데이터 배치, 입력 이벤트 리스너 등록.
- `update()`: (필요 시) 실시간 UI 업데이트.

### 3.2 `GridManager`
- 그리드 초기화 및 렌더링.
- `getCell(x, y)`: 특정 좌표의 `Cell` 반환.
- `isAllFilled()`: 모든 셀이 채워졌는지 확인.

### 3.3 `PathManager`
- `startPath(cell)`: 드래그 시작 시 호출.
- `extendPath(cell)`: 드래그 중 새로운 셀 진입 시 호출.
- `validatePath(cell)`: 교차 여부 확인 및 기존 경로 끊기 로직 처리.
- `pathsByColor`: 색상별로 완성되거나 진행 중인 경로 저장.

## 4. 핵심 로직 및 알고리즘

### 4.1 경로 그리기 (Path Drawing)
1. 사용자가 특정 색상의 점 위에서 클릭/터치 시작.
2. 드래그 이동 시 인접한 셀(상하좌우)로만 이동 허용.
3. 새로운 셀로 이동 시:
    - 다른 색상의 선이 있다면: 해당 선을 끊음(초기화).
    - 같은 색상의 선이 이미 있다면: 해당 지점까지 경로를 되돌림(Backtrack).
    - 다른 색상의 점이 있다면: 진입 금지.
4. 목표 점(동일 색상)에 도달하면 경로 완성.

### 4.2 승리 판정 (Win Condition)
1. 모든 색상의 점 쌍이 연결됨.
2. 그리드의 모든 `Cell.isOccupied`가 `true`임.
3. 위 두 조건 충족 시 승리 알림 표시.

## 5. UI 및 시각적 요소
- **선(Pipe)**: `Graphics` 객체를 사용하여 부드러운 선 표현.
- **점(Dot)**: `Circle` 프리미티브 사용.
- **HUD**: 화면 상단에 현재 레벨 및 이동 횟수 표시.
- **Reset**: 화면 하단에 현재 레벨 초기화 버튼 배치.

```

---

### 예제 2 — 레벨 시스템 확장 (데이터/시스템)

```markdown
# 설계서: 레벨 시스템 확장 (LinkDots)

## 1. 데이터 구조 설계 (JSON Schema)

### 1.1 `levels.json` 구조
```json
{
  "packs": [
    {
      "name": "5x5 Basic",
      "levels": [
        {
          "id": 1,
          "size": 5,
          "dots": [
            { "color": "0xFF0000", "positions": [[0, 0], [4, 4]] },
            { "color": "0x00FF00", "positions": [[0, 4], [4, 0]] }
          ]
        }
      ]
    }
  ]
}
```

## 2. 클래스 및 로직 설계

### 2.1 `MainScene` 확장
- `preload()`: `this.load.json('levels', 'assets/levels/levels.json')` 추가.
- `init(data)`: 다른 씬으로부터 `levelId`를 전달받아 현재 레벨 설정.
- `create()`: `this.cache.json.get('levels')`로 데이터를 읽어 현재 레벨에 맞는 그리드 생성.

### 2.2 레벨 전환 로직
1. `checkWinCondition()`에서 승리 판정 시 `showWinMessage()` 호출.
2. `showWinMessage()` 내부에 'NEXT LEVEL' 버튼(Phaser Text) 생성.
3. 버튼 클릭 시:
    - 현재 레벨 인덱스 증가.
    - 다음 레벨이 존재하면 `this.scene.restart({ levelId: nextId })` 호출.
    - 마지막 레벨이면 "All Levels Completed!" 메시지 표시 후 메인으로 이동.

## 3. UI/UX 설계

### 3.1 승리 팝업 (Win Overlay)
- 배경: 반투명 검은색 사각형.
- 텍스트: "PERFECT!", "Level Complete".
- 버튼: "NEXT LEVEL" (Interactive Text).

### 3.2 HUD 업데이트
- 상단 중앙: `Level {PackName} - {LevelId}` 형식으로 표시.

## 4. 상세 구현 흐름
1. `assets/levels/` 디렉토리 생성 및 `levels.json` 작성.
2. `MainScene.js`의 `create()` 로직을 `levelId` 기반으로 동적 초기화되도록 수정.
3. `checkWinCondition` → `showWinMessage` → `Next Level` 버튼 로직 순차 구현.
4. 레벨 전환 시 `this.pathManager.reset()` 등 정리 작업 확인 (Scene Restart 시 자동 처리됨).

```

---

## 유형 B: 버그 수정 설계

### 예제 1 — 이벤트·상태 로직 수정 (이벤트 시스템 확장 포함)

```markdown
# 설계서: 이동 횟수(Moves) 카운팅 로직 최적화

## 1. 개요
단순 터치나 동일 색상의 반복 수정을 제외하고, 실질적인 파이프 그리기 작업이 발생했을 때만 이동 횟수를 증가시키는 로직을 설계합니다.

## 2. 세부 설계

### 2.1 이벤트 시스템 확장 (`PathManager.js`)
- **기존**: `startPath` (터치 시점)에서 `path-started` 이벤트 발생.
- **변경**: 파이프의 마디가 2개가 되는 시점에 새로운 이벤트 발생.
- **신규 이벤트**: `path-extended-first`
    - **발생 조건**: `this.currentPath.length === 2`인 순간.
    - **전달 데이터**: `this.currentColor`.

### 2.2 상태 관리 및 카운팅 로직 (`MainScene.js`)
- **상태 변수**: `this.lastMovedColor` (마지막으로 이동 횟수를 소모한 색상).
- **로직 흐름**:
    1. `path-extended-first` 이벤트 수신.
    2. 수신된 색상이 `this.lastMovedColor`와 같은지 확인.
    3. **다르다면**: `this.moves++`, `this.lastMovedColor = currentColor`, `uiManager.updateStats()` 호출.
    4. **같다면**: 무시 (동일 파이프 수정으로 간주).

### 2.3 예외 상황 및 초기화
- **레벨 초기화(Reset)**: `this.moves = 0`, `this.lastMovedColor = null`로 초기화.
- **파이프 끊김**: 다른 파이프를 가로질러 끊겨도 `lastMovedColor`는 유지됨. 끊긴 파이프를 다시 그릴 때 새로운 이동으로 카운트.

## 3. 상세 수정 스펙

| 파일 | 위치 | 수정 내용 |
| :--- | :--- | :--- |
| `src/classes/PathManager.js` | `extendPath()` | 길이가 2가 되는 시점에 `path-extended-first` 이벤트 emit. |
| `src/scenes/MainScene.js` | `create()` / `setupEvents()` | 이동 횟수 증가 조건을 `lastMovedColor` 비교 로직으로 변경. |
| `src/scenes/MainScene.js` | `init()` / `create()` | `lastMovedColor` 변수 초기화 로직 추가. |

## 4. 기대 결과
- 점을 터치만 하는 실수로 인한 이동 횟수 낭비 방지.
- 같은 색상의 파이프를 여러 번 시도해도 다른 색상을 만지기 전까지 1회의 이동으로 보호.

```

---

### 예제 2 — 코드 수정 설계 (코드 스니펫 포함)

```markdown
# 설계서: 파이프 점유율 계산 버그 수정

## 1. 개요
그리드 점유율 계산 시 점(Dot)의 존재 여부를 배제하고, 실제 플레이어가 그린 파이프의 점유 상태만을 반영하도록 로직을 수정합니다.

## 2. 로직 수정 설계

### 2.1 `GridManager.getFillPercentage()` 수정
- **기존 로직**: `isOccupied || dotColor` 인 경우를 모두 점유된 것으로 카운트.
- **수정 로직**: 오직 `isOccupied`가 `true`인 경우만 카운트.
- **이유**: `PathManager`는 파이프가 시작되는 점(Dot) 위치의 `isOccupied`도 `true`로 설정하므로, `isOccupied`만 체크하는 것이 실제 점유 상태를 가장 정확하게 나타냄.

```javascript
getFillPercentage() {
    let occupied = 0;
    const total = this.size * this.size;
    for (let y = 0; y < this.size; y++) {
        for (let x = 0; x < this.size; x++) {
            if (this.cells[y][x].isOccupied) { // dotColor 체크 제거
                occupied++;
            }
        }
    }
    return Math.floor((occupied / total) * 100);
}
```

### 2.2 `GridManager.isAllFilled()` 검토
- 승리 판정도 동일한 철학 유지.
- 모든 칸의 `isOccupied`가 `true`인지만 확인하도록 단순화.

## 3. 상세 수정 스펙

| 파일 | 위치 | 수정 내용 |
| :--- | :--- | :--- |
| `src/classes/GridManager.js` | `getFillPercentage()` | `dotColor` 조건 제거, `isOccupied`로만 판정. |
| `src/classes/GridManager.js` | `isAllFilled()` | 모든 셀의 `isOccupied` 여부만 체크하도록 최적화. |

## 4. 검증 계획
1. **초기 상태 확인**: 레벨 진입 직후(파이프 0개) 점유율이 0%인지 확인.
2. **그리기 도중 확인**: 파이프를 한 칸 그릴 때마다 점유율이 1/N씩 증가하는지 확인.
3. **삭제 시 확인**: 파이프를 지웠을 때 점유율이 다시 감소하고, 모두 지우면 0%가 되는지 확인.
4. **승리 시 확인**: 모든 칸이 채워졌을 때 100%와 함께 승리 팝업이 뜨는지 확인.

```

---

## 유형 C: UI/UX 설계

### 예제 1 — 인터랙션 요소 설계 (터치 하이라이트)

```markdown
# 설계서: 터치 하이라이트 효과 추가

## 1. 하이라이트 객체 사양
- **타입**: `Phaser.GameObjects.Arc` (Circle)
- **크기**: `cellSize * 0.8` (그리드 셀 크기의 약 80%를 지름으로 설정)
- **투명도**: `0.35` (반투명)
- **뎁스(Depth)**: `15` (그리드: 0, 파이프: 5, 점: 10보다 상위)
- **색상**: `pathManager.currentColor`를 동적으로 할당.

## 2. 입력 상태별 동작 흐름 (`MainScene.js`)

### 2.1 Pointer Down (터치 시작)
1. 사용자가 점을 터치하여 경로 그리기가 가능해지면:
2. `highlightCircle.setVisible(true)` 호출.
3. `highlightCircle.setFillStyle(currentColor, 0.35)`로 색상 동적 변경.
4. `highlightCircle.setPosition(pointer.x, pointer.y)` 초기 위치 설정.

### 2.2 Pointer Move (드래그 중)
- `highlightCircle.setPosition(pointer.x, pointer.y)`를 호출하여 실시간으로 위치 추적.

### 2.3 Pointer Up (터치 종료)
- `highlightCircle.setVisible(false)` 호출하여 화면에서 즉시 제거.

## 3. 구현 방식 설계

### 3.1 객체 생성 (`MainScene.create`)
```javascript
this.touchHighlight = this.add.circle(0, 0, this.gridManager.cellSize * 0.4, 0xffffff, 0.35);
this.touchHighlight.setVisible(false);
this.touchHighlight.setDepth(15);
```

### 3.2 이벤트 핸들러 업데이트 (`MainScene.setupInputs`)
- `pointerdown`: `this.touchHighlight.setFillStyle(this.pathManager.currentColor, 0.35); this.touchHighlight.setVisible(true);`
- `pointermove`: `if (pointer.isDown) this.touchHighlight.setPosition(pointer.x, pointer.y);`
- `pointerup`: `this.touchHighlight.setVisible(false);`

## 4. 예외 처리 및 고려 사항
- **색상 없음**: 빈 공간 터치 시 `pathManager.currentColor` 유무를 체크하여 하이라이트 미표시.
- **성능**: 매 프레임 업데이트 대신 `pointermove` 이벤트 발생 시에만 좌표를 갱신.

```

---

### 예제 2 — 화면 전체 레이아웃 재설계 (9:16 화면 비율)

```markdown
# 설계서: 화면 비율 최적화 (9:16 Aspect Ratio)

## 1. 게임 엔진 및 전역 설정

### 1.1 Phaser Config 수정
- **기존 해상도**: 600 x 800 (3:4)
- **변경 해상도**: 720 x 1280 (9:16)
- **위치**: `src/main.js`

## 2. 동적 UI 시스템 설계 (`UIManager.js`)

### 2.1 공통 수식
- **Center X**: `this.width / 2`
- **Center Y**: `this.height / 2`
- **Bottom Margin**: `this.height - 100`

### 2.2 UIManager 메서드 수정
- `createTitle`: y 좌표를 `this.height * 0.08`로 조정.
- `createMenuListItem`: x 시작 위치를 80에서 `this.width * 0.1`로 변경.
- `createPagination`: y 위치를 740에서 `this.height * 0.9`로 변경.

## 3. Scene별 레이아웃 재설계

### 3.1 TitleScene (메인 메뉴)
- **로고 ("flow")**: y: 220 → 320 (상단 여백 확보).
- **메뉴 리스트**: 시작 y: 420 → 580, 간격: 100px.
- **하단 버튼**: y: `height - 80` → `height - 120`.

### 3.2 MainScene (게임 플레이)
- **그리드 크기 (`gridSize`)**: 400 → 550 (화면 너비 720px 기준 좌우 여백 각 85px).
- **그리드 위치**: 화면 중앙 배치를 위한 `offsetY` 계산식 정교화.
- **HUD**: 상단 여백 확대 (`height * 0.05` 지점).

## 4. 시각적 자산 및 효과 조정
- **TitleScene 배경**: `Phaser.Math.Between(0, 720)`, `Phaser.Math.Between(0, 1280)`으로 생성 범위 확장.
- **Win Overlay**: 화면 전체를 덮는 `rectangle` 크기를 720x1280으로 자동 확장.

```

---

## 수정 스펙 표 작성 패턴

### 버그 수정용 (필수)

```markdown
## 3. 상세 수정 스펙

| 파일 | 위치 | 수정 내용 |
| :--- | :--- | :--- |
| `src/classes/PathManager.js` | `endPath()` | `if (this.currentPath && this.currentPath.length < 2) this.clearPath(this.currentColor)` 로직 추가. |
| `src/scenes/MainScene.js` | `setupInputs()` | `pointerdown` 콜백 내부에 `draw()` 및 `updateHUD()` 추가. |
| `src/scenes/MainScene.js` | `setupInputs()` | `pointerup` 콜백 내부에 `draw()` 및 `updateHUD()` 추가. |
```

### 검증 계획 작성 패턴

검증 계획은 버그 수정 설계에서 "기대 결과" 대신 사용할 수 있습니다. 단계별 확인 순서로 작성합니다.

```markdown
## 4. 검증 계획
1. **[초기 상태]**: [확인 조건].
2. **[동작 중]**: [확인 조건].
3. **[해제/취소 시]**: [확인 조건].
4. **[완료 상태]**: [확인 조건].
```
