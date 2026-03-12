# 검증 방법 레퍼런스

작업 유형별 파일 검사 쿼리와 판정 기준 모음.

---

## 패키지 설치 확인

### 검사 파일
```
Read: Packages/manifest.json
```

### 판정 기준

| 결과 | 조건 |
|------|------|
| ✅ | `dependencies`에 `"[패키지명]"` 키가 존재하고 버전이 일치 |
| ⚠️ | 키는 존재하지만 버전이 가이드와 다름 |
| ❌ | `dependencies`에 키가 없음 |

### 예시 쿼리
```
Grep: "[패키지명]"   path: Packages/manifest.json
```

---

## Assembly Definition 확인

### 파일 존재 여부
```
Glob: Assets/**/*.asmdef
Glob: Assets/[예상 경로]/[어셈블리명].asmdef
```

| 결과 | 조건 |
|------|------|
| ✅ | 예상 경로에 .asmdef 파일 존재 |
| ❌ | 파일 없음 |

### 참조(References) 확인
```
Read: [어셈블리명].asmdef
```

.asmdef는 JSON 형식이므로 `references` 배열에서 확인:

```json
{
  "name": "MyProject.Core",
  "references": [
    "GUID:...",
    "MyProject.Shared"
  ],
  "includePlatforms": [],
  "excludePlatforms": [],
  "allowUnsafeCode": false,
  "overrideReferences": false,
  "precompiledReferences": [],
  "autoReferenced": true,
  "defineConstraints": [],
  "versionDefines": [],
  "noEngineReferences": false
}
```

| 결과 | 조건 |
|------|------|
| ✅ | `references` 배열에 기대 어셈블리명 또는 GUID 포함 |
| ⚠️ | 일부 참조만 포함 |
| ❌ | `references` 배열이 비어 있거나 기대 항목 없음 |
| 🔍 | GUID 형식(`GUID:...`)만 있는 경우 — GUID와 실제 파일 매핑 수동 확인 필요 |

### 테스트 어셈블리 설정 확인

```
Read: [어셈블리명].asmdef
```

- `"includePlatforms": ["Editor"]` 또는 `"Editor"` 포함 여부 확인 (EditMode)
- `overrideReferences`가 `true`이고 `precompiledReferences`에 `"nunit.framework.dll"` 포함 여부

---

## Project Settings 확인

### Tags 확인
```
Read: ProjectSettings/TagManager.asset
```

YAML 형식 파일에서 `tags:` 섹션 탐색:
```yaml
tags:
- [태그명1]
- [태그명2]
```

```
Grep: "[태그명]"   path: ProjectSettings/TagManager.asset
```

| 결과 | 조건 |
|------|------|
| ✅ | `tags:` 섹션에 기대 태그명 존재 |
| ❌ | 태그명 없음 |

### Layers 확인
```
Read: ProjectSettings/TagManager.asset
```

`layers:` 섹션에서 확인:
```yaml
layers:
-
-
-
-
-
-
-
-
- [레이어명]    ← User Layer 8부터
```

```
Grep: "[레이어명]"   path: ProjectSettings/TagManager.asset
```

### Scripting Define Symbols 확인
```
Read: ProjectSettings/ProjectSettings.asset
```

플랫폼별 키를 확인:
```yaml
scriptingDefineSymbols:
  1: SYMBOL_A;SYMBOL_B        # Standalone
  13: SYMBOL_A;SYMBOL_B       # Android
  …
```

```
Grep: "[심볼명]"   path: ProjectSettings/ProjectSettings.asset
```

| 결과 | 조건 |
|------|------|
| ✅ | 대상 플랫폼 키의 값에 심볼명 포함 |
| ⚠️ | 일부 플랫폼에만 포함 |
| ❌ | 어떤 플랫폼 키에도 포함되지 않음 |

---

## ScriptableObject 에셋 확인

### 파일 존재 여부
```
Glob: Assets/[예상 경로]/[에셋명].asset
```

| 결과 | 조건 |
|------|------|
| ✅ | 예상 경로에 .asset 파일 존재 |
| ❌ | 파일 없음 |

### 타입 확인
```
Read: [에셋명].asset   (첫 5줄)
```

Unity 에셋 파일 YAML 헤더에서 스크립트 참조 확인:
```yaml
MonoBehaviour:
  m_Script: {fileID: 11500000, guid: [guid], type: 3}
```

- `guid` 값이 대상 ScriptableObject 스크립트의 GUID와 일치하는지 확인
- GUID는 `[스크립트명].cs.meta` 파일에서 확인:
  ```
  Read: Assets/[경로]/[스크립트명].cs.meta
  Grep: "^guid:"   path: [스크립트명].cs.meta
  ```

| 결과 | 조건 |
|------|------|
| ✅ | GUID 일치 |
| ⚠️ | 파일은 있으나 GUID 불일치 (다른 타입의 에셋) |
| 🔍 | GUID 확인 불가 — Inspector에서 타입 직접 확인 필요 |

### Inspector 필드값 확인
```
Read: [에셋명].asset
Grep: "[필드명]:"   path: [에셋명].asset
```

YAML에서 직렬화된 필드 탐색:
```yaml
  m_[필드명]: [값]
  [필드명]: [값]
```

| 결과 | 조건 |
|------|------|
| ✅ | 필드명과 값이 가이드 기대값과 일치 |
| ⚠️ | 필드는 존재하지만 값이 다름 |
| ❌ | 필드가 기본값(0, "", null)으로 설정됨 |
| 🔍 | Object 참조 필드 — GUID로만 확인 가능, 실제 에셋 연결은 수동 확인 |

---

## Prefab 확인

### 파일 존재 여부
```
Glob: Assets/[예상 경로]/[Prefab명].prefab
```

### 컴포넌트 존재 확인
```
Read: [Prefab명].prefab
Grep: "m_Script"   path: [Prefab명].prefab
```

Prefab YAML에서 컴포넌트 블록 탐색:
```yaml
MonoBehaviour:
  m_Script: {fileID: 11500000, guid: [컴포넌트 스크립트 GUID], type: 3}
```

```
Grep: "[컴포넌트명]"   path: [Prefab명].prefab
```

| 결과 | 조건 |
|------|------|
| ✅ | 컴포넌트 스크립트 GUID가 Prefab에 포함됨 |
| ❌ | 컴포넌트 블록 없음 |
| 🔍 | Inspector 필드 연결 (Object 참조) — 수동 확인 필요 |

---

## Addressables 설정 확인

### 초기화 여부
```
Glob: Assets/AddressableAssetsData/AddressableAssetSettings.asset
```

| 결과 | 조건 |
|------|------|
| ✅ | 파일 존재 |
| ❌ | 파일 없음 (Addressables 초기화 미수행) |

### 그룹 존재 확인
```
Glob: Assets/AddressableAssetsData/AssetGroups/[그룹명].asset
```

### 에셋 주소 등록 확인
```
Glob: Assets/AddressableAssetsData/AssetGroups/*.asset
Grep: "[Addressables 키값]"   path: Assets/AddressableAssetsData/
```

에셋 설정 파일에서 `m_Address` 필드 탐색:
```yaml
m_Address: [키값]
```

| 결과 | 조건 |
|------|------|
| ✅ | 키값이 AssetGroups 내 파일에 포함됨 |
| ❌ | 키값 없음 |
| 🔍 | 빌드 여부 — Play Mode Script 설정은 수동 확인 필요 |

---

## Input Action Asset 확인

### 파일 존재 여부
```
Glob: Assets/[예상 경로]/[액션셋명].inputactions
```

### Action Map / Action 확인
```
Read: [액션셋명].inputactions
```

`.inputactions`는 JSON 형식:
```json
{
  "name": "[액션셋명]",
  "maps": [
    {
      "name": "[맵명]",
      "actions": [
        {
          "name": "[액션명]",
          "type": "[Button|Value|PassThrough]",
          "bindings": [...]
        }
      ]
    }
  ]
}
```

```
Grep: "\"[맵명]\""     path: [액션셋명].inputactions
Grep: "\"[액션명]\""   path: [액션셋명].inputactions
```

| 결과 | 조건 |
|------|------|
| ✅ | 맵명·액션명·타입이 모두 일치 |
| ⚠️ | 맵 또는 액션 중 하나만 존재 |
| ❌ | 파일 없음 또는 맵·액션 미존재 |
| 🔍 | Binding 경로(키 매핑) — 값은 확인되나 올바른 키인지는 Play 모드에서 확인 필요 |

---

## 판정 불가 항목 (항상 🔍)

파일 검사만으로는 확인이 불가능하여 **반드시 Editor에서 수동 확인**이 필요한 항목:

| 항목 | 수동 확인 방법 |
|------|---------------|
| Prefab Inspector 오브젝트 참조 연결 | Prefab Edit 모드 → Inspector 필드에 에셋 표시 여부 |
| ScriptableObject 오브젝트 참조 필드 | 에셋 선택 → Inspector에서 필드 연결 에셋 확인 |
| Player Input 컴포넌트의 Actions 필드 | GameObject Inspector → Player Input > Actions 확인 |
| Renderer Feature 추가 | URP Renderer 에셋 Inspector 확인 |
| Test Runner 테스트 인식 | `Window > General > Test Runner`에서 목록 확인 |
| Build Settings 씬 등록 | `File > Build Settings`에서 씬 목록 확인 |
