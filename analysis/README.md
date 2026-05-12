# Open Design 소스 분석 보고서

이 폴더는 `open-design` 모노레포(`v0.6.0`)의 소스 구조를 한국어로 정리한 분석 문서입니다.
대상 커밋: `main` (`23218ea`, 2026-05-12).

## 분석 범위

Open Design은 **"로컬 우선 디자인 제품 — 설치된 코딩 에이전트 CLI를 자동 감지해서 디자인 스킬 + 디자인 시스템을 돌리고, 아티팩트를 샌드박스 미리보기로 스트리밍한다"** 라는 단일 목적의 모노레포입니다. (`package.json:7`)

문서는 **5개 주제 그룹**으로 계층화되어 있고, 각 문서는 [`svg/`](./svg/) 폴더의 다이어그램이 본문 상단에 임베드되어 있습니다.

## 폴더 구조

```
analysis/
├── README.md
├── 00-overview/        — 모노레포 개요와 4계층 설계 원칙
├── 10-structure/       — apps, packages, tools 코드 구조
├── 20-content/         — skills/design-systems 콘텐츠 시스템과 카탈로그
├── 30-runtime/         — 에이전트 스폰, 프롬프트, SSE 채팅, 시퀀스
├── 40-data/            — SQLite 영속성과 .od/ 레이아웃
├── 50-integration/     — BYOK 프록시, 미디어 생성, 보안 모델
└── svg/                — 13개 SVG 다이어그램 (각 문서에서 임베드)
```

## 문서 인덱스

### 00-overview/ — 광역

| 문서 | 내용 |
|---|---|
| [01-architecture.md](./00-overview/01-architecture.md) | 모노레포 4계층 다이어그램, 7가지 핵심 설계 원칙, 컴포넌트 책임 매트릭스, 4종 배포 형태 |

### 10-structure/ — 코드 구조

| 문서 | 내용 |
|---|---|
| [02-apps.md](./10-structure/02-apps.md) | `apps/` 6개 앱(daemon, web, desktop, packaged, landing-page, telemetry-worker) |
| [03-packages.md](./10-structure/03-packages.md) | `packages/` 4개 패키지(contracts, sidecar-proto, sidecar, platform) |
| [04-tools.md](./10-structure/04-tools.md) | `tools/` 3개 컨트롤 플레인(dev, pack, pr) |

### 20-content/ — 콘텐츠 시스템

| 문서 | 내용 |
|---|---|
| [05-content-catalog.md](./20-content/05-content-catalog.md) | `skills/`, `design-systems/`, `design-templates/`, `craft/`, `prompt-templates/`, `specs/` 카탈로그 |
| [13-skill-catalog.md](./20-content/13-skill-catalog.md) | `skills.ts` 정규화 함수, shadowing 패턴, derived examples, 새 스킬 추가 절차 |

### 30-runtime/ — 런타임 동작

| 문서 | 내용 |
|---|---|
| [06-data-flows.md](./30-runtime/06-data-flows.md) | 라이프사이클 부트스트랩, 3-턴 채팅, 데스크탑 IPC, HMAC 폴더 import, 패키지 부트스트랩, PR 트리아주 — 6개 시퀀스 |
| [07-agent-runtime.md](./30-runtime/07-agent-runtime.md) | 16개 코딩 에이전트 CLI 어댑터, 4종 스트림 포맷 정규화, capability 프로브, child process 생애주기 |
| [08-prompt-engine.md](./30-runtime/08-prompt-engine.md) | 3-턴 결정론적 프롬프트(Turn 1 폼 → Turn 2 방향 → Turn 3+ 빌드), 11-레이어 system prompt 조립, 5개 OKLch 팔레트, anti-AI-slop 체크리스트 |
| [09-sse-chat-pipeline.md](./30-runtime/09-sse-chat-pipeline.md) | `POST /api/chat` 전체 흐름: 검증 → 프롬프트 합성 → 스폰 → 스트림 정규화 → SSE 이벤트 → 인터럽트/취소/영속화 |

### 40-data/ — 영속성

| 문서 | 내용 |
|---|---|
| [10-persistence.md](./40-data/10-persistence.md) | SQLite 12+ 테이블 스키마, `.od/` 디렉토리 레이아웃, 누적식 마이그레이션, 경로 검증, `OD_DATA_DIR` 오버라이드 |

### 50-integration/ — 외부 통합과 보안

| 문서 | 내용 |
|---|---|
| [11-byok-and-media.md](./50-integration/11-byok-and-media.md) | BYOK 프록시(Anthropic/OpenAI/Azure/Google), SSE 정규화, SSRF 가드, 미디어 생성(gpt-image-2, Seedance, HyperFrames), media_tasks 비동기 추적 |
| [12-security-model.md](./50-integration/12-security-model.md) | HMAC 데스크탑 게이트, `od://` 프로토콜, srcdoc iframe 샌드박스, 경로 검증, anti-AI-slop 린터, OAuth/MCP 토큰, 위협 모델 매트릭스 |

## 한눈에 보는 숫자

- 워크스페이스: `apps/* + packages/* + tools/* + e2e` (pnpm 10.33.2, Node ~24)
- 활성 앱: 6개 (`apps/daemon`, `apps/web`, `apps/desktop`, `apps/packaged`, `apps/landing-page`, `apps/telemetry-worker`)
- 활성 패키지: 4개 (`contracts`, `sidecar-proto`, `sidecar`, `platform`)
- 컨트롤 플레인: 3개 (`tools-dev`, `tools-pack`, `tools-pr`)
- 지원 코딩 에이전트 CLI: **16개** (Claude Code, Codex, Devin for Terminal, Cursor Agent, Gemini, OpenCode, Qwen, Qoder, GitHub Copilot, Hermes, Kimi, Pi, Kiro, Kilo, Mistral Vibe, DeepSeek)
- `skills/` 디렉토리: 100+ 스킬 (`design-system`, `image`, `prototype`, `deck`, `video`, `audio` 등의 mode)
- `design-systems/`: 150여 개 브랜드별 `DESIGN.md`
- `design-templates/`: 100여 개 렌더링 템플릿
- `craft/`: 보편 디자인 규칙 12개 (typography, color, anti-ai-slop 등)
- `prompt-templates/`: 100여 개 이미지/비디오 프롬프트 (JSON)

## 빠른 진입점

- 개발자 신규 합류 → [`00-overview/01-architecture.md`](./00-overview/01-architecture.md) 1~2장만 읽기
- 새 에이전트 CLI 추가 → [`30-runtime/07-agent-runtime.md`](./30-runtime/07-agent-runtime.md) §11 체크리스트
- 채팅 라우트 / SSE 흐름 변경 → [`30-runtime/09-sse-chat-pipeline.md`](./30-runtime/09-sse-chat-pipeline.md)
- 프롬프트 커스터마이즈 → [`30-runtime/08-prompt-engine.md`](./30-runtime/08-prompt-engine.md) §14 가이드
- DB 마이그레이션 추가 → [`40-data/10-persistence.md`](./40-data/10-persistence.md) §12
- 새 LLM 프로바이더 또는 미디어 모델 추가 → [`50-integration/11-byok-and-media.md`](./50-integration/11-byok-and-media.md) §14
- 보안 변경 (HMAC, iframe, 경로 검증) → [`50-integration/12-security-model.md`](./50-integration/12-security-model.md)
- 새 빌트인 스킬 추가 → [`20-content/13-skill-catalog.md`](./20-content/13-skill-catalog.md) §13
- 사이드카/IPC 경계 손대기 → [`10-structure/03-packages.md`](./10-structure/03-packages.md) sidecar-proto / sidecar 절
- 릴리스/패키징 → [`10-structure/04-tools.md`](./10-structure/04-tools.md) `tools/pack` 절

## 다이어그램 인덱스

| SVG | 문서 | 주제 |
|---|---|---|
| [01-layered-architecture](./svg/00-overview/01-layered-architecture.svg) | 00-overview/01 | Content → apps → packages → tools 4계층 |
| [02-apps-graph](./svg/10-structure/02-apps-graph.svg) | 10-structure/02 | 6개 앱의 HTTP/IPC 통신 토폴로지 |
| [03-packages-deps](./svg/10-structure/03-packages-deps.svg) | 10-structure/03 | 4-패키지 순환 없는 의존 그래프 |
| [04-tools-control-plane](./svg/10-structure/04-tools-control-plane.svg) | 10-structure/04 | tools-dev/pack/pr 입출력 |
| [05-content-catalog](./svg/20-content/05-content-catalog.svg) | 20-content/05 | 콘텐츠 → 데몬 카탈로그화 → 시스템 프롬프트 주입 |
| [06-runtime-graph](./svg/30-runtime/06-runtime-graph.svg) | 30-runtime/06 | 사용자 → 데몬 → 에이전트 → 아티팩트 통합 그래프 |
| [07-agent-pipeline](./svg/30-runtime/07-agent-pipeline.svg) | 30-runtime/07 | 16 어댑터 → 4 파서 → ChatSseEvent union |
| [08-prompt-engine](./svg/30-runtime/08-prompt-engine.svg) | 30-runtime/08 | 3-턴 패턴 + 11-레이어 시스템 프롬프트 스택 |
| [09-sse-sequence](./svg/30-runtime/09-sse-sequence.svg) | 30-runtime/09 | POST /api/chat 시퀀스 다이어그램 |
| [10-er-diagram](./svg/40-data/10-er-diagram.svg) | 40-data/10 | SQLite 12 테이블 ER 다이어그램 |
| [11-byok-proxy](./svg/50-integration/11-byok-proxy.svg) | 50-integration/11 | BYOK 프록시 라우트 + SSRF 가드 |
| [12-hmac-gate](./svg/50-integration/12-hmac-gate.svg) | 50-integration/12 | HMAC 데스크탑 인증 게이트 시퀀스 |
| [13-skill-shadowing](./svg/20-content/13-skill-shadowing.svg) | 20-content/13 | listSkills 6단계 + shadowing 패턴 |

## 분석 방법

- 루트 + 영역별 `AGENTS.md`(repo가 명시한 단일 진실 공급원)를 1차 소스로 사용
- 4개의 영역(apps/packages/tools/content)에 대해 병렬 코드 탐색을 수행한 뒤 합성
- 가능한 곳마다 `path/to/file.ts:line` 형식의 인용을 유지
- 분량 압축이 필요한 곳은 "패턴 → 대표 사례"로 추상화
