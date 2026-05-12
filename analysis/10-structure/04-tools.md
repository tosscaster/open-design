# 04. tools/ — 3개 컨트롤 플레인

`tools/`는 모노레포의 **개발·패키징·메인테이너 관리** 자동화를 담당하는 3개의 컨트롤 플레인입니다. 모두 `cac` 기반 CLI이며 `pnpm tools-<name>`으로 진입합니다.

![tools 컨트롤 플레인](../svg/10-structure/04-tools-control-plane.svg)

## 1. tools/dev — 로컬 라이프사이클 컨트롤 플레인

**역할**: 데몬·웹·데스크탑 사이드카의 시작/중지/관찰/검사를 통합한 **유일한** 개발 진입점. 루트에 `pnpm dev`나 `pnpm start` 별칭을 두지 않는다 — 모든 흐름이 여기를 통과해야 stamp/네임스페이스/로그 경로가 일관된다. (`AGENTS.md:80`)

### 1-1. package.json

- name: `@open-design/tools-dev` (v0.6.0)
- bin: `./bin/tools-dev.mjs`
- deps: `@open-design/platform`, `@open-design/sidecar`, `@open-design/sidecar-proto` (workspace), `cac@6.7.14`

### 1-2. 디렉토리

```
tools/dev/src/
├── index.ts              # CLI 진입점, 8개 서브명령
├── config.ts             # 런타임 경로 해석, 앱 설정 빌드
├── sidecar-client.ts     # 데몬/웹/데스크탑 IPC 호출
├── diagnostics.ts        # 로그 에러 패턴 감지
└── desktop-auth-gate.ts  # 데스크탑 보안: 데몬 재시작 조정
```

### 1-3. 서브명령

| 명령 | 대상 | 설명 |
|---|---|---|
| `start [app]` | daemon \| web \| desktop | 백그라운드로 시작. 앱 미지정 시 전부 |
| `run [app]` | daemon \| web | 포그라운드 실행 (Ctrl+C로 중단). Playwright webServer 흐름용 |
| `stop [app]` | … | 우아한 종료(IPC SHUTDOWN) → 강제 종료(fallback) |
| `restart [app]` | … | stop → start |
| `status [app]` | … | `{ state, pid, url, windowVisible }` 스냅샷 |
| `logs [app]` | … | 200라인 테일 + 진단 (`diagnostics.ts` 패턴 매칭) |
| `inspect <app> [target]` | … | 데몬/웹: status. 데스크탑: status/eval/screenshot/console/click |
| `check [app]` | … | 통합 진단: 상태 + 로그 + 에러 감지 |

### 1-4. 주요 플래그

- `--namespace <name>` — 런타임 격리 키 (기본: `default`)
- `--daemon-port <port>` — 데몬 포트 고정 (충돌 시 즉시 실패)
- `--web-port <port>` — 웹 포트 고정
- `--tools-dev-root <path>` — tools-dev 런타임 루트 재정의
- `--json` — JSON 출력
- `--prod` — 프로덕션 빌드 사용 (web)
- 데스크탑 inspect 전용: `--expr <js>`, `--path <file>`, `--selector <css>`, `--timeout <s>`

### 1-5. 사이드카 스폰 흐름

```typescript
// tools/dev/src/index.ts (요약)
async function spawnSidecarRuntime(request): Promise<{ pid: number }> {
  const { args: stampArgs, env } = createAppStamp(config, appName);
  const spawned = await spawnBackgroundProcess({
    args: [tsxCliPath, sidecarEntryPath, ...stampArgs],
    command: process.execPath,
    detached: true,
    env: { ...process.env, ...launchEnv, ...requestEnv },
    logFd: logHandle.fd,
  });
  return { pid: spawned.pid };
}
```

- `createAppStamp`가 stamp 인자(앱 ID, IPC 경로, 네임스페이스, 소스)를 자동 조립 — 사람이 `--od-stamp-*`를 손으로 쓰지 않는다.
- `detached: true`로 부모와 독립.
- 로그는 파일 디스크립터로 스트리밍 → `.tmp/tools-dev/<namespace>/<app>/latest.log`.

### 1-6. 데스크탑 IPC 호출

```typescript
async function inspectDesktop(config, target, options) {
  switch (target) {
    case "status":
      return await inspectDesktopRuntime(runtimeLookup(config), 1000);
    case "eval":
      return await requestJsonIpc<DesktopEvalResult>(
        config.apps.desktop.ipcPath,
        { input: { expression: options.expr }, type: SIDECAR_MESSAGES.EVAL },
        { timeoutMs }
      );
    // SCREENSHOT, CONSOLE, CLICK 동일 패턴
  }
}
```

`sidecar-client.ts:resolveDesktopIpcPath`가 IPC 소켓 경로 계산, `requestJsonIpc`(sidecar 패키지)가 JSON 메시지 왕복.

### 1-7. 포트 환경 변수

```typescript
async function spawnDaemonRuntime(config, options) {
  const daemonPort = parsePortOption(options.daemonPort, "--daemon-port");
  return await spawnSidecarRuntime({
    env: {
      [SIDECAR_ENV.DAEMON_PORT]: String(daemonPort ?? 0),  // OD_PORT
      [SIDECAR_ENV.WEB_PORT]: String(webPort),             // OD_WEB_PORT
    },
  });
}
```

- `SIDECAR_ENV.DAEMON_PORT = "OD_PORT"`, `SIDECAR_ENV.WEB_PORT = "OD_WEB_PORT"` (sidecar-proto)
- `spawnWebRuntime`은 데몬 status 조회 후 데몬 포트를 web 환경으로 전달
- 0이면 OS가 자유 포트 할당

## 2. tools/pack — 패키지 빌드·라이프사이클

**역할**: Mac/Win/Linux 데스크탑 패키지를 electron-builder 위에서 빌드하고, 설치·시작·로그·정리·언인스톨까지 OS별 분기를 한 곳에 인코딩. macOS 베타 릴리스 아티팩트 준비.

### 2-1. package.json

- name: `@open-design/tools-pack` (v0.6.0)
- bin: `./bin/tools-pack.mjs`
- deps: `electron-builder@26.8.1`, `@electron/rebuild@4.0.4`, `@electron/notarize@3.1.1`, `cac`, workspace 사이드카 패키지들

### 2-2. 디렉토리

```
tools/pack/src/
├── index.ts                      # CLI 진입점
├── config.ts                     # 플랫폼/경로 해석
├── cache.ts, lock.ts             # 빌드 캐시 + 잠금
├── resources.ts                  # 리소스 복사
├── package-source-hash.ts        # workspace 해시
├── workspace-build.ts            # 사전 빌드 체크
├── mac-prebundle.ts              # Mac pre-build
├── mac/
│   ├── build.ts                  # 7단계 packMac 메인 파이프라인
│   ├── builder.ts                # electron-builder 호출
│   ├── app-config.ts             # 패키지 앱 설정 생성
│   ├── lifecycle.ts              # install/start/stop/uninstall
│   └── {artifacts,commands,constants,fs,manifest,paths,report,types,workspace,app}.ts
├── win/
│   ├── build.ts, builder.ts, lifecycle.ts
│   ├── custom-installer.ts       # NSIS 커스텀 로직
│   ├── registry.ts               # Windows 레지스트리
│   ├── identity.ts               # 코드 서명
│   └── {constants,fs,manifest,nsis,paths,report,types}.ts
└── linux.ts                      # Linux 명령 일체형
```

### 2-3. 플랫폼별 서브명령

**Mac (`tools-pack mac <action>`)**
- `build` — `--to all|app|dmg|zip`, `--signed`, `--mac-compression`
- `install` — DMG에서 `/Applications`로 설치
- `start`, `stop`, `logs`, `inspect`
- `uninstall` — `/Applications`에서 제거
- `cleanup` — 로컬 빌드 네임스페이스 정리

**Windows (`tools-pack win <action>`)**
- 위와 동일 + `list`(설치된 네임스페이스), `reset`(모든 네임스페이스 정리)
- `--remove-data`, `--remove-logs`, `--remove-product-user-data`, `--remove-sidecars` 플래그

**Linux (`tools-pack linux <action>`)**
- 기본 명령 동일
- `--containerized` — Docker `electronuserland/builder`에서 빌드
- `--headless` — Electron 없는 서버 버전 (CI/원격 호스트)

### 2-4. 공통 플래그

- `--namespace <name>`, `--dir <path>`, `--cache-dir <path>`
- `--app-version <version>` — 패키지 버전 오버라이드
- `--portable` — 로컬 tools-pack 루트를 앱에 베이크하지 않음
- `--signed` (Mac) — 서명/공증 활성
- `--to <target>` — 빌드 산출물 선택 (플랫폼별)
- `--json`

### 2-5. Mac 빌드 파이프라인 (`mac/build.ts`)

```typescript
export async function packMac(config): Promise<MacPackResult> {
  const targets = resolveElectronBuilderTargets(config.to);

  // Phase 1: workspace 빌드 사전 검증
  await runPhase("workspace-build", async () => {
    await ensureMacWorkspaceBuild(config, cache);
  });

  // Phase 2-5: 앱 구성, workspace 아카이브, 리소스 복사
  const tarballs = await runPhase("workspace-tarballs", () => collectWorkspaceTarballs(...));
  await runPhase("assembled-app", () => writeAssembledApp(...));

  // Phase 6: electron-builder
  await runPhase("electron-builder", () => runElectronBuilder(config, paths, targets));

  // Phase 7: 산출물 정리, 크기 리포트
  const artifacts = await runPhase("artifacts", () => finalizeMacArtifacts(...));
}
```

각 단계 타이밍이 리포트에 기록됩니다.

**Target 매핑**:
```typescript
function resolveElectronBuilderTargets(to: MacBuildOutput): ElectronBuilderTarget[] {
  switch (to) {
    case "app": return ["dir"];
    case "dmg": return ["dir", "dmg"];
    case "zip": return ["dir", "zip"];
    case "all": return ["dir", "dmg", "zip"];
  }
}
```

### 2-6. macOS 서명/공증

```typescript
const hookConfig = {
  macAdhocBundleSign: !config.signed,      // --signed=false → ad-hoc
  resourceName: WEB_STANDALONE_RESOURCE_NAME,
  standaloneSourceRoot: join(webRoot, ".next", "standalone"),
};
```

`--signed` 플래그가 공증 경로 제어. 미설정 시 ad-hoc 서명(로컬 개발/CI에서 정식 서명 없이도 작동).

### 2-7. Linux 컨테이너 빌드

```typescript
const INTERNAL_PACKAGES = [
  { directory: "packages/contracts", name: "@open-design/contracts" },
  { directory: "packages/sidecar-proto", name: "@open-design/sidecar-proto" },
  // ...
];
```

Internal 패키지 목록으로 tarball 생성 전략을 결정하고, `--containerized` 플래그가 spawn 호출을 Docker로 변경합니다.

### 2-8. 패키징 스코프 가드 (`tools/AGENTS.md:14`)

- 런타임 업데이터 제품 통합은 보류 (later phase)
- 데이터/로그/런타임/캐시 경로는 **네임스페이스로 스코프** — 포트는 일시적 전송 디테일이므로 경로 결정에 참여 금지
- 루트 `pnpm build` 집계 없음 — 소스는 `pnpm --filter`, 패키지 빌드는 `pnpm tools-pack`

## 3. tools/pr — 메인테이너 PR-duty 컨트롤 플레인

**역할**: `gh` CLI를 얇게 감싼 **read-only** 분석 도구. 리뷰 lane 도출, 금지 표면 탐지, 팩트 기반 태그 생성, 검증 명령 도출. **부작용 금지** — approve/merge/comment/close는 메인테이너가 직접 `gh`로 실행한다. (`tools/AGENTS.md:12`)

### 3-1. package.json

- name: `@open-design/tools-pr` (v0.6.0)
- bin: `./bin/tools-pr.mjs`
- deps: `cac@6.7.14` (사이드카 패키지 의존 없음 — 독립적)

### 3-2. 디렉토리

```
tools/pr/src/
├── index.ts         # CLI 진입점, 4개 명령
├── list.ts          # list: 트리아주 큐 스캔
├── view.ts          # view: 단일 PR 요약
├── assignment.ts    # assignment: 담당자 관점
├── classify.ts      # classify: 팩트 기반 태그
├── lane.ts          # lane: 경로 → lane 매핑, 금지 표면
├── tags.ts          # tag: 15+ 디텍터
├── gh.ts            # gh 명령 래퍼
├── bot.ts           # bot 검출: Looper 마커
├── types.ts         # 타입 정의
└── AGENTS.md        # 태그 사전, 검증 전략 문서
```

### 3-3. 서브명령

| 명령 | 설명 |
|---|---|
| `list` | 트리아주 큐: bucket × lane × bucket × author 필터 |
| `view <num>` | 단일 PR 요약: lane + 금지 표면 + 검증 명령 + 리뷰 요약 + CI 상태 |
| `assignment` | 담당자 관점: 누가 뭘 가지고 있는가, 대기 시간, 블로커 |
| `classify [num]` | 팩트 기반 태그. `--all`로 전체 큐, `--json` 출력, `.tmp/tools-pr/classify/` 저장 |

### 3-4. 주요 플래그

- `--json`
- `--namespace <name>` (현재 미사용, 예약)
- list 전용: `--limit <n>`, `--lane <list>`, `--bucket <list>`, `--author <list>`, `--include-drafts`
- classify 전용: `--all`, `--name <stem>`, `--print`

### 3-5. Lane 도출 로직 (`lane.ts:43`)

```typescript
export function deriveLane(paths: string[]): { lane: Lane; hits: Set<Lane> } {
  const hits = new Set<Lane>();
  for (const filePath of paths) {
    if (SKILL_DIR.test(filePath)) hits.add("skill");
    else if (DESIGN_DIR.test(filePath)) hits.add("design-system");
    else if (CRAFT_DIR.test(filePath)) hits.add("craft");
    else if (CONTRACT_PATHS.some((rx) => rx.test(filePath))) hits.add("contract");
    if (!DOCS_ONLY.some((rx) => rx.test(filePath))) allDocs = false;
  }
  if (hits.size === 0 && allDocs) return { lane: "docs", hits: new Set(["docs"]) };
  if (hits.size === 0) return { lane: "default", hits: new Set(["default"]) };
  if (hits.size === 1) return { lane: only as Lane, hits };
  return { lane: "multi", hits };
}
```

정규식:
- `SKILL_DIR = /^skills\/[^/]+\//`
- `DESIGN_DIR = /^design-systems\/[^/]+\//`
- `CRAFT_DIR = /^craft\/[^/]+\.md$/`
- `CONTRACT_PATHS` — `packages/contracts/`, `packages/sidecar-proto/`, `apps/daemon/src/*-routes.ts`

### 3-6. 금지 표면 디텍터

```typescript
export function deriveForbidden(paths: string[]): ForbiddenHit[] {
  const hits: ForbiddenHit[] = [];
  if (paths.some((p) => p.startsWith("apps/nextjs/"))) hits.push("restores-apps/nextjs");
  if (paths.some((p) => p.startsWith("packages/shared/"))) hits.push("restores-packages/shared");
  return hits;
}
```

`apps/nextjs/`, `packages/shared/`는 제거된 디렉토리 — 복원 시도를 PR diff 경로로 감지.

### 3-7. 팩트 기반 태그 사전 (`tags.ts`)

15+ 디텍터, 각각 `{ name, reason, source, [awaitingHours] }` 구조:

| 태그 | 조건 |
|---|---|
| `bot-only-approval` | `reviewDecision = APPROVED` 인데 모든 APPROVED 리뷰가 bot(Looper 마커) |
| `needs-rebase` | `mergeStateStatus ∈ {DIRTY, BEHIND}` |
| `forbidden-surface` | `deriveForbidden` 적중 |
| `unlabeled` | size/, risk/, type/ 레이블 누락 |
| `duplicate-title` | 같은 작가의 다른 PR과 제목 동일 |
| `non-ascii-slug` | design-system 디렉토리 이름이 `[a-z0-9-]+` 위반 |
| `maintainer-edits-disabled` | fork PR에서 `maintainerCanModify=false` |
| `org-member` | author가 리포 조직 멤버 |
| `unresolved-changes-requested` | CHANGES_REQUESTED 리뷰 미해결 |
| `stale-approval` | APPROVED 리뷰의 커밋 OID가 현재 HEAD와 다름 |
| `awaiting-author-response-24h` | 리뷰어 신호 후 24h+ 작가 무응답 |
| `awaiting-reviewer-response-24h` | 작가 신호 후 24h+ 리뷰어 무응답 |
| `awaiting-first-review-24h` | 인간 리뷰어 신호 없고 PR 생성 후 24h+ |

**Bot 검출** (`bot.ts`):
```typescript
const BOT_MARKERS = [
  /<!--\s*looper:/i,
  /Powered by\s*<a[^>]*>Looper<\/a>/i,
  /\[bot\]/i,
];
```

### 3-8. 검증 명령 자동 도출 (`view.ts`)

```typescript
function deriveValidation(paths: string[]): ValidationCommand[] {
  const cmds = [];
  cmds.push({ command: "pnpm guard", reason: "TS-first + .js allowlist gate" });
  cmds.push({ command: "pnpm typecheck", reason: "workspace-wide typecheck" });

  if (touched("apps/web/")) {
    cmds.push({ command: "pnpm --filter @open-design/web typecheck", ... });
    cmds.push({ command: "pnpm --filter @open-design/web test", ... });
    cmds.push({ command: "pnpm --filter @open-design/web build", ... });
  }
  if (touched("packages/sidecar-proto/")) {
    cmds.push({ command: "pnpm --filter @open-design/sidecar-proto test", ... });
  }
  // ...각 touched 패키지별 검증 명령 생성
}
```

PR이 건드린 경로에서 **자동으로** 필요한 검증 명령 목록을 만들어 리뷰어가 무엇을 돌려야 할지 알려줍니다.

### 3-9. 부작용 금지 보장

`tools-pr` 자체는 lane/태그/검증 도출만 — diff 콘텐츠 검사는 `pnpm guard`의 책임으로 명확히 분리됩니다. 따라서 tools-pr은 PR API 호출이 모두 read-only(`gh pr list`, `gh pr view`, `gh api`)이며 approve/comment/merge/close 같은 mutate 호출이 없습니다.

## 4. 비교 요약

| 측면 | tools-dev | tools-pack | tools-pr |
|---|---|---|---|
| 역할 | 로컬 개발 라이프사이클 | 데스크탑 패키징/배포 | PR 트리아주 분석 |
| 서브명령 수 | 8 | 플랫폼당 8개 × 3 OS | 4 |
| 상태 출처 | 사이드카 IPC + 프로세스 스캔 | 파일시스템 + electron-builder | `gh` API |
| 멀티테넌시 | namespace 격리 | namespace 격리 | 없음 |
| 사이드카 의존 | 완전 의존 (proto + sidecar + platform) | 패키지 앱 환경 변수 설정 | 없음 (gh 래퍼만) |
| 부작용 | 프로세스 spawn/kill, 파일 쓰기 | 빌드/설치 | 없음 (read-only) |
