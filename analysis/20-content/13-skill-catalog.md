# 13. 스킬 카탈로그 내부 동작 — frontmatter 정규화, shadowing, derived examples

`apps/daemon/src/skills.ts`는 Open Design의 콘텐츠 시스템 심장입니다. 디스크에 있는 100여 개의 `SKILL.md`를 파싱하고, frontmatter를 정규화하고, 사용자가 빌트인 스킬을 편집하면 shadow 폴더로 클론하고, `examples/*.html`을 derived card로 표면화합니다.

![listSkills 흐름과 shadowing](../svg/20-content/13-skill-shadowing.svg)

## 1. listSkills — 진입 함수

`apps/daemon/src/skills.ts:122-275`:

```typescript
export async function listSkills(skillsRoots: string | string[]): Promise<SkillInfo[]> {
  const roots = Array.isArray(skillsRoots) ? skillsRoots : [skillsRoots];
  const seenIds = new Set<string>();
  const out: SkillInfo[] = [];

  for (let rootIdx = 0; rootIdx < roots.length; rootIdx++) {
    const root = roots[rootIdx];
    const source: SkillSource = rootIdx === 0 ? "user" : "built-in";

    let entries;
    try { entries = await readdir(root, { withFileTypes: true }); }
    catch { continue; }

    for (const entry of entries) {
      if (!entry.isDirectory() && !entry.isSymbolicLink()) continue;
      const dir = path.join(root, entry.name);
      const skillPath = path.join(dir, "SKILL.md");

      try { await stat(skillPath); }
      catch { continue; }

      const raw = await readFile(skillPath, "utf8");
      const { data, body } = parseFrontmatter(raw);

      // 1. ID 도출 + shadow 검사
      const rawName = typeof data.name === "string" && data.name.trim()
        ? data.name.trim()
        : entry.name;
      const parentId = rawName;
      if (seenIds.has(parentId)) continue;        // shadowing!
      seenIds.add(parentId);

      // 2. 정규화
      const description = String(data.description ?? '');
      const od = (data.od as Record<string, unknown>) ?? {};
      const mode = normalizeMode(od.mode, body, description);
      const surface = normalizeSurface(od.surface, mode);
      const platform = normalizePlatform(od.platform);
      const scenario = normalizeScenario(od.scenario, body, description);
      const category = normalizeCategory(od.category);
      const triggers = normalizeTriggers(data.triggers);
      const craftRequires = normalizeCraftRequires(od.craft);

      // 3. 부수 파일 발견 + 프리앰블
      const hasAttachments = await dirHasAttachments(dir);
      const bodyWithPreamble = hasAttachments
        ? withSkillRootPreamble(body, dir)
        : body;

      // 4. 부모 카드 push
      out.push({
        id: parentId, name: parentId, description, triggers,
        mode, surface, source, craftRequires, platform, scenario, category,
        previewType: 'html',
        designSystemRequired: od.design_system?.requires ?? true,
        defaultFor: normalizeDefaultFor(od.default_for),
        upstream: typeof od.upstream === 'string' ? od.upstream : null,
        featured: typeof od.featured === 'number' ? od.featured : null,
        fidelity: normalizeFidelity(od.fidelity),
        speakerNotes: od.speaker_notes ?? null,
        animations: od.animations ?? null,
        examplePrompt: typeof od.example_prompt === 'string' ? od.example_prompt : null,
        aggregatesExamples: false,                // 아래에서 update
        body: bodyWithPreamble,
        dir,
      });

      // 5. derived examples
      const derivedExamples = await collectDerivedExamples(dir);
      if (derivedExamples.length > 0) {
        out[out.length - 1].aggregatesExamples = true;
        for (const example of derivedExamples) {
          out.push({
            id: `${parentId}:${example.key}`,
            name: humanizeExampleName(example.key),
            // 부모의 모드/플랫폼/시나리오/설명 상속
            description, mode, surface, source, platform, scenario, category,
            triggers,
            featured: null,                       // magazine row 제외
            aggregatesExamples: false,            // 연쇄 파생 차단
            body: bodyWithPreamble,
            dir,
            // ... 기타 필드
          });
        }
      }
    }
  }

  return out;
}
```

핵심 흐름 6단계:
1. 루트 정규화 (배열 또는 단일 문자열)
2. 각 루트 순회, `readdir` + `SKILL.md` 존재 검증
3. Frontmatter 파싱 (parentId 도출)
4. **Shadowing** — `seenIds`에 이미 등록된 ID 건너뜀
5. 메타데이터 정규화 + 프리앰블 주입
6. Derived examples 수집 + 부모 카드의 `aggregatesExamples` 플래그 갱신

## 2. parseFrontmatter — 의존성 제로 YAML 파서

`apps/daemon/src/frontmatter.ts:1-160`:

```typescript
const FRONTMATTER_RE = /^---\r?\n([\s\S]*?)\r?\n---\r?\n?([\s\S]*)$/;

export function parseFrontmatter(raw: string): { data: Record<string, unknown>; body: string } {
  const m = FRONTMATTER_RE.exec(raw);
  if (!m) return { data: {}, body: raw };
  const yaml = m[1];
  const body = m[2] ?? '';
  const data = parseYaml(yaml);
  return { data, body };
}
```

`parseYaml`(26-144행)은 외부 라이브러리(`yaml`, `js-yaml`) 미사용. 직접 스택 기반 재귀 파서:
- 들여쓰기 추적으로 중첩 구조 인식
- 배열 (`- `) 및 키-값 쌍 구분
- 블록 리터럴 (`|`, `>`) 멀티라인 문자열
- 따옴표 친화적 타입 강제 (`"42"`는 string으로 유지, `42`만 number)

타입 강제(`coerce`, 147-159행)는 빈 문자열 → null, `true|false` → boolean, 숫자 → Number.

## 3. SkillInfo 인터페이스 전체

`apps/daemon/src/skills.ts:51-80`:

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string | 정규화된 스킬 식별자 (`name` 또는 폴더명) |
| `name` | string | 표시명 (id와 동일) |
| `description` | string | frontmatter `description` |
| `triggers` | string[] | 정렬된 트리거 |
| `mode` | `image\|video\|audio\|deck\|design-system\|template\|prototype\|utility` | 스킬 타입 |
| `surface` | `web\|image\|video\|audio` | 출력 매체 |
| `source` | `user\|built-in` | 출처 태깅 (UI 배지) |
| `craftRequires` | string[] | `od.craft.requires` 슬러그 (소문자, 공백 제거) |
| `platform` | `desktop\|mobile\|null` | 모바일 전용 필터 |
| `scenario` | string | 업무 맥락 (finance, marketing, design, …) |
| `category` | string? | 필터 슬러그 (소문자 + 대시) |
| `previewType` | string | 예제 MIME (기본 `"html"`) |
| `designSystemRequired` | boolean | 기본 true |
| `defaultFor` | string[] | 특정 프로젝트 종류에 기본 선택 |
| `upstream` | string? | 출처 URL |
| `featured` | number? | magazine row 순서 (낮을수록 상단) |
| `fidelity` | `wireframe\|high-fidelity\|null` | 프로토타입 힌트 |
| `speakerNotes` | boolean?/string? | 덱 힌트 |
| `animations` | boolean?/string? | 덱 힌트 |
| `examplePrompt` | string? | "Use this prompt" 버튼 |
| `aggregatesExamples` | boolean | `examples/*.html` 존재 시 true |
| `body` | string | 프리앰블 포함 본문 |
| `dir` | string | 절대 경로 |

## 4. 정규화 함수들

### normalizeMode (skills.ts:527-533)
```typescript
function normalizeMode(value, body, description): SkillMode {
  if (value === "image" | "video" | ... "prototype") return value;
  return inferMode(body, description);
}

function inferMode(body, description): SkillMode {
  const hay = `${description}\n${body}`.toLowerCase();
  if (/\bimage|poster|illustration|photography|图片|海报|插画/.test(hay)) return "image";
  if (/\bvideo|motion|shortform|animation|视频|动效|短片/.test(hay)) return "video";
  if (/\baudio|music|jingle|tts|sound|音频|音乐|配音|音效/.test(hay)) return "audio";
  if (/\bppt|deck|slide|presentation|幻灯|投影/.test(hay)) return "deck";
  if (/\bdesign[- ]system|\bdesign\.md|\bdesign tokens/.test(hay)) return "design-system";
  if (/\btemplate\b/.test(hay)) return "template";
  return "prototype";    // 기본값
}
```

영어 + 한자(`视频`, `设计系统`) 다국어 키워드 지원.

### normalizeScenario (skills.ts:589-606)
오타/누락 시 본문 키워드 추론 — `finance` → revenue/balance/income, `marketing` → campaign/cta/persona, …

### normalizeCategory (skills.ts:581-587)
```typescript
const value = String(raw).toLowerCase().trim();
if (!/^[a-z0-9][a-z0-9-]*$/.test(value)) return null;
return value;
```

소문자 + 대시 + 숫자만 허용.

### normalizeCraftRequires (skills.ts:444-456)
```typescript
function normalizeCraftRequires(rawCraft): string[] {
  if (!rawCraft || typeof rawCraft !== 'object') return [];
  const requires = (rawCraft as Record<string, unknown>).requires;
  if (!Array.isArray(requires)) return [];
  const out = new Set<string>();
  for (const item of requires) {
    if (typeof item !== 'string') continue;
    const slug = item.trim().toLowerCase();
    if (slug && /^[a-z0-9][a-z0-9-]*$/.test(slug)) out.add(slug);
  }
  return Array.from(out).sort();
}
```

frontmatter `od.craft.requires: [typography, anti-ai-slop]` → 검증된 슬러그 배열.

## 5. Shadowing 패턴

### 5-1. 다중 루트 우선순위

```typescript
const roots = Array.isArray(skillsRoots) ? skillsRoots : [skillsRoots];
for (let rootIdx = 0; rootIdx < roots.length; rootIdx++) {
  const source: SkillSource = rootIdx === 0 ? "user" : "built-in";
  // ...
  if (seenIds.has(parentId)) continue;   // 이미 본 ID는 스킵
  seenIds.add(parentId);
}
```

`listSkills([USER_SKILLS_DIR, BUILTIN_SKILLS_DIR])` 호출 시:
- USER_SKILLS_DIR(첫 루트, `.od/skills/`)이 우선
- 같은 ID가 BUILTIN_SKILLS_DIR에도 있으면 건너뜀
- 결과 `source: "user"`로 마킹되어 UI에 "수정됨" 배지 표시

### 5-2. 자동 클론 — updateUserSkill

`apps/daemon/src/skills.ts:791-846`. 사용자가 빌트인 스킬을 처음 편집하면:

```typescript
export async function updateUserSkill(
  userRoot: string,
  builtinRoot: string,
  skillId: string,
  patch: { body?: string; frontmatter?: object },
) {
  const userDir = path.join(userRoot, skillId);
  const exists = await dirExists(userDir);

  if (!exists) {
    const builtinDir = path.join(builtinRoot, skillId);
    await mkdir(userDir, { recursive: true });
    await cloneSkillSideFiles(builtinDir, userDir);   // 부수 파일 복사
  }

  // 사용자 편집 SKILL.md만 신규 폴더에 작성
  const newSkillMd = buildSkillMd(patch.body, patch.frontmatter);
  await writeFile(path.join(userDir, 'SKILL.md'), newSkillMd, 'utf8');
}
```

### 5-3. cloneSkillSideFiles (skills.ts:853-875)

`SKILL.md`를 **제외하고** `assets/`, `references/`, `examples/`, `scripts/` 등을 복사:

```typescript
async function cloneSkillSideFiles(src: string, dst: string) {
  const entries = await readdir(src, { withFileTypes: true });
  for (const entry of entries) {
    if (entry.name === 'SKILL.md') continue;
    const srcPath = path.join(src, entry.name);
    const dstPath = path.join(dst, entry.name);
    if (entry.isDirectory()) {
      await mkdir(dstPath, { recursive: true });
      await cloneSkillSideFiles(srcPath, dstPath);
    } else if (entry.isFile()) {
      await copyFile(srcPath, dstPath);
    }
  }
}
```

결과: 사용자 편집 SKILL.md + 원본 부수 파일 — 다음 `listSkills()` 호출 시 user 루트가 우선되어 shadow 활성화.

## 6. collectDerivedExamples

`apps/daemon/src/skills.ts:290-309`:

```typescript
async function collectDerivedExamples(dir: string): Promise<DerivedExample[]> {
  const examplesDir = path.join(dir, "examples");
  let entries;
  try { entries = await readdir(examplesDir, { withFileTypes: true }); }
  catch { return []; }

  const out: DerivedExample[] = [];
  for (const entry of entries) {
    if (!entry.isFile()) continue;
    if (!entry.name.toLowerCase().endsWith(".html")) continue;
    const key = entry.name.replace(/\.html$/i, "");
    if (!isSafeExampleKey(key)) continue;
    out.push({ key });
  }
  out.sort((a, b) => a.key.localeCompare(b.key));
  return out;
}

function isSafeExampleKey(key: string): boolean {
  if (!key || key.startsWith(".")) return false;
  if (key.includes(":")) return false;                   // 합성 ID와 충돌 방지
  return /^[A-Za-z0-9._-]+$/.test(key);
}
```

`examples/demo.html`과 `examples/advanced.html`이 있으면:
- 부모 카드 `id="live-artifact"`, `aggregatesExamples=true`
- 파생 카드 `id="live-artifact:advanced"`, `name="Advanced"`
- 파생 카드 `id="live-artifact:demo"`, `name="Demo"`

파생 카드는 부모의 모드/플랫폼/시나리오/설명을 상속하되 `featured: null` (magazine row 제외)과 `aggregatesExamples: false` (연쇄 파생 차단)을 가짐.

## 7. withSkillRootPreamble

`apps/daemon/src/skills.ts:379-415`:

```typescript
function withSkillRootPreamble(body: string, dir: string): string {
  const referencedFiles = collectReferencedSideFiles(body);
  if (referencedFiles.length === 0) return body;

  const folder = path.basename(dir);
  const skillRootRel = `${SKILLS_CWD_ALIAS}/${folder}`;     // .od-skills/<folder>/

  const preamble = [
    "> **Skill root (relative to project):** `" + skillRootRel + "/`",
    "> **Skill root (absolute fallback):** `" + dir + "`",
    ">",
    "> This skill ships side files alongside `SKILL.md`. When the workflow",
    "> below references side files such as `" + referencedFiles[0] + "`, prefer the",
    "> relative form rooted at the first path above — e.g. open `" +
      skillRootRel + "/" + referencedFiles[0] + "`.",
    ">",
    "> Known side files in this skill: " +
      referencedFiles.map((f) => "`" + f + "`").join(", ") + ".",
    "",
  ].join("\n");

  return preamble + body;
}

function collectReferencedSideFiles(body: string): string[] {
  const files = new Set<string>();
  const matches = body.matchAll(/\b(?:assets|references)\/[A-Za-z0-9._-]+\b/g);
  for (const match of matches) files.add(match[0]);
  if (/\bexample\.html\b/.test(body)) files.add("example.html");
  return Array.from(files).sort();
}
```

### 왜 두 경로?

`apps/daemon/src/cwd-aliases.ts:2-30`이 정의하는 `SKILLS_CWD_ALIAS`:
- **상대 경로** (`.od-skills/<folder>/`) — 채팅 핸들러가 turn 시작 시 `stageActiveSkill()`로 프로젝트 cwd에 복사. agent의 cwd 내이므로 권한 정책 차단 없음.
- **절대 경로** (fallback) — projectId 없는 API 호출 (`/api/runs` 직접) 또는 스테이징 실패 시. Claude/Copilot에 `--add-dir`로 전달.

## 8. resolveDerivedExamplePath

`apps/daemon/src/skills.ts:340-343`:

```typescript
export function resolveDerivedExamplePath(parentDir: string, childKey: string): string | null {
  if (!isSafeExampleKey(childKey)) return null;
  return path.join(parentDir, "examples", `${childKey}.html`);
}

export function splitDerivedSkillId(id: unknown): DerivedSkillIdParts | null {
  if (typeof id !== "string") return null;
  const idx = id.indexOf(":");
  if (idx <= 0 || idx === id.length - 1) return null;
  const parentId = id.slice(0, idx);
  const childKey = id.slice(idx + 1);
  if (!isSafeExampleKey(childKey)) return null;
  return { parentId, childKey };
}
```

`/api/skills/live-artifact:demo/example` 요청 시:
1. `splitDerivedSkillId("live-artifact:demo")` → `{ parentId, childKey }`
2. `findSkillById(skills, "live-artifact")` → 부모 dir
3. `resolveDerivedExamplePath(parentDir, "demo")` → `<parentDir>/examples/demo.html`
4. `fs.existsSync()` 검증 + asset URL 재쓰기 후 응답

## 9. 데몬 API 라우트

`apps/daemon/src/static-resource-routes.ts:26-550`:

| 엔드포인트 | 메서드 | 기능 |
|---|---|---|
| `/api/skills` | GET | 모든 스킬 목록 (body/dir 제외) |
| `/api/skills/:id` | GET | 특정 스킬 상세 (full body 포함) |
| `/api/skills/import` | POST | 사용자 스킬 생성 |
| `/api/skills/:id` | PUT | 사용자 스킬 편집 (빌트인 shadow 자동 클론) |
| `/api/skills/:id/files` | GET | 스킬 폴더의 파일 트리 (Settings 패널용) |
| `/api/skills/:id/example` | GET | 예제 HTML (6단계 폴백 체인) |
| `/api/skills/:id/assets/*` | GET | 정적 자산 (이미지/CSS/JS) |

## 10. design-systems.ts — 유사 카탈로그

`apps/daemon/src/design-systems.ts:23-54`. SKILL.md 대신 `DESIGN.md` 스캔:
- H1 헤딩에서 제목 추출
- `> Category: <name>` 블록쿼트에서 카테고리 파싱
- 색상 토큰 추출 정규식 (`/^[\s>*-]*\**([A-Za-z...]):\s*(#[0-9a-fA-F]{3,8})/gm`)으로 swatch 배열 생성

`DesignSystemSummary` 타입 (11-19행):
```typescript
type DesignSystemSummary = {
  id: string;              // 폴더명
  title: string;           // H1
  category: string;        // blockquote
  summary: string;         // 첫 단락 (240자 제한)
  swatches: string[];      // 선택된 4 색상 (#hex)
  surface: DesignSystemSurface;
  body: string;            // 원본
};
```

## 11. Craft 섹션 시스템 프롬프트 주입

스킬 frontmatter `od.craft.requires: [typography, color, anti-ai-slop]` → `craftRequires: ["anti-ai-slop", "color", "typography"]` (정렬됨).

chat 핸들러가 turn 직전에 각 슬러그를 `craft/<slug>.md`로 읽어 `craftBody`로 합쳐 `composeSystemPrompt`에 전달. system.ts(`241-249행`):

```typescript
if (craftBody && craftBody.trim().length > 0) {
  const sectionLabel = Array.isArray(craftSections) && craftSections.length > 0
    ? ` — ${craftSections.join(', ')}`
    : '';
  parts.push(
    `\n\n## Active craft references${sectionLabel}\n\n` +
    `The following craft rules are universal — they apply on top of the active ` +
    `design system above, regardless of brand. On any conflict between a craft ` +
    `rule and a brand DESIGN.md, the brand wins for token values; craft rules ` +
    `still apply to anything the brand does not override...\n\n` +
    craftBody.trim()
  );
}
```

## 12. 모드별 시스템 프롬프트 차이

| mode | DECK_FRAMEWORK 주입 | MEDIA_CONTRACT 주입 | tools.css 처리 |
|---|---|---|---|
| `prototype` | 아니오 | 아니오 | 디자인시스템 CSS 변수 inject |
| `deck` | skill seed 없을 때만 | 아니오 | 디자인시스템 inject |
| `image/video/audio` | 아니오 | 예 | 아니오 (HTML 생성 금지) |
| `design-system` | 아니오 | 아니오 | 아니오 (DESIGN.md만) |
| `template` | 아니오 | 아니오 | 디자인시스템 inject |

`apps/daemon/src/prompts/system.ts:277-282`:
```typescript
const isDeckProject = skillMode === 'deck' || metadata?.kind === 'deck';
const hasSkillSeed = !!skillBody && /assets\/template\.html/.test(skillBody);
if (isDeckProject && !hasSkillSeed) {
  parts.push(`\n\n---\n\n${DECK_FRAMEWORK_DIRECTIVE}`);
}
```

`guizang-ppt`나 `html-ppt`처럼 자기 framework가 있는 스킬은 skill seed 검출되어 generic skeleton 건너뜀.

## 13. 새 빌트인 스킬 추가 절차

### Step 1 — 폴더 구조
```bash
mkdir -p skills/your-skill
touch skills/your-skill/SKILL.md
```

### Step 2 — SKILL.md
```yaml
---
name: "your-skill"
description: "Short one-liner"
triggers:
  - "Trigger phrase"
od:
  mode: prototype                    # 또는 deck, image, video, audio, design-system, template
  surface: web
  platform: desktop                   # 또는 mobile
  scenario: general                   # finance, marketing, design, engineering, …
  category: web-components            # 옵션, lowercase + dashes
  featured: 2                         # 옵션, magazine 순서
  upstream: "https://github.com/owner/repo"
  preview:
    type: html
  craft:
    requires:
      - typography
      - anti-ai-slop
  fidelity: high-fidelity
  speaker_notes: true
  animations: true
  example_prompt: "Build a dashboard with..."
---

## Workflow

Reference `assets/template.html` and `references/checklist.md` as needed.

When you copy the seed template, bind the active design system's tokens into its `:root` block.
```

### Step 3 — 부수 파일 (옵션)
```bash
mkdir -p skills/your-skill/assets skills/your-skill/references skills/your-skill/examples
echo "<!doctype html>..." > skills/your-skill/assets/template.html
echo "# Checklist\n- [ ] P0 ..." > skills/your-skill/references/checklist.md
echo "<!doctype html>..." > skills/your-skill/examples/basic.html
echo "<!doctype html>..." > skills/your-skill/examples/advanced.html
```

### Step 4 — 검증
```bash
curl http://127.0.0.1:7456/api/skills | jq '.skills[] | select(.id == "your-skill")'
curl http://127.0.0.1:7456/api/skills/your-skill | jq
```

### Step 5 — 커밋
```bash
git add skills/your-skill/
git commit -m "feat(skills): add your-skill"
```

### 주의사항

1. **ID 충돌** — frontmatter `name`이 기존 스킬 ID와 겹치면 후속 스캔에서 건너뜀.
2. **부수 파일 경로** — SKILL.md 본문에서 `references/checklist.md` 또는 `assets/template.html` 언급 시 자동으로 `.od-skills/<folder>/` 프리앰블이 inject됨.
3. **Craft requires** — `od.craft.requires: [slug1]`은 `craft/slug1.md` 존재 가정. 누락이면 조용히 무시.
4. **examples 폴더 레이아웃** — `examples/<name>.html` 단일 파일만 derived card가 됨. `examples/<name>/template.html` 같은 폴더 구조는 표면화되지 않음.
5. **Mode 추론** — `od.mode` 생략 시 description/body에서 추론. 명시 권장.

## 14. 요약

`apps/daemon/src/skills.ts`는 다음 5가지 패턴을 결합:

1. **다중 루트 + 첫 루트 우선 shadowing** — 사용자 편집이 빌트인을 가림
2. **frontmatter 정규화 + 본문 추론 폴백** — 누락/오타 안전
3. **부수 파일 발견 + 프리앰블 주입** — agent가 자동으로 정확한 경로 사용
4. **examples/*.html → derived card** — 같은 워크플로우의 여러 변형을 별도 카드로 노출
5. **자동 클론** — 빌트인 편집 시 부수 파일까지 복사해 일관성 유지

`apps/daemon/src/design-systems.ts`가 유사 패턴을 DESIGN.md에 적용하여 디자인시스템 카탈로그도 동일한 방식으로 동작합니다.
