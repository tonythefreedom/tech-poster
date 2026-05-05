# tech-poster

반야 AI 콘텐츠 파이프라인 통합 워크스페이스입니다. `auto-poster`, `tech-blog`, `gdoc-fixer` 세 프로젝트를 Git 서브모듈로 묶어 한 작업 환경에서 관리합니다.

## 구성

```
tech-poster/
├── auto-poster/    # 콘텐츠 생성 + 멀티 채널 포스팅 오케스트레이터
├── tech-blog/      # Firebase Hosting 기반 기술 블로그 (https://tony.banya.ai)
└── gdoc-fixer/     # AI HTML 문서 편집 + 프레젠테이션 생성 웹앱 (+ 부속 도구)
```

### auto-poster
- MD → HTML 변환 (Gemini 기반)
- Firestore/Firebase Storage 업로드
- LinkedIn, YouTube, Slack 등 멀티 채널 포스팅
- FastAPI 기반 웹 앱 (`web_app/`)
- Android 앱 (`android/`)
- upstream: `kr-ai-dev-association/auto-poster`

### tech-blog
- React 19 + Vite 기반 프론트엔드
- Firestore 실시간 콘텐츠 조회 (`static-wiki`, `banya-official-news` 컬렉션)
- SEO 페이지 자동 생성 및 Firebase Hosting 배포
- GitHub Actions로 6시간 주기 자동 배포
- upstream: `kr-ai-dev-association/tech-blog`

### gdoc-fixer
- AI HTML 문서 편집기 + AI 프레젠테이션 생성 웹앱 (`web/`, React 19 + Firebase + Gemini 3-모델 파이프라인)
- DOCX/HWP 임포트, AI 문서 수정, AI 슬라이드 편집, 차트 렌더링, PNG/PDF/DOCX 내보내기
- 부속 도구: Google Docs Apps Script (`Code.gs`), HTML→PNG 캡처기 (`html_to_png.py`)
- **tech-blog 게시 기능** — 한글 HTML을 영문 자동 번역 후 `static-wiki` 컬렉션에 게시 + tech-blog의 `deploy-seo` 워크플로우 자동 트리거 (super_admin 전용, Cloud Function `publishToTechBlog`)
- upstream: `tonythefreedom/gdoc-fixer`

## 파이프라인 흐름

콘텐츠를 `tech-blog`로 흘려보내는 두 가지 경로가 있습니다.

### 경로 A — auto-poster 자동 파이프라인

```
[auto-poster] MD 작성
      ↓ Gemini로 HTML 변환
      ↓ Firebase Storage(이미지) + Firestore(static-wiki / banya-official-news)
[tech-blog] 실시간 조회 + SEO 페이지 빌드 (GitHub Actions, 6h 주기)
      ↓ Firebase Hosting → https://tony.banya.ai
[auto-poster] LinkedIn / YouTube / Slack 멀티채널 포스팅
```

### 경로 B — gdoc-fixer 수동 게시 (super_admin)

```
[gdoc-fixer/web] HTML 문서 편집
      ↓ "tech-blog 게시" 버튼 (super_admin)
      ↓ Cloud Function publishToTechBlog
        ① 권한 검증 (userProfiles.role === 'super_admin')
        ② Gemini 2.5 Pro로 한→영 번역
        ③ Gemini 2.5 Flash로 메타 추출 (titles, slug, excerpt)
        ④ Secondary Admin SDK(tonys-tech-note service account)
        ⑤ GitHub repository_dispatch (event_type: deploy-seo)
      ↓ tonys-tech-note Firestore (static-wiki 컬렉션)
[tech-blog] https://tony.banya.ai/report/{id} 즉시 노출 (Home의 위키 문서 최상위 latest)
      ↓ deploy-seo 워크플로우 자동 실행 (~3-5분)
      ↓ /dist/report/{id}/index.html 생성 (OG 메타 포함)
[tech-blog Hosting] Slack/LinkedIn 미리보기 정상 동작
```

## 서브모듈 통합 메커니즘

| 결합 형태 | 매개체 | 비고 |
|----------|--------|------|
| `auto-poster` → `tech-blog` | Firestore (`static-wiki`, `banya-official-news`) | 코드 직접 의존 없음 |
| `auto-poster` → `tech-blog` | GitHub API (`youtube_poster/deploy_seo.py`) | SEO 재빌드 트리거 |
| `gdoc-fixer` → `tech-blog` | Firestore (`static-wiki`) | 크로스 프로젝트, service account 인증 |
| `gdoc-fixer` → `tech-blog` | GitHub repository_dispatch (`deploy-seo`) | 게시 직후 SEO 자동 빌드 |

세 서브모듈은 모두 **데이터 결합도(Firestore 매개)** 만 가지며 코드 레벨 의존이 없어 독립적으로 배포·릴리스됩니다.

## 클론

서브모듈을 포함해 클론해야 합니다.

```bash
git clone --recurse-submodules https://github.com/tonythefreedom/tech-poster.git
```

이미 클론한 경우:
```bash
git submodule update --init --recursive
```

> macOS 사용자 주의: `gdoc-fixer` upstream에 `README.md`/`readme.md` 두 파일이 분리 트래킹되어 있어, case-insensitive 파일시스템(APFS 기본)에서는 git status에 `README.md modified` 가 상시 표시될 수 있습니다. 무해한 잔존 상태이며 작업에 영향이 없습니다.

## 서브모듈 최신화

```bash
git submodule update --remote --merge
```

각 서브모듈에서 개별 작업 후 상위 저장소에서 포인터 커밋:
```bash
cd auto-poster && git pull origin main && cd ..
git add auto-poster
git commit -m "chore: bump auto-poster submodule"
```

## gdoc-fixer publish 기능 배포 (참고)

`gdoc-fixer/web/functions/publishToTechBlog.js` 가 동작하려면 다음 시크릿 3개가 필요합니다:

| Secret | 값 |
|--------|---|
| `GEMINI_API_KEY` | Google Gemini API 키 (서버 사이드용) |
| `TECH_BLOG_SERVICE_ACCOUNT` | tonys-tech-note GCP service account JSON 전체 (Cloud Datastore User 권한 필요) |
| `GITHUB_DISPATCH_TOKEN` | `kr-ai-dev-association/tech-blog` 의 `Contents: write` 권한 PAT (deploy-seo 워크플로우 자동 트리거용) |

```bash
cd gdoc-fixer/web
firebase functions:secrets:set GEMINI_API_KEY
firebase functions:secrets:set TECH_BLOG_SERVICE_ACCOUNT
firebase functions:secrets:set GITHUB_DISPATCH_TOKEN
firebase deploy --only functions:publishToTechBlog,hosting
```

> tech-blog의 `deploy-seo.yml` 워크플로우는 `repository_dispatch types: [deploy-seo]` 를 받도록 이미 설정되어 있으며, gdoc-fixer 측 변경만으로 자동 빌드가 트리거됩니다 (tech-blog 코드 변경 불필요, Blaze 요금제 불필요).
