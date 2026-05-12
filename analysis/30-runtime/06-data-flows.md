# 06. 주요 데이터 흐름과 시퀀스

이 문서는 Open Design의 **핵심 런타임 시나리오 6개**를 시퀀스 형태로 정리합니다. 앞 문서들의 컴포넌트 지식을 전제로 합니다.

![통합 호출 그래프](../svg/30-runtime/06-runtime-graph.svg)

## 1. `pnpm tools-dev start` — 로컬 라이프사이클 부트스트랩

```
사용자
  │ pnpm tools-dev start --namespace default
  ▼
tools/dev/src/index.ts (cac CLI)
  │ 1. config 해석: namespaceRoot = .tmp/tools-dev/default
  │    ipcPath = /tmp/open-design/ipc/default/{daemon,web,desktop}.sock
  ▼
spawnDaemonRuntime()
  │ 2. createAppStamp() → ["--od-stamp-app=daemon",
  │                       "--od-stamp-mode=runtime",
  │                       "--od-stamp-namespace=default",
  │                       "--od-stamp-ipc=...",
  │                       "--od-stamp-source=tools-dev"]
  │ 3. spawnBackgroundProcess({
  │      command: node,
  │      args: [tsx, apps/daemon/sidecar/, ...stampArgs],
  │      detached: true,
  │      env: { OD_PORT: 7456, OD_WEB_PORT: 0, OD_SIDECAR_* }
  │    })
  ▼
apps/daemon/sidecar/ (별도 프로세스)
  │ 4. bootstrapSidecarRuntime() → SidecarRuntimeContext
  │ 5. createJsonIpcServer() listening on /tmp/open-design/ipc/default/daemon.sock
  │ 6. startServer() → Express on 127.0.0.1:7456
  │ 7. SQLite open: .od/app.sqlite (WAL)
  │ 8. registerXxxRoutes() — 10여 개 도메인 라우트
  │ 9. writeJsonFile(current.json) — 자식 포인터
  ▼
spawnWebRuntime()
  │ 10. await requestJsonIpc(daemon.sock, {type: STATUS}) → url
  │ 11. spawnBackgroundProcess(web sidecar, env: OD_PORT=daemonPort)
  ▼
apps/web/sidecar (Next dev 서버)
  │ 12. listen on free port (또는 --web-port)
  │ 13. createJsonIpcServer() on web.sock
  │ 14. /api/*, /artifacts/*, /frames/* → http://127.0.0.1:OD_PORT 로 rewrite
  ▼
tools-dev: 사용자에게 web URL 출력
```

**불변(invariant)**:
- 데몬→웹 통신은 HTTP origin/port. Unix socket 전환은 Next.js SSR 프록시 가정 때문에 보류.
- 데이터/로그/런타임 경로는 namespace로만 스코프 — 포트가 경로에 끼지 않는다.
- 데스크탑은 포트를 추측하지 않고 `web.sock`의 STATUS로만 URL을 얻는다.

## 2. 채팅 요청 — Turn 1 discovery form

```
브라우저 (apps/web/src/App.tsx)
  │ 1. 사용자가 EntryView에 "make me a magazine-style pitch deck"
  │    skill: web-prototype, design-system: linear (선택)
  │ POST /api/chat { agentId, message, projectId?, skillIds, designSystemId }
  ▼
apps/web → rewrite → apps/daemon (Express)
  ▼
chat-routes.ts: handleChatRequest()
  │ 2. SSE 응답 시작 (Content-Type: text/event-stream)
  │ 3. 프로젝트 없으면 새로 생성 (SQLite INSERT INTO projects)
  │ 4. discovery 검사: 첫 턴인가? → composeDiscoveryPrompt()
  ▼
prompts/discovery.ts
  │ 5. discovery form spec emit:
  │    <question-form>
  │      surface: [website|landing|deck|mobile-app|...]
  │      audience: [...]
  │      tone: [...]
  │      brand_context: [...]
  │      scale: [...]
  │    </question-form>
  │ 6. event: start → event: agent {kind: question_form, payload} → event: end
  ▼
브라우저
  │ 7. <QuestionForm/> 렌더 (라디오 5개)
  │ 8. 사용자 응답 → POST /api/chat { previousAnswers: {surface: ..., ...} }
```

## 3. 채팅 요청 — Turn 2+ 에이전트 스폰 + 스트림

```
chat-routes.ts: handleChatRequest() (재진입)
  │ 1. discovery 완료 + brand 미정 → directions form (5개 방향)
  │    또는 brand 정함 → 바로 build
  │ 2. composeSystemPrompt() — apps/daemon/src/prompts/system.ts
  │    a. 활성 스킬의 SKILL.md 본문 주입
  │    b. design-system DESIGN.md 주입 (디렉토리 풀-텍스트)
  │    c. craft.requires 섹션들 주입 (typography, anti-ai-slop, …)
  │    d. 사용자 memory 주입
  │    e. discovery 답변 주입
  ▼
runtimes/defs/<agentId>.ts (예: claude.ts)
  │ 3. buildArgs(prompt, imagePaths, extraAllowedDirs, options)
  │    → ["--output-format=stream-json", "--add-dir=...", ...]
  │ 4. promptViaStdin: true → stdin으로 prompt 전송
  ▼
child_process.spawn(claudeBin, args, { cwd: .od/projects/<id> })
  │ 5. agent 실행 (실제 Read/Write/Bash/WebFetch 가능)
  ▼
agent stdout (claude-stream-json)
  │ {"type":"assistant","content":[{"type":"text","text":"..."}]}
  │ {"type":"assistant","content":[{"type":"tool_use","name":"TodoWrite","input":{...}}]}
  │ {"type":"user","content":[{"type":"tool_result","tool_use_id":"...","content":...}]}
  ▼
runtimes/<agent>/parser.ts: 정규화
  │ 6. claude-stream-json → ChatSseEvent (contracts/sse/chat.ts)
  │    agent {kind: status, payload: {...}}
  │    agent {kind: text_delta, payload: "..."}
  │    agent {kind: thinking_delta, payload: "..."}
  │    agent {kind: tool_use, payload: {name, input}}
  │    agent {kind: tool_result, payload: {tool_use_id, content}}
  │    agent {kind: usage, payload: {inputTokens, outputTokens}}
  ▼
SSE 스트림 → 브라우저
  │ 7. <TodoCard/> 실시간 업데이트 (in_progress → completed)
  │ 8. <ArtifactPreview/> srcdoc iframe 렌더
  ▼
브라우저: 사용자가 mid-flight 인터럽트 가능
  │ POST /api/chat/cancel → daemon이 child process SIGTERM → 5초 → SIGKILL
  │ (platform.stopProcesses)
```

**아티팩트 emit 규약**: agent는 `<artifact>` 태그 한 번 emit → 데몬이 `.od/projects/<id>/files/` 또는 `live-artifacts/`에 기록 → `GET /api/artifacts/:id`로 미리보기.

## 4. 사이드카 IPC — 데스크탑 inspect

```
tools-dev inspect desktop screenshot --path /tmp/od.png
  │
  ▼
tools/dev/src/index.ts: inspectDesktop()
  │ 1. resolveDesktopIpcPath(config) → /tmp/open-design/ipc/default/desktop.sock
  │ 2. requestJsonIpc(socketPath, {
  │      type: "SCREENSHOT",
  │      input: { path: "/tmp/od.png" }
  │    }, { timeoutMs: 5000 })
  ▼
Unix Domain Socket
  │ {"type":"SCREENSHOT","input":{"path":"/tmp/od.png"}}\n
  ▼
apps/desktop/src/main/index.ts: createJsonIpcServer handler
  │ 3. normalizeDesktopSidecarMessage(payload) (sidecar-proto)
  │ 4. switch (type) { case SCREENSHOT: await window.webContents.capturePage(...) }
  │ 5. fs.writeFile("/tmp/od.png", buffer)
  ▼
응답: {"ok":true,"result":{"path":"/tmp/od.png","width":1440,"height":900}}\n
  ▼
tools-dev: 사용자에게 JSON/텍스트 출력
```

다른 메시지(EVAL, CONSOLE, CLICK, SHUTDOWN, STATUS)도 동일 패턴 — sidecar-proto의 메시지 타입에 따라 분기합니다.

## 5. 데스크탑 폴더 import — HMAC 인증 게이트

폴더 import는 데몬의 파일시스템에 임의 경로를 노출할 수 있어 위험합니다. 따라서 데스크탑↔데몬 HMAC 시그니처로 보호됩니다.

```
부팅 시 (한 번)
  │
apps/packaged → desktop runtime 시작
  │ 1. registerDesktopAuthWithDaemon():
  │    HMAC secret = randomBytes(32)
  │    requestJsonIpc(daemon.sock, {
  │      type: "REGISTER_DESKTOP_AUTH",
  │      input: { secret }
  │    })
  ▼
apps/daemon: chat-routes 또는 import-routes에 저장
  │ 2. desktopAuthSecret ← secret
  │ 3. DaemonStatusSnapshot.desktopAuthGateActive = true


사용자가 폴더 import 버튼 클릭
  │
apps/desktop: pickAndImportFolder()
  │ 4. dialog.showOpenDialog({properties: ["openDirectory"]})
  │ 5. validateExistingDirectory(folderPath)
  │ 6. token = HMAC-SHA256(desktopAuthSecret, JSON({nonce, path, expires: now+60s}))
  ▼
POST /api/import/folder
  Authorization: OD-Desktop <token>
  Body: { path, nonce, expires }
  ▼
apps/daemon: import-export-routes.ts
  │ 7. verifyDesktopImportToken(req): HMAC 검증 + expires 검증 + nonce 미사용 검증
  │ 8. 성공: 파일시스템에서 project 생성
  │    실패: 401 → 데스크탑 인증 게이트가 동작
```

## 6. 패키지 부트스트랩 — `apps/packaged` 단일 번들

```
사용자: Open Design.app 실행 (macOS)
  │
Electron 메인 프로세스 진입
  │
apps/packaged/src/index.ts: createDesktopRuntime()
  │ 1. paths = resolvePackagedPaths({appPath, namespace: "default"})
  │ 2. startPackagedSidecars(runtime, paths, {
  │      appVersion, daemonCliEntry, webSidecarEntry,
  │      requireDesktopAuth: true
  │    })
  ▼
sidecars.ts: 3개 사이드카 동시 부트스트랩
  │ 3. spawn daemon sidecar (apps/daemon/sidecar/ + stamp args)
  │    cwd = paths.namespaceRoot
  │    env: OD_PORT=0 (자유 포트), OD_RESOURCE_ROOT=resources/
  │ 4. await daemon ready (waitForHttpOk)
  │ 5. spawn web sidecar (Next standalone server)
  │    env: OD_PORT=daemonStatus.url의 포트
  │ 6. await web ready
  │ 7. createDesktopRuntime (Electron BrowserWindow)
  │    BrowserWindow.loadURL("od://app/")
  ▼
apps/packaged/src/protocol.ts: protocol.handle("od", handler)
  │ 8. od://app/foo → handleOdRequest()
  │    → fetch http://127.0.0.1:<webPort>/foo 프록시
  │    (loopback만 허용, 그 외 도메인은 거부)
  ▼
사용자에게 BrowserWindow 표시
```

**경로 격리**: 모든 데이터/로그/런타임/캐시는 `paths.namespaceRoot` 아래로 — 포트 정보가 경로에 들어가지 않으므로 재시작 시 포트가 바뀌어도 동일 데이터에 접근.

## 7. PR 트리아주 — `tools-pr list`

```
사용자: pnpm tools-pr list --bucket=merge-ready --json
  │
tools/pr/src/list.ts: handleList()
  │ 1. ghPrList({limit: 1000, includeDrafts}) → gh pr list --json ...
  │ 2. 병렬 enrichment (per-PR):
  │    a. ghPrView(num) → files, reviews, statuses
  │    b. deriveLane(files) → "skill" | "design-system" | "contract" | …
  │    c. deriveForbidden(files) → [restores-apps/nextjs?, restores-packages/shared?]
  │    d. derivePrBucket(...) →
  │       "merge-ready" | "approved-blocked" | "changes-requested" |
  │       "new" | "stale" | "needs-rebase" | "draft"
  │ 3. tag detectors (tags.ts) → 15+ 팩트 태그
  │ 4. 필터링 (bucket/lane/author)
  │ 5. JSON 또는 표 출력
```

**핵심 가치**: `tools-pr`는 read-only — 어떤 호출도 mutate 하지 않습니다. 메인테이너가 출력을 보고 직접 `gh pr review --approve`, `gh pr comment` 등을 실행합니다.

## 8. 통합 시야: 전체 호출 그래프

```
사용자
  │
  ▼
  ┌─ pnpm tools-dev (개발) ─┐
  │                          │
  │                          │ stamp + spawn
  │                          ▼
  │           ┌───────────────────────┐
  │           │ apps/daemon (Express) │ ◄────── apps/web (Next.js)
  │           │   SQLite, 16 agents   │  HTTP   │ rewrites /api/*
  │           └─────────┬─────────────┘         └──────┬─────────┘
  │                     │ spawn child                  │
  │                     ▼                              │
  │            child_process (claude/codex/...)        │
  │                     │ stream-json                  │
  │                     ▼                              │
  │              SSE → /api/chat ──────────────────────┘
  │
  │           apps/desktop (Electron)
  │              │ sidecar IPC ──► daemon, web
  │              └─ od:// → web HTTP 프록시
  │
  ├─ pnpm tools-pack (패키지) ─►  Mac/Win/Linux 번들
  │                                  └─ apps/packaged/src/sidecars.ts
  │
  └─ pnpm tools-pr (메인테이너) ─► gh API (read-only)
```

## 9. 요약된 불변(invariant) 목록

1. 데몬 외 어떤 앱도 SQLite에 직접 쓰지 않는다.
2. 데스크탑은 web 포트를 추측하지 않고 sidecar IPC STATUS로만 알아낸다.
3. 데이터/로그/런타임 경로는 **namespace**로만 스코프 — 포트는 일시적.
4. 사이드카 stamp는 **정확히 5필드** (app, mode, namespace, ipc, source).
5. 모든 외부 노출 API는 `packages/contracts`의 DTO를 통과한다.
6. `apps/web`은 `apps/daemon/src/**`를 import하지 않는다.
7. 코딩 에이전트 CLI 출력은 모두 `ChatSseEvent` union으로 정규화된다.
8. 위험한 데몬 API(폴더 import 등)는 HMAC 데스크탑 인증 게이트로 보호된다.
9. `tools-pr`는 read-only — mutate gh 호출이 없다.
10. 콘텐츠 자산(skills/design-systems/design-templates/craft)은 데몬이 부팅 시 frontmatter를 정규화·shadowing하여 카탈로그화한다.
