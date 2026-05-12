# 02. apps/ — 런타임 6개

`apps/`에는 6개의 독립적인 런타임이 있으며 pnpm workspace로 묶여 있습니다. 핵심은 **daemon(Express+SQLite) ↔ web(Next.js) ↔ desktop(Electron) ↔ packaged(번들 진입점)** 4개이고, `landing-page`(Astro)와 `telemetry-worker`(Cloudflare Worker)는 보조 역할을 합니다.

![apps 통신 토폴로지](../svg/10-structure/02-apps-graph.svg)

## 1. apps/daemon — 로컬 데몬과 `od` CLI

**역할**: 16개 코딩 에이전트 CLI를 자식 프로세스로 스폰·정규화하고, 프로젝트/채팅/배포/스킬/디자인시스템/MCP를 REST+SSE로 노출하는 단일 권한 프로세스. (`apps/AGENTS.md:8`)

### 1-1. package.json 핵심

```json
{
  "name": "@open-design/daemon",
  "main": "./dist/cli.js",
  "bin": { "od": "./dist/cli.js" },
  "dependencies": {
    "express": "^4.19.2",
    "better-sqlite3": "^12.9.0",
    "@open-design/contracts": "workspace:*",
    "@open-design/platform": "workspace:*",
    "@open-design/sidecar": "workspace:*",
    "@open-design/sidecar-proto": "workspace:*"
  }
}
```

### 1-2. 디렉토리 레이아웃

- `apps/daemon/src/cli.ts` — `od` CLI 진입점. 3개 서브커맨드: `od`(기본 데몬 + 브라우저), `od media …`(이미지/비디오/오디오 생성), `od mcp …`(MCP 서버 관리).
- `apps/daemon/src/server.ts` — Express 부트스트랩, 라우트 등록, SQLite 연결, 데스크탑 인증 게이트. **2,000+ 라인**으로 가장 큰 단일 파일.
- `apps/daemon/src/runtimes/defs/*.ts` — 16개 에이전트 정의(선언형).
- `apps/daemon/src/runtimes/{capabilities,detection}.ts` — capability 플래그, PATH 스캔 검출.
- `apps/daemon/src/prompts/{discovery,directions,system}.ts` — 3-턴 프롬프트 엔진.
- `apps/daemon/src/*-routes.ts` — 10여 개 도메인 라우트 모듈 (아래 참조).
- `apps/daemon/src/{skills,memory,design-systems}.ts` — 콘텐츠 카탈로그 로더.
- `apps/daemon/src/db.ts` — SQLite 스키마 (better-sqlite3, WAL).
- `apps/daemon/src/acp.ts` — Anthropic Compute Platform 어댑터.
- `apps/daemon/src/pi-rpc.ts` — Pi RPC 세션 (JSON-RPC over stdio).
- `apps/daemon/src/langfuse-bridge.ts` — 텔레메트리 브리지(→ `apps/telemetry-worker`로 릴레이).
- `apps/daemon/sidecar/` — 데몬 사이드카 진입점 (tools-dev/packaged에서 호출).
- `apps/daemon/tests/` — Vitest 테스트.

### 1-3. 라우트 모듈 카탈로그

`apps/AGENTS.md:19`가 강제하는 규칙: 새 도메인 엔드포인트는 **서버 부트스트랩이 아니라** 해당 도메인 라우트 모듈에 등록하라. `/api/health`, `/api/version` 같은 부트스트랩 수준만 `server.ts`에 둔다.

| 라우트 모듈 | 주요 엔드포인트 | 도메인 |
|---|---|---|
| `chat-routes.ts` (~34 KB) | `POST /api/chat` (SSE 스트림) | 에이전트 스폰, 프롬프트 합성, 스트림 정규화 |
| `project-routes.ts` | `GET/POST/PATCH /api/projects`, 파일 CRUD | 프로젝트 라이프사이클 |
| `live-artifact-routes.ts` | `GET /api/artifacts/:id` | 라이브 HTML 미리보기 |
| `import-export-routes.ts` | `POST /api/import/folder`, `/api/import/claude-design` | 외부 도구로부터 import |
| `deploy-routes.ts` | `POST /api/deploy` | Vercel / Cloudflare Pages 배포 |
| `routine-routes.ts` | `GET/POST /api/routines` | cron 스케줄 |
| `mcp-routes.ts` | `GET /api/mcp/config`, OAuth | MCP 서버 |
| `active-context-routes.ts` | `/api/active` | 일시적 UI 포커스 상태 |
| `connector/routes.ts` | Composio 커넥터 | 외부 도구 연동 |
| `proxy/anthropic/openai/azure/google` | `POST /api/proxy/<provider>/stream` | BYOK 폴백 (SSRF 차단) |

`server.ts`는 각 라우트를 `register<Domain>Routes(app, ctx)` 형태로 등록하고, 모든 라우트에 공유 `ServerContext`를 주입합니다.

### 1-4. 16개 에이전트 어댑터

각 에이전트는 `apps/daemon/src/runtimes/defs/*.ts`에 **선언형**으로 정의됩니다.

```typescript
// 예: apps/daemon/src/runtimes/defs/claude.ts
export const claudeAgentDef = {
  id: 'claude',
  bin: 'claude',
  buildArgs: (prompt, imagePaths, extraAllowedDirs, options) => [...],
  promptViaStdin: true,
  streamFormat: 'claude-stream-json',
  fallbackModels: [{ id: 'sonnet', label: '...' }],
  capabilityFlags: { '--add-dir': 'addDir', ... }
}
```

**스폰 어댑터 16종**: claude, codex, devin, cursor-agent, gemini, opencode, qwen, qoder, copilot, hermes(ACP), kimi(ACP), pi(RPC), kiro(ACP), kilo(ACP), mistral-vibe(ACP), deepseek.

**스트림 포맷별 정규화**: `claude-stream-json` / `codex-events` / `acp-jsonrpc` / `pi-rpc` / 일반 텍스트가 모두 `contracts`의 `ChatSseEvent` union으로 변환되어 클라이언트에는 동일하게 도착합니다.

**Windows 안전성**: 모든 어댑터는 argv 길이 `ENAMETOOLONG`을 대비해 stdin 또는 임시 프롬프트 파일 폴백을 가집니다.

### 1-5. 3-턴 프롬프트 엔진

`apps/daemon/src/prompts/discovery.ts`(~24 KB)가 강제하는 패턴(`README.md`의 "30 seconds of radios beats 30 minutes of redirects" 슬로건):

- **Turn 1**: discovery form 발행 — surface, audience, tone, brand context, scale를 라디오로 잠금.
- **Turn 2**: 사용자가 브랜드를 가지고 있으면 그 브랜드 spec 추출, 아니면 5개 큐레이션된 방향(Editorial Monocle / Modern Minimal / Tech Utility / Brutalist / Soft Warm) 중 선택. 각 방향은 결정론적 OKLch 팔레트 + 폰트 스택. (`apps/daemon/src/prompts/directions.ts`)
- **Turn 3+**: TodoWrite 계획을 라이브 카드로 스트리밍 → 빌드 → 5차원 자기 비평 → `<artifact>` 단일 emit.

`apps/daemon/src/prompts/system.ts`(~43 KB)는 위 단계마다 활성 스킬·디자인시스템·craft 섹션·메모리를 system prompt에 주입하고 최종 prompt를 조립합니다.

### 1-6. SQLite 스키마 (apps/daemon/src/db.ts)

WAL 모드, ~1,300 라인. 주요 테이블:

- `projects` — `id`, `name`, `skill_id`, `design_system_id`, `metadata_json`
- `conversations`, `messages` — 채팅 히스토리
- `preview_comments` — 미리보기에 단 코멘트
- `deployments` — Vercel/Cloudflare 배포 추적
- `routines`, `routine_runs` — 크론 스케줄과 실행 기록
- `tabs` — UI 탭 상태

인덱스: `idx_conv_project`, `idx_messages_conv`(시간 정렬).

실제 파일(HTML 아티팩트, 업로드)은 SQLite 밖 `.od/projects/<id>/files/`에 저장 — GC 안전성을 위한 분리.

### 1-7. `.od/` 디렉토리

```
.od/
├── app.sqlite              # 메타데이터
├── projects/<id>/
│   ├── files/              # HTML 아티팩트, 업로드, 스케치
│   ├── conversations/      # 대화 JSON
│   └── live-artifacts/     # 라이브 업데이트 캐시
├── memories/               # 사용자 메모리 (Markdown)
├── skills/                 # 사용자 임포트/편집 스킬 (shadowing)
├── artifacts/              # 저장된 export
└── media-config.json       # BYOK API 자격증명
```

저장 루트 오버라이드 우선순위: `OD_MEDIA_CONFIG_DIR > OD_DATA_DIR > <projectRoot>/.od`.

## 2. apps/web — Next.js 16 + React 18

**역할**: 데몬의 REST/SSE를 소비하는 React SPA. `next.config.ts`가 dev에서 `/api/*`, `/artifacts/*`, `/frames/*`를 `OD_PORT`로 rewrite합니다. (`apps/AGENTS.md:7`)

### 2-1. 디렉토리

- `apps/web/app/` (App Router)
  - `layout.tsx` — 전역 레이아웃, 테마 초기화 inline 스크립트, I18n provider
  - `[[...slug]]/page.tsx` — catch-all 라우트 (정적 export 시 generateStaticParams는 빈 배열)
  - `[[...slug]]/client-app.tsx` — 실제 SPA 로직
- `apps/web/src/`
  - `App.tsx` — 메인 셸 (`useRoute()` 기반 라우팅, 프로젝트 CRUD, 설정/메모리/펫 오버레이)
  - `components/` — EntryView, ProjectView, SettingsDialog, PetOverlay 등
  - `state/` — `config.ts`(데몬 설정 동기화), `projects.ts`, `appearance.ts`(테마)
  - `providers/registry.ts` — `fetchAppVersionInfo`, `fetchAgents`, `fetchSkills` 등 API 호출 헬퍼
  - `runtime/` — 채팅 클라이언트, 에이전트 실행 상태
  - `hooks/`, `utils/`, `i18n/`

### 2-2. next.config.ts 빌드 모드

- **dev**: `rewrites()`로 daemon 프록시.
- **static export**(`output: 'export'`): 데몬 단일 프로세스가 `out/`를 서브.
- **standalone**(`output: 'standalone'`): packaged 데스크탑에서 SSR로 사용.

### 2-3. Vitest 위치

`apps/web/tests/`. **`src/` 안에 `*.test.ts` 추가 금지** (`apps/AGENTS.md:28-32`). Playwright UI 자동화는 `e2e/ui/`로.

## 3. apps/desktop — Electron 셸

**역할**: BrowserWindow 호스트, 사이드카 IPC 클라이언트, PDF export, 폴더 import 브릿지. **웹 포트를 추측하지 않고** 사이드카 IPC로 런타임 상태를 조회해서 보고된 web URL을 연다. (`apps/AGENTS.md:9`)

### 3-1. 디렉토리

- `apps/desktop/src/main/index.ts` — `createDesktopRuntime()`, Electron BrowserWindow 생성, 데몬과 HMAC secret 등록(`registerDesktopAuthWithDaemon`), web discovery(`createWebDiscovery()` — IPC로 web 사이드카 상태 조회).
- `apps/desktop/src/main/runtime.ts` — Window open 핸들러(보안: `od://` 프로토콜 + loopback 호스트만), PDF export(`buildDesktopPdfExportInput`), 폴더 import(`pickAndImportFolder`, `validateExistingDirectory`).
- `apps/desktop/src/main/pdf-export.ts` — electron-pdf 통합.

### 3-2. 데스크탑 IPC 메시지 (sidecar-proto 정의)

- `STATUS` — `{ url, version, ready }`
- `EVAL` — DevTools에서 JS 실행
- `SCREENSHOT` — 화면 캡처 (E2E용)
- `CONSOLE` — 콘솔 로그 조회
- `CLICK` — CSS 셀렉터로 요소 클릭
- `SHUTDOWN` — 프로세스 종료
- `REGISTER_DESKTOP_AUTH` — 데스크탑이 데몬에 HMAC secret 등록 (folder import 게이트)

## 4. apps/packaged — Electron 번들 진입점

**역할**: 데몬 + 웹 + 데스크탑을 단일 번들로 묶고, `od://` 프로토콜 핸들러, 사이드카 라이프사이클을 소유. (`apps/AGENTS.md:10`)

### 4-1. 핵심 파일

- `apps/packaged/src/index.ts` — 메인 진입점, `createDesktopRuntime()` 호출.
- `apps/packaged/src/launch.ts` — 네임스페이스 경로 + Electron 경로 오버라이드.
- `apps/packaged/src/protocol.ts` — `od://` 프로토콜 등록:
  ```typescript
  protocol.handle(OD_SCHEME, async (request) =>
    handleOdRequest(request, webRuntimeUrl)
  );
  ```
  `od://app/*` → web 사이드카 HTTP 프록시.
- `apps/packaged/src/sidecars.ts` — daemon + web + desktop 3개 사이드카 부트스트랩.
- `apps/packaged/src/headless.ts` — daemon + web 헤드리스 모드 (데스크탑 제외, Linux 서버 빌드용).

### 4-2. 패키지 경로 불변

- 패키지 web은 Next.js SSR을 web 사이드카로 띄움. **데몬의 `OD_RESOURCE_ROOT`에 Next.js 출력을 두지 않는다**.
- `OD_RESOURCE_ROOT`는 데몬의 비-Next 읽기 전용 리소스(`skills/`, `design-systems/`, `frames/`) 전용.
- 데이터/로그/런타임/캐시 경로는 **포트가 아닌 namespace로** 스코프 — 포트는 일시적 전송 디테일.
- 데몬↔웹 패키지 트래픽은 여전히 HTTP origin/port를 쓰는데, 이는 Next.js SSR 프록시가 HTTP origin을 가정하기 때문. Unix socket 전환은 Next 내부 패치가 필요해 보류.

## 5. apps/landing-page — Astro

`apps/landing-page/astro.config.ts`. `astro dev --host 127.0.0.1 --port 17574`로 개발. 빌드는 정적 HTML/CSS/JS. workspace 의존성 없음 — 완전 독립.

## 6. apps/telemetry-worker — Cloudflare Worker

`apps/telemetry-worker/src/index.ts` (~200 라인). `POST /api/langfuse`로 데몬의 langfuse 텔레메트리를 받아 Langfuse upstream에 인증된 요청으로 릴레이.

기능: body size/JSON 스키마 검증, Durable Object 기반 rate limit (client + IP), allowed event type 화이트리스트(`trace-create`, `span-create`, `generation-create` 등). `GET /health`는 설정 상태 보고.

## 7. 패키지 의존 그래프

```
apps/web         → contracts, platform, sidecar, sidecar-proto
apps/daemon      → contracts, platform, sidecar, sidecar-proto
apps/desktop     → platform, sidecar, sidecar-proto
apps/packaged    → daemon, desktop, web, platform, sidecar, sidecar-proto
apps/landing-page → (없음)
apps/telemetry-worker → (없음)
```

## 8. 일관된 패턴

1. **선언형 에이전트 정의** — 새 CLI 추가는 `apps/daemon/src/runtimes/defs/<id>.ts` 하나만 작성하면 16개와 동일한 인터페이스에 묶임.
2. **사이드카 IPC를 통한 데스크탑↔웹 디스커버리** — 데스크탑이 포트를 추측하지 않음.
3. **HMAC 데스크탑 인증 게이트** — folder import 등 위험한 호출에 60초 TTL 서명 토큰.
4. **`OD_*` 환경 변수만 사용** — `NEXT_PORT` 같은 비표준 변수 금지.
5. **라우트 모듈 자동 등록** — `server.ts`의 단일 부트스트랩 함수가 모든 도메인 라우트를 한 번에 wire.
