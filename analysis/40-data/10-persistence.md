# 10. 영속성 레이어 — SQLite + `.od/` 디렉토리

데몬의 영속성은 **두 축**으로 나뉩니다.

- **`apps/daemon/src/db.ts`** (1,342 라인) — 메타데이터/관계 데이터를 담는 SQLite (`app.sqlite`, WAL 모드, 12+ 테이블).
- **`.od/` 디렉토리 트리** — 실제 아티팩트, 대화 로그, 메모리, MCP 설정, BYOK 자격증명 파일.

![SQLite ER 다이어그램](../svg/40-data/10-er-diagram.svg)

이 분리는 **GC 안전성**(SQLite 백업으로 대용량 HTML 아티팩트가 누락되지 않게)과 **git 친화성**(GitHub 연결 프로젝트는 `metadata.baseDir`로 사용자 폴더에 직접 작업)을 동시에 보장합니다.

## 1. SQLite 연결과 Pragma

`apps/daemon/src/db.ts:29-42` (요약):

```typescript
export function openDatabase(projectRoot, { dataDir } = {}) {
  const dir = dataDir ? path.resolve(dataDir) : path.join(projectRoot, '.od');
  const file = path.join(dir, 'app.sqlite');
  if (dbInstance && dbFile === file) return dbInstance;
  if (dbInstance) closeDatabase();
  fs.mkdirSync(dir, { recursive: true });
  const db = new Database(file);
  db.pragma('journal_mode = WAL');     // Write-Ahead Logging
  db.pragma('foreign_keys = ON');      // FK 제약 활성
  migrate(db);
  dbInstance = db;
  dbFile = file;
  return db;
}
```

주요 결정:
- **WAL 모드** — 읽기 중 쓰기 가능. WAL 모드에서 `synchronous`는 자동으로 NORMAL로 조정되어 안전성/속도 균형.
- **`foreign_keys = ON`** — `ON DELETE CASCADE` 정상 동작 보장.
- **싱글턴 인스턴스** — 같은 파일 재오픈 방지.
- **`synchronous`, `cache_size`, `mmap_size` 미설정** — SQLite 기본값 사용.

## 2. 누적식 마이그레이션 시스템

`apps/daemon/src/db.ts:51-260`의 `migrate(db)` (요약):

```typescript
function migrate(db) {
  db.exec(`
    CREATE TABLE IF NOT EXISTS projects (...);
    CREATE TABLE IF NOT EXISTS conversations (...);
    CREATE TABLE IF NOT EXISTS messages (...);
    -- 모든 베이스 테이블 IF NOT EXISTS로
  `);

  // 신규 컬럼은 PRAGMA로 검사
  const cols = db.prepare(`PRAGMA table_info(projects)`).all();
  if (!cols.some((c) => c.name === 'metadata_json')) {
    db.exec(`ALTER TABLE projects ADD COLUMN metadata_json TEXT`);
  }

  // 도메인별 모듈 마이그레이션 호출
  migrateCritique(db);
  migrateMediaTasks(db);
}
```

특징:
1. **IF NOT EXISTS 패턴** — 멱등성 보장.
2. **PRAGMA table_info 체크** — SQLite는 `ALTER TABLE IF NOT EXISTS COLUMN` 미지원이므로 직접 검사.
3. **누적식** — 과거 마이그레이션을 지우지 않아 기존 DB도 안전 업그레이드.
4. **도메인 모듈 분리** — `critique/persistence.ts`, `media-tasks.ts`가 자기 테이블을 책임.

## 3. 테이블 스키마 카탈로그

### 3-1. projects (db.ts:53)
```sql
CREATE TABLE projects (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  skill_id TEXT,
  design_system_id TEXT,
  pending_prompt TEXT,
  metadata_json TEXT,             -- JSON: baseDir, kind, fidelity, …
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);
```

`id`는 `isSafeId(id)` 검증 (`apps/daemon/src/projects.ts:1039-1044`): `[A-Za-z0-9._-]+` 1–128자, 순수 `.`-시퀀스 거부.

### 3-2. conversations (db.ts:73)
```sql
CREATE TABLE conversations (
  id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL,
  title TEXT,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  FOREIGN KEY(project_id) REFERENCES projects(id) ON DELETE CASCADE
);
CREATE INDEX idx_conv_project ON conversations(project_id, updated_at DESC);
```

### 3-3. messages (db.ts:85)
```sql
CREATE TABLE messages (
  id TEXT PRIMARY KEY,
  conversation_id TEXT NOT NULL,
  role TEXT NOT NULL,              -- 'user' | 'assistant'
  content TEXT NOT NULL,
  agent_id TEXT,
  agent_name TEXT,
  events_json TEXT,                -- JSON: tool_use/tool_result/usage 이벤트 배열
  attachments_json TEXT,           -- JSON: 첨부 파일/이미지
  produced_files_json TEXT,        -- JSON: AI 생성 파일 목록
  feedback_json TEXT,              -- JSON: 👍/👎 등
  comment_attachments_json TEXT,   -- JSON: 인라인 코멘트
  started_at INTEGER,
  ended_at INTEGER,
  position INTEGER NOT NULL,       -- 메시지 순서 (0부터)
  created_at INTEGER NOT NULL,
  -- 마이그레이션 컬럼
  run_id TEXT,
  run_status TEXT,                 -- queued|running|succeeded|failed|canceled
  last_run_event_id TEXT,
  FOREIGN KEY(conversation_id) REFERENCES conversations(id) ON DELETE CASCADE
);
CREATE INDEX idx_messages_conv ON messages(conversation_id, position);
```

### 3-4. preview_comments (db.ts:106)
```sql
CREATE TABLE preview_comments (
  id TEXT PRIMARY KEY,             -- cmt_xxxxxxxx
  project_id TEXT NOT NULL,
  conversation_id TEXT NOT NULL,
  file_path TEXT NOT NULL,         -- 아티팩트 파일
  element_id TEXT NOT NULL,        -- DOM element id
  selector TEXT NOT NULL,          -- CSS 선택자
  label TEXT NOT NULL,
  text TEXT NOT NULL,              -- 요소 텍스트 (≤160자)
  position_json TEXT NOT NULL,     -- JSON: {x, y, width, height}
  html_hint TEXT NOT NULL,         -- 요소 HTML 스니펫 (≤180자)
  note TEXT NOT NULL,
  status TEXT NOT NULL,            -- open|attached|applying|needs_review|resolved|failed
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  -- 마이그레이션 컬럼 (멀티 요소 pod 코멘트)
  selection_kind TEXT,             -- 'element' | 'pod'
  member_count INTEGER,
  pod_members_json TEXT,
  UNIQUE(project_id, conversation_id, file_path, element_id),
  FOREIGN KEY(project_id) REFERENCES projects(id) ON DELETE CASCADE,
  FOREIGN KEY(conversation_id) REFERENCES conversations(id) ON DELETE CASCADE
);
CREATE INDEX idx_preview_comments_conversation
  ON preview_comments(project_id, conversation_id, updated_at DESC);
```

### 3-5. tabs (db.ts:129)
```sql
CREATE TABLE tabs (
  project_id TEXT NOT NULL,
  name TEXT NOT NULL,
  position INTEGER NOT NULL,
  is_active INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY(project_id, name),
  FOREIGN KEY(project_id) REFERENCES projects(id) ON DELETE CASCADE
);
CREATE INDEX idx_tabs_project ON tabs(project_id, position);
```

### 3-6. deployments (db.ts:141)
```sql
CREATE TABLE deployments (
  id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL,
  file_name TEXT NOT NULL,
  provider_id TEXT NOT NULL,        -- cloudflare-pages | github-pages | vercel | …
  url TEXT NOT NULL,
  deployment_id TEXT,
  deployment_count INTEGER NOT NULL DEFAULT 1,
  target TEXT NOT NULL DEFAULT 'preview',
  status TEXT NOT NULL DEFAULT 'ready',
  status_message TEXT,
  reachable_at INTEGER,
  provider_metadata_json TEXT,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  UNIQUE(project_id, file_name, provider_id),
  FOREIGN KEY(project_id) REFERENCES projects(id) ON DELETE CASCADE
);
CREATE INDEX idx_deployments_project ON deployments(project_id, updated_at DESC);
```

### 3-7. routines + routine_runs (db.ts:163)
```sql
CREATE TABLE routines (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  prompt TEXT NOT NULL,
  schedule_kind TEXT NOT NULL,      -- once|hourly|daily|weekly|cron
  schedule_value TEXT NOT NULL,      -- (레거시)
  schedule_json TEXT,                -- (신규) RoutineSchedule 객체
  project_mode TEXT NOT NULL,        -- all|specific
  project_id TEXT,
  skill_id TEXT,
  agent_id TEXT,
  enabled INTEGER NOT NULL DEFAULT 1,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);

CREATE TABLE routine_runs (
  id TEXT PRIMARY KEY,
  routine_id TEXT NOT NULL,
  trigger TEXT NOT NULL,             -- scheduled|manual|webhook
  status TEXT NOT NULL,              -- queued|running|succeeded|failed
  project_id TEXT NOT NULL,
  conversation_id TEXT NOT NULL,
  agent_run_id TEXT NOT NULL,
  started_at INTEGER NOT NULL,
  completed_at INTEGER,
  summary TEXT,
  error TEXT,
  FOREIGN KEY(routine_id) REFERENCES routines(id) ON DELETE CASCADE
);
CREATE INDEX idx_routine_runs_routine ON routine_runs(routine_id, started_at DESC);
```

### 3-8. critique_runs (`apps/daemon/src/critique/persistence.ts:174`)
```sql
CREATE TABLE critique_runs (
  id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL,
  conversation_id TEXT,
  artifact_path TEXT,
  status TEXT NOT NULL CHECK (status IN
    ('shipped','below_threshold','timed_out','interrupted','degraded','failed','legacy','running')),
  score REAL,
  rounds_json TEXT NOT NULL DEFAULT '[]',
  transcript_path TEXT,
  protocol_version INTEGER NOT NULL,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  FOREIGN KEY(project_id) REFERENCES projects(id) ON DELETE CASCADE,
  FOREIGN KEY(conversation_id) REFERENCES conversations(id) ON DELETE SET NULL
);
CREATE INDEX idx_critique_runs_project ON critique_runs(project_id, updated_at DESC);
CREATE INDEX idx_critique_runs_status ON critique_runs(status);
```

### 3-9. media_tasks (`apps/daemon/src/media-tasks.ts:99`)
```sql
CREATE TABLE media_tasks (
  id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL,
  status TEXT NOT NULL CHECK (status IN
    ('queued','running','done','failed','interrupted')),
  surface TEXT,
  model TEXT,
  progress_json TEXT NOT NULL DEFAULT '[]',
  file_json TEXT,
  error_json TEXT,
  started_at INTEGER NOT NULL,
  ended_at INTEGER,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  FOREIGN KEY(project_id) REFERENCES projects(id) ON DELETE CASCADE
);
CREATE INDEX idx_media_tasks_project ON media_tasks(project_id, updated_at DESC);
CREATE INDEX idx_media_tasks_status ON media_tasks(status, updated_at DESC);
```

### 3-10. templates (db.ts:64)
```sql
CREATE TABLE templates (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  description TEXT,
  source_project_id TEXT,
  files_json TEXT NOT NULL,
  created_at INTEGER NOT NULL
);
```

## 4. JSON 컬럼 정책

`apps/daemon/src/db.ts:411-414` + `db.ts:1103-1110`:

```typescript
function stringifyJsonObjectOrNull(value) {
  if (!value || typeof value !== 'object' || Array.isArray(value)) return null;
  return Object.keys(value).length > 0 ? JSON.stringify(value) : null;
}

function parseJsonOrUndef(s) {
  if (typeof s !== 'string' || !s) return undefined;
  try { return JSON.parse(s); }
  catch { return undefined; }    // 손상된 JSON은 undefined (에러 throw 안 함)
}
```

읽기 시 모든 `*_json` 컬럼은 `parseJsonOrUndef`로 복원, 실패 시 undefined → UI는 빈 상태로 fallback. 쓰기 시 빈 객체는 `null`로 저장 (공간 절약).

## 5. ER 다이어그램 (텍스트)

```
projects (id PK)
  ├─ conversations (id PK, project_id FK CASCADE)
  │   └─ messages (id PK, conversation_id FK CASCADE)
  │
  ├─ preview_comments (id PK, project_id FK CASCADE, conversation_id FK CASCADE)
  │   UNIQUE(project_id, conversation_id, file_path, element_id)
  │
  ├─ tabs (project_id PK, name PK, position, is_active)
  │
  ├─ deployments (id PK, project_id FK CASCADE)
  │   UNIQUE(project_id, file_name, provider_id)
  │
  ├─ critique_runs (id PK, project_id FK CASCADE, conversation_id FK SET NULL)
  │
  └─ media_tasks (id PK, project_id FK CASCADE)

routines (id PK)
  └─ routine_runs (id PK, routine_id FK CASCADE)

templates (id PK)   -- 독립 (project 참조 없음)
```

## 6. `.od/` 디렉토리 레이아웃

![.od/ 디렉토리 트리](../svg/40-data/10b-od-directory-tree.svg)

```
<projectRoot>/.od/                 (또는 OD_DATA_DIR)
├── app.sqlite                     # 메인 DB
├── app.sqlite-shm                 # WAL 공유 메모리 (임시)
├── app.sqlite-wal                 # WAL 로그 (임시)
├── app-config.json                # 앱 설정 (onboarding, agent/skill 선택)
├── media-config.json              # BYOK 자격증명
├── projects/
│   └── <projectId>/
│       ├── index.html             # 메인 아티팩트
│       ├── *.html, *.css, *.js
│       ├── *.sketch.json          # 편집 가능 스케치
│       ├── *.png|jpg              # 이미지
│       ├── .live-artifacts/       # 실시간 미리보기 캐시
│       └── *.artifact.json        # 아티팩트 메타데이터
├── artifacts/                     # 공유 아티팩트 저장소
├── critique-artifacts/             # 평가 결과
│   └── <critiqueRunId>.json
├── design-systems/                # 사용자 추가 디자인시스템 (shadow)
├── design-templates/              # 사용자 추가 템플릿
├── skills/                        # 사용자 추가/편집 스킬 (shadow)
├── memories/                      # 사용자 메모리 (Markdown)
├── mcp/                           # MCP 서버 설정
└── connectors/                    # 외부 커넥터 상태
```

### Git 연결 프로젝트

`apps/daemon/src/projects.ts:41-48`:

```typescript
export function resolveProjectDir(projectsRoot, projectId, metadata) {
  if (typeof metadata?.baseDir === 'string') {
    const p = path.normalize(metadata.baseDir);
    if (path.isAbsolute(p)) return p;     // 사용자 폴더 직접 사용
  }
  if (!isSafeId(projectId)) throw new Error('invalid project id');
  return path.join(projectsRoot, projectId);
}
```

- **표준 프로젝트** → `.od/projects/<id>/` (생성된 파일이 격리된 영역에 보관)
- **Git 연결 프로젝트** → `metadata.baseDir`가 가리키는 사용자 폴더 (DB는 여전히 중앙 `.od/app.sqlite` 사용)

### 숨김 폴더 화이트리스트

파일 목록 조회 시 제외되는 디렉토리: `.git`, `node_modules`, `.next`, `.nuxt`, `.cache`, `.turbo`, `.od`, `.tmp`, …

## 7. 경로 검증 (보안)

`apps/daemon/src/projects.ts:908-943` (`resolveSafe` + `resolveSafeReal`):

```typescript
function resolveSafe(dir, name) {
  const safePath = validateProjectPath(name);
  const target = path.resolve(dir, safePath);
  if (!target.startsWith(dir + path.sep) && target !== dir) {
    throw new Error('path escapes project dir');
  }
  return target;
}

async function resolveSafeReal(dir, name) {
  const candidate = resolveSafe(dir, name);
  const rootReal = await realpath(dir).catch(() => dir);
  let real;
  try { real = await realpath(candidate); }
  catch (err) {
    if (err.code !== 'ENOENT') throw err;
    real = await resolveExistingPrefix(candidate);
  }
  if (!real.startsWith(rootReal + path.sep) && real !== rootReal) {
    throw new Error('path escapes project dir via symlink');
  }
  return real;
}
```

- 문자열 prefix 검증 — `..` 으로 디렉토리 탈출 차단
- **realpath 해석 후** prefix 재검증 — symlink escape 차단

## 8. OD_DATA_DIR / OD_MEDIA_CONFIG_DIR 오버라이드

두 변수는 **독립적**이다 (`server.ts:924-959` `resolveDataDir`, `media-config.ts:122-129` `configFile`).

- SQLite / projects / artifacts 등 모든 일반 데이터: `OD_DATA_DIR` → 없으면 `<projectRoot>/.od` (`openDatabase`의 `dataDir` 파라미터에 주입).
- `media-config.json` 전용: `OD_MEDIA_CONFIG_DIR` > `OD_DATA_DIR` > `<projectRoot>/.od` (즉 media-config는 두 변수 모두 우선순위 적용).

### 홈 확장 (`apps/daemon/src/home-expansion.ts:23-32`)

```typescript
const HOME_BARE_TOKENS = new Set(['~', '$HOME', '${HOME}']);
const HOME_PREFIX_RE = /^(~|\$\{HOME\}|\$HOME)[/\\](.*)$/;

export function expandHomePrefix(raw: string): string {
  const home = os.homedir();
  if (HOME_BARE_TOKENS.has(raw)) return home;
  const match = HOME_PREFIX_RE.exec(raw);
  if (match) return path.join(home, match[2] ?? '');
  return raw;
}
```

지원: `~`, `~/path`, `$HOME`, `${HOME}/path`. 절대/상대 경로는 그대로.

## 9. 트랜잭션 패턴

### 9-1. setTabs — 원자 swap

`apps/daemon/src/db.ts:1329-1342` (요약):

```typescript
export function setTabs(db, projectId, names, activeName) {
  const tx = db.transaction(() => {
    db.prepare(`DELETE FROM tabs WHERE project_id = ?`).run(projectId);
    const ins = db.prepare(
      `INSERT INTO tabs (project_id, name, position, is_active) VALUES (?, ?, ?, ?)`,
    );
    names.forEach((name, i) => {
      ins.run(projectId, name, i, name === activeName ? 1 : 0);
    });
  });
  tx();   // 실행 (실패 시 자동 롤백)
  return listTabs(db, projectId);
}
```

### 9-2. upsertMessage — 준원자

신규면 INSERT (position 자동 계산), 기존이면 UPDATE. 별도 SQL로 `conversations.updated_at` 갱신 — **같은 트랜잭션 아님** (대화 정렬 최신성 유지 목적).

## 10. 데이터 무결성 가드

### Foreign Key
- `conversations.project_id` → `projects.id` CASCADE
- `messages.conversation_id` → `conversations.id` CASCADE
- `preview_comments.{project_id, conversation_id}` → CASCADE
- `critique_runs.project_id` → CASCADE, `conversation_id` → SET NULL
- `media_tasks.project_id` → CASCADE
- `routine_runs.routine_id` → CASCADE

### UNIQUE
- `deployments(project_id, file_name, provider_id)` — 같은 파일/제공자당 최대 1개
- `preview_comments(project_id, conversation_id, file_path, element_id)` — 같은 요소당 최대 1개
- `tabs(project_id, name)` — 프로젝트당 파일명 유일

### CHECK
- `media_tasks.status IN ('queued','running','done','failed','interrupted')`
- `critique_runs.status IN ('shipped','below_threshold','timed_out','interrupted','degraded','failed','legacy','running')`

## 11. 적극적 인덱싱

모든 주요 쿼리 경로에 인덱스:
- `idx_conv_project (project_id, updated_at DESC)`
- `idx_messages_conv (conversation_id, position)`
- `idx_preview_comments_conversation (project_id, conversation_id, updated_at DESC)`
- `idx_deployments_project (project_id, updated_at DESC)`
- `idx_tabs_project (project_id, position)`
- `idx_routine_runs_routine (routine_id, started_at DESC)`
- `idx_critique_runs_project (project_id, updated_at DESC)`
- `idx_critique_runs_status (status)`
- `idx_media_tasks_project (project_id, updated_at DESC)`
- `idx_media_tasks_status (status, updated_at DESC)`

better-sqlite3는 내부적으로 prepared statement 결과를 캐싱하므로 같은 SQL 문자열은 자동 재사용됨.

## 12. 마이그레이션 추가 가이드

### 신규 테이블 (기본 도메인)
1. `db.ts`의 `migrate()` 함수에 `CREATE TABLE IF NOT EXISTS` 추가
2. Normalize/Insert/Update/Delete 헬퍼 작성
3. DB 인스턴스 닫고 재오픈해서 마이그레이션 확인

### 기존 테이블 컬럼 추가
```typescript
const cols = db.prepare(`PRAGMA table_info(existing_table)`).all();
if (!cols.some((c) => c.name === 'new_column')) {
  db.exec(`ALTER TABLE existing_table ADD COLUMN new_column TYPE DEFAULT value`);
}
```
Normalize 함수에 `newColumn: row.newColumn ?? undefined` 추가, Insert/Update SQL 업데이트.

### 신규 도메인 (전문 모듈)
1. `<domain>/persistence.ts` 또는 `<domain>-tasks.ts` 작성 — `critique/persistence.ts`, `media-tasks.ts` 참고
2. `migrateDomain(db)` export
3. `db.ts`의 `migrate()` 끝에서 호출
4. 도메인 모듈은 자기 테이블 + 인덱스 + Normalize 함수까지 책임

### 검증 및 롤아웃
- 로컬: 기존 DB + 신규 마이그레이션 → 하위 호환 확인
- 스테이징: 실제 데이터셋으로 PRAGMA 검사 성능 측정
- 프로덕션: 메인 배포 시 모든 데몬이 자동으로 마이그레이션 (재시작 시 `migrate()` 실행)

## 13. DB 헬퍼 API 정리

자주 쓰이는 함수들 (`apps/daemon/src/db.ts`):

| 범주 | 함수 |
|---|---|
| Projects | `getProject`, `listProjects`, `insertProject`, `updateProject`, `deleteProject` |
| Conversations | `getConversation`, `listConversations`, `insertConversation`, `updateConversation`, `deleteConversation` |
| Messages | `listMessages`, `upsertMessage`, `deleteMessage` |
| Preview Comments | `listPreviewComments`, `upsertPreviewComment`, `updatePreviewCommentStatus`, `deletePreviewComment` |
| Deployments | `listDeployments`, `getDeployment`, `upsertDeployment` |
| Routines | `listRoutines`, `getRoutine`, `insertRoutine`, `updateRoutine`, `deleteRoutine`, `listRoutineRuns`, `getLatestRoutineRun`, `insertRoutineRun`, `updateRoutineRun` |
| Tabs | `listTabs`, `setTabs` |
| Critique | `insertCritiqueRun`, `getCritiqueRun`, `updateCritiqueRun`, `listCritiqueRunsByProject` |
| Media Tasks | `insertMediaTask`, `getMediaTask`, `updateMediaTask`, `listMediaTasksByProject` |

## 14. 요약

- **중앙화된 메타데이터** — SQLite ACID + WAL 모드
- **분리된 아티팩트** — `.od/projects/<id>/` 파일시스템 (GC 안전)
- **누적식 마이그레이션** — IF NOT EXISTS + PRAGMA 검증
- **경로 보안** — `resolveSafe`/`resolveSafeReal`로 path traversal + symlink escape 차단
- **확장 포인트** — `<domain>/persistence.ts` 모듈로 새 도메인 추가가 격리됨
- **인덱싱** — 모든 정렬/필터 컬럼에 인덱스
- **트랜잭션** — `db.transaction()`으로 멀티 INSERT/DELETE 원자성 보장

---

## 15. 심층 노트

### 15-1. 핵심 코드 발췌

```typescript
// apps/daemon/src/db.ts (요약) — 멱등 마이그레이션
function migrate(db: SqliteDb): void {
  db.exec(`CREATE TABLE IF NOT EXISTS projects (...); ...`);
  const cols = db.prepare(`PRAGMA table_info(projects)`).all() as DbRow[];
  if (!cols.some(c => c.name === 'metadata_json')) {
    db.exec(`ALTER TABLE projects ADD COLUMN metadata_json TEXT`);
  }
  migrateCritique(db);
  migrateMediaTasks(db);
}
```

```typescript
// apps/daemon/src/db.ts — 트랜잭션 패턴
export function setTabs(db, projectId, names, activeName) {
  const tx = db.transaction(() => {
    db.prepare(`DELETE FROM tabs WHERE project_id = ?`).run(projectId);
    const ins = db.prepare(`INSERT INTO tabs (...) VALUES (?, ?, ?, ?)`);
    names.forEach((n, i) => ins.run(projectId, n, i, n === activeName ? 1 : 0));
  });
  tx();   // 실패 시 자동 ROLLBACK
}
```

```typescript
// apps/daemon/src/projects.ts — symlink-aware 경로 검증
async function resolveSafeReal(baseDir: string, relPath: string) {
  const candidate = resolveSafe(baseDir, relPath);
  const rootReal = await realpath(baseDir).catch(() => baseDir);
  const real = await realpath(candidate).catch((err) => {
    if (err.code === 'ENOENT') return resolveExistingPrefix(candidate);
    throw err;
  });
  if (!real.startsWith(rootReal + path.sep) && real !== rootReal)
    throw new Error('path escapes project dir via symlink');
  return real;
}
```

### 15-2. 엣지 케이스 + 에러 패턴

- **WAL 모드 + 동시 쓰기**: better-sqlite3는 단일 프로세스, single-threaded 핸들. 동시 쓰기 시 직렬화 (자동). 외부 프로세스가 같은 DB 열면 WAL race — 데몬은 단일.
- **마이그레이션 부분 실패**: ALTER TABLE 중 SQLite 에러 (디스크 full 등) → 부분 적용 가능. 다음 부팅 시 PRAGMA check가 누락 컬럼 발견 → 다시 ALTER.
- **JSON 컬럼 손상**: 외부 도구가 잘못 편집했거나 클라이언트가 잘못 입력 시 `parseJsonOrUndef` 가 try-catch로 silent fail → undefined.
- **FK CASCADE 폭주**: project 삭제 시 conversations + messages + comments + deployments + critique + media_tasks 모두 CASCADE. 큰 프로젝트 (메시지 수만 개) 삭제 ~수 초.
- **`.od/projects/<id>/` + `.od/app.sqlite` 불일치**: 사용자가 수동으로 폴더 삭제 시 DB에 project 레코드만 남음 → daemon이 "missing files" 표시. cleanup 스크립트 (manual).
- **`OD_DATA_DIR` + `OD_MEDIA_CONFIG_DIR` 동시 지정**: media-config.json만 별도 위치. SQLite은 OD_DATA_DIR 사용.

### 15-3. 트레이드오프 + 설계 근거

- **SQLite vs PostgreSQL/embedded option**: SQLite는 zero-config + 단일 파일 + WAL 성능 충분. 비용은 multi-process 동시 쓰기 불가 — 데몬 단일 프로세스 모델과 잘 맞음.
- **누적식 마이그레이션**: 마이그레이션 번호 시스템 (v1, v2) 없이 IF NOT EXISTS + PRAGMA check. 단순성 vs version-aware. 비용은 마이그레이션 순서 의존성 불명확 (현재는 부팅 시 한 번 실행).
- **JSON 컬럼 vs 정규화 테이블**: events/attachments 같은 schemaless 데이터에 JSON. 비용은 쿼리 시 JSON path 어렵 → 보통 row 전체 fetch 후 JS에서 처리.
- **파일 vs SQLite BLOB**: HTML 아티팩트, 이미지 등은 파일시스템. SQLite BLOB은 단일 파일 backup 편하지만 GC/streaming 비용.
- **realpath + prefix 검증**: symlink escape 차단. 비용은 매 호출 시 fs.realpath syscall (~ms).

### 15-4. 알고리즘 + 성능

- **SQLite WAL write**: ~0.1-1ms per simple INSERT. fsync(checkpoint)는 ~5-20ms (디스크 동기화).
- **인덱스 사용 query**: `idx_messages_conv (conversation_id, position)`로 대화 메시지 조회 O(log N + K) — N=메시지 수, K=결과 수.
- **트랜잭션 batch INSERT**: setTabs 1000 entries → ~10-30ms (트랜잭션 없으면 ~수 초).
- **WAL checkpoint**: auto-checkpoint 1000 pages 기본. 데이터 ~ 4MB 누적 시 자동 합병.
- **DB 사이즈**: 평균 사용자 ~50-200 MB (대화 텍스트가 주요). HTML 아티팩트는 파일 → DB 안 들어옴.
- **마이그레이션 실행**: 모든 IF NOT EXISTS 합쳐 ~10-30ms 콜드 (no-op 시 ~1ms).

## 16. 함수·라인 단위 추적

### 16-1. `migrate(db)` — `apps/daemon/src/db.ts:51-260`

기본 도메인 8개 테이블을 단일 `db.exec()` 블록(`db.ts:52-196`)으로 모두 만든 뒤, 신규 컬럼을 `PRAGMA table_info`로 검사하여 누락분만 `ALTER TABLE`로 보강한다. 순서대로 분해하면:

| 라인 | 동작 | 영향 |
|------|------|------|
| `52-62` | `CREATE TABLE projects` | 메인 엔티티. `id`/`name`/`metadata_json` 등 8 컬럼. |
| `64-71` | `CREATE TABLE templates` | 프로젝트 → 템플릿 변환 결과. FK 없음 (소스 프로젝트 삭제돼도 유지). |
| `73-80` | `CREATE TABLE conversations` | `FOREIGN KEY(project_id) ON DELETE CASCADE`. |
| `82-83` | `CREATE INDEX idx_conv_project` | `(project_id, updated_at DESC)` — listConversations 핫패스. |
| `85-101` | `CREATE TABLE messages` | 13 컬럼. JSON: `events_json`, `attachments_json`, `produced_files_json`, `feedback_json`. |
| `103-104` | `CREATE INDEX idx_messages_conv` | `(conversation_id, position)` — listMessages 핫패스. |
| `106-124` | `CREATE TABLE preview_comments` | UNIQUE 키 `(project_id, conversation_id, file_path, element_id)` 로 멱등. |
| `126-127` | `CREATE INDEX idx_preview_comments_conversation` | 대화별 코멘트 조회. |
| `129-136` | `CREATE TABLE tabs` | 복합 PK `(project_id, name)`. `is_active INTEGER` 로 boolean. |
| `138-139` | `CREATE INDEX idx_tabs_project` | `(project_id, position)`. |
| `141-158` | `CREATE TABLE deployments` | UNIQUE `(project_id, file_name, provider_id)`; `provider_metadata_json` JSON. |
| `160-161` | `CREATE INDEX idx_deployments_project` | 프로젝트별 배포 목록. |
| `163-177` | `CREATE TABLE routines` | FK 없음 (글로벌). `schedule_kind`/`schedule_value` legacy + `schedule_json` 권위. |
| `179-192` | `CREATE TABLE routine_runs` | `FK(routine_id) ON DELETE CASCADE`. |
| `194-195` | `CREATE INDEX idx_routine_runs_routine` | `(routine_id, started_at DESC)`. |

ALTER 블록 (`db.ts:197-257`):

| 라인 | 검사 대상 | 추가 컬럼 |
|------|-----------|----------|
| `199-202` | `projects` | `metadata_json TEXT` |
| `203-208` | `messages` | `agent_id`, `agent_name` |
| `210-217` | `messages` | `run_id`, `run_status`, `last_run_event_id` |
| `219-224` | `messages` | `comment_attachments_json`, `feedback_json` |
| `226-235` | `preview_comments` | `selection_kind`, `member_count`, `pod_members_json` |
| `236-248` | `deployments` | `status NOT NULL DEFAULT 'ready'`, `status_message`, `reachable_at`, `provider_metadata_json` |
| `254-257` | `routines` | `schedule_json` (테이블이 존재할 때만 — 새 DB는 inline 정의로 이미 포함) |

마지막에 `migrateCritique(db)` (`db.ts:258`)과 `migrateMediaTasks(db)` (`db.ts:259`)가 도메인 마이그레이션을 위임받아 각각 `critique_*` 테이블, `media_tasks` 테이블을 생성한다.

### 16-2. `resolveSafeReal(dir, name)` — `apps/daemon/src/projects.ts:925-943` (호출 `resolveSafe` @ 908-915, `validateProjectPath` @ 965-981, `resolveExistingPrefix` @ 945-958)

심볼릭 링크 우회까지 막는 경로 검증. 호출 흐름:

```
resolveSafeReal(dir, name)                            // projects.ts:925
 ├─ resolveSafe(dir, name)                            // projects.ts:908
 │   ├─ validateProjectPath(name)                     // projects.ts:965 — NUL/Windows-drive/절대경로/`..`/예약어 차단
 │   └─ path.resolve(dir, safePath)                   // string-prefix 확인
 ├─ realpath(dir)                                     // projects.ts:927 — rootReal 확정
 ├─ realpath(candidate)                               // projects.ts:930 — 실재 시
 │   └─ ENOENT → resolveExistingPrefix(candidate)     // projects.ts:945 — 가장 긴 실재 prefix 찾기 (재귀적 realpath)
 └─ real.startsWith(rootReal + sep) 확인              // projects.ts:937 — 실패 시 EPATHESCAPE
```

`resolveExistingPrefix` (`projects.ts:945-958`)는 path를 sep으로 자른 뒤 뒤에서부터 realpath()를 시도한다. 가장 깊은 실재 prefix가 발견되면 tail을 단순 concat — 미실재 디렉토리에 쓰기 시도(첫 write)도 검증 가능. `validateProjectPath` (`projects.ts:965-981`)의 차단 규칙:

| 차단 패턴 | 라인 | 사유 |
|-----------|------|------|
| `typeof !== 'string'` 또는 빈 문자열 | `966` | 형식 위반 |
| `\0` (NUL) | `970` | C-string 종료 트릭 |
| `^[A-Za-z]:` | `970` | Windows 드라이브 절대 경로 |
| `^/` (선행 슬래시) | `970` | POSIX 절대 경로 |
| `FORBIDDEN_SEGMENT` 매치 (`^$\|^\.\.?$` — 빈 세그먼트/`.`/`..`) | `974` | path traversal |
| `RESERVED_PROJECT_FILE_SEGMENTS` (실제 항목: `.live-artifacts`) | `977` | 프로젝트 내부 예약 디렉토리 침범 |

### 데이터 페이로드 샘플

`projects` row (실제 SQLite 덤프 형태):

```
id           = 'proj_2024-04-12T09-15-22-abc123'
name         = 'landing-page-v3'
skill_id     = 'web-app'
design_system_id = 'shadcn-ui'
pending_prompt = NULL
metadata_json = '{"baseDir":"/Users/me/Code/landing","github":{"repo":"owner/landing","branch":"main"},"createdVia":"import"}'
created_at   = 1712910922340
updated_at   = 1712914533102
```

`conversations` row:

```
id           = 'conv_01HV9XK7QY8Q3Z5N4M2P1R'
project_id   = 'proj_2024-04-12T09-15-22-abc123'
title        = 'Add hero CTA + pricing comparison'
created_at   = 1712911000123
updated_at   = 1712914500987
```

`messages` row (assistant 응답, JSON 컬럼 포함):

```
id                = 'msg_01HV9XKBR2D5...'
conversation_id   = 'conv_01HV9XK7QY8Q3Z5N4M2P1R'
role              = 'assistant'
content           = 'I added the hero CTA section…'
agent_id          = 'claude-sonnet-4-5'
agent_name        = 'Claude'
events_json       = '[{"t":"tool_use","name":"write_file","path":"app/page.tsx"},{"t":"text","delta":"I added…"}]'
attachments_json  = '[]'
produced_files_json = '["app/page.tsx","app/pricing.tsx"]'
feedback_json     = NULL
run_status        = 'done'
position          = 7
created_at        = 1712913004511
```

`messages` 스키마 덤프 (`PRAGMA table_info(messages)`):

```
0|id|TEXT|1||1
1|conversation_id|TEXT|1||0
2|role|TEXT|1||0
3|content|TEXT|1||0
4|agent_id|TEXT|0||0
5|agent_name|TEXT|0||0
6|events_json|TEXT|0||0
7|attachments_json|TEXT|0||0
8|produced_files_json|TEXT|0||0
9|feedback_json|TEXT|0||0
10|started_at|INTEGER|0||0
11|ended_at|INTEGER|0||0
12|position|INTEGER|1||0
13|created_at|INTEGER|1||0
14|run_id|TEXT|0||0
15|run_status|TEXT|0||0
16|last_run_event_id|TEXT|0||0
17|comment_attachments_json|TEXT|0||0
```

### 불변(invariant) 매트릭스

| 작업 | 필요한 변경 | 검증 |
|------|------------|------|
| 신규 테이블 추가 | `db.ts:52-196` 블록에 `CREATE TABLE IF NOT EXISTS` 추가, 필요 시 인덱스 추가, FK CASCADE 결정 | `pnpm --filter @open-design/daemon test`, 신/구 DB 모두에서 `migrate()` 멱등 확인 |
| 컬럼 추가 | inline 정의 추가 + `PRAGMA table_info` 검사 + `ALTER TABLE` 분기(`db.ts:197-257` 패턴) | 신규 DB: inline 사용; 기존 DB: ALTER 적용. 두 케이스 다 테스트 필요 |
| 인덱스 추가 | `CREATE INDEX IF NOT EXISTS` 한 줄. 컬럼 순서가 selectivity 결정 (앞 컬럼이 equality, 뒤가 range) | SQLite `EXPLAIN QUERY PLAN` 으로 인덱스 사용 확인 |
| FK CASCADE 변경 | SQLite는 `ALTER TABLE`로 FK 변경 불가 — 새 테이블 만들고 데이터 복사 후 swap | 마이그레이션 전후 row count 비교 |
| JSON 컬럼 추가 | `TEXT` 컬럼으로 추가. read 시 `parseJsonOrUndef` (null/invalid silent fail) | 손상 JSON 입력으로 fallback 검증 |

### 성능·리소스 실측

| 쿼리/연산 | 인덱스 | 예상 비용 |
|----------|--------|----------|
| `listMessages` (`SELECT … WHERE conversation_id=? ORDER BY position`) | `idx_messages_conv (conversation_id, position)` | O(log N + K). 300개 메시지 대화: ~0.5-2ms |
| `listConversations` (`WHERE project_id=? ORDER BY updated_at DESC`) | `idx_conv_project` | O(log N + K). 50개 대화 프로젝트: ~0.3-1ms |
| `getDeployment` (UNIQUE 검색) | 자동 UNIQUE 인덱스 | O(log N). ~0.1-0.5ms |
| `listMediaTasks` (`WHERE project_id=? ORDER BY updated_at DESC LIMIT 50`) | `idx_media_tasks_project` | O(log N + 50). ~1-2ms |
| `setTabs` (트랜잭션 DELETE + N × INSERT) | PK `(project_id, name)` | 100 tabs ~5-15ms (트랜잭션). 트랜잭션 없으면 ~수백 ms |
| `upsertMessage` (`INSERT OR REPLACE`) | PK | ~0.5-2ms 단발 |

WAL checkpoint: 기본 auto-checkpoint 1000 pages (~4MB). 활성 사용자가 1000 메시지 추가 시 자동 합병 1-3회 발생. `app.sqlite` 평균 사이즈는 메시지 1000개 기준 ~5-15MB (메시지 평균 ~5-15KB 가정, JSON 컬럼 포함). HTML/이미지 등 큰 페이로드는 모두 `.od/projects/<id>/` 파일이므로 DB 외부 — DB 증가가 선형으로 통제 가능.

`fs.realpath` syscall 비용: macOS APFS ~수십 μs, 경로당 1-3회 호출 (`resolveSafeReal` 1회 + `resolveExistingPrefix` 재귀). 핫패스에서는 캐싱 안 됨 → 매 호출 syscall.
