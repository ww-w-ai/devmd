# DevMD — Development as Markdown

> 소프트웨어 개발 전체를 .md 파일로 표준화하는 오픈 규격
> Domain: devmd.ai (2026-05-13 구입)
> GitHub: github.com/ww-w-ai/devmd (예정)

## 이 프로젝트가 뭔지

Google이 DESIGN.md로 UI 디자인을 표준화했듯, **DevMD는 소프트웨어 개발 전체를 .md 파일로 표준화**하는 오픈 규격이다.

핵심 한 줄: **"사람이 md 파일을 채우면, AI가 서비스를 만든다."**

## 어떻게 이 아이디어가 나왔는지 (전체 맥락)

2026-05-13, top-ai-influencers 서비스를 만드는 세션에서 자연스럽게 도출됨:

1. **top-ai-influencers 서비스 구현** — Discovery(72회 웹 리서치 + C-Suite 10관점) → Plan → 코드 구현 완료 (CF Workers + React 19 + D1)
2. **반복 패턴 발견** — peery, seochi, top-ai 3개 프로젝트에서 동일한 보일러플레이트 반복 (Layout, Tailwind, Workers 엔트리, D1 헬퍼, Legal 페이지 등)
3. **@ww-w/stack 구상** — 공통 코드를 npm 패키지로 추출. private npm 패키지 방식 확정
4. **DESIGN.md 리서치** — Google이 2026.04 공개한 DESIGN.md 스펙 발견. 4대 소스(awesome-design-md 74.3K stars, designmd.app 454개, getdesign.md 71개, awesome-claude-design 68개) 조사
5. **전 도메인 확산 발견** — DESIGN.md 패턴이 백엔드(Kiro), QA(BMAD), 접근성(ACCESSIBILITY.md), 보안(specs.md), 기획(specs.md) 등으로 확산 중. 5개 프레임워크(Kiro, specs.md, BMAD, MAGI, ACCESSIBILITY.md) 딥다이브
6. **빈자리 3+1개 발견** — UI.md, INFRA.md, RUNTIME.md, SEO.md는 아직 아무도 표준을 제안하지 않음
7. **38개 도메인 매핑** — 10개 소프트웨어 방법론(SDLC, Agile, DevOps, DDD, Clean Architecture, 12-Factor, OWASP, WCAG, SRE, Product Management)에서 38개 도메인 추출 → AI 가치 최상위 17개 선별
8. **DevMD 명명** — "Development as Markdown". devmd.ai 도메인 확보

## 회사 미션과의 연결

ww-w.ai 미션: **"Do it once, share with all"** — 중복 토큰 사용을 줄여 유저·사회·환경에 기여.

Token Saving 3레벨 중 **도구 레벨**:
- 10억 명이 각자 프로젝트 스펙을 정의하는 대신, DevMD 템플릿 채우기
- AI가 DevMD를 읽고 구현 → 시행착오 토큰 절감
- "에이전트를 각자 처음부터 만드는 시행착오 N회 → DevMD 템플릿 1회"

## 17개 표준 파일

### 기존 표준 채택 (4개)

| 파일 | 출처 | 영역 |
|---|---|---|
| CLAUDE.md | Anthropic | 코딩 컨벤션 + 프로젝트 규칙 |
| AGENTS.md | AAIF (Linux Foundation) | AI 에이전트 지시 |
| DESIGN.md | Google Labs (2026.04) | UI 디자인 토큰 |
| SECURITY.md | GitHub | 보안 정책 |

### DevMD 오리지널 (9개)

| 파일 | 영역 |
|---|---|
| PRODUCT.md | 제품 비전, 타겟 유저, 가치 제안, 성공 기준 |
| GLOSSARY.md | 도메인 용어 사전 (DDD 유비쿼터스 언어) |
| ARCHITECTURE.md | 시스템 구조, 레이어, 의존성 방향, ADR |
| SCHEMA.md | DB 테이블, 관계, 인덱스, 마이그레이션 규칙 |
| API.md | 엔드포인트 네이밍, 에러 형식, 버전 정책 |
| ERRORS.md | 에러 코드 체계, 예외 계층, 재시도 정책 |
| LOGGING.md | 로그 레벨, 구조화 형식, 트레이스 전파 |
| TESTING.md | 테스트 피라미드, 커버리지 기준, 범위 |
| BRAND.md | 브랜드 보이스, 톤, 카피 규칙 |

### DevMD 고유 제안 — 업계 빈자리 (4개)

| 파일 | 영역 | 왜 빈자리인가 |
|---|---|---|
| **UI.md** | 프론트엔드 구조/레이아웃/플로우 | DESIGN.md는 시각만, 구조 스펙 없었음 |
| **SEO.md** | SEO + GEO 전략 | seochi 실전 노하우 기반. 선언적 SEO 스펙 없었음 |
| **INFRA.md** | 인프라 의도 정의 | Terraform=HOW(코드), 의도(WHAT) 레벨 스펙 없었음 |
| **RUNTIME.md** | 에이전트 실행 명세 | 에이전트 정의(BMAD)는 있지만 실행 스펙 없었음 |

## 설계 철학 (3가지)

1. **체크리스트 효과**: 빈 템플릿의 모든 섹션이 "이걸 정의했나?"를 AI에게 묻는다. 항목이 있으므로 놓치지 않음
2. **추상화로 토큰 절감**: `type: card-feed`라고 쓰면 AI가 카드 리스트의 모든 디테일을 이미 알고 생성. 페이지당 ~4,000 토큰 절감
3. **`@파일#섹션` 참조**: 다른 .md 파일을 참조하여 중복 제거. 한 곳에서 수정하면 전체 반영

## 경쟁 환경 분석 (딥다이브 완료)

### 가장 참고할 것들

| 프레임워크 | 핵심 배울 점 | 우리와의 차이 |
|---|---|---|
| **BMAD-METHOD** | 7단계 에이전트 부팅 시퀀스, 3-layer 커스터마이제이션(base/team/user), persistent_facts, menu dispatch | BMAD는 정의 표준(정적), DevMD는 전체 개발 스펙 + 실행까지 |
| **Kiro (AWS)** | 3파일 체계(requirements/design/tasks), EARS 수락 기준, 양방향 싱크 | Kiro는 IDE 종속($20-200/mo), DevMD는 도구 비종속 오픈 표준 |
| **specs.md (fabriqai)** | Bolt(인텐트 단위 실행), ADR, AI-DLC 3단계 | specs.md는 프레임워크, DevMD는 파일 규격 표준 |
| **MAGI** | ai-script 코드블록(문서 내 LLM 실행 지시), typed relationship | RC 단계, 채택 미미. 개념만 참고 |
| **ACCESSIBILITY.md** | CI 게이트 통합, AGENTS.md 정렬 포맷 | 실험 단계(48 stars). 패턴만 참고 |
| **Google DESIGN.md** | YAML frontmatter + markdown 본문 2층 구조 | DevMD가 이 포맷을 전 도메인으로 확장 |

### DevMD의 포지셔닝

```
Google DESIGN.md  = 디자인 1개 파일의 표준 규격
BMAD-METHOD       = 에이전트 정의의 베스트 프랙티스
Kiro              = IDE 종속 3파일 시스템 (유료)
specs.md          = 방법론 프레임워크

DevMD             = 소프트웨어 개발 전체의 .md 파일 표준 규격 (오픈, 무료, 도구 비종속)
```

아무도 "전체를 커버하는 .md 파일 세트"를 표준으로 제안하지 않았음. 이 빈자리가 DevMD.

## RUNTIME.md 상세 — 에이전트 실행 스펙

BMAD가 "에이전트를 정의하는 법"이라면, RUNTIME.md는 "정의된 에이전트를 돌아가게 만드는 실행 명세서". Dockerfile이 앱을 만드는 게 아니라 "이미 만든 앱을 어떻게 띄울지" 정의하듯.

```
정의 레이어 (BMAD/SKILL.md)  →  뭘 하는지
실행 레이어 (RUNTIME.md)     →  언제, 어떻게, 어떤 조건으로
런타임 (AgentRunner)         →  실제로 돌리는 인프라
```

첫 구현 대상: madori 마케팅 에이전트 팀 (10개 에이전트) 자동화.
상세 스펙은 wiki: `10_프로젝트 기획/컨셉/에이전트-런타임-스펙-표준화.md`

## UI.md 상세 — DESIGN.md의 컴패니언

```
DESIGN.md = 어떻게 생겼는지 (시각: 색상, 폰트, 간격)
UI.md     = 무엇이 어디에 있는지 (구조: 레이아웃, 플로우, 패턴)
```

추상화 타입(`card-feed`, `tab-filter`, `email-cta` 등)을 정의하여 타입명 하나가 수백 토큰의 디테일을 대체.
노코드 툴이 GUI로 하던 것을 마크다운으로 선언. 더 정밀하고 버전 관리되고 AI가 읽기 쉬움.

## SEO.md 상세 — seochi 실전 노하우 기반

seochi 프로젝트(가장 시간 많이 투자한 프로젝트 중 하나)에서 실전 구현된 모든 SEO/GEO 패턴 추출:

- SSR HTML-first: AI가 JS 없이 HTML만으로 전체 콘텐츠 확인 (`html_first: true`)
- `/:slug.md` 마크다운 엔드포인트: AI 에이전트가 raw markdown으로 직접 접근
- AI 크롤러별 명시 허용 + Content-Signal 헤더 (`ai-train=no, search=yes, ai-input=yes`)
- speakable JSON-LD (음성 검색 대응)
- articleBody 전문 JSON-LD 포함 (LLM 인용 최적화)
- News Sitemap 별도 (`/sitemap-news.xml`)
- OG Image 자동 생성 (satori → SVG → resvg-wasm → PNG)
- Cache: 콘텐츠 1년 immutable + CUD 시 자동 퍼지
- hreflang 자동 매핑

## INFRA.md 상세 — Infrastructure as Markdown

```
Terraform    = HOW (인프라를 코드로)
INFRA.md     = WHAT (인프라를 의도로)
```

AI가 INFRA.md 읽고 → wrangler.toml / Terraform / docker-compose 등 어떤 형식이든 생성. 런타임 비종속.
bkend 프로젝트와 직결: 유저가 INFRA.md 작성 → bkend가 자동 프로비저닝.

## DESIGN.md 도구 가이드

DESIGN.md를 기존 사이트에서 자동 추출하는 도구:
- **design.dev** — URL 입력 → 스크린샷+스타일 분석 → DESIGN.md 생성
- **Chrome 확장** (bergside/design-md-chrome) — 웹페이지에서 원클릭 추출
- **Figma 플러그인** (bergside/design-md-figma) — Figma 스타일에서 추출
- **context.dev** — AI 기반 분석 + 생성
- 레퍼런스 컬렉션: awesome-design-md (74.3K stars), designmd.app (454개)

## 기술 스택 (이 프로젝트 자체)

- DevMD 규격 문서: 마크다운 (dogfooding — DevMD 자체가 .md로 정의됨)
- CLI: `npx devmd init` (추후 개발)
- 웹사이트: devmd.ai (CF Workers + Astro 또는 React — @ww-w/stack 사용)
- GitHub: github.com/ww-w-ai/devmd

## 연관 프로젝트

- **@ww-w/stack**: DevMD의 런타임 구현체. DevMD 스펙 파일을 읽어서 코드 생성하는 npm 패키지
- **top-ai-influencers**: DevMD의 첫 번째 실전 검증 프로젝트
- **seochi**: SEO.md 스펙의 노하우 원천
- **noul**: Clean Architecture + Atomic Design 패턴의 원천
- **peery**: CF Workers + React 19 스택의 레퍼런스
- **bkend**: INFRA.md의 비즈니스 연결점
- **AgentRunner**: RUNTIME.md의 실행 환경
- **madori**: RUNTIME.md의 첫 구현 대상 (10개 에이전트 팀 자동화)

## wiki 참조 (상세 분석 문서)

모든 리서치와 상세 분석은 www-wiki에 저장됨:

- `30_참고자료/기술-노하우/md-스펙-파일-v1-세트.md` — 17개 파일 상세 스펙 + UI.md 풀 YAML + SEO.md 풀 YAML
- `30_참고자료/기술-노하우/마크다운-스펙-파일-생태계.md` — 5개 프레임워크 딥다이브 (Kiro, specs.md, BMAD, MAGI, ACCESSIBILITY.md)
- `30_참고자료/기술-노하우/DESIGN-md-도구-가이드.md` — DESIGN.md 도구 5종 + 4대 소스별 베스트 샘플
- `30_참고자료/기술-노하우/ww-w-stack-표준화-구상.md` — @ww-w/stack 패키지 설계
- `10_프로젝트 기획/컨셉/에이전트-런타임-스펙-표준화.md` — RUNTIME.md 상세 + BMAD 참고 패턴 + madori 자동화 계획
- `10_프로젝트 기획/컨셉/DevMD-표준-규격.md` — DevMD 프로젝트 개요
- `10_프로젝트 기획/ww-w-stack-brainstorming/` — @ww-w/stack brainstorming + plan
- `50_노트/회사-법인-회계/미션-비전.md` — 회사 미션 "Do it once, share with all"

## 완료된 것 (2026-05-13)

1. ~~templates/ 디렉토리에 빈 템플릿 생성~~ → **25개 완성** (1,729줄)
2. ~~spec/ 디렉토리에 규격 문서 작성~~ → **26개 완성** (2,388줄, RFC 2119)
3. ~~실전 예시 작성~~ → **Twenty CRM (44K stars) + Documenso (9K stars)** 각 25개 파일
4. ~~gap 분석~~ → **106개 gap 발견, 25개로 93% 해소 증명** (docs/gap-analysis.md)
5. ~~README 작성~~ → Product Hunt급, Why 25 실증 근거 포함
6. ~~HARNESS.md 설계~~ → noul/seochi/madori 분석 기반 13개 섹션
7. ~~DESIGN.md 생태계 조사~~ → 배포 모델, 채택 경로, 도구 현황 파악

## 현재 프로젝트 구조

```
devmd/
├── README.md                           ← 소개 + Why 25 + 티어
├── CLAUDE.md                           ← 이 파일
├── docs/gap-analysis.md                ← 106개 gap 실증 분석
├── spec/ (26개, 2,388줄)               ← 정규 규격 (RFC 2119)
├── templates/ (25개, 1,729줄)          ← 빈 골격
├── examples/documenso/ (25개, 6,369줄) ← 실전 예시
└── examples/twenty/ (25개, 8,294줄)    ← 실전 예시
                             104 파일, ~19,400줄
```

## 25개 표준 파일 (17개 → 25개 확장)

기존 17개 + FLOWS.md, SCREENS.md, HARNESS.md, LIFECYCLE.md, CONFIG.md, DEVOPS.md, OPERATIONS.md, CHANGELOG.md

업계 최초 제안 6개: UI.md, SEO.md, INFRA.md, HARNESS.md, LIFECYCLE.md, RUNTIME.md

## 다음 단계

1. **이 CLAUDE.md를 읽고 맥락 파악** ← 새 세션 시작 시 여기부터
2. **GitHub 공개** — github.com/ww-w-ai/devmd에 push
3. **devmd.ai 웹사이트 구축** — 규격 소개 + 템플릿 다운로드 + 예시
4. **콜드스타트 테스트** — noul 소스코드만으로 DevMD 자동 생성 → `npx devmd scan` feasibility 검증
5. **awesome-devmd 큐레이션** — 커뮤니티 예시 수집 (DESIGN.md는 큐레이션이 spec의 6배 인기)

## 백로그

- **🔥 DESIGN.md → Theme Pipeline (Killer Feature)** — DESIGN.md frontmatter(colors, typography, spacing) 파싱 → CSS custom properties 자동 생성 → Tailwind v4 @theme 주입 → UI 동적 반영. "DESIGN.md 교체 = UI 테마 교체". 테스트베드: thessen-ai-ws. 목표: `npx devmd theme-gen` CLI + guide/scan 통합.
- **CLI 개발** — `npx devmd init` (대화형 템플릿), `npx devmd scan` (코드→DevMD 자동생성), `npx devmd lint` (스펙 검증), `npx devmd check` (파일 간 충돌 감지), `npx devmd export` (DevMD→코드 생성), `npx devmd diff` (버전 비교), `npx devmd theme-gen` (DESIGN.md→CSS)
- **npm 패키지** — @devmd/cli 또는 devmd
- **VS Code 확장** — DevMD 파일 자동완성, 크로스 레퍼런스 네비게이션
- **Chrome 확장** — 기존 사이트에서 DevMD 추출 (DESIGN.md Chrome 확장 참고)
- **Figma 플러그인** — DESIGN.md + UI.md + SCREENS.md 연동
