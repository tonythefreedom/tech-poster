# tech-poster

반야 AI 콘텐츠 파이프라인 통합 워크스페이스입니다. `auto-poster`, `tech-blog`, `gdoc-fixer` 세 프로젝트를 Git 서브모듈로 묶어 한 작업 환경에서 관리합니다.

```
tech-poster/                # 빈 래퍼 저장소 (서브모듈 포인터만 보유)
├── auto-poster/    # 콘텐츠 생성 + 멀티 채널 포스팅 오케스트레이터
├── tech-blog/      # Firebase Hosting 기반 기술 블로그 (https://tony.banya.ai)
└── gdoc-fixer/     # AI HTML 문서 편집 + 프레젠테이션 생성 웹앱
```

`gdoc-fixer`만 owner가 다릅니다 (`tonythefreedom` 개인 계정). 클론/push 권한이 다른 두 개와 별개입니다.

---

## 서브모듈

### 1. auto-poster — Python / FastAPI

**역할**: 주제 입력 → 슬라이드 기획 → 이미지·MP4 생성 → 멀티채널 포스팅까지 잇는 오케스트레이터.

**구조**:
- `web_app/main.py` (~85KB) — FastAPI 엔드포인트 + Alpine.js 프론트엔드 단일 거대 파일
- `web_app/services/` — 기능 단위 서비스 모듈
  - `content_generator_service.py` — Gemini 기반 기획안(JSON) 생성, 서브슬라이드 시스템 핵심
  - `pdf2mp4_service.py` (~124KB) — PDF → MP4 (Basic/Smart 모드, Whisper 타이밍, NVENC 가속)
  - `auto_pipeline_service.py` — 7단계 오케스트레이터 + 큐 시스템
  - `crypto_service.py` + `firebase_service.py` — Fernet 대칭키로 service account/client secrets DB 암호화
- `web_app/core/` — SQLAlchemy 모델(`User`, `SecureFile`) + SQLite (`autoposter.db`)
- `core/` — 레거시 CLI 스크립트 (`linkedin_poster.py`, `summarizer.py`)
- `youtube_poster/`, `android/` — 별개의 구버전 CLI / 초기 Android 앱

**배포 환경**:
- 자체 서버에서 `uvicorn main:app --host 0.0.0.0 --port 8000` 으로 운영
- `.env` 에 `GEMINI_API_KEY`, LinkedIn 토큰, `YOUTUBE_API_KEY`, `GITHUB_TOKEN`, `SUPER_ADMIN_*`, `SECRET_KEY`
- 프로덕션: `ENVIRONMENT=production` 으로 두면 보안 키 파일의 로컬 폴백 차단 → `/admin/secure-files` UI 에서 DB 암호화 업로드 필수
- 시스템 의존: `ffmpeg` (PDF→MP4), Whisper (Smart 모드 타이밍)

**Upstream**: `kr-ai-dev-association/auto-poster`

---

### 2. tech-blog — React 19 + Vite

**역할**: 외부 노출용 기술 블로그. 정적 위키(`html/`) + Firestore 동적 콘텐츠 양쪽을 한 라우터로 렌더링.

**구조**:
- `src/App.jsx`, `src/pages/` — React Router 라우트
- `src/pages/Report.jsx` — **가장 복잡한 페이지**. 정적 HTML 위키를 런타임에 정제(수식 처리/CSS 스코핑/이미지 경로 보정). 큰 본문은 `contentUrl` 기반 GCS JSON fetch 분기 포함.
- `src/index.css` — 사이트 전역 스타일 + 모바일 반응형
- `src/components/WikiContent.jsx` — 리포트 본문 래퍼 (이미지 모달 / 공유 버튼)
- `src/services/firebase.js` — Firebase 클라이언트 초기화 (`.env.local` 의 `VITE_FIREBASE_*`)
- `scripts/`
  - `deploy-html-wiki.js` — `html/` 의 로컬 HTML → `static-wiki.json` 생성
  - `generate-report-seo.js` — Firestore + `static-wiki.json` → `dist/report/{id}/index.html` (OG/Twitter 메타)
  - `upload_wiki.py` — Firebase Admin SDK 로 HTML + 이미지 Firestore/Storage 업로드 + GitHub Actions 트리거
  - `trigger-github-deploy.{py,sh}` — 외부 시스템에서 deploy-seo 워크플로우 수동 트리거

**빌드 체인** (`npm run build` 3단계):
1. `deploy-html-wiki` — `html/` 처리 → `static-wiki.json`
2. `vite build` — 클라이언트 번들 (`dist/`)
3. `generate-seo` — Firestore + JSON → 정적 SEO 페이지

각 단계 의존:
- `deploy-html-wiki`: `html/` 디렉터리 존재 + `serviceAccountKey.json`
- `vite build`: `.env.local` 의 `VITE_FIREBASE_*` (없으면 Firebase 미초기화로 클라이언트 깨짐)
- `generate-seo`: Firestore 접근 권한 (`serviceAccountKey.json`)

**배포 환경**:
- **GitHub Actions 가 정식 배포 경로** — `.github/workflows/deploy-seo.yml`
  - 트리거: `workflow_dispatch` / `repository_dispatch (deploy-seo)` / 6 시간 cron
  - Secrets: `VITE_FIREBASE_*` 7개 + `FIREBASE_SERVICE_ACCOUNT`
  - 빌드 → `FirebaseExtended/action-hosting-deploy` 로 hosting 채널 `live` 배포
- 로컬에서 `firebase deploy --only hosting` 으로 직접 배포 시 **`.env.local` 없으면 Firebase 미초기화 빌드가 라이브로 올라가 메인 페이지가 깨짐**. 응급 복구는 GitHub Actions 재실행 (`gh workflow run deploy-seo.yml --repo kr-ai-dev-association/tech-blog --ref main`).
- 별도 워크플로우 `generate-sitemap.yml` — 일 1회 sitemap.xml 갱신
- 호스팅 사이트: `tonys-tech-note` (default URL: `https://tonys-tech-note.web.app`), 커스텀 도메인 `https://tony.banya.ai`

**Upstream**: `kr-ai-dev-association/tech-blog`

---

### 3. gdoc-fixer — React 19 + Vite + Firebase Functions

**역할**: HWP/DOCX/HTML 임포트 → AI 문서 편집 → AI 슬라이드 생성 → 공유 링크/PNG/PDF/DOCX 내보내기 → tech-blog 게시까지의 인하우스 편집 환경.

**구조**:
- `web/` — Vite + React 19 + Zustand + Tailwind v4
- `web/src/`
  - `components/` — `Layout`, `Sidebar`, `MainPanel`, `ContentListPage`, `PreviewIframe`, `SlideEditor`, `ShareView`, `PublishModal` 등
    - `Layout.jsx` — `currentView === 'contents'` 분기로 ContentListPage / 편집기 토글. 시작 시 `useAppStore.currentView` 기본값이 `'contents'` 라 컨텐츠 리스트가 첫 화면.
    - `ContentListPage.jsx` — 세 섹션 순서: **Shared (배포된 링크) → Files → Presentations**
    - `PreviewIframe.jsx` — srcDoc 직전에 `patchYoutubeThumbnails` + `injectMathJax` 적용. iframe `load` 이벤트로 MathJax fallback 강제 주입.
  - `utils/`
    - `injectMathJax.js` — `$$...$$` / `\(...\)` / `\[...\]` 패턴 감지 시 MathJax 로더 + inline-block CSS 자동 주입. LaTeX 영역 내 이중 백슬래시(`\\beta` → `\beta`) 정규화 포함.
    - `geminiApi.js` — Gemini 3-모델 파이프라인 (Flash 콘텐츠 분석 → Flash-Image 이미지 → Pro HTML 슬라이드 조합) + JSON 이스케이프 자동 보정
    - `storage.js` — Firestore 파일 CRUD + 클라이언트 사이드 GCS 이미지 업로드 (service account JWT 직접 서명)
    - `publishApi.js` — `publishToTechBlog` Callable 호출 (20분 timeout)
- `web/functions/` — Firebase Cloud Functions (Gen 2, Node 20)
  - `index.js` — `shareOg` (SSR `/share/:id` OG 메타, iframe srcdoc 안에 MathJax 주입)
  - `publishToTechBlog.js` — tech-blog 게시 함수 (자세한 흐름은 아래 섹션)
- `Code.gs`, `appsscript.json` — Google Docs 연동 Apps Script

**iframe sandbox** (Preview / Share / SlideEditor 공통):
```
allow-scripts allow-same-origin allow-popups allow-popups-to-escape-sandbox
```
`target="_blank"` 링크가 새 탭에서 정상 로드되도록 popup 두 토큰 필수.

**GCS 사용**:
- 이미지 업로드 (`uploadDocumentImages`, `uploadSlideImages`) — 클라이언트가 `bucket-access@banya2025.iam.gserviceaccount.com` 의 service account JWT 로 `banya_public2` bucket 에 직접 PUT
- 큰 본문 분리 저장 — Cloud Function 이 같은 service account credentials 로 `banya_public2/wiki-content/{docId}.json` 에 업로드

**배포 환경**:

| 영역 | 명령 | 비고 |
|------|------|------|
| Hosting | `npm run build && npx firebase deploy --only hosting` | `web/.env` 의 `VITE_*` 가 빌드에 inline. |
| Functions | `npx firebase deploy --only functions:publishToTechBlog,functions:shareOg` | secrets 자동 권한 부여 |
| Functions secrets | `firebase functions:secrets:set <NAME>` | 아래 표 참조 |

| Secret | 값 | 출처 |
|--------|-----|------|
| `GEMINI_API_KEY` | Google Gemini API 키 (서버 사이드) | Google AI Studio |
| `TECH_BLOG_SERVICE_ACCOUNT` | `tonys-tech-note` GCP service account JSON 전체 | tech-blog Firestore 쓰기 (Cloud Datastore User) |
| `GITHUB_DISPATCH_TOKEN` | `kr-ai-dev-association/tech-blog` 의 `Contents: write` PAT | deploy-seo 워크플로우 자동 트리거 |
| `GCS_BUCKET` | `banya_public2` | 큰 본문 GCS 분리 저장용 bucket |
| `GCS_SA_EMAIL` | `bucket-access@banya2025.iam.gserviceaccount.com` | 위 bucket 에 객체 생성 권한 가진 SA |
| `GCS_PRIVATE_KEY` | 해당 SA 의 PEM private key (개행은 실제 `\n`) | 클라이언트 `.env` 의 `VITE_GCS_PRIVATE_KEY` 와 동일 |

**환경 변수** (`web/.env`):
- Firebase 웹 앱 7개 (`VITE_FIREBASE_*`)
- `VITE_GEMINI_API_KEY` — 클라이언트 사이드 Gemini 호출용
- `VITE_GCS_BUCKET`, `VITE_GCS_SA_EMAIL`, `VITE_GCS_PRIVATE_KEY` — 이미지 직접 업로드용 (클라이언트 번들에 포함되므로 권한 최소화 필수)

**Upstream**: `tonythefreedom/gdoc-fixer`

---

## publishToTechBlog 함수 상세

gdoc-fixer 의 HTML 을 tech-blog 의 `static-wiki` 컬렉션에 게시하는 핵심 함수. 함수 한도: `timeoutSeconds: 1200` (20분), `memory: 1GiB`.

**입력 / 출력 한도**:
- `MAX_INPUT_BYTES = 1500 * 1024` — 클라이언트에서 받는 HTML 한도
- `MAX_DOC_BYTES = 1000 * 1024` — 최종 Firestore doc 한도 (1MiB - 안전 마진)
- `INLINE_CONTENT_THRESHOLD = 400 * 1024` — ko/en 한 쪽이라도 이 한도를 넘으면 GCS 분리 저장

**처리 흐름**:

```
[클라이언트] activeFileContent (자기완결 HTML) → callable
      ↓
[함수] 권한 검증 (tony@banya.ai 또는 userProfiles.role === 'super_admin')
      ↓
[함수] 입력 크기 검증
      ↓
[정규화] normalizeHtmlDeterministic(html) — Gemini 호출 X, Node 정규식
   - <!DOCTYPE>/<html>/<head>/<body>/<script>/<style>/<meta>/<title>/<link> 제거
   - 모든 class / inline style 속성 제거 (prose 가 typography 책임)
   - 첫 <h1> 을 헤더 div 로 분리
   - <header>/<footer> wrapper 풀기
   - 최종: <article class="wiki-content"> ... <div class="wiki-html-content prose prose-slate max-w-none ..."> {본문} </div> </article>
      ↓
[번역] translateInChunks(normalizedKo) — Gemini Flash, SSE 스트리밍, 30000 chars 블록 단위 chunk
   - <section/div/h*/table/ul/ol/blockquote/pre/figure/hr> 경계에서 분할
   - 각 chunk 6분 timeout, thinkingBudget: 0
   - 헤더(제목)는 별도 1회 번역, 본문 chunks 순차 번역 후 합침
      ↓
[메타] Gemini Flash 로 ko_title / en_title / slug / excerpt JSON 추출
      ↓
[저장 분기]
   - koBytes/enBytes > 400KB OR 합계 > Firestore 한도 → GCS 분리
     → uploadContentToGcs(docId, ko, en) → banya_public2/wiki-content/{docId}.json
     → Firestore 에는 contentUrl 만 + content: {}
   - 그 외 → Firestore 에 content.{ko,en} inline
      ↓
[Firestore] static-wiki/{slug-shortId6} set (Secondary Admin SDK)
      ↓
[GitHub] repository_dispatch (event_type: deploy-seo)
      ↓ ~1-3분
[tech-blog deploy-seo.yml] 자동 빌드 + Firebase Hosting 배포
```

**`callGemini` 구현 특이사항**:
- Cloud Run outbound idle timeout (300초) 회피를 위해 `streamGenerateContent + alt=sse` 사용
- SSE 이벤트 구분자는 `\r?\n\r?\n` (Google SSE 는 CRLF)
- 응답 `parts` 배열 전체 순회하면서 `thought: true` 인 part 제외, `text` 만 누적 (Gemini 2.5 thinking 모델 대응)
- Pro 는 `thinkingBudget: 0` 거부 → 최소값 128 명시 / Flash 는 0 허용
- transient 에러 (`fetch failed`/`ECONNRESET`/`AbortError`) 자동 1회 재시도 (현재 maxAttempts=1, 옵션으로 확장 가능)

---

## 파이프라인 흐름 요약

콘텐츠를 `tech-blog` 로 흘려보내는 두 가지 경로:

### 경로 A — auto-poster 자동 파이프라인

```
[auto-poster] MD → Gemini HTML 변환 → Firebase Storage(이미지) + Firestore
[tech-blog] 실시간 조회 + SEO 빌드 (GitHub Actions, 6h 주기) → https://tony.banya.ai
[auto-poster] LinkedIn / YouTube / Slack 멀티채널 포스팅
```

### 경로 B — gdoc-fixer 수동 게시 (super_admin)

```
[gdoc-fixer/web] HTML 문서 편집 → "tech-blog 게시"
      ↓ Cloud Function publishToTechBlog (위 상세 흐름)
      ↓ tonys-tech-note Firestore + (큰 본문은 banya_public2 GCS)
[tech-blog] https://tony.banya.ai/report/{id} 즉시 노출 (Home latest)
      ↓ deploy-seo 워크플로우 자동 실행 (~1-3분)
[tech-blog Hosting] OG 메타 페이지 생성 → Slack/LinkedIn 미리보기 정상
```

## 서브모듈 통합 메커니즘

| 결합 형태 | 매개체 | 비고 |
|----------|--------|------|
| `auto-poster` → `tech-blog` | Firestore (`static-wiki`, `banya-official-news`) | 코드 직접 의존 없음 |
| `auto-poster` → `tech-blog` | GitHub API (`youtube_poster/deploy_seo.py`) | SEO 재빌드 트리거 |
| `gdoc-fixer` → `tech-blog` | Firestore (`static-wiki`) | 크로스 프로젝트, service account 인증 |
| `gdoc-fixer` → `tech-blog` | GCS `banya_public2/wiki-content/*.json` | 1MiB 초과 본문 분리 저장 |
| `gdoc-fixer` → `tech-blog` | GitHub `repository_dispatch (deploy-seo)` | 게시 직후 SEO 자동 빌드 |

세 서브모듈은 모두 **데이터 결합도(Firestore/GCS 매개)** 만 가지며 코드 레벨 의존이 없어 독립적으로 배포·릴리스됩니다.

## 클론

```bash
git clone --recurse-submodules https://github.com/tonythefreedom/tech-poster.git
```

이미 클론한 경우:
```bash
git submodule update --init --recursive
```

> macOS 사용자 주의: `gdoc-fixer` upstream 에 `README.md`/`readme.md` 두 파일이 분리 트래킹되어 있어, case-insensitive 파일시스템(APFS 기본)에서는 git status 에 `README.md modified` 가 상시 표시될 수 있습니다. 무해한 잔존 상태입니다.

## 서브모듈 작업 흐름

```bash
# 서브모듈 안에서 변경 → 커밋 → push
cd gdoc-fixer && git add . && git commit -m "..." && git push && cd ..
# 최상위에서 포인터 bump
git add gdoc-fixer && git commit -m "chore: bump gdoc-fixer submodule" && git push
```

`git submodule update --remote --merge` 로 리모트 main 일괄 동기화 가능.

> 최상위에서 `git add gdoc-fixer` 만으로는 서브모듈 내부 변경이 커밋되지 않습니다. 포인터가 아직 push 되지 않은 커밋을 가리키게 될 수 있으니, 서브모듈 내부 push 를 먼저 끝내야 합니다.
