# CI 파이프라인

`.github/workflows/ci.yml` 은 `push` 와 `pull_request` (대상 브랜치: `main`,
`cleanup`, 그리고 `pr*/**` 패턴의 작업 브랜치) 마다 코드를 검증합니다.
세 개의 **독립 job** 으로 구성되며, 서로 `needs` 의존이 없어 **한 job 이 실패해도
나머지는 끝까지 실행**됩니다(사실상 fail-fast 없음). 같은 ref 에 새 push 가 오면
진행 중이던 run 만 취소됩니다(`concurrency.cancel-in-progress`).

> GitHub Pages **배포** 는 이 워크플로우가 아니라 `.github/workflows/docs.yml`
> 이 전담합니다. `ci.yml` 의 docs job 은 **검증(strict 빌드)만** 수행합니다.

## job 1 — frontend (typecheck + build)

| 단계 | 내용 |
| ---- | ---- |
| setup | Node 22 (`actions/setup-node`, npm 캐시) |
| install | `frontend/` 에서 `npm ci` (lockfile 기준 재현 설치) |
| typecheck | `npx tsc --noEmit` — 타입 오류 시 실패 (하드 게이트) |
| build | `npm run build` (`next build`). `next.config.ts` 가 `typescript.ignoreBuildErrors=false` 라 빌드도 타입을 한 번 더 검증합니다. |

외부 백엔드는 빌드 타임에 호출되지 않으므로 더미 rewrite 백엔드
(`NEXT_PUBLIC_API_URL_REWRITE_BACKEND=http://127.0.0.1:8000`) 만 넣어 둡니다.

## job 2 — backend (pytest)

`backend/tests/` 는 **실제로 백엔드를 띄운 상태에서 HTTP 레벨로** 라우터를 훑는
스모크 테스트입니다(`conftest.py` 가 `/health` 200 을 기다린 뒤 `super_admin`
으로 로그인). 따라서 CI 는 다음 순서로 진행합니다.

1. **Postgres 서비스 컨테이너** 기동 — `pgvector/pgvector:pg16`
   (docker-compose 와 동일 이미지). `pg_isready` 헬스체크 통과까지 대기.
2. **확장 생성** — 서비스 컨테이너는 `init-db.sql` 을 자동 실행하지 않으므로
   `CREATE EXTENSION vector / pg_trgm` 을 명시 실행.
3. **의존성 설치** — `requirements*.txt` 가 있으면 `pip` 으로, 없으면(현재 구성)
   `uv sync --frozen --group dev` (pyproject.toml + uv.lock).
4. **백엔드 기동** — `uvicorn app.main:app --port 8000` 을 백그라운드로 실행,
   로그를 `/tmp/ci-logs/backend.log` 로 리다이렉트.
5. **헬스 대기** — `127.0.0.1:8000/health` 가 `200` 을 줄 때까지 폴링(최대 ~120초).
6. **pytest** — `pytest tests/ -v`.
7. **실패 시 로그 dump** — 테스트가 깨지면 `backend.log` 를 step 출력으로 덤프.

### 설정·비밀 주입

`pydantic-settings` 는 **환경변수가 `.env` 보다 우선**하므로, CI 는 더미값을
env 로 주입합니다(`ci.yml` 의 `backend.env` 참고).

- `JWT_SECRET` — 부팅 가드(32자 이상/placeholder 거부) 통과용 더미.
- `BOOTSTRAP_ADMIN_*` — 최초 부팅 시 super_admin 자동 생성, pytest 가 이 계정으로 로그인.
- `TEST_BASE_URL` / `TEST_ADMIN_EMAIL` / `TEST_ADMIN_PASSWORD` — `conftest.py` 가 읽는 값.
- `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` / `GOOGLE_API_KEY` — **빈 값**.
  외부 LLM 호출 테스트(`/chat/stream`, 이미지 생성 등)는 테스트 스위트에서
  제외되어 있어 CI 에서 과금/네트워크가 발생하지 않습니다.
- `SANDBOX_REQUIRE_DOCKER=false` — Docker 미가용 러너용. Docker 의존 테스트
  (`test_sandbox_isolation.py`)는 자체 `skipif` 마커로 건너뜁니다.

### 포트 격리

로컬 dev 백엔드는 **8001**, CI 백엔드는 **8000** 으로 분리하여 포트 충돌을 피합니다.
Postgres 도 CI 러너에서는 호스트 `5432` 로 노출(로컬 5432/5433 과 무관).

## job 3 — docs (mkdocs build --strict)

Python 3.12 + `mkdocs-material` 설치 후 `mkdocs build --strict` 를 실행합니다.
`--strict` 라 **끊긴 내부 링크** 가 있으면 실패합니다. 사내 전용 링크 평문화는
별도 sync 스크립트가 전담하므로, CI 는 **코드 리포 `docs/` 원본 기준** 으로
검증해 깨진 문서가 곧바로 드러나게 합니다.

> nav 에 포함되지 않은 페이지(`ci.md`, `known-issues.md`)는 strict 에서 `INFO`
> 레벨이라 빌드를 실패시키지 않습니다. 빌드를 **실패** 시키는 것은 `WARNING`
> (예: 존재하지 않는 대상으로의 끊긴 링크) 입니다.

### strict 빌드 현황

현재 `mkdocs build --strict` 는 **통과**합니다(exit 0). `adr/0001-sandbox-runtime.md`
가 백엔드 소스 파일(`backend/app/services/code_sandbox.py` 등)을 언급하지만, 이는
마크다운 링크가 아니라 **인라인 코드 스팬**이라 strict 가 링크로 해석하지 않습니다.
ADR 내부의 문서 간 링크(`../sandbox.md`, `0002-...md` 등)는 모두 실재 파일을 가리킵니다.

남는 출력은 nav 미포함 페이지(`ci.md`, `known-issues.md`)와 일부 앵커 누락 등
**INFO** 레벨뿐이며, 이는 strict 빌드를 실패시키지 않습니다(실패는 `WARNING`,
예: 존재하지 않는 대상으로의 끊긴 링크). 세 job 은 `needs` 의존이 없어 서로
영향을 주지 않고 끝까지 실행됩니다.

## 로컬에서 동일하게 검증하기

```bash
# ── frontend ──────────────────────────────────────────────
cd frontend
npm ci
npx tsc --noEmit
npm run build

# ── backend ───────────────────────────────────────────────
# 1) Postgres(pgvector) 기동 — 루트 compose 사용 (포트 5433)
docker compose up -d db          # init-db.sql 로 vector/pg_trgm 자동 생성

cd backend
uv sync --group dev

# 2) 백엔드를 띄운다 (tmux 등 별도 세션 권장)
bash run.sh                      # uvicorn :8000

# 3) 다른 셸에서 health 확인 후 테스트
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8000/health   # 200
TEST_ADMIN_PASSWORD="${BOOTSTRAP_ADMIN_PASSWORD}" uv run pytest tests -v

# ── docs ──────────────────────────────────────────────────
pip install "mkdocs-material>=9.5" "mkdocs>=1.6" pymdown-extensions
mkdocs build --strict
```

> 로컬 compose 의 Postgres 는 **5433**, CI 서비스 컨테이너는 **5432** 입니다.
> 로컬에서 CI 와 똑같은 주소로 돌리려면 `DATABASE_URL` 을 환경변수로 덮어쓰면 됩니다
> (env 가 `.env` 보다 우선).
