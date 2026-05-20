# 나머지 백엔드 모듈 — 짧은 워크스루

작은 라우터·서비스 모듈은 한 곳에 모아 요약합니다. 더 자세한 흐름이 필요해질 때 개별 파일로 분리합니다.

## chatbots (`features/chatbots/router.py`)

- CRUD + `/chatbots/{id}/share` (cross-team 공유).
- `services/chatbot_service.get_for_use(db, user, id)` 가 권한 검증 헬퍼 — private → 본인만, public → 같은 팀, share 받음 → cross-team.
- `document_ids` / `tool_slugs` 는 PATCH 시 **fully replace** (ARRAY 컬럼이라 부분 갱신이 비효율).
- 생성 시 `model_id` 가 시스템에 등록된 모델인지 검증 — `core/models.py` 의 카탈로그 lookup.

## tools (`features/tools/router.py` + `services/tools/`)

- 두 출처 머지: 내장 catalog (`services/tools/registry.py`) + DB 등록 사용자 도구.
- `ToolKind` — builtin / mcp / user_http. user_http 는 등록 시 URL/method/input_schema 만 입력하면 됨.
- `ToolRegistry.dispatch(slug, input, ctx)` 가 단일 진입점. agent 도 워크플로 엔진도 모두 이 함수 호출.
- `ToolCredential` — 사용자별로 도구 인증 정보(API key 등) 저장. 비공개, 같은 사용자/팀만 접근.
- 좋아요/즐겨찾기는 UserToolUpvote / UserToolFavorite 별도 테이블 (dedup 보장).

## skills (`features/skills/router.py`)

- 스킬 마켓의 GET/POST/PATCH/DELETE + `/upvote` + `/favorite` + `/fork`.
- 카테고리 트리는 `backend/config/skill_categories.yaml` 시드에서 SkillCategory 로 적재.
- 정렬 옵션: `recent` / `popular` (upvote_count) / `trending` (최근 14일 가중) / `mine`.
- 포크는 새 Skill row + `forked_from_id` 셋팅 + 본문 복사. 좋아요/즐겨찾기 카운터는 초기화.

## conversations (`features/conversations/router.py`)

- `GET /conversations?q=...` — 본인 대화 전체. q 가 있으면 title + 메시지 본문 ts_query 검색.
- `GET /conversations/{id}/messages` — 페이지네이션.
- `PATCH /conversations/{id}` — 제목 수동 변경.
- `DELETE /conversations/{id}` — soft delete 아님, 정말 삭제. cascade 로 Message 함께.

## admin (`features/admin/router.py`)

- `/admin/me_role` — 본인의 권한 요약 (can_manage_team / can_manage_system 등). 프론트가 UI 가드에 사용.
- `/admin/users` — 사용자 목록. super_admin → 전체, team_admin → 본인 팀만.
- `PATCH /admin/users/{id}` — role / approval_status / is_active 변경. **본인을 비활성화 불가** 가드.
- `/admin/teams` — 팀 CRUD + invite_code 재발급.
- 로깅이 풍부 — 모든 admin 동작은 `app.audit` 로거에 별도 남김 (감사 추적).

## team (`features/team/router.py`)

- `/team/members` — 팀 멤버 목록.
- `/team/audit/conversations` — 팀 단위 대화 감사 — team_auditor 이상.
- `/team/audit/usage` — 팀 단위 사용량 감사.

## notices (`features/notices/router.py`)

- DB 기반 공지사항 — `Notice` 테이블 + `/notices/latest?limit=N` (핀 우선).
- 작성/수정/삭제 — `team_admin / super_admin` 만.
- 채팅 좌측 하단의 NoticeMiniList 도 같은 API 사용.

## usage (`features/usage/router.py`)

- `/usage/me/summary?window=30d` — 본인의 토큰·비용 합산.
- 비용은 USD 와 KRW 동시 — `USD_KRW_FALLBACK_RATE` 환경변수 or 한국은행 API.
- `services/metrics.py` 의 `usage_events` 큐가 비동기로 적재 → 이 API 가 집계.

## schedules (`features/schedules/router.py` + `services/schedule_runner.py`)

- 사용자가 등록한 예약 작업 (chat 메시지 자동 실행 / workflow 트리거 / 알림).
- APScheduler `AsyncIOScheduler` 가 in-process. main.py 의 lifespan 에서 start.
- 트리거는 cron 또는 datetime — `ScheduleTriggerType` enum.
- 실행 결과는 `ScheduledQueryRun` 에 적재. 실패도 row 로 남김.

## images (`features/images/router.py`)

- `POST /images/generate` — DALL·E 3 호출. 결과는 `data/uploads/images/` 에 저장 + DB Document 으로 등록.
- 사용량은 `usage_events` 에 `kind=image_gen` 으로 기록.

## themes (`features/themes/router.py`)

- 브랜드 토큰 catalog. 16개 시드 + 사용자 정의 (super_admin 만).
- GET `/themes` 응답에 `tokens` dict 포함 → 프론트 라디얼 피커가 미리보기 색을 즉시 표시.

## metrics (`features/metrics/router.py`)

- `/admin/metrics` — 전사 집계 대시보드. 백엔드 SQL 집계 + 최근 30일 추세.
- 카테고리별 에러 분포는 `error_failures` 테이블 (`signature` 로 dedup).

## models_catalog (`features/models_catalog/router.py`)

- `/models` — provider 별 모델 목록 + 표시 라벨 + capabilities (tools / vision / json).
- 정적 카탈로그 + 동적 검증 (해당 provider 의 API key 가 .env 에 있는지).

## 공통 dependencies (`core/`)

- `core/config.py` — `Settings(BaseSettings)` + `get_settings()` lru_cache.
- `core/security.py` — JWT (PyJWT) + passlib bcrypt.
- `core/deps.py` — `get_current_user`, `require_team`, `require_role(role)` 등 RBAC 가드.
- `core/middleware/request_logging.py` — 모든 요청에 짧은 ID 부여 + 응답 시간 기록.
- `core/error_handlers.py` — HTTPException / ValidationError / etc. → 사용자 친화적 JSON.
