# tech-poster

반야 AI 콘텐츠 파이프라인 통합 워크스페이스입니다. `auto-poster`와 `tech-blog` 두 프로젝트를 Git 서브모듈로 묶어 한 작업 환경에서 관리할 수 있도록 구성했습니다.

## 구성

```
tech-poster/
├── auto-poster/     # 콘텐츠 생성 및 멀티 플랫폼 포스팅 오케스트레이터
└── tech-blog/       # Firebase Hosting 기반 기술 블로그 (https://tony.banya.ai)
```

### auto-poster
- MD → HTML 변환 (Gemini 기반)
- Firestore/Firebase Storage 업로드
- LinkedIn, YouTube, Slack 등 멀티 채널 포스팅
- FastAPI 기반 웹 앱 (`web_app/`)
- Android 앱 (`android/`)

### tech-blog
- React + Vite 기반 프론트엔드
- Firestore 실시간 콘텐츠 조회
- SEO 페이지 자동 생성 및 Firebase Hosting 배포
- GitHub Actions로 6시간 주기 자동 배포

## 파이프라인 흐름

```
[auto-poster] MD 작성
      ↓
[auto-poster] Gemini로 HTML 변환
      ↓
[Firebase Storage] 이미지 업로드
[Firestore]        HTML 저장 (static-wiki, banya-official-news)
      ↓
[tech-blog]  실시간 조회 + SEO 페이지 빌드
      ↓
[Firebase Hosting] https://tony.banya.ai 배포
      ↓
[auto-poster] LinkedIn/기타 채널 포스팅
```

## 클론

서브모듈을 포함해 클론해야 합니다.

```bash
git clone --recurse-submodules https://github.com/tonythefreedom/tech-poster.git
```

이미 클론한 경우:
```bash
git submodule update --init --recursive
```

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
