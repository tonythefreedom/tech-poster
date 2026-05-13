# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 저장소 개요

`tech-poster`는 반야 AI 콘텐츠 파이프라인의 **통합 워크스페이스**입니다. 실제 코드는 전부 세 개의 Git 서브모듈 안에 있으며 이 최상위 저장소에는 의미 있는 소스가 없습니다.

```
tech-poster/                # 빈 래퍼 저장소 (서브모듈 포인터만 보유)
├── auto-poster/    ←  submodule: kr-ai-dev-association/auto-poster
├── tech-blog/      ←  submodule: kr-ai-dev-association/tech-blog
└── gdoc-fixer/     ←  submodule: tonythefreedom/gdoc-fixer  (다른 owner!)
```

**참고**: `gdoc-fixer`만 owner가 다릅니다(`tonythefreedom` 개인 계정). 클론/push 권한 설정이 다른 두 개와 별개라는 점에 주의하세요.

작업할 때 항상 이 구조를 먼저 확인하세요. 파일을 수정하려면 해당 서브모듈 디렉터리로 들어가야 하며, 각 서브모듈은 **독립적인 git 저장소**입니다(다른 리모트, 다른 브랜치). 최상위에서 `git status`를 실행하면 서브모듈 자체의 파일 변경은 보이지 않고, 오직 서브모듈 포인터(커밋 해시) 변경만 감지됩니다.

## 서브모듈 작업 흐름

신선한 클론의 경우:
```bash
git submodule update --init --recursive
```

서브모듈 내용을 최신화하려면 (리모트의 main을 가져옴):
```bash
git submodule update --remote --merge
```

각 서브모듈에서 개별 작업한 뒤에는 **두 번 커밋**해야 합니다 — 먼저 서브모듈 내부에서, 그 다음 최상위에서 포인터를 올립니다:
```bash
cd auto-poster && git add . && git commit && git push
cd .. && git add auto-poster && git commit -m "chore: bump auto-poster submodule"
```

최상위 저장소에서 `git add auto-poster`만으로는 서브모듈 내부 변경이 커밋되지 않습니다 — 포인터가 아직 누구도 push하지 않은 커밋을 가리키게 될 수 있으니 서브모듈 내부의 push를 먼저 끝내세요.

## 세 구성 요소의 역할

데이터 흐름은 **단방향**입니다: `auto-poster`와 `gdoc-fixer`가 각각 콘텐츠를 생성/업로드하고, `tech-blog`가 읽어서 보여줍니다. 세 구성 요소는 직접 호출 관계가 없으며, **Firebase(Firestore + Cloud Storage) + GitHub Actions 트리거**를 통해서만 통신합니다.

```
[auto-poster] MD/PDF → Gemini로 HTML 변환    [gdoc-fixer] HTML → AI 슬라이드 변환
                     ↓                                    ↓
[Firestore]  static-wiki / banya-official-news 컬렉션에 HTML 저장
[Storage]    wiki-images/{wiki_id}/*.png 에 이미지 저장
                     ↓
[GitHub Actions]  deploy-seo.yml 트리거 → tech-blog 빌드 + SEO 페이지 생성
                     ↓
[tech-blog]  실시간 구독 + `npm run build` 시 SEO 페이지 생성
                     ↓
[Firebase Hosting]  https://tony.banya.ai 배포
                     ↓
[auto-poster] LinkedIn/YouTube/Slack 등으로 포스팅
```

핵심 계약은 Firestore의 `static-wiki` 문서 구조입니다 (`titles.{ko,en}`, `content.{ko,en}`, `thumbnailUrl`, `lastUpdated`, `type: "firestore-content"`). **세 구성 요소 중 어느 쪽이든** 이 스키마를 바꾸면 나머지 쪽도 함께 업데이트해야 합니다.

게시 후 SEO 페이지 생성은 `auto-poster`와 `gdoc-fixer` 양쪽 모두 `GITHUB_TOKEN`을 사용해 tech-blog 저장소의 `deploy-seo.yml` 워크플로우를 `workflow_dispatch`로 호출하는 방식입니다. 이 토큰이 빠지면 Firestore 저장은 성공해도 정적 SEO 페이지(`dist/report/{id}/index.html`)는 갱신되지 않아 SNS 링크 프리뷰가 깨집니다.

### auto-poster (Python / FastAPI)

- **`web_app/main.py`**: FastAPI 엔드포인트와 Alpine.js 프론트엔드를 서빙하는 **단일 거대 파일(~85KB)**. 라우트는 여기에 모여 있고, 기능 로직은 `web_app/services/`의 서비스 모듈에 위임합니다.
- **`web_app/services/`**: 각 기능 한 파일. 특히 주목할 것들:
  - `content_generator_service.py` — Gemini 기반 기획안(JSON) 생성. 서브슬라이드 시스템의 핵심. 레거시 기획안 호환을 위한 `get_flat_sub_slides()` 헬퍼가 여기 있음.
  - `pdf2mp4_service.py` (124KB) — PDF → MP4 변환 파이프라인. Basic/Smart 모드, Whisper 기반 타이밍 매칭, NVENC 가속 모두 이 안에서 처리.
  - `auto_pipeline_service.py` — 주제 입력 → YouTube 업로드까지 7단계 오케스트레이터. 큐 시스템(`/api/pipeline/queue/*` 엔드포인트)도 여기.
  - `crypto_service.py` + `firebase_service.py` — Fernet 대칭키로 `serviceAccountKey.json`/`client_secrets.json`을 DB 암호화 저장하는 경로. 프로덕션에서는 로컬 파일 폴백이 차단됨(`ENVIRONMENT=production`).
- **`web_app/core/`**: SQLAlchemy DB 설정(`database.py`)과 모델(`models.py` — `User`, `SecureFile`). DB는 `web_app/autoposter.db` (SQLite).
- **`core/`** (web_app 외부): 레거시 Python 스크립트(`linkedin_poster.py`, `summarizer.py`, `auth_helper.py`). 웹 UI가 대체했지만 일부 모듈을 계속 import하는 경우가 있으므로 제거 전 확인.
- **`youtube_poster/`**, **`android/`**: 각각 별개의 구버전 CLI와 초기 Android 앱. 주력은 `web_app`.

### gdoc-fixer (React 19 + Vite, 별도 Firebase 프로젝트)

- **`web/`**: 메인 React 앱. HTML 문서 편집기 + AI 슬라이드 생성기. Tailwind v4, Zustand.
- **`web/src/`**: Gemini 3-모델 파이프라인 (Flash 콘텐츠 분석 → Flash-Image 이미지 생성 → Pro HTML 슬라이드 조합) + Canvas BFS 기반 투명 배경 처리 + 뷰포트 자동 보정(스크린샷 → Gemini Flash 멀티모달 피드백).
- **`web/functions/`**: Firebase Cloud Functions (Gen 2). `/share/:id` 경로에서 OG 메타 태그 SSR.
- **`web/.env`**: Firebase + Gemini + **GCS 직접 접근용 서비스 계정 JWT** (`VITE_GCS_BUCKET`, `VITE_GCS_SA_EMAIL`, `VITE_GCS_PRIVATE_KEY`). 클라이언트 사이드에서 GCS REST API + JWT 서명. 서비스 계정 키가 번들에 들어가니 노출 범위 주의.
- **`Code.gs`, `appsscript.json`**: Google Apps Script — Google Docs 연동 진입점.
- **tech-blog와의 연계**: 슬라이드/문서를 Firestore `static-wiki`에 게시한 뒤 GitHub Actions로 SEO 빌드 트리거 (auto-poster와 동일한 트리거 패턴).

### tech-blog (React / Vite)

- **`src/App.jsx`** + **`src/pages/`**: React Router 라우트. `Report.jsx`는 정적 HTML 위키를 런타임에 정제하는 가장 복잡한 페이지로, **수식 처리/CSS 스코핑/이미지 경로 보정이 이 파일 안에서 일어남**. 정적 리포트가 깨질 때 먼저 볼 곳.
- **`src/index.css`**: 사이트 전역 스타일 + 모바일 반응형. 줄간격/테이블 스크롤 관련 이슈는 여기서 수정.
- **`src/components/WikiContent.jsx`**: 리포트 본문 래퍼. 이미지 모달, 상단 공유 버튼 레이아웃 담당.
- **`src/services/firebase.js`**: Firebase 클라이언트 초기화. `.env.local`의 `VITE_FIREBASE_*` 변수를 사용.
- **`scripts/`** (중요 — 빌드 체인이 이것들에 의존):
  - `deploy-html-wiki.js` — `html/` 디렉터리의 로컬 HTML을 처리해 `static-wiki.json` 생성.
  - `generate-report-seo.js` — Firestore + `static-wiki.json`에서 리포트를 읽어 `dist/report/{id}/index.html`에 OG/Twitter 메타 태그가 박힌 페이지를 생성.
  - `upload_wiki.py` — **Python** 스크립트. Firebase Admin SDK로 HTML + 이미지 쌍을 Firestore/Storage에 한 번에 올리고 옵션으로 GitHub Actions까지 트리거.
  - `trigger-github-deploy.{py,sh}` — 외부 시스템에서 Firestore에 저장한 뒤 배포 워크플로우를 수동 트리거할 때 사용.
- **`.github/workflows/`**: `deploy-seo.yml`(수동 트리거 또는 6시간마다 자동 배포), `generate-sitemap.yml`(일 1회 sitemap.xml 갱신).

## 자주 쓰는 명령어

### tech-blog (서브모듈 내부에서 실행)
```bash
cd tech-blog
npm run dev                  # Vite 개발 서버
npm run build                # deploy-html-wiki → vite build → generate-seo (3단계 체인)
npm run lint                 # ESLint
npm run generate-seo         # SEO 페이지만 재생성 (빌드 없이)
npm run deploy-html-wiki     # static-wiki.json만 재생성
npm run reset-admin          # SSH 인증 기반 관리자 계정 초기화
firebase deploy --only hosting   # dist/ 를 Firebase Hosting에 배포
node scripts/generate-sitemap.js # sitemap.xml 수동 재생성
node scripts/upload-news.js      # banya-official-news 컬렉션에 샘플 업로드
```

`npm run build`가 실패할 때는 3단계 중 어느 단계에서 깨졌는지 확인 — `deploy-html-wiki`는 `html/` 디렉터리와 `serviceAccountKey.json`을 요구하고, `generate-seo`는 Firestore 접근 권한을 요구합니다.

### auto-poster (서브모듈 내부에서 실행)
```bash
cd auto-poster
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
# README의 "빠른 시작"에는 requirements.txt에 없는 추가 패키지 목록이 있음
# (fastapi, sqlalchemy, google-cloud-storage 등) — 새 환경에서는 README 참조

cd web_app
../venv/bin/python -m uvicorn main:app --host 0.0.0.0 --port 8000 --reload
# → http://localhost:8000

# ffmpeg이 시스템에 설치되어 있어야 함 (brew install ffmpeg)
# Smart 모드 PDF→MP4는 openai-whisper 필요 (requirements.txt에 포함)
```

환경 변수는 `auto-poster/.env`에 둡니다 (`ENVIRONMENT`, `SUPER_ADMIN_ID`, `SUPER_ADMIN_PW`, `SECRET_KEY`, `GEMINI_API_KEY`, LinkedIn 토큰, `YOUTUBE_API_KEY`, `GITHUB_TOKEN`). 프로덕션에서는 `ENVIRONMENT=production`으로 두고 보안 키 파일은 `/admin/secure-files` UI에서 DB 암호화 업로드합니다 — 로컬 `secrets/` 디렉터리 폴백이 차단됩니다.

`GITHUB_TOKEN`은 SEO 빌드 자동 트리거에 필수입니다(권한: `repo`, `workflow`). 없으면 Firestore 저장은 성공해도 `dist/report/{id}/index.html` 생성이 안 됩니다.

### gdoc-fixer (서브모듈 내부에서 실행)
```bash
cd gdoc-fixer/web
npm install
npm run dev                                     # Vite dev 서버
npm run build && npx firebase deploy --only hosting
npx firebase deploy --only functions            # /share/:id OG SSR Cloud Function
```

`web/.env`에 Firebase 웹 앱 설정 + `VITE_GEMINI_API_KEY` + GCS 서비스 계정 (`VITE_GCS_BUCKET`, `VITE_GCS_SA_EMAIL`, `VITE_GCS_PRIVATE_KEY`)이 필요합니다. **`VITE_GCS_PRIVATE_KEY`는 클라이언트 번들에 들어가므로** 해당 서비스 계정의 권한을 GCS 업로드 최소권한으로 한정하세요 (Storage Admin 같은 광범위 권한 금지).

## auto-poster에서 주의할 제약

- **`content_generator_service.py`의 프롬프트/JSON 스키마는 다른 단계와 강하게 결합되어 있습니다.** `slides[].sub_slides[]`, `narration`, `image_prompt` 등의 필드 이름을 바꾸면 이미지 생성 → PDF 메타데이터 임베드 → Smart 모드 타이밍 매칭까지 연쇄적으로 깨집니다. 필드를 추가/변경할 때는 `auto_pipeline_service.py`와 `pdf2mp4_service.py`의 소비 지점을 함께 확인하세요.
- **서브슬라이드 파일명 규칙**: `{plan_id}_slide_{slide:02d}_{sub:02d}.png` — PDF 페이지 수 = 총 서브슬라이드 수. Smart 모드 타이밍 알고리즘이 이 가정에 의존합니다.
- **레거시 기획안**(sub_slides 없음)은 `get_flat_sub_slides()` 헬퍼로 자동 변환됩니다 — 새 코드에서 `slides[].sub_slides`에 직접 접근하기 전에 이 헬퍼를 거치도록 하세요.
- 대용량 PDF(>50MB)는 업로드 전에 PyPDF2 + img2pdf로 자동 압축되고, 1차 압축이 부족하면 JPEG 품질/DPI를 낮춰 2차 압축합니다. PDF 처리 로직 변경 시 양쪽 경로를 모두 확인.

## tech-blog에서 주의할 제약

- **`Report.jsx`는 Firestore의 원본 HTML을 런타임에 크게 정제**합니다: 수식 앞뒤 태그 제거, `max-w-4xl`/`p-8` 등의 레이아웃 클래스 제거, `../images/`/`./images/` 상대 경로를 `/static-wiki/images/`로 치환, CSS 스코핑 적용. 리포트 렌더링이 이상할 때는 원본 HTML을 건드리기 전에 이 파일의 정제 파이프라인을 먼저 점검하세요.
- **한/영 매칭**은 파일명이 `_ko` / `_en`으로만 다르면 자동이고, 구글 드라이브 동기화 문서는 설명(Description) 필드의 공통 `translationKey`로 묶입니다. macOS NFD(자모 분리) 인코딩 문제는 이미 대응되어 있으니 파일명 정규화 로직을 건드릴 때 회귀에 주의.
- **MathJax 설정**은 섬세합니다 — 로딩 순서를 바꾸거나 수식 주변 `white-space: nowrap`을 제거하면 수식이 줄바꿈으로 깨집니다.
