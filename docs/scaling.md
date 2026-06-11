# 스케일 대비 — 사용자 수천~수백만 시나리오

> 현재 단일 백엔드 + 단일 Postgres + 단일 워커 구성. 사용자가 늘었을 때 막히는
> 지점을 미리 표시하고, 어디까지 손대야 하는지를 단계별로 정리합니다.
> **이 문서의 Phase 2/3 는 아직 적용 안 됨** — 트래픽이 그만큼 와야 의미가 있음.

## 0. 현재 상태 (Phase 1 완료)

| 항목 | 상태 |
|---|---|
| ID 전략 | UUID (uuid4) — 분산·합치기 안전. 정렬은 별도 `created_at` |
| 비동기 처리 | Postgres `LISTEN/NOTIFY` + `SELECT FOR UPDATE SKIP LOCKED` 큐 (`ingest_jobs`) |
| 스케줄러 | APScheduler (in-process) + `scheduled_queries` 테이블 |
| 캐시 | in-process TTL dict (`app/services/cache.py`) — yaml catalog 는 `@lru_cache` |
| 페이지네이션 | 유틸 통일 (`app/core/pagination.py`). 적용: skills / faqs / comments / conversations / workflow runs / notices / team members |
| 인덱스 | 모든 조회 hot path 에 복합 인덱스 (다음 §2) |

## 1. 페이지네이션 적용 매트릭스

> `/skills` · `/workflows` · `/tools` · `/schedules` 라우트는 **개발 전용(운영 미노출, ADR-0009)**
> 입니다 — 운영 빌드에서는 `FEATURE_*` 가 OFF 라 라우터가 등록되지 않습니다(404). 아래 매트릭스는
> 전 기능(개발) 기준입니다.

이미 적용된 라우트:
- `GET /skills` (limit/offset) — *개발 전용*
- `GET /chatbots/{id}/faqs` (PageParams)
- `GET /faqs/{id}/comments` (PageParams)
- `GET /conversations` (limit, max 200)
- `GET /workflows/{id}/runs` (limit, max 200) — *개발 전용*
- `GET /team/members` (limit, max 200)
- `GET /notices/*` (limit)

아직 미적용 — 트래픽 증가 시 추가 필요:
- `GET /chatbots` (팀당 챗봇 50~100개 예상 시 추가)
- `GET /tools` (사용자 등록 도구 100+개 시 추가) — *개발 전용*
- `GET /themes` (현재 16개라 무관, 캐시로 충분)
- `GET /documents` (사용자당 문서 1000+ 가능 — **우선순위 높음**)

규칙:
- 100건 이상 노출 가능한 라우트는 반드시 페이지네이션
- offset 페이지네이션은 ~10k 행까지 OK
- 그 이상은 cursor (created_at desc + id) 로 전환 (deep pagination 성능 ↓ 방지)

## 2. 인덱스 설계 — 자주 조회 일괴

이미 깔린 인덱스 (요약):

```
documents           : ix_documents_team_status, ix_documents_owner_scope, ix_documents_uploaded_at_desc
document_chunks     : ix_document_chunks_content_tsv (GIN), HNSW on embedding
usage_events        : (team, kind, created_at), (kind, created_at), (created_at)
skills              : (team_id, is_active), (category_id), (source), (visibility), triggers GIN, tags GIN
chatbot_faqs        : (chatbot_id, created_at), (status), (team_id)
faq_comments        : (faq_id, created_at), (parent_comment_id), (author_kind)
ingest_jobs         : (status, priority), (kind)
workflows           : (team_id), (author_user_id), (visibility)
workflow_runs       : (workflow_id, started_at), (status)
```

추가 권장 (Phase 2 — 트래픽 ↑ 시):
- `messages (conversation_id, created_at)` — 채팅 페이지 진입 시 매번 last 24 fetch
- `tools (visibility, upvote_count desc)` — 인기순 카탈로그 정렬
- `themes (is_active, name)` — 캐시로 우선 회피

EXPLAIN ANALYZE 로 Seq Scan 보이면 즉시 추가.

## 3. ID 전략 — 왜 UUID 인가

| 전략 | 장점 | 단점 | 우리 선택 |
|---|---|---|---|
| Auto increment | 정렬·인덱스 효율 ↑ | 분산 시 충돌, 추측 가능 (보안) | ❌ |
| UUID v4 | 분산 안전, 추측 불가 | 16B (vs 4B), 정렬 무의미 | ✅ |
| UUID v7 / ULID | 시간 정렬 가능 | 라이브러리 의존 | 후순위 |
| Snowflake | 64bit, 시간 정렬, 고성능 | 인프라 (워커별 ID 발급) 필요 | 후순위 |

현재는 UUID v4 + `created_at` 인덱스로 충분. 100M 행 이상 갈 때 UUID v7 / ULID 로 전환 검토 (insert order 가 인덱스 cache friendly).

## 4. 파티셔닝 로드맵 (Phase 2 — 아직 미적용)

대상 테이블 (시간 기반 데이터 — append-mostly):

| 테이블 | 예상 행수/년 | 파티션 키 | 시점 |
|---|---|---|---|
| `usage_events` | ~50M | RANGE (created_at, monthly) | 1M 행 넘어가면 |
| `failure_logs` | ~5M | RANGE (created_at, monthly) | 100K 행 넘어가면 |
| `workflow_runs` | ~10M | RANGE (started_at, monthly) | 500K 행 넘어가면 |
| `messages` | ~100M | RANGE (created_at, monthly) | **가장 먼저 필요** |
| `metrics_*` | ~1B | RANGE (timestamp, daily) | 2M 행/일 넘어가면 |

Postgres declarative partitioning 패턴:
```sql
CREATE TABLE messages (...)
  PARTITION BY RANGE (created_at);

CREATE TABLE messages_2026_05 PARTITION OF messages
  FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

자동화: `pg_partman` 확장 또는 cron 으로 매월 새 파티션 생성. 오래된 파티션은 `DETACH` 후 cold storage.

## 5. 샤딩 로드맵 (Phase 3 — 사용자 100만+ 단계)

샤딩은 마지막 카드. 트레이드오프 큼 (조인·트랜잭션·집계 어려움).

도입 시점 시그널:
- 단일 Postgres 가 IOPS 한계 (디스크 또는 CPU 포화)
- 백업·복구 시간이 운영상 허용 한도 넘어감
- 단일 노드 메모리 부족 (working set > RAM)

샤딩 키 후보:
- `team_id` — 같은 팀 데이터는 같은 샤드 (조인 안전), 균형 잡기 위해 hash
- `user_id % N` — 개인 데이터 위주일 때

대안 우선 검토:
- Read replica (PgPool, pgBouncer + replica)
- Citus / TimescaleDB hyperscale
- Aurora Postgres
- 마지막에 application-level sharding

## 6. 캐싱 (적용 중)

`app/services/cache.py`:
- in-memory TTL dict (단일 워커 기본)
- `REDIS_URL` 감지 시 RedisCache 로 swap 준비 hook
- 적용처:
  - `tools_for_agent` (yaml) — `@lru_cache(maxsize=1)`
  - `themes:list:active` — 60s TTL
  - 추가 예정: `model_catalog`, `skill_categories:tree`

원칙:
- read-heavy, low-cardinality, 변경 빈도 낮음
- 변경 발생 라우트에서 `cache.invalidate_prefix(...)` 호출
- TTL 짧게 (60-300s) — invalidate 실수 안전망

## 7. 비동기 처리 (적용 중)

| 영역 | 큐 | 워커 | 백오프 |
|---|---|---|---|
| 문서 ingest | `ingest_jobs` (Postgres) | `worker.ingest.runner` | attempts++ + max_attempts |
| 메트릭 집계 | in-process queue | `metrics_worker` | drop on backpressure |
| 스케줄 쿼리 | APScheduler | in-process | retry count + cooldown |

Phase 2 — 메시지 브로커 도입:
- Postgres LISTEN/NOTIFY 한계: 단일 인스턴스, payload 8KB, 워커 분산 X
- Redis Streams 또는 NATS 로 전환 시 멀티 워커 horizontal 가능

## 8. 모니터링 (관측성 — scale 결정 근거)

이미 있는 것:
- `usage_events` 테이블 (요청별 토큰/비용)
- 백엔드 로그 (`data/logs/backend.log`, 일자 회전)
- `app.metrics` 워커 — DB 배치 적재

추가 필요:
- p50/p95 latency per endpoint (현재는 평균만)
- DB 슬로우쿼리 로그 (`log_min_duration_statement = 200ms`)
- 한 시간당 활성 사용자 (HAU) — 캐파시티 계획 지표

## 결정 트리

```
사용자 수    → 작업
─────────────────────────────────────
< 100        : 페이지네이션, 인덱스만 — 현재 충분
100~10k      : 캐시 강화 (Redis 전환), read replica 1대
10k~100k     : 메시지 브로커 분리, 파티셔닝 시작 (messages)
100k+        : 샤딩 검토, 마이크로서비스 분리 (외부 tool-server 는 이미 분리 — 단 도구 마켓 자체가 개발 전용·운영 미노출, ADR-0009)
1M+          : 멀티 리전, 글로벌 CDN, edge compute
```

각 단계는 **trigger 시그널** 이 와야 적용. 미리 만들지 말 것.
