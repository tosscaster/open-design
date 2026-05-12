# 12. 보안 모델

Open Design은 **로컬 데몬 + Electron 데스크탑 + 웹 렌더러** 삼층 구조에서 신뢰 경계를 명확히 그어야 합니다. 이 문서는 8개 계층의 보안 메커니즘을 코드 수준에서 정리합니다.

![HMAC 데스크탑 인증 게이트 시퀀스](../svg/50-integration/12-hmac-gate.svg)

## 1. HMAC 데스크탑 인증 게이트 (PR #974)

**위협**: 로컬 네트워크 공격자나 렌더러 프로세스가 데몬의 `/api/import/folder` 같은 위험한 엔드포인트를 직접 호출해 임의 디렉토리를 프로젝트로 등록.

### 1-1. 핵심 코드

`apps/daemon/src/server.ts` (line ~350-450, ~1800-2100):

```typescript
const DESKTOP_IMPORT_TOKEN_TTL_MS = 60_000;
const DESKTOP_IMPORT_TOKEN_FIELD_SEP = '~';

let desktopAuthSecret: Buffer | null = null;
let desktopAuthEverRegistered = process.env.OD_REQUIRE_DESKTOP_AUTH === '1';
const consumedImportNonces = new Map<string, number>();

export function signDesktopImportToken(
  secret: Buffer,
  baseDir: string,
  options: { nonce: string; exp: string },
): string {
  const signature = createHmac('sha256', secret)
    .update(`${baseDir}\n${options.nonce}\n${options.exp}`)
    .digest('base64url');
  return [options.nonce, options.exp, signature].join(DESKTOP_IMPORT_TOKEN_FIELD_SEP);
}

export function verifyDesktopImportToken(
  secret: Buffer,
  baseDir: string,
  token: string,
  now: number,
  consumedNonces: Map<string, number>,
): DesktopImportTokenVerification {
  const parts = token.split(DESKTOP_IMPORT_TOKEN_FIELD_SEP);
  if (parts.length !== 3) return { ok: false, reason: 'token shape invalid' };
  const [nonce, expISO, signature] = parts;

  const expMs = Date.parse(expISO);
  if (expMs <= now) return { ok: false, reason: 'token expired' };
  if (expMs - now > DESKTOP_IMPORT_TOKEN_TTL_MS * 2)
    return { ok: false, reason: 'token expiry exceeds permitted window' };

  const expected = createHmac('sha256', secret)
    .update(`${baseDir}\n${nonce}\n${expISO}`)
    .digest('base64url');

  if (!timingSafeStringEquals(signature, expected))
    return { ok: false, reason: 'invalid signature' };

  if (consumedNonces.has(nonce))
    return { ok: false, reason: 'nonce already consumed' };
  consumedNonces.set(nonce, expMs);
  return { ok: true };
}
```

### 1-2. 작동 흐름

1. **부팅 시 secret 등록**: Desktop 런타임이 시작되면 sidecar IPC로 32바이트 secret을 데몬에 등록 (`REGISTER_DESKTOP_AUTH` 메시지).
2. **토큰 생성**: 폴더 import 시 Desktop이 `baseDir + nonce + 60초 TTL`로 HMAC-SHA256 토큰 생성.
3. **헤더 첨부**: `X-OD-Desktop-Import-Token` 헤더로 데몬 호출.
4. **검증**: 데몬이 (a) 서명, (b) TTL, (c) 시계 왜곡 한계(2×TTL), (d) nonce 중복을 모두 검사.

### 1-3. 핵심 방어 기법

- **`timingSafeStringEquals`** — 타이밍 공격 방지 (서명 비교는 항상 같은 시간)
- **Nonce 일회용** — `consumedImportNonces` Map이 replay 차단
- **2× TTL 상한** — 시계 왜곡(clock skew) 허용 범위 제한

테스트: `apps/daemon/tests/desktop-import-token-gate.test.ts` — 20+ 케이스로 위조/만료/경로 미일치/nonce replay 모두 거부.

## 2. `od://` 프로토콜 핸들러

**위협**: 렌더러가 외부 도메인 fetch나 임의 file URI 접근으로 SSRF.

### 2-1. 코드 (`apps/packaged/src/protocol.ts`)

```typescript
const OD_SCHEME = "od";
const OD_ENTRY_URL = `${OD_SCHEME}://app/`;

protocol.registerSchemesAsPrivileged([
  {
    privileges: {
      corsEnabled: true,
      secure: true,
      standard: true,
      stream: true,
      supportFetchAPI: true,
    },
    scheme: OD_SCHEME,
  },
]);

export async function handleOdRequest(
  request: Request,
  webRuntimeUrl: string,
  fetchImpl: typeof fetch = fetch,
): Promise<Response> {
  const target = toWebRuntimeUrl(webRuntimeUrl, request.url);
  try {
    return await fetchImpl(new Request(target, request));
  } catch (error) {
    return buildProxyErrorResponse(error, target);
  }
}

function toWebRuntimeUrl(webRuntimeUrl: string, requestUrl: string): string {
  const incoming = new URL(requestUrl);
  const target = new URL(webRuntimeUrl);
  target.pathname = incoming.pathname;
  target.search = incoming.search;
  target.hash = incoming.hash;
  return target.toString();           // host는 고정 (webRuntimeUrl만)
}
```

### 2-2. 핵심 보장

- 모든 `od://` 요청을 **명시적으로 localhost web sidecar로 프록시**.
- pathname/search/hash만 치환, **호스트는 고정** → SSRF 불가능.
- undici 예외(`setTypeOfService EINVAL`) 처리로 네이티브 다이얼로그 방지.

## 3. 샌드박스 아이프레임 격리

**위협**: LLM 생성 HTML이 렌더러 프로세스의 localStorage/쿠키/부모 DOM에 접근해 세션 탈취.

### 3-1. iframe 설정 (`apps/web/src/components/PreviewModal.tsx`)

```typescript
<iframe
  title={`${title} ${activeView?.label ?? ''}`}
  sandbox="allow-scripts allow-popups allow-popups-to-escape-sandbox"
  srcDoc={srcDoc}
/>
```

**핵심**: `allow-same-origin`이 **없다**. 즉 iframe 내 script는 부모 origin의 localStorage/cookie에 접근 불가.

### 3-2. Storage shim (`apps/web/src/runtime/srcdoc.ts`)

대다수 deck/landing artifact가 `localStorage.getItem()`을 호출하는데, sandbox iframe에서 `SecurityError`를 던지면 페이지가 깨짐. 따라서 in-memory stub을 주입:

```typescript
function injectSandboxShim(doc: string): string {
  const shim = `<script data-od-sandbox-shim>(function(){
    function makeStore(){
      var data = {};
      return {
        getItem: function(k){ return data[k] ?? null; },
        setItem: function(k, v){ data[k] = String(v); },
        removeItem: function(k){ delete data[k]; },
        clear: function(){ data = {}; },
      };
    }
    tryShim('localStorage');
    tryShim('sessionStorage');
    // 링크 처리: target="_blank"는 URL 화이트리스트만
  })();</script>`;
  return injectBeforeBodyEnd(doc, shim);
}
```

### 3-3. postMessage 브릿지

iframe ↔ 부모 통신:
- iframe → 부모: 슬라이드 상태, 선택된 요소 (코멘트/inspect)
- 부모 → iframe: 팔레트 변경 (`od:palette`), inspect 모드 (`od:inspect-set`)

전송되는 CSS 값은 `UNSAFE_VALUE` 정규식(`/[;{}<>\n\r]/`)으로 검증 후 `!important` 주입 — CSS injection 차단.

### 3-4. 링크 처리

`<a href>` 클릭 시:
- `#anchor`, `http(s)://`, `mailto:`만 허용
- `javascript:` URL, 빈 href 차단

### 3-5. 검증 신호

```typescript
expect(wrapper).toContain('sandbox="allow-scripts"');
expect(wrapper).not.toContain('allow-same-origin');
```

## 4. Window.open 핸들러 가드

**위협**: 렌더러가 `window.open('file:///etc/passwd')` 호출.

### 4-1. 코드 (`apps/desktop/src/main/runtime.ts:91-100`)

```typescript
export async function fetchResolvedProjectDir(
  apiBaseUrl: string,
  projectId: string,
  fetchImpl = globalThis.fetch,
): Promise<{ ok: true; context } | { ok: false; reason: string }> {
  if (!/^[A-Za-z0-9._-]{1,128}$/.test(projectId)) {
    return { ok: false, reason: "project id contains disallowed characters" };
  }
  let resp = await fetchImpl(`${apiBaseUrl}/api/projects/${encodeURIComponent(projectId)}`);
  // ...
}

export function isOpenPathAllowedForProject(context): { ok: true } | { ok: false; reason: string } {
  if (context.hasBaseDir && !context.fromTrustedPicker) {
    return { ok: false, reason: "project did not come from the trusted picker flow" };
  }
  return { ok: true };
}

export async function validateExistingDirectory(p: string): Promise<PathValidationResult> {
  if (!isAbsolute(p)) return { ok: false, reason: "path must be absolute" };

  let resolvedReal = await realpath(p);     // symlink 해석
  let st = await stat(resolvedReal);
  if (!st.isDirectory()) return { ok: false, reason: "path is not a directory" };

  // macOS .app 번들 차단 — shell.openPath는 앱 실행 의도이므로
  if (resolvedReal.toLowerCase().endsWith(".app"))
    return { ok: false, reason: "application bundles are not project directories" };

  return { ok: true, resolved: resolvedReal };
}
```

### 4-2. 방어 기법

- **렌더러는 projectId만** 제공 → 메인 프로세스가 데몬에서 경로 조회
- **`realpath`** — symlink 해석 후 재검증
- **`.app` 차단** — macOS 앱 번들 실행 방지
- **`fromTrustedPicker` 마커** — folder import HMAC 흐름과 웹 모드 구분

## 5. 파일 경로 검증

`apps/daemon/src/projects.ts:25-50`:

```typescript
const FORBIDDEN_SEGMENT = /^$|^\.\.?$/;

export function isSafeId(projectId: unknown): boolean {
  if (typeof projectId !== 'string') return false;
  const trimmed = projectId.trim();
  if (trimmed.length === 0 || trimmed.length > 128) return false;
  return /^[A-Za-z0-9._-]{1,128}$/.test(trimmed);
}

async function resolveSafeReal(baseDir: string, relPath: string) {
  // 1. realpath(baseDir + relPath)
  // 2. prefix 검증: resolved가 baseDir로 시작하는지
  // 3. path traversal (..) 및 symlink escape 차단
}
```

핵심:
- 프로젝트 ID 정규식(`[A-Za-z0-9._-]{1,128}`)
- `realpath` 해석 후 prefix 검증 → `..` + symlink escape 모두 차단
- 경로 정규화 + `FORBIDDEN_SEGMENT` 체크

## 6. BYOK 프록시 SSRF 가드

`packages/contracts/src/api/connectionTest.ts:51-125` — [11-byok-and-media.md §5](./11-byok-and-media.md#5-ssrf-가드--핵심-보안-레이어) 참조.

요약:
- IPv4 사설 범위(`10/8`, `172.16/12`, `192.168/16`, `127/8`, `169.254/16`, `224/4`) 차단
- IPv6 `fc00::/7`, `fe80::/10`, IPv4-mapped 언래핑 후 재검증
- `validateBaseUrl()`가 4개 BYOK 프록시 라우트에서 모두 호출됨

## 7. 데몬 권한 범위

### 7-1. Agent permission-mode

`apps/daemon/src/runtimes/defs/claude.ts:65`:
```typescript
args.push('--permission-mode', 'bypassPermissions');
```

Claude는 데몬이 이미 `.od/projects/<id>/` cwd로 격리했으므로 내부 권한 prompt를 우회 (사용자 편의).

### 7-2. 허용 디렉토리

`extraAllowedDirs` — 활성 스킬/디자인시스템 폴더만 `--add-dir`로 추가:
```typescript
if (dirs.length > 0 && caps.addDir !== false) {
  args.push('--add-dir', ...dirs);
}
```

### 7-3. CWD 격리

- 표준 프로젝트: `.od/projects/<id>/` (데몬 통제 영역)
- Git 연결 프로젝트: `metadata.baseDir` (사용자가 명시 선택한 경로)

## 8. 이미지 업로드 격리

`apps/daemon/src/server.ts:3203`:
```typescript
const safeImages = imagePaths.filter((p) => {
  const resolved = path.resolve(p);
  return resolved.startsWith(UPLOAD_DIR + path.sep) && fs.existsSync(resolved);
});
```

`UPLOAD_DIR` 화이트리스트 — 그 외 경로는 모두 거부.

Pi RPC는 추가로 base64 인코딩 시 `realpath`로 symlink escape 재검증 (`apps/daemon/src/pi-rpc.ts:399`).

## 9. anti-AI-slop 린터

**위협**: AI 생성 콘텐츠가 도메인 신호로 작용해 anti-design 패턴이 트렌드로 굳어짐.

### 9-1. P0 패턴 (`apps/daemon/src/lint-artifact.ts:38-110`)

```typescript
const PURPLE_HEXES = [
  '#a855f7', '#9333ea', '#7c3aed', '#6d28d9', '#581c87',
  '#6366f1', '#4f46e5', '#4338ca', '#3730a3', '#312e81',
];

const SLOP_EMOJI = [
  '✨', '🚀', '🎯', '⚡', '🔥', '💡', '📈', '🎨', '🛡️', '🌟',
  '💪', '🎉', '👋', '🙌', '✅', '⭐', '🏆',
];

const INVENTED_METRIC_PATTERNS = [
  /\b10×\s+(faster|better|easier)\b/i,
  /\b100×\s+(faster|better)\b/i,
  /\b99\.\d+%\s+uptime\b/i,
  /\bzero[- ]downtime\b/i,
];

const DISPLAY_SANS_RE = /(?:h1|h2|h3|\.h-?(?:hero|xl|lg|md))[^{}]*\{[^}]*font-family\s*:\s*["']?(?:Inter|Roboto|Arial)/i;

export function lintArtifact(rawHtml): LintFinding[] {
  const out: LintFinding[] = [];
  const html = rawHtml.replace(/<!--[\s\S]*?-->/g, '');   // 주석 제거

  // P0: 보라색 그라데이션
  for (const hex of PURPLE_HEXES) {
    const re = new RegExp(`linear-gradient\\([^)]*${escapeRe(hex)}[^)]*\\)`, 'i');
    if (re.exec(html)) {
      out.push({ severity: 'P0', id: 'purple-gradient', message: `...` });
      break;
    }
  }
  // P0: emoji-as-icons
  for (const emoji of SLOP_EMOJI) {
    if (html.includes(emoji)) {
      out.push({ severity: 'P0', id: 'slop-emoji', message: `...` });
      break;
    }
  }
  // P1: invented metrics, P2: lorem ipsum
  return out;
}
```

### 9-2. 운영 모델

- **P0** (블로킹) — 데몬이 다음 turn에 에이전트에게 self-correct 피드백
- **P1/P2** (경고) — UI에 노출, 사용자 결정

## 10. OAuth & MCP 토큰

`apps/daemon/src/mcp-oauth.ts:98-150`:

```typescript
const VERIFIER_LEN = 64;   // RFC 7636: 43–128

export function generateCodeVerifier(): string {
  return base64url(randomBytes(VERIFIER_LEN));
}
export function deriveCodeChallenge(verifier: string): string {
  return base64url(createHash('sha256').update(verifier).digest());
}
export function generateState(): string {
  return base64url(randomBytes(32));
}
```

방어 기법:
- **PKCE (RFC 7636)** — code interception 차단
- **Dynamic Client Registration** — `.od/mcp-oauth-clients.json`에 per-server 캐시
- **In-memory state cache** — authorization request 추적, `state` 파라미터로 CSRF 방지
- **Token refresh rotation** — access_token 만료 시 refresh_token 사용

## 11. 미디어 자격증명 저장소

`.od/media-config.json`:
- **암호화 없음** — 데몬이 localhost만 listen하므로 workspace 신뢰 가정
- **환경 변수 우선** — `OD_OPENAI_API_KEY` 등을 쓰면 파일에 저장 안 함
- **읽기 시 마스킹** — `apiKeyTail: entry.apiKey.slice(-4)`만 노출
- **쓰기 시 wipe 방지** — force=true 없으면 빈 config 거부

## 12. 위협 모델 매트릭스

| 자산 | 위협 | 공격자 | 완화 |
|---|---|---|---|
| 폴더 import API | 임의 디렉토리 프로젝트화 (SSRF/path injection) | 로컬 렌더러, 네트워크 공격자 | HMAC-SHA256 토큰 (60초 TTL, nonce 일회용, 2×TTL 시계 왜곡 상한) |
| 외부 도메인 fetch | `od://` 또는 `file://` 통한 데이터 exfil | 렌더러 | `od://` 핸들러가 호스트를 webRuntimeUrl로 고정 프록시 |
| localStorage / 쿠키 | XSS via 생성 HTML | LLM 출력, 손상된 artifact | `sandbox="allow-scripts"` (allow-same-origin **제외**) + in-memory shim |
| 폴더 열기 (system shell) | symlink escape → /etc/passwd 열기 | metadata 위조 | `realpath` 후 prefix 검증 + `.app` 차단 + `fromTrustedPicker` 마커 |
| 외부 API base URL | 내부 네트워크 스캔 (SSRF) | 사용자 입력/스크립트 | `validateBaseUrl()` 사설/loopback/link-local 정규식 차단 |
| API 키 | 자격증명 유출 | 파일시스템 접근, 메모리 덤프 | env 우선, 파일 시 마스킹, wipe 방지 |
| Artifact 품질 | AI slop 패턴 고착 | LLM 생성 | grep 기반 P0/P1/P2 anti-slop 린터 |
| OAuth code/state | 토큰 탈취 | 로컬 네트워크 도청 | PKCE verifier, in-memory state cache, refresh rotation |
| 데몬 명령 실행 | 권한 확장 | 악의적 agent/skill | `permission-mode bypassPermissions` + `extraAllowedDirs` 화이트리스트 + cwd 격리 |
| 이미지 업로드 | path escape | 사용자 입력 | `UPLOAD_DIR` 화이트리스트 + `realpath` 재검증 |

## 13. Critical Security Invariants

1. **Desktop-Daemon trust boundary** — 데몬은 `OD_REQUIRE_DESKTOP_AUTH=1` 설정 시에만 HMAC 토큰 요구 (웹 모드는 허용).
2. **Sandbox integrity** — 모든 untrusted artifact는 `sandbox="allow-scripts"` (allow-same-origin 제외) iframe에서만 렌더.
3. **Path canonicalization** — `realpath` 해석 후 prefix 검증으로 symlink/`..` escape 모두 차단.
4. **Replay protection** — nonce + TTL 결합.
5. **Timing-safe comparison** — HMAC 서명 검증은 `timingSafeEqual`.
6. **Config wiping prevention** — 미디어 설정 write 시 `force=true` 없으면 빈 config 거부.
7. **Localhost-only binding** — 데몬은 `127.0.0.1:<port>`로만 수신.
8. **Render-time linting** — artifact 저장 시 anti-slop 검사, P0 위반 시 에이전트 피드백 루프.

## 14. 결론

Open Design의 보안 모델은 세 가지 축으로 구성됩니다.

1. **거리 검증** — 어떤 코드가 어떤 자산에 접근할 자격이 있는가를 토큰/마커로 명시
2. **암호화 토큰** — HMAC-SHA256 + nonce + TTL로 위조/replay 방지
3. **격리된 렌더링** — iframe sandbox, `od://` 프록시, cwd 격리로 untrusted 코드의 영향 범위를 제한

특히 PR #974에서 표준화된 HMAC 데스크탑 게이트, web-side sandbox 격리, daemon-side 경로 검증의 조합이 *신뢰할 수 없는 artifact를 안전하게 실행*하는 핵심 메커니즘입니다.
