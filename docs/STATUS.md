# 프로젝트 현재 상태 (Status & Handover)

> 마지막 업데이트: 2026-05-29
> 이 문서는 **살아있는 핸드오버 문서**입니다. 작업 재개 시 가장 먼저 읽어주세요.
> 정보가 바뀌면 즉시 갱신해서 다음 세션이 5분 안에 컨텍스트 회복할 수 있게 유지합니다.

---

## 1. 프로젝트 한 줄 소개
**my-claude-os** — Claude Code를 학습하면서 "나만의 클로드 OS"를 점진적으로 구축하는 학습용 레포. 슬래시 커맨드 → 훅 → 서브 에이전트로 추상화를 단계별로 쌓아 올린다.

## 2. 현재 단계
- **완료:** 기초 슬래시 커맨드 4종, 스킬 사용 시간 추적 훅, 웹앱 개발 에이전트 9개 정의, 표준 디렉토리 골격
- **진행 중 / 보류:** 에이전트 실제 동작 검증 — 환경 한계로 막혀 있음 (§5 참조)
- **다음 단계 후보:** §6 참조

## 3. 구축된 자산 한눈에

### 슬래시 커맨드 (`.claude/commands/`)
| 커맨드 | 역할 |
|--------|------|
| `/git-commit` | git 변경사항 분석 → 커밋 메시지 제안 → 커밋 |
| `/daily-log` | 그날의 git 이력을 기반으로 일일 작업 로그 작성 (`logs/YYYY-MM-DD.md`) |
| `/tech-qna` | 기술 개념을 학습용 포맷(정의→필요성→동작→오해→다음→출처)으로 설명 |
| `/skill-stat` | `logs/skill-usage.jsonl` 집계 → 스킬별 호출/평균/최대/최소 시간 표 |

### 훅 (`.claude/settings.json`)
- `UserPromptSubmit`: 슬래시 커맨드 호출 시 시작 시각 기록 (`/tmp/claude-skill-current.json`)
- `Stop`: 응답 종료 시 소요 시간 계산 후 `logs/skill-usage.jsonl` 에 한 줄 append

### 웹앱 개발 에이전트 (`.claude/agents/` — 등록 미인식 상태, §5 참조)
| # | 에이전트 | 입력 | 출력 |
|---|----------|------|------|
| 0 | `dev-orchestrator` | 사용자 요청 | 단계별 위임 + 체크포인트 조정 |
| 1 | `requirements-interviewer` | (사용자 인터뷰) | `docs/01-requirements.md` |
| 2 | `spec-writer` | `01-requirements.md` | `docs/02-spec.md` (+ RTM) |
| 3 | `system-architect` | `02-spec.md` | `docs/03-system-architecture.md` (+ Mermaid) |
| 4 | `data-modeler` | `02-spec.md`, `03-system-architecture.md` | `docs/04-data-model.md` (+ ERD + DDL) |
| 5 | `software-architect` | `02`, `03`, `04` | `docs/05-software-architecture.md` (+ API) |
| 6 | `prototyper` | `02`, `04`, `05` | `src/`, `docs/06-prototype-notes.md` |
| 7 | `test-writer` | `02`, `05`, `src/` | `tests/` |
| 8 | `code-reviewer` | `02`, `05`, `src/`, `tests/` | `docs/reviews/NN-review.md` |

전체 다이어그램은 `README.md` 참조.

### 표준 디렉토리 골격
```
docs/        ← 에이전트 산출물 01~06, STATUS.md, reviews/
src/         ← 프로토타입 코드 (prototyper)
tests/       ← 테스트 코드 (test-writer)
migrations/  ← DB 마이그레이션
scripts/     ← 시드/유틸 스크립트
logs/        ← 일일 작업 로그 + 스킬 사용 통계
```

## 4. 프로젝트 핵심 철학 (README 발췌)
1. **사이드이펙트 방지** — 테스트 코드로 강제. 통합 테스트는 실제 DB 사용 (mock 금지)
2. **DDD, 객체지향, 팀 컨벤션** 기반의 클린 코드 유지
3. **비즈니스 도메인 무결성** — 도메인 지식을 컨텍스트로 체계적 관리

각 에이전트의 행동 원칙은 이 철학에서 도출됨. 새 에이전트 추가 시에도 이 세 가지를 기준으로 행동 정의.

## 5. 알려진 이슈와 우회법

### 🔴 [Blocker] 프로젝트 로컬 `.claude/agents/` 미스캔
**증상:** 새 세션을 시작해도 `Agent` 도구의 `subagent_type` 으로 우리 에이전트(예: `dev-orchestrator`)를 지정하면 `Agent type 'dev-orchestrator' not found` 에러. `/agents` Library 탭에도 우리 9개 에이전트가 표시되지 않음.

**확인된 조건:**
- 파일 위치: `프로젝트/.claude/agents/*.md` ✅
- frontmatter 형식: `name`, `description`, `tools` ✅
- 세션 재시작: ✅ (효과 없음)
- 글로벌 `~/.claude/agents/`: 디렉토리 자체가 없음

**원인 추정:** 현재 실행 환경(FleetView 호스팅 추정)이 표준 Claude Code CLI 와 다른 동작 — 프로젝트 로컬 에이전트 디스코버리가 비활성일 가능성.

**우회 옵션 (검증 안 됨, 다음 세션에 시도 가능):**
1. **글로벌 위치로 심볼릭 링크** — `mkdir -p ~/.claude/agents && ln -s "$PWD/.claude/agents/"*.md ~/.claude/agents/` 후 세션 재시작
2. **글로벌 위치로 파일 복사** — 위와 동일하나 `cp`. 단점은 수동 동기화
3. **`general-purpose` 위임** — 동작 검증 대신 에이전트 정의 품질만 검증. 사용자가 가상 사용자 답변까지 제공하는 시뮬레이션 진행
4. **표준 Claude Code CLI 환경에서 테스트** — 로컬 머신에서 직접 실행

## 6. 다음 액션 후보 (우선순위 순)

### A. 에이전트 등록 우회 시도
- 1순위 검증: `ln -s` 로 글로벌 등록 → `/agents` 에 보이는지 확인
- 안 되면 `cp` 로 복사 → 재확인
- 둘 다 안 되면 환경 한계 확정, B로 이동

### B. `general-purpose` 위임으로 정의 품질 검증
- To-do 웹앱 가상 시나리오 (이전 세션에서 합의됨)
- 가상 사용자 답변 4 Phase 분량을 미리 작성해서 한 호출에 전달
- 결과물(`docs/01-requirements.md` 시뮬레이션)과 에이전트 정의에서 발견한 모호점/개선점 리포트 수집

### C. 보완할 수 있는 에이전트 영역
검증 완료 후, 다음 추가 고려:
- **observability 에이전트** — 로깅/메트릭/알람 설계 전담 (현재 `system-architect` 안에 흡수)
- **migration 에이전트** — DB 마이그레이션 안전 적용 전담
- **release 에이전트** — 배포/롤백 절차 자동화

### D. CLAUDE.md 보강
- 9개 에이전트가 추가되면서 프로젝트가 학습용을 넘어 워크플로우 도구가 되어가는 중 → CLAUDE.md 에 "에이전트 호출 순서", "산출물 표준 경로" 등 명시하면 새 세션이 컨텍스트 빠르게 회복.

## 7. 참고 위치
- 에이전트 정의: `.claude/agents/*.md`
- 슬래시 커맨드: `.claude/commands/*.md`
- 설정/훅: `.claude/settings.json`
- 일일 작업 로그: `logs/YYYY-MM-DD.md`
- 스킬 사용 통계 원본: `logs/skill-usage.jsonl` (`/skill-stat` 으로 집계)
- 워크플로우 다이어그램: `README.md` 의 "워크플로우" / "오케스트레이터 호출 관계" 섹션

## 8. 변경 로그 (이 문서)
- 2026-05-29: 최초 작성. 에이전트 등록 이슈와 다음 액션 후보 정리.
