# readIt — 웹 콘텐츠 AI 요약·스크랩 서비스 (Backend)

URL이나 텍스트를 넣으면 백그라운드에서 본문을 수집해 LLM으로 요약하고, 요약 결과를 카테고리별 스크랩으로 저장·공유하는 서비스의 백엔드입니다.

- 기간: 2025.06 – 2025.08
- 인원: 백엔드 2명 ([@Leegyu66](https://github.com/Leegyu66), [@shinHyungii](https://github.com/shinHyungii))
- 상태: 기능 구현 완료, 서비스 미출시 (프론트엔드 연동 단계에서 중단)
- 원본 저장소: [MAKE-VALUE-GACHI/3_readIt_backend_](https://github.com/MAKE-VALUE-GACHI/3_readIt_backend_)

## 주요 기능

- **요약 요청 → 비동기 처리 → 상태 조회**: `POST /scrap/summaries`는 DB에 `processing` 상태의 스크랩을 만들고 `task_id`를 즉시 반환(202). Celery 워커가 크롤링·요약을 끝내면 `completed`/`failed`로 갱신하고, 클라이언트는 `GET /scrap/summaries/{task_id}`로 폴링
- **요약 유형 4가지**: 한 줄(`oneline`), 뉴스레터형(`fiveline`), 인사이트(`insight`), 기본(`basic`)
- **장문 처리**: 본문이 8,000자를 넘으면 청크(200자 오버랩)로 나눠 병렬 요약 → 최종 통합 요약(map-reduce)
- **스크랩 관리**: 수정·삭제, 공개 여부, 카테고리 CRUD(기본 카테고리 "기타" 자동 할당)
- **탐색**: 좋아요순·조회수순·최신순 페이지네이션 조회
- **소셜**: 공감(좋아요)·공감 취소, 댓글
- **인증**: Google OAuth2 로그인 → JWT 발급, 토큰 갱신
- **기타**: 회원 정보 조회·수정, 약관 조회

## 아키텍처

```
Client ──► FastAPI (uvicorn :8080)
              │  INSERT scrap(status=processing)
              ├──► MySQL 8
              │  send_task(task_id, type, url|text)
              └──► Redis (Celery broker/backend)
                      │
                      ▼
                 Celery worker
                   1. httpx로 페이지 수집 → BeautifulSoup으로 <p> 추출 (또는 입력 텍스트 사용)
                   2. OpenAI Chat API 요약 (장문이면 chunk map-reduce)
                   3. "제목: / 내용:" 파싱
                   4. UPDATE scrap SET status, subject, content
```

API 서버와 워커는 같은 이미지를 공유하고, 워커는 자체 async 세션으로 DB를 갱신합니다. 요약에 수십 초가 걸릴 수 있어 HTTP 요청을 붙잡아 두지 않고 작업 큐로 넘기는 구조를 택했습니다.

### 코드 구조

3-layer(controller → service → repository)로 도메인별 분리.

```
app/
├── api/
│   ├── auth/        Google OAuth2, JWT 발급·갱신
│   ├── user/        회원 조회·수정, 내 스크랩 목록
│   ├── scrap/       요약 요청·상태 조회, 수정·삭제, 공감, 정렬 조회
│   ├── category/    카테고리 CRUD
│   ├── comment/     댓글 등록·조회
│   ├── terms/       약관
│   ├── common_schema.py   공통 응답(CommonRes), 페이징 파라미터
│   └── routers.py
├── client/          OAuth 클라이언트
├── db/session.py    SQLAlchemy async engine / session
├── dependency/celery_service.py   Celery app, 요약 태스크
├── exceptions/      CustomException + 핸들러
├── models/          SQLAlchemy 모델, Enum
├── security.py      JWT provider, get_current_user
└── main.py
tests/               pytest + Alembic 마이그레이션 기반 테스트 DB
```

### 데이터 모델

| 테이블 | 비고 |
|---|---|
| `user` | OAuth provider, login_id(unique), soft delete(`deleted_at`) |
| `scrap` | `task_id`(unique) · `status` · `type` · `subject`/`content` · `origin_url`/`text` · `is_public` · `like_count`/`view_count`, FK → user, category |
| `category` | 사용자별 카테고리 |
| `comment` | FK → user, scrap |
| `scrap_like` | (`user_id`, `scrap_id`) 복합 PK |

## 기술 스택

| 구분 | 사용 |
|---|---|
| Framework | FastAPI 0.115, Pydantic v2 |
| DB | MySQL 8, SQLAlchemy 2.0 (async, aiomysql), Alembic |
| 비동기 작업 | Celery 5.5, Redis 7 |
| 인증 | Google OAuth2, PyJWT |
| LLM / 수집 | OpenAI API, httpx, BeautifulSoup(lxml) |
| 인프라 | Docker, Docker Compose |
| 테스트 | pytest, TestClient |

## API

모든 응답은 `CommonRes { code, message, data }` 형태로 감쌉니다. 🔒 표시는 `Authorization: Bearer <JWT>` 필요.

| Method | Path | 설명 |
|---|---|---|
| GET | `/auth/login/{provider}` | 소셜 로그인 리다이렉트 |
| GET | `/auth/callback/{provider}` | OAuth 콜백 → JWT 발급 |
| GET | `/auth/refresh` | 토큰 갱신 |
| GET | `/user` 🔒 | 내 정보 |
| PUT | `/user` 🔒 | 이름 변경 |
| GET | `/user/scraps` 🔒 | 내 스크랩 목록 (카테고리 필터) |
| PATCH | `/user/scrap/visibility` 🔒 | 스크랩 공개 여부 변경 |
| POST | `/scrap/summaries` 🔒 | 요약 요청 (202, `task_id` 반환) |
| GET | `/scrap/summaries/{task_id}` | 요약 상태·결과 조회 |
| PATCH | `/scrap/{id}` 🔒 | 스크랩 수정 |
| DELETE | `/scrap/{id}` 🔒 | 스크랩 삭제 |
| PATCH | `/scrap/{id}/like` · `/unlike` 🔒 | 공감 / 공감 취소 |
| GET | `/scrap/likes` · `/views` · `/latest` | 정렬별 목록 (`page`, `size`) |
| POST/PATCH/DELETE/GET | `/category` 🔒 | 카테고리 CRUD |
| POST / GET | `/comment` 🔒 | 댓글 등록·조회 |
| GET | `/terms`, `/terms/{n}` | 약관 |

서버 실행 후 `/docs`에서 Swagger UI로 전체 스펙을 확인할 수 있습니다.

## 실행

```bash
# 프로젝트 루트에 .env 작성 (아래 변수) 후
docker compose up --build
```

`.env`

```
DATABASE_HOST=db
DATABASE_NAME=readit
DATABASE_USER=readit
DATABASE_PASSWORD=
MYSQL_ROOT_PASSWORD=
MYSQL_DATABASE=readit
MYSQL_USER=readit
MYSQL_PASSWORD=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_REDIRECT_URI=http://localhost:8080/auth/callback/google
SECRET_KEY=
OPENAI_API_KEY=
```

- API: http://localhost:8080 (Swagger: `/docs`)
- MySQL: 호스트 포트 3307
- 컨테이너: `readit_api`, `celery`, `redis`, `db`

### 테스트

```bash
pytest tests/
```

`tests/conftest.py`가 테스트용 DB를 생성하고 Alembic 마이그레이션(`tests/migrations`)을 적용한 뒤, 테스트 종료 시 삭제합니다.

## 담당 범위

| 영역 | Leegyu66 (최규영) | shinHyungii (신현기) |
|---|---|---|
| 스크랩 도메인 (요약 요청·상태·수정·삭제, 정렬 조회·페이지네이션) | ● | |
| Celery 워커 — 크롤링, LLM 요약(map-reduce), 상태 갱신 | ● | |
| 카테고리 CRUD, 약관, CORS | ● | |
| Docker Compose (api·worker·redis·mysql) 구성 | ● | |
| 프로젝트 초기 구조(3-layer), DB 세션·Alembic 세팅 | | ● |
| Google OAuth2 · JWT · 토큰 갱신, 예외 핸들러 | | ● |
| 회원, 마이페이지, 댓글, 공감 | | ● |

## 알려진 한계 · 개선 과제

- `like_count` 증감이 읽기-수정-쓰기로 되어 있어 동시 요청 시 lost update 가능 → `UPDATE ... SET like_count = like_count + 1` 또는 `scrap_like` 집계로 전환
- 목록 조회가 OFFSET 기반이라 깊은 페이지에서 비효율 → 커서(keyset) 페이지네이션으로 전환
- 요약 결과 파싱이 "제목:/내용:" 문자열 분리에 의존 → structured output(JSON mode)으로 교체
- 워커 실패 시 재시도 없음 → Celery `autoretry_for`, 최대 재시도·백오프 설정
- 본문 추출이 `<p>` 태그에 한정 → readability 계열 라이브러리로 보강
