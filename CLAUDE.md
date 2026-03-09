# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 저장소 개요

Claude Code 플러그인 마켓플레이스 저장소로, Unity 개발을 위한 플러그인과 스킬을 포함합니다. Claude Code의 플러그인 시스템에 정의된 구조를 따릅니다.

## 저장소 구조

2단계 플러그인 아키텍처를 사용합니다:

```
.
├── .claude-plugin/
│   └── marketplace.json          # 마켓플레이스 설정 (owner: DarkNaku)
└── unity-plugin/
    ├── .claude-plugin/
    │   └── plugin.json            # 플러그인 메타데이터
    ├── agents/                    # 커스텀 에이전트 (현재 비어있음)
    └── skills/                    # Unity 전문 스킬
        ├── unity-vcontainer/      # VContainer DI 스킬
        │   ├── SKILL.md
        │   └── references/
        │       ├── vcontainer-patterns.md
        │       └── vcontainer-integrations.md
        └── unity-r3/              # R3 반응형 프로그래밍 스킬 (준비중)
```

### 주요 설정 파일

**marketplace.json** (`.claude-plugin/marketplace.json`):
- 마켓플레이스 소유자 정의 (DarkNaku)
- 사용 가능한 플러그인 목록과 소스 경로, 설명 포함
- 상대 경로를 사용하여 플러그인 디렉토리 지정

**plugin.json** (`unity-plugin/.claude-plugin/plugin.json`):
- 플러그인 메타데이터 포함 (name, description, version, author)
- 현재 "unity-plugin" v0.0.1 정의

## 스킬 시스템

스킬은 Claude Code가 특정 작업에 사용할 수 있는 전문 지식 모듈입니다. 각 스킬은 다음 구조를 따릅니다:

### 스킬 구성

**SKILL.md**: YAML frontmatter가 있는 메인 스킬 정의 파일
- `name`: 스킬 식별자
- `description`: 스킬 전문성과 사용 시점에 대한 상세 설명
- 콘텐츠 섹션: 개요, 빠른 시작, 핵심 개념, 참고 문서, 모범 사례

**references/**: 심층 참조 문서
- 상황별 사용 패턴 가이드 (예: `vcontainer-patterns.md`)
- 다른 라이브러리 또는 프레임워크와 통합 가이드 (예: `vcontainer-integrations.md`)

### 기존 스킬

**unity-vcontainer**: VContainer 의존성 주입 전문가
- IoC 컨테이너 구성, 라이프사이클 관리, Unity 최적화 DI 패턴 전문
- 포함 내용: 의존성 해결, 스코프 컨테이너, 테스트 가능한 아키텍처
- 사용 시점: VContainer 설정, 서비스 등록, SOLID 원칙 구현
- 참조 문서: 등록 패턴, LifetimeScope 계층 구조, Mock을 사용한 테스트, 팩토리 패턴, EntryPoint 패턴, 멀티 씬 아키텍처

**unity-r3**: 현재 준비 중 (SKILL.md 없음)

## 새 스킬 추가 방법

새 스킬을 생성할 때:

1. `unity-plugin/skills/<skill-name>/` 디렉토리 생성
2. 적절한 frontmatter와 함께 `SKILL.md` 추가:
   ```yaml
   ---
   name: skill-identifier
   description: 한글 또는 영어로 상세 설명
   ---
   ```
3. 섹션 포함: 개요, 빠른 시작, 핵심 개념, 참고 문서, 모범 사례
4. 상세 패턴 및 통합을 위한 `references/` 하위 디렉토리에 참조 문서 추가
5. 적절한 C# 구문 강조가 있는 코드 예제 사용
6. `unity-vcontainer`의 문서 스타일을 템플릿으로 따르기

## 언어 규칙

- 스킬 문서는 주로 한글로 작성
- 코드 예제는 헝가리안 표기법 접두사를 사용한 영어 식별자 사용 (예: `mPlayerService`, `mConfig`)
- 코드 예제의 주석은 한글

## Git 워크플로우

- 메인 브랜치: `main`
- 저장소는 표준 git 워크플로우 사용
- 최근 커밋은 점진적 스킬 추가를 보여줌 (VContainer 스킬이 최근 추가됨)

## 개발 노트

- 콘텐츠 중심 저장소 (문서 및 설정 파일)
- 빌드, 테스트 또는 컴파일 명령 불필요
- 스킬 문서 품질 및 구조 유지에 집중
- 스킬 수정 시 YAML frontmatter가 유효한지 확인
- 참조 문서는 상세하고 예제가 풍부하게 유지
