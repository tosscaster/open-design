# 01. 모노레포 아키텍처 개요

## 1. 한 문장 요약

Open Design은 **로컬 데몬이 설치된 코딩 에이전트 CLI를 자동 감지해서, 31개 스킬 × 129개 디자인 시스템을 컨텍스트로 묶어 디자인 아티팩트를 생성하고, Next.js 웹/Electron 데스크탑에서 샌드박스 iframe으로 스트리밍하는** 오픈소스 디자인 워크벤치입니다.

Claude Design(Anthropic, 2026-04-17 출시, 폐쇄형)의 오픈 대안을 목표로 합니다. (`README.md:35`)

## 2. 레이어 다이어그램

![4계층 아키텍처](../svg/00-overview/01-layered-architecture.svg)

```
┌──────────────────────────────────────────────────────────────────┐
│ 콘텐츠 자산 (Content Layer)                                       │
│   skills/  design-systems/  design-templates/                    │
│   craft/   prompt-templates/  specs/                             │
└─────────────────────────────┬────────────────────────────────────┘
                              │ 데몬이 파일시스템 스캔
┌─────────────────────────────▼────────────────────────────────────┐
│ apps/  (런타임 6개)                                              │
│  ┌──────────────┐  HTTP/SSE  ┌──────────────┐                   │
│  │   apps/web   │ ◄────────► │ apps/daemon  │ ─► .od/app.sqlite │
│  │ Next.js 16   │            │ Express+SQLi │ ─► 코딩 에이전트   │
│  └──────────────┘            └──────┬───────┘    CLI 스폰         │
│         ▲                           │ sidecar IPC                │
│         │ web URL                   │                            │
│  ┌──────┴───────┐            ┌──────▼───────┐                   │
│  │ apps/desktop │            │ apps/packaged│ (Electron 번들)   │
│  │  (Electron)  │ ◄────────► │ od:// 핸들러 │                   │
│  └──────────────┘  sidecar   └──────────────┘                   │
│                                                                  │
│  apps/landing-page (Astro)   apps/telemetry-worker (Cloudflare)  │
└─────────────────────────────┬────────────────────────────────────┘
                              │ depends
┌─────────────────────────────▼────────────────────────────────────┐
│ packages/  (4계층 순수 TS)                                       │
│   contracts       ─ web/daemon DTO/SSE 이벤트/에러 코드          │
│   sidecar-proto   ─ stamp 5필드, IPC 메시지, 상태 형태           │
│   sidecar         ─ 범용 IPC 전송, 경로 해석, JSON 헬퍼          │
│   platform        ─ OS 프로세스, stamp 직렬화, toolchain 탐색    │
└─────────────────────────────┬────────────────────────────────────┘
                              │ 호출
┌─────────────────────────────▼────────────────────────────────────┐
│ tools/  (컨트롤 플레인)                                          │
│   tools-dev   ─ 로컬 라이프사이클 (start/stop/run/status/logs)   │
│   tools-pack  ─ Mac/Win/Linux 패키지 빌드/설치/시작/정리         │
│   tools-pr    ─ 메인테이너 PR-duty (read-only `gh` 래퍼)         │
└──────────────────────────────────────────────────────────────────┘
```

## 3. 핵심 설계 원칙 (AGENTS.md 기반)

루트 [`AGENTS.md`](../AGENTS.md)와 영역별 `AGENTS.md`가 강제하는 **불변(invariant)** 들:

### 3-1. 단일 라이프사이클 진입점
- 모든 로컬 개발은 `pnpm tools-dev`만 사용. 루트에 `pnpm dev`, `pnpm start`, `pnpm daemon` 같은 별칭을 **금지**.
- 포트는 `--daemon-port`, `--web-port` 플래그로만 지정. 내부 env는 `OD_PORT`, `OD_WEB_PORT`. `NEXT_PORT` 사용 금지.

### 3-2. 앱 경계 — 데몬과 웹은 HTTP로만 만난다
- `apps/web/**`는 `apps/daemon/src/**`를 import할 수 없다.
- 모든 web↔daemon 통합은 **`packages/contracts`의 DTO**와 HTTP API를 통한다.
- 교차 앱 정합성 검사는 `e2e/tests/`에 둔다 (`AGENTS.md` 참조).

### 3-3. 사이드카 다섯 필드
- 모든 사이드카 프로세스는 정확히 5개 필드 stamp: `app`, `mode`, `namespace`, `ipc`, `source`.
- `--od-stamp-*` 인자를 수동 조립하지 말고 `@open-design/platform`의 `createProcessStampArgs()`를 통해야 한다.

### 3-4. 패키지 경계
- `contracts`는 **순수 TS 타입만** — Next.js/Express/Node FS/process/SQLite/브라우저 API 의존 금지.
- `sidecar-proto`는 Open Design 비즈니스 프로토콜 — IPC 메시지 스키마, 5-stamp 디스크립터.
- `sidecar`는 **범용** 런타임 프리미티브 — OD 앱 키를 하드코딩하지 않는다.
- `platform`은 **범용** OS 프로세스 프리미티브 — `--od-stamp-*` 이름을 하드코딩하지 않고 `ProcessStampContract`를 인자로 받는다.

### 3-5. 런타임 데이터 경로
기본 저장 루트: `<projectRoot>/.od/` (override: `OD_DATA_DIR > OD_MEDIA_CONFIG_DIR`)
- SQLite: `.od/app.sqlite`
- 에이전트 CWD: `.od/projects/<id>/`
- 아티팩트: `.od/artifacts/`
- 미디어 자격증명: `.od/media-config.json`

POSIX IPC 소켓 경로(고정): `/tmp/open-design/ipc/<namespace>/<app>.sock`.

### 3-6. 테스트 레이아웃
- 패키지/앱/툴 테스트는 **`src/`의 형제 `tests/`**에 둔다. `src/` 내부에는 `*.test.ts` 추가 금지.
- Playwright UI 자동화는 **`e2e/ui/`** 전용. 앱 디렉토리에 두지 않는다.

### 3-7. 커밋 정책
- Git 커밋에 `Co-authored-by` 트레일러를 **포함하지 않는다** — repo가 명시적으로 차단.

## 4. 4대 외부 영감 (README.md:45)

Open Design은 4개의 오픈소스 어깨 위에 서있습니다:

1. **`alchaincyf/huashu-design`** — Junior-Designer 워크플로우, 5단계 브랜드 자산 프로토콜, anti-AI-slop 체크리스트, 5차원 자기 비평. → `apps/daemon/src/prompts/discovery.ts`로 증류.
2. **`op7418/guizang-ppt-skill`** — 매거진 풍 데크. `skills/guizang-ppt/`에 LICENSE 보존하며 그대로 번들.
3. **`OpenCoworkAI/open-codesign`** — UX 북극성, 가장 가까운 동료. 스트리밍 아티팩트 루프, 샌드박스 iframe 미리보기, 5포맷 export.
4. **`multica-ai/multica`** — 데몬-앤-런타임 아키텍처. PATH 스캔 에이전트 검출.

## 5. 컴포넌트 책임 매트릭스

| 책임 | 위치 |
|---|---|
| 코딩 에이전트 CLI 스폰 + 프롬프트 조립 + 스트림 정규화 | `apps/daemon/src/runtimes/`, `apps/daemon/src/prompts/` |
| HTTP/SSE 라우트, 라이브 아티팩트, 배포, 루틴, MCP, 미디어 | `apps/daemon/src/*-routes.ts` (10여 개) |
| 메타데이터 영속화(프로젝트/대화/메시지/탭/루틴/배포) | `apps/daemon/src/db.ts` (SQLite WAL) |
| React 셸, 라우팅, 상태, 미리보기 iframe | `apps/web/src/App.tsx`와 `components/`, `state/` |
| 데스크탑 셸, BrowserWindow, PDF export, 폴더 import | `apps/desktop/src/main/` |
| `od://` 프로토콜, 데몬+웹+데스크탑 통합 부트스트랩 | `apps/packaged/src/` |
| web/daemon 간 API 타입/에러/SSE 이벤트 | `packages/contracts/src/` |
| 사이드카 IPC 메시지 스키마, 5-stamp 디스크립터 | `packages/sidecar-proto/src/index.ts` |
| 범용 IPC 전송, 경로 해석, JSON 파일 헬퍼 | `packages/sidecar/src/index.ts` |
| 프로세스 stamp 직렬화, ps/PowerShell 스캔, toolchain 탐색 | `packages/platform/src/index.ts` |
| 로컬 라이프사이클(start/stop/run/status/logs/inspect/check) | `tools/dev/src/index.ts` |
| 패키지 빌드/설치/시작/정리 (Mac/Win/Linux) | `tools/pack/src/{mac,win,linux}/` |
| PR 트리아주 분석 (lane 도출, 금지 표면, 팩트 태그) | `tools/pr/src/{lane,tags,classify}.ts` |
| 스킬 카탈로그 정규화 / shadowing / derived examples | `apps/daemon/src/skills.ts` |
| 31개 스킬 콘텐츠, frontmatter 메타데이터 | `skills/<id>/SKILL.md` |
| 150여 개 브랜드별 디자인 가이드 | `design-systems/<brand>/DESIGN.md` |
| 100여 개 렌더링 템플릿 (decks/prototypes) | `design-templates/<id>/` |
| 12개 보편 craft 규칙 (typography, anti-ai-slop, …) | `craft/<rule>.md` |

## 6. 배포 형태 4종

같은 코드베이스가 4가지 형태로 배포됩니다.

1. **로컬 개발**: `pnpm tools-dev` → 데몬(7456) + 웹(Next dev) + 옵션 데스크탑
2. **로컬 단일 프로세스 CLI**: `od` 바이너리 — 데몬이 정적 export된 웹을 같은 포트에서 직접 서브
3. **Vercel 웹 레이어**: `apps/web`의 standalone 빌드 + 별도 호스팅된 데몬 (Vercel은 데몬 미배포)
4. **패키지 데스크탑 앱**: macOS Apple Silicon / Windows x64 / Linux AppImage — `tools-pack`이 electron-builder로 데몬+웹+데스크탑을 단일 번들로 묶음
