# 11. BYOK 프록시와 미디어 생성

Open Design의 "BYOK at every layer" 슬로건은 두 곳에서 구현됩니다.

1. **BYOK 프록시** — `/api/proxy/{anthropic,openai,azure,google}/stream`. 클라이언트가 baseUrl + apiKey + model을 주면 데몬이 정규화된 SSE로 변환해 돌려줌.
2. **미디어 생성** — `/api/projects/:id/media/generate` + `od media generate` CLI. gpt-image-2 (이미지), Seedance (비디오), HyperFrames (HTML→MP4) 등 멀티 프로바이더.

![BYOK 프록시 흐름과 SSRF 가드](../svg/50-integration/11-byok-proxy.svg)

## 1. BYOK 프록시 엔드포인트

`apps/daemon/src/chat-routes.ts`에 4개 핵심 경로가 통합되어 있습니다.

| 엔드포인트 | 인증 헤더 | 라인 |
|---|---|---|
| `/api/proxy/anthropic/stream` | `x-api-key` | ~549 |
| `/api/proxy/openai/stream` | `Authorization: Bearer <key>` | ~646 |
| `/api/proxy/azure/stream` | `api-key` | ~758 |
| `/api/proxy/google/stream` | `x-goog-api-key` | ~856 |

추가로 Ollama 호환 엔드포인트도 제공.

## 2. 요청 정규화

클라이언트는 통일된 페이로드를 보냅니다:

```typescript
{ baseUrl, apiKey, model, systemPrompt, messages, maxTokens }
```

각 프로바이더의 고유 형식으로 변환:

### OpenAI-style (`chat-routes.ts:626-637`)
```typescript
const payloadMessages = Array.isArray(messages) ? [...messages] : [];
if (typeof systemPrompt === 'string' && systemPrompt) {
  payloadMessages.unshift({ role: 'system', content: systemPrompt });
}
const payload = {
  model,
  messages: payloadMessages,
  max_tokens: typeof maxTokens === 'number' && maxTokens > 0 ? maxTokens : 8192,
  stream: true,
};
```

### Google Gemini (`chat-routes.ts:834-847`)
```typescript
const contents = (Array.isArray(messages) ? messages : []).map((message) => ({
  role: message.role === 'assistant' ? 'model' : 'user',
  parts: [{ text: message.content }],
}));
const payload = {
  contents,
  generationConfig: { maxOutputTokens },
  ...(systemPrompt ? { systemInstruction: { parts: [{ text: systemPrompt }] } } : {}),
};
```

Anthropic은 `messages` 배열을 그대로 전달하되 `system` 필드를 별도 처리. Azure는 경로 구조 감지 후 `?api-version=...` 자동 부여, deployment 기반이면 본문 `model` 필드 생략.

## 3. 엔드포인트 경로 자동 추론

`chat-routes.ts:346-366`:

```typescript
const appendVersionedApiPath = (baseUrl: string, path: string) => {
  const url = new URL(baseUrl);
  const trimmed = url.pathname.replace(/\/+$/, '');
  url.pathname = /\/v\d+(\/|$)/.test(trimmed)
    ? `${trimmed}${path}`
    : `${trimmed}/v1${path}`;
  return url.toString();
};
```

4가지 케이스 처리:
1. 호스트만: `api.openai.com` → `/v1/chat/completions`
2. `/vN` 포함: `api.openai.com/v1` → `/v1/chat/completions` (중복 방지)
3. 서브패스 `/vN/...`: `api.deepinfra.com/v1/openai` → `/v1/openai/chat/completions`
4. 비-버전 서브패스: `api.minimaxi.com/anthropic` → `/v1/anthropic/chat/completions`

### Azure 분기 (`chat-routes.ts:716-733`)
```typescript
const basePath = url.pathname.replace(/\/+$/, '');
const usesVersionedOpenAIPath = /\/openai\/v\d+(?:$|\/)/.test(basePath);
url.pathname = usesVersionedOpenAIPath
  ? `${basePath}/chat/completions`
  : `${basePath}/openai/deployments/${encodeURIComponent(model)}/chat/completions`;
```

## 4. SSE 응답 정규화

### SSE 프레임 파서 (`chat-routes.ts:368-413`)
```typescript
const collectSseFrame = (frame: string) => {
  const lines = frame.replace(/\r/g, '').split('\n');
  const dataLines = [];
  let event = 'message';
  for (const line of lines) {
    if (line.startsWith('event:')) { event = line.slice(6).trim(); continue; }
    if (!line.startsWith('data:')) continue;
    let value = line.slice(5);
    if (value.startsWith(' ')) value = value.slice(1);
    dataLines.push(value);
  }
  const payload = dataLines.join('\n');
  if (!payload) return { event, payload: '', data: null };
  if (payload === '[DONE]') return { event, payload, data: null };
  try { return { event, payload, data: JSON.parse(payload) }; }
  catch { return { event, payload, data: null }; }
};
```

### 프로바이더별 컨텐츠 추출

- **Anthropic**: `content_block_delta` 이벤트의 `data.delta.text` → text_delta로 전달, `message_stop` 이벤트로 종료.
- **OpenAI / Azure**: `data: [DONE]` 페이로드로 종료 감지, `choices[0].delta.content` 또는 `choices[0].text` 추출.
- **Google Gemini**: `candidates[0].content.parts[].text` 집계, `promptFeedback.blockReason` 또는 `finishReason != "STOP"` 감지.

모든 프로바이더 출력이 같은 `ChatSseEvent`로 통합되어 클라이언트는 차이를 모름.

## 5. SSRF 가드 — 핵심 보안 레이어

검증 함수는 **`packages/contracts/src/api/connectionTest.ts:110-125`**에서 export됩니다.

```typescript
export function validateBaseUrl(baseUrl: string): BaseUrlValidationResult {
  let parsed: ParsedBaseUrl;
  try {
    parsed = new URL(String(baseUrl).replace(/\/+$/, ''));
  } catch {
    return { error: 'Invalid baseUrl' };
  }
  if (!['http:', 'https:'].includes(parsed.protocol)) {
    return { error: 'Only http/https allowed' };
  }
  const hostname = parsed.hostname.toLowerCase();
  if (!isLoopbackApiHost(hostname) && isBlockedExternalApiHostname(hostname)) {
    return { error: 'Internal IPs blocked', forbidden: true };
  }
  return { parsed };
}
```

### 차단 IP 범위 (`connectionTest.ts:51-64`)

**IPv4**:
- `0.0.0.0/8`
- `10.0.0.0/8` (private A)
- `127.0.0.0/8` (loopback — 외부 프록시는 차단, 로컬 프록시는 허용)
- `169.254.0.0/16` (link-local)
- `172.16.0.0/12` (private B)
- `192.168.0.0/16` (private C)
- `224.0.0.0/4` (multicast)

**IPv6**:
- `::` (unspecified)
- `fc00::/7` (unicast private)
- `fe80::/10` (link-local)
- IPv4-mapped `::ffff:...` 언래핑 후 IPv4 검증

### 정규화 (`connectionTest.ts:23-31`)
- 대괄호 제거: `[::1]` → `::1`
- 후행 점 제거: `localhost.` → `localhost`
- 소문자 변환

### 라우트 적용 패턴 (`chat-routes.ts:516-524`)
```typescript
const validated = validateExternalApiBaseUrl(baseUrl);
if (validated.error) {
  return sendApiError(
    res,
    validated.forbidden ? 403 : 400,
    validated.forbidden ? 'FORBIDDEN' : 'BAD_REQUEST',
    validated.error,
  );
}
```

4개 프록시 모두 동일 패턴으로 검증 호출 (`chat-routes.ts:516`, `611`, `706`, `818`).

## 6. 에러 처리

### 상태 코드 매핑 (`chat-routes.ts:326-332`)
```typescript
const proxyErrorCode = (status: number) => {
  if (status === 401) return 'UNAUTHORIZED';
  if (status === 403) return 'FORBIDDEN';
  if (status === 404) return 'NOT_FOUND';
  if (status === 429) return 'RATE_LIMITED';
  return 'UPSTREAM_UNAVAILABLE';
};
```

### `retryable` 플래그
- 429 → `retryable: true`
- 5xx → `retryable: true`
- 4xx → `retryable: false`

### 에러 SSE 전파 (`chat-routes.ts:334-344`)
```typescript
const sendProxyError = (sse, message, init = {}) => {
  sse.send('error', {
    message,
    error: {
      code: init.code || 'UPSTREAM_UNAVAILABLE',
      message,
      ...(init.details === undefined ? {} : { details: init.details }),
      ...(init.retryable === undefined ? {} : { retryable: init.retryable }),
    },
  });
};
```

## 7. 취소와 타임아웃 (`chat-routes.ts:110-120`)
```typescript
const controller = new AbortController();
const abortIfRequestAborted = () => {
  if ((req.aborted || !req.complete) && !res.writableEnded) controller.abort();
};
const abortIfResponseClosed = () => {
  if (!res.writableEnded) controller.abort();
};
req.on('close', abortIfRequestAborted);
res.on('close', abortIfResponseClosed);
```

클라이언트가 SSE를 닫으면 upstream fetch도 자동 abort.

## 8. 미디어 생성 시스템

`apps/daemon/src/media.ts`, `apps/daemon/src/media-config.ts`, `apps/daemon/src/media-routes.ts`, `apps/daemon/src/media-tasks.ts`가 협력.

### 8-1. 지원 모델

`apps/daemon/src/media-models.ts`:

| Surface | 모델 |
|---|---|
| 이미지 | `gpt-image-2`, `gpt-image-1.5`, `dall-e-3`, `seedream`, `grok-imagine-image`, `gemini-3.1-flash-image-preview` |
| 비디오 | `seedance-pro`, `seedance-lite`, `grok-imagine-video`, `hyperframes` |
| 오디오 | `openai-tts-1`, `openai-tts-1-hd`, `elevenlabs-v3-multilingual`, `fish-speech-2`, `minimax`, `suno-v5`, `udio-v1` |

각 모델은 `{ id, name, surface, provider, integrated }`. `provider` 필드가 라우팅 결정자.

### 8-2. 라우팅 (`media.ts:382-437`)
- OpenAI + image → `renderOpenAIImage()`
- OpenAI + audio(speech) → `renderOpenAISpeech()`
- Volcengine + video → `renderVolcengineVideo()`
- Grok + image → `renderGrokImage()`
- HyperFrames + video → `renderHyperFramesViaCli()`
- 기타 → 스텁 폴백 (요구만 받고 데모 응답)

### 8-3. 자격증명 해석 (`media-config.ts:151-159`)

우선순위 (높음 → 낮음):
1. `OD_<PROVIDER>_API_KEY` (정책 우선)
2. 표준 공급업체 env (`OPENAI_API_KEY`, `XAI_API_KEY`, …)
3. `.od/media-config.json` 저장 자격증명
4. OAuth 토큰 폴백 (Hermes/Codex)

```typescript
function readEnvKey(providerId: string): string | null {
  const keys = ENV_KEYS[providerId];
  if (!keys) return null;
  for (const k of keys) {
    const v = process.env[k];
    if (typeof v === 'string' && v.trim()) return v.trim();
  }
  return null;
}
```

### 8-4. 파일 경로 (`media-config.ts:122-129`)
```typescript
function configFile(projectRoot: string): string {
  const dir =
    envOverrideDir('OD_MEDIA_CONFIG_DIR', projectRoot)
    ?? envOverrideDir('OD_DATA_DIR', projectRoot)
    ?? path.join(projectRoot, '.od');
  return path.join(dir, 'media-config.json');
}
```

### 8-5. 보안: API 키 마스킹

`GET /api/media/config`은 키 끝 4글자만 노출:
```typescript
providers[id] = {
  configured: Boolean(envKey || hasStoredKey || oauth?.apiKey),
  source: envKey ? 'env' : hasStoredKey ? 'stored' : oauth?.source || 'unset',
  apiKeyTail: hasStoredKey && entry.apiKey ? entry.apiKey.slice(-4) : '',
  baseUrl: entry.baseUrl || '',
};
```

### 8-6. 쓰기 보호 (`media-config.ts`)
빈 config 쓰기 시도 시 거부:
```typescript
if (Object.keys(next).length === 0 && priorIds.length > 0) {
  if (!force) {
    const err = new Error(`refusing to wipe ${priorIds.length} configured provider(s)`);
    err.status = 409;
    throw err;
  }
}
```

## 9. 출력 파일 명명과 검증

### 명명 규칙 (`media.ts:529-539`)
```typescript
function autoOutputName(surface, model, audioKind?) {
  const base = DEFAULT_OUTPUT_BY_SURFACE[surface] || 'artifact.bin';
  const stamp = Date.now().toString(36);
  const slug = String(model).toLowerCase().replace(/[^a-z0-9]+/g, '-').slice(0, 32);
  const tag = surface === 'audio' && audioKind ? `${audioKind}-${slug}` : slug;
  // ...
  return `${stem}-${tag}-${stamp}${ext}`;
}
```

예: `image-gpt-image-2-1vqc8a8.png`

### 입력 이미지 검증 (`media.ts:143-197`)
```typescript
async function resolveProjectImage(rel: unknown, projectDir: string): Promise<ImageRef | null> {
  if (typeof rel !== 'string' || !rel.trim()) return null;
  const projectRootResolved = path.resolve(projectDir);
  const abs = path.resolve(projectRootResolved, rel.trim());

  // 프로젝트 디렉토리 escape 차단
  if (abs !== projectRootResolved && !abs.startsWith(projectRootResolved + path.sep)) {
    throw new Error(`--image path "${rel}" resolves outside the project directory.`);
  }
  // 크기 제한
  const MAX_IMAGE_BYTES = 16 * 1024 * 1024;
  if (info.size > MAX_IMAGE_BYTES) {
    throw new Error(`--image too large (${info.size} bytes; max ${MAX_IMAGE_BYTES}).`);
  }
}
```

MIME 화이트리스트: PNG, JPEG, WebP, GIF.

## 10. 비동기 작업 추적

### media_tasks 라이프사이클

`apps/daemon/src/media-tasks.ts`. 상태:
- `queued` — 초기
- `running` — 활성
- `done` — 성공
- `failed` — 프로바이더/시스템 오류
- `interrupted` — 데몬 재시작으로 중단

### 진행도 스트리밍 (`media-routes.ts:251-290`)
```typescript
app.post('/api/media/tasks/:id/wait', async (req, res) => {
  const task = getLiveMediaTask(taskId);
  if (task.status === 'done' || task.status === 'failed'
      || task.progress.length > since) {
    return respond();    // 즉시
  }
  const timeoutMs = Math.min(Math.max(requestedTimeout, 0), 25_000);
  task.waiters.add(wake);
  const timer = setTimeout(wake, timeoutMs);
  res.on('close', wake);
});
```

최대 25초 long-poll. 진행률은 `progress_json` 배열에 row별로 append → 다음 wait 호출 시 `since`로 증분만 받음.

## 11. HyperFrames 통합 (HTML→MP4)

`apps/daemon/src/media.ts:1566-1736`. HTML 컴포지션을 헤드리스 Chrome으로 렌더링해 MP4로 변환.

### 컴포지션 구조
```
<compositionDir>/
├── hyperframes.json    # 메타데이터
├── meta.json           # 타이밍 데이터
└── index.html          # window.__timelines 등록 필수
```

### 검증 (`media.ts:1566-1612`)
```typescript
const projectRootResolved = path.resolve(projectDir);
const compAbs = path.resolve(projectRootResolved, compRel);
if (compAbs !== projectRootResolved && !compAbs.startsWith(projectRootResolved + path.sep)) {
  throw new Error(`compositionDir "${compRel}" resolves outside the project directory.`);
}
```

### 실행 (`media.ts:1650-1736`)
```typescript
const child = spawn(
  'npx',
  ['-y', 'hyperframes', 'render', compAbs, '--output', tmpOutput, '--workers', '1'],
  { env: process.env, stdio: ['ignore', 'pipe', 'pipe'] }
);
```

- **타임아웃**: 5분
- **워커**: 1 (메모리 제약 ~256 MB)
- **ANSI 제거**: `s.replace(/\x1b\[[0-9;?]*[A-Za-z]/g, '').replace(/\x1b\[\?[0-9]+[hl]/g, '')`
- **임시 디렉토리 cleanup**: `await rm(tmpRoot, { recursive: true, force: true })`

## 12. CLI 인터페이스 (`od media generate`)

`apps/daemon/src/cli.ts:327-525`:

```typescript
async function runMediaGenerate(rawArgs) {
  const body = {
    surface,
    model: flags.model,
    prompt: flags.prompt,
    output: flags.output,
    aspect: flags.aspect,
    voice: flags.voice,
    audioKind: flags['audio-kind'],
    compositionDir: flags['composition-dir'],
    image: flags.image,
    language: flags.language,
  };
  const url = `${daemonUrl}/api/projects/${projectId}/media/generate`;
  const resp = await fetch(url, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify(body),
  });
  const { taskId } = await resp.json();
  await pollUntilDoneOrBudget(daemonUrl, taskId, 0);
}
```

### 폴링 (`cli.ts:427-525`)
- 총 예산: 25초
- 폴링 간격: 4초
- 예산 소진 시 `od media wait <taskId> --since <n>`로 에이전트 런타임 재개 가능

### 출력 형태
```json
{
  "file": {
    "name": "image-gpt-image-2-xyz.png",
    "size": 102400,
    "mtime": 1715623200000,
    "mime": "image/png",
    "providerNote": "openai/gpt-image-2 · 1:1 · 102.4 KB",
    "providerId": "openai",
    "providerError": null,
    "warnings": ["--length 9999999 clamped to 10"]
  }
}
```

## 13. 보안 요약

| 위협 | 완화 |
|---|---|
| 외부 API 호출로 내부 네트워크 스캔 (SSRF) | `validateBaseUrl()` — 사설/loopback/link-local IP 정규식 차단, IPv4-mapped IPv6 언래핑 |
| API 키 유출 | env 변수 우선 + 파일 시 끝 4글자만 마스킹 |
| 미디어 입력 escape | 경로 prefix 검증, MIME 화이트리스트, 16 MB 상한 |
| 무한 미디어 task 누적 | `media_tasks` 인덱스, status 화이트리스트 (CHECK 제약) |
| HyperFrames 임시 파일 누적 | `try/finally`로 `rm(tmpRoot, recursive: true, force: true)` |
| Upstream 무한 hang | `AbortController` + req/res close 이벤트 |

## 14. 확장 포인트

- **새 LLM 프로바이더**: `chat-routes.ts`에 `/api/proxy/<id>/stream` 추가, `validateBaseUrl` 재사용
- **새 미디어 프로바이더**: `media-models.ts`에 entry 추가, `media.ts`에 `render<X>` 함수 작성, 라우팅 분기에 등록
- **자격증명 저장소**: `media-config.ts`의 `ENV_KEYS` 맵 확장
- **OAuth 토큰 풀**: `media-config.ts` OAuth fallback 로직 확장

이 아키텍처는 BYOK 철학을 일관되게 구현 — 사용자가 자기 키를 로컬 데몬에 보관, 보안 검증 후 외부 프로바이더로 프록시하며, 클라이언트는 모든 프로바이더가 같은 SSE 인터페이스로 보입니다.
