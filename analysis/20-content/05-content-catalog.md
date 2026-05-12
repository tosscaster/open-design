# 05. 콘텐츠 카탈로그 — skills, design-systems, design-templates, craft, prompt-templates, specs

Open Design은 **소스 코드와 동등하게 중요한 콘텐츠 자산**을 6개 디렉토리로 관리합니다. 데몬은 부팅 시 이 디렉토리들을 파일시스템 스캔으로 카탈로그화하고, frontmatter 메타데이터를 React UI/시스템 프롬프트로 흘려보냅니다.

![콘텐츠 → 데몬 카탈로그 → 시스템 프롬프트](../svg/20-content/05-content-catalog.svg)

## 1. 콘텐츠 레이어 구분

| 디렉토리 | 역할 | 입력 | 출력 |
|---|---|---|---|
| `skills/` | Agent가 mid-task에 invoke하는 기능성 스킬 | 사용자 의도 + 컨텍스트 | 패키지/유틸리티/디자인-시스템 헬퍼 |
| `design-templates/` | 렌더링 카탈로그 (decks, prototypes, image/video/audio) | 활성 스킬 + 디자인시스템 | 단일 HTML/MP4/PNG 아티팩트 |
| `design-systems/` | 브랜드별 디자인 가이드 (`DESIGN.md`) | 시각 디자인 언어 | 색상/타이포/컴포넌트 규칙 (시스템 프롬프트에 주입) |
| `craft/` | 브랜드 무관 보편 규칙 (typography, anti-ai-slop, …) | 모든 출력 | 토큰 효율적 opt-in 규칙 모음 |
| `prompt-templates/` | 이미지/비디오 프롬프트 라이브러리 (JSON) | 사용자 요청 | gpt-image-2 / Seedance / HyperFrames 호출 페이로드 |
| `specs/` | 아키텍처 명세, 로드맵, 결정 기록 | — | (개발자용 도큐먼트) |

> 핵심 구분: **skills 는 "도우미"**, **design-templates 는 "렌더링 모양"**, **design-systems 는 "시각 언어"**, **craft 는 "보편 규칙"**.

## 2. skills/ — 기능성 스킬

100여 개의 디렉토리 (`skills/<id>/SKILL.md`)가 frontmatter `od.mode`별로 분류됩니다.

### 2-1. mode 분포 (실측)

| mode | 개수 | 예 |
|---|---:|---|
| `design-system` | ~36 | creative-director, ad-creative |
| `image` | ~24 | gpt-image-2, midjourney 등 생성 헬퍼 |
| `prototype` | ~21 | dashboard, web-prototype |
| `deck` | ~5 | html-ppt, guizang-ppt |
| `video` | ~8 | seedance, hyperframes |
| `audio` | ~4 | (음악/오디오 생성) |
| `template` | ~8 | digital-eguide |
| `utility` | ~1 | 순수 유틸 |

README는 "31 skills ship in the box"라고 명시하는데, 이는 **사용자 노출용 카탈로그 기준** 수치이며 디스크의 SKILL.md 파일 수와는 약간 다를 수 있습니다(번들된 design-system 헬퍼 등이 포함되는지에 따라).

### 2-2. 표준 폴더 구조

```
skills/<skill-id>/
├── SKILL.md          # 유일한 필수 파일 (frontmatter + 마크다운)
├── assets/           # (선택) 템플릿 HTML, CSS, fonts
├── examples/         # (선택) .html 샘플 → daemon이 derived card로 표면화
└── references/       # (선택) 상세 가이드
```

가장 단순한 스킬은 `SKILL.md` 한 파일(예: `ad-creative`), 가장 큰 스킬은 36개 테마 + 31개 레이아웃 + 15개 full-deck 템플릿 + 키보드 런타임 + 27개 CSS 애니메이션을 포함(`html-ppt`, `guizang-ppt`).

### 2-3. Frontmatter 스키마

```yaml
---
name: <skill-id>
description: <single-line description>
triggers:
  - "keyword1"
  - "keyword2"
od:
  # 1. 분류
  mode: image|video|audio|deck|design-system|template|prototype|utility
  category: <category-slug>
  scenario: design|marketing|operation|engineering|product|finance|hr|sale|personal
  platform: desktop|mobile|null
  fidelity: wireframe|high-fidelity|null

  # 2. 광고/표시
  featured: <int>            # 매거진 행 순서 (낮을수록 우선)
  default_for: [<surface>]   # 예: [deck]
  upstream: "https://github.com/..."
  example_prompt: "..."      # "Use this prompt" 버튼 텍스트
  speaker_notes: <bool>
  animations: <bool>

  # 3. 컨텍스트 주입
  craft:
    requires: [typography, color, anti-ai-slop, …]
  design_system:
    requires: <bool>
    sections: [color, typography, components, …]

  # 4. 미리보기
  preview:
    type: html
    entry: index.html
---
```

### 2-4. 데몬과의 통합 (`apps/daemon/src/skills.ts`)

핵심 함수:

```typescript
// 모든 스킬 디렉토리를 재귀 스캔, frontmatter 정규화, SkillInfo[] 반환
async function listSkills(skillsRoots: string[]): Promise<SkillInfo[]>;

// SkillInfo: id, name, description, triggers, mode, surface, platform,
//   scenario, category, fidelity, craftRequires, designSystemRequired,
//   previewType, featured, upstream, examplePrompt, aggregatesExamples,
//   body (정규화된 frontmatter), dir (디스크 경로)

// frontmatter 값 검증 + 폴백 (mode 누락 시 본문/description으로 추론)
function normalizeMode() / normalizeSurface() / normalizeScenario();

// examples/*.html 발견 → derived card 생성 (예: examples/demo.html → <parent>:demo)
function collectDerivedExamples(dir);

// 첨부 파일 있는 스킬에 경로 프리앰블 추가
//   .od-skills/<folder>/ (CWD 별칭) + 절대 경로 폴백
function withSkillRootPreamble(body, dir);
```

**Shadowing 패턴**: `listSkills([USER_SKILLS_DIR, BUILTIN_SKILLS_DIR])` — 첫 루트(사용자)가 두 번째 루트(번들)를 가립니다. 사용자가 빌트인 스킬을 편집하면 데몬이 자동으로 `USER_SKILLS_DIR/<id>/`로 클론하고 그 사본을 우선 노출합니다.

### 2-5. 대표 스킬

- **html-ppt** (mode: deck, featured: 19) — 36개 테마 + 15개 full-deck + speaker notes + presenter mode(`S` 키로 현재/다음/스피커노트/타이머 마그네틱 카드)
- **creative-director** (mode: design-system) — 20+ 방법론(SIT, TRIZ, SCAMPER, Synectics), 3축 평가, 5단계 프로세스
- **ad-creative** (mode: design-system, category: marketing-creative) — 광고 헤드라인/설명/주요 텍스트 반복
- **dashboard** (mode: prototype, scenario: operations) — KPI 카드, 차트, 네비게이션
- **dating-web** (mode: prototype, featured: 5) — 매거진 풍 컨슈머 대시보드

### 2-6. Bundled-Verbatim: guizang-ppt

`design-templates/guizang-ppt/`는 완전 자체-포함 PPT 엔진으로 외부 레포에서 그대로 번들합니다:

- **LICENSE**: MIT, Copyright (c) 2026 op7418 (歸藏) — 라이선스를 폴더 루트에 정확히 보존
- 36개 테마, 15개 전체 데크 템플릿 (pitch-deck, tech-sharing, weekly-report, xhs-post, presenter-mode-reveal 등), 31개 레이아웃, 27개 CSS 애니메이션 + 20개 canvas FX
- Presenter Mode: 자기 스크립트(逐字稿) 지원, `S` 키로 현재/다음/스피커노트/타이머 마그네틱 카드 팝업

## 3. design-systems/ — 브랜드 디자인 가이드

150여 개 디렉토리 (`design-systems/<brand>/DESIGN.md`). 각 폴더에는 **DESIGN.md 한 파일만** 있습니다 (~5-10 KB 마크다운, frontmatter 없음 — 데몬이 디렉토리 이름으로 brand id 유도).

### 3-1. 표준 섹션

1. 제목 + 카테고리 — `# Design System Inspired by <Brand>`
2. **Visual Theme & Atmosphere** — 시각 철학 (다크/라이트, 미니멀/복잡함)
3. **Color Palette & Roles** — primary, secondary, accent, surface, neutrals, semantic, gradients (CSS 변수 형태)
4. **Typography Rules** — 폰트 패밀리, 가중치, 크기 계층, 행간, 자간 규칙 테이블
5. **Components** — 버튼, 입력, 카드, 네비게이션 스타일
6. **Voice & Tone** — 텍스트 원칙
7. **Additional Sections** — motion, icons, spacing, borders 등 (브랜드별 차이)

### 3-2. 대표 브랜드

- **기술**: Claude, OpenAI, Anthropic, Hugging Face, Mistral AI, Cohere, Together AI, Replicate, Ollama
- **SaaS/생산성**: Linear, Notion, Slack, Figma, Framer, Webflow, Intercom, Raycast, Lovable, Posthog
- **커머스/리테일**: Shopify, Stripe, Airbnb, Uber, Lyft, Wise, Revolut
- **엔터테인먼트**: Spotify, Netflix, YouTube, Discord, Twitch
- **자동차**: Tesla, BMW, Ferrari, Lamborghini, Bugatti, Renault
- **기타**: Apple, Google (Material), Microsoft (Fluent), Duolingo, Xiaohongshu(小红书), Wechat

### 3-3. 두 가지 예시

**Linear** (`design-systems/linear-app/DESIGN.md`)
- 다크 모드 네이티브: `#08090a` 배경, `#0f1011` 패널
- Inter Variable + `"cv01", "ss03"` OpenType 피처
- 서명 가중치 510 (Regular와 Medium 사이)
- 인디고-바이올렛 단일 액센트: `#5e6ad2` (bg) / `#7170ff` (interactive)
- 초박형 반투명 흰 테두리: `rgba(255,255,255,0.05)`

**Airbnb** (`design-systems/airbnb/DESIGN.md`)
- Rausch coral-pink 단일 액센트: `#ff385c`
- Airbnb Cereal VF 단일 폰트 패밀리 (모든 크기)
- 전전폭 4:3 사진 (hero scale)
- 제품 계층 색상 코딩 (Plus magenta, Luxe purple)
- Guest Favorite 수상 로고

## 4. design-templates/ — 렌더링 카탈로그

100여 개 렌더링 템플릿. skills와 달리 **출력 모양** 중심입니다.

### 4-1. mode 분포

| mode | 개수 | 예 |
|---|---:|---|
| `deck` | ~55 | html-ppt 내 15개 full-deck + guizang-ppt + 단독 데크 |
| `prototype` | ~43 | dashboard, blog-post, dating-web, finance-report |
| `template` | ~2 | digital-eguide |
| `video` | ~2 | hyperframes |
| `image` | ~1 | poster |
| `audio` | ~1 | |

### 4-2. 표준 구조

```
design-templates/<template-id>/
├── SKILL.md              # Frontmatter + 디자인 워크플로우
├── assets/               # CSS, fonts, runtime JS, 테마
├── templates/            # 또는 examples/ — 레이아웃, 전체 데크 샘플
├── scripts/              # 빌드/렌더 스크립트
├── references/           # 상세 카탈로그, 저작 가이드
├── examples/             # 미리-구워진 .html 샘플 (derived card)
├── README.md             # 여러 언어
└── LICENSE               # (선택)
```

### 4-3. skills/ 와 design-templates/ 차이

| 차원 | skills/ | design-templates/ |
|---|---|---|
| 역할 | Mid-task invoke 기능 (유틸, 도움말) | Artifact 렌더링 형태 |
| 주요 mode | design-system, utility | prototype, deck, template |
| 구조 | 단순 SKILL.md (대부분) | 복잡: assets + templates + scripts + examples |
| 출력 | 조언, 패키지, 참조 링크 | 렌더링된 HTML/정적 페이지 |
| `design_system.requires` | false | true (색상/타이포 주입) |
| `featured` | 적음 (갤러리 미포함) | 많음 (매거진 갤러리) |
| API | `/api/skills*` | `/api/design-templates*` (분리 계획 — `specs/current/skills-and-design-templates.md`) |

## 5. craft/ — 보편 디자인 규칙

12개 마크다운 파일. **브랜드 무관 보편 가이드라인**으로, 스킬이 `od.craft.requires`로 opt-in합니다.

### 5-1. 항목

| 파일 | 섹션명 | 언제 필요 |
|---|---|---|
| `typography.md` | typography | 모든 타이핑 스킬 |
| `typography-hierarchy.md` | typography-hierarchy | 강한 진입점, 다양한 레벨 |
| `typography-hierarchy-editorial.md` | typography-hierarchy-editorial | 블로그/문서/이북 |
| `color.md` | color | 모든 스타일 출력 |
| `anti-ai-slop.md` | anti-ai-slop | 마케팅/랜딩/데크 (**P0: auto-lint**) |
| `state-coverage.md` | state-coverage | 상태 있는 UI (대시보드/폼/테이블) |
| `animation-discipline.md` | animation-discipline | 모션 (모바일 앱/마이크로인터랙션) |
| `accessibility-baseline.md` | accessibility-baseline | 상호작용 UI |
| `rtl-and-bidi.md` | rtl-and-bidi | 로컬라이제이션 |
| `form-validation.md` | form-validation | 폼 |
| `laws-of-ux.md` | laws-of-ux | 인지 제약 (Hick's law, Choice Overload, Zeigarnik) |

### 5-2. Opt-in 모델

스킬 frontmatter:
```yaml
od:
  craft:
    requires: [typography, color, anti-ai-slop]
```

데몬은 **요청된 섹션만 시스템 프롬프트에 주입** → 토큰 효율성. 모든 규칙을 항상 주입하지 않음.

### 5-3. 시행 수준

- **Auto-checked (P0/P1 배지)**: `anti-ai-slop.md`의 P0 규칙들이 데몬의 lint-artifact 로직에 인코딩됨 — Tailwind indigo accent, two-stop hero gradients, emoji-as-icons 등 사용 시 빌드 실패
- **Guidance**: 나머지 — agent가 읽고 따르되, linter는 검사하지 않음

## 6. prompt-templates/ — 이미지/비디오 프롬프트

100여 개 JSON 프롬프트. `image/`(gpt-image-2 ~43-45개) + `video/`(Seedance ~39-57개) + 11개 HyperFrames(README는 43+39+11=93로 명시).

### 6-1. 구조

```json
{
  "id": "3d-stone-staircase-evolution-infographic",
  "surface": "image",
  "title": "...",
  "summary": "...",
  "category": "Infographic",
  "tags": ["3d-render"],
  "model": "gpt-image-2",
  "aspect": "1:1",
  "prompt": {
    "type": "...",
    "instruction": "...",
    "style": "...",
    "layout": "...",
    "centerpiece": "..."
  },
  "previewImageUrl": "https://...",
  "source": {
    "repo": "YouMind-OpenLab/awesome-gpt-image-2",
    "license": "CC-BY-4.0",
    "author": "知识猫图解",
    "url": "https://x.com/..."
  }
}
```

각 프롬프트는 **저자/라이선스/저장소 출처**를 추적해 attribution을 잃지 않습니다.

## 7. specs/ — 아키텍처 명세

**주요 파일**:
- `specs/current/skills-and-design-templates.md` — `skills/`와 `design-templates/` 분리 MVP 리팩터(Phase 0: 파일시스템 분리 + API 미러링, Phase 1: Skills CRUD UI, Phase 2: 폴더/ZIP 임포트)
- `specs/current/architecture-boundaries.md` — apps/daemon/, packages/*, tools/* 간 경계
- `specs/current/runtime-adapter.md` — 에이전트 런타임 어댑터
- `specs/current/critique-theater.md` — 크리틱 피드백 시스템
- `specs/current/maintainability-roadmap.md` — 유지보수성 로드맵 (루트 AGENTS.md 참조)

## 8. 데몬 측 카탈로그 API (추정)

`apps/daemon/src/skills.ts` 기반으로 추측되는 라우트:

- `GET /api/skills` — 모든 스킬 카탈로그
- `GET /api/skills/:id` — 단일 스킬 또는 derived card
- `GET /api/skills/:id/example` — `resolveDerivedExamplePath()` → HTML 미리보기
- `POST /api/skills` (Phase 1) — 새 스킬 작성
- `POST /api/skills/import-folder`, `/import-zip` (Phase 2)
- `GET /api/design-systems` — 모든 브랜드 카탈로그
- `GET /api/design-systems/:brand` — DESIGN.md 본문
- `GET /api/design-templates` (Phase 0 후 분리) — 렌더링 카탈로그

## 9. 결론

Open Design은 **콘텐츠를 코드와 동등한 1급 시민**으로 다룹니다. frontmatter 정규화 + shadowing + derived examples + craft opt-in 모델이 결합되어, 사용자가 콘텐츠를 자유롭게 편집/추가/오버라이드 하더라도 시스템 프롬프트 토큰은 효율적으로 유지되고, 빌트인 자산은 유실되지 않습니다.
