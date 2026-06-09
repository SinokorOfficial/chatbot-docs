# 성능 프로파일링 — chat / RAG / document 경로

> 작성: 2026-06-09. 측정 근거는 chat 라우터가 이미 적재하는 **latency meta**
> (`search_ms` · `rerank_ms` · `rag_status` — `record_llm` meta 로 기록)다.
> 별도 부하 도구 없이 실 요청의 구간별 ms 를 그대로 읽어 병목을 특정했다.

---

## 1. 측정 방법 (latency meta)

RAG 파이프라인(`backend/app/services/chatbot_rag.py:search_for_chatbot`)은 호출자가
넘긴 `timings: dict` 에 구간 소요를 in-place 로 적재한다:

| 키 | 구간 | 적재 위치 |
| --- | --- | --- |
| `search_ms` | 쿼리 임베딩 + dense(HNSW) + lexical(tsvector) + RRF | `chatbot_rag.py:172` |
| `rerank_ms` | LLM 크로스-인코더 재랭킹 | `chatbot_rag.py:181` |
| `rag_status` | `ok` / `failed` (#8 가드) | `features/chat/router.py:556,566,583` |

chat 라우터(`features/chat/router.py:1672-1673`)가 이 값을 SSE 종료 메타와
`record_llm` 에 실어, 운영 중 실 요청별로 구간 분해가 남는다.

### 관측된 병목 (상대 비중)

1. **`rerank_ms` — 가장 큼.** LLM 왕복 1회(크로스-인코더). 단일 호출이라 본질적으로
   네트워크+생성 지연이 지배적. `rerank_enabled=True` 가 기본이라 *모든* RAG 질의가
   이 비용을 낸다.
2. **`search_ms` — 그다음.** 직렬 구조에서 `dense → lexical` 가 순차였다. 두 쿼리는
   상호 독립인데 DB 왕복이 더해져 누적됐다.
3. **document 인제스트** — 임베딩 API 왕복이 청크 수/배치크기에 비례. chat 경로는
   아니지만 업로드 후 `ready` 까지의 체감 지연을 좌우.

---

## 2. 적용한 최적화 (검증 완료 · 결과 불변)

### (A) RAG 검색 병렬화 — `search_ms` 단축
`chatbot_rag.py:164-169`. dense(HNSW)와 lexical(tsvector)은 상호 독립 read-only
SELECT 이므로 `asyncio.gather` 로 병렬 실행한다. 단 SQLAlchemy `AsyncSession` 은
동일 커넥션 동시 사용을 금지(`InterfaceError: another operation is in progress`)하므로,
lexical 은 **전용 단기 세션**(`_lexical_top_own_session`, 같은 풀 공유)에서 돌린다.
RRF 입력 리스트가 동일하므로 **랭킹 결과는 완전 불변**, 두 DB 왕복의 max 만큼만 든다.

### (B) 인제스트 임베딩 배치 64 — API 왕복 절감
`ingest.py:253-256`. `embed_texts(..., batch_size=64)`. 임베딩은 입력별 독립이라
배치 그룹과 무관하게 **벡터 결과가 동일**(`embeddings.py` 는 배치별 순차 호출 후
`extend`). OpenAI embeddings 요청당 입력 한도(2048개) 내라 rate-limit 위험 없이
API 왕복만 절반(32→64)으로 줄인다. 인제스트(배치) 경로 전용.

### (C) RAG 가속 인덱스 명시 보장 — seq-scan 회귀 차단
`schema_upgrade.py:259-263`. `create_all` 은 *새 테이블* 의 Index 만 만들고 운영 중이던
기존 `document_chunks` 에는 추가하지 않는다. 누락 시 HNSW/tsvector 검색이 seq-scan 으로
떨어져 `search_ms` 가 급증한다. `models.DocumentChunk.__table_args__` 와 동일 정의의
`ix_document_chunks_embedding_hnsw`(HNSW, m=16/ef_construction=64) · `ix_document_chunks_content_tsv`(GIN)
를 `CREATE INDEX IF NOT EXISTS` 로 멱등 보장.

### (D) 캐시 멀티테넌시 키 규약 (가드레일)
`cache.py` docstring. 전역 단일 키 공간 캐시라, 테넌트별로 달라지는 값을 캐싱할 때
호출부가 키에 스코프(`team:{id}`/`user:{id}`)를 직접 박도록 규약화. 스코프 없는 키로
테넌트별 값을 캐싱하면 Team A 값이 Team B 로 새는 *교차오염*을 막는 안전장치
(현 유일 호출부 `themes:list:active` 는 공개 전역 목록이라 스코프 불필요).

---

## 3. 남은(care-needed) 항목 — 미적용, 다음 작업 후보

아래는 측정상 이득이 있으나 **정확도 회귀 위험 / 추가 검증 필요**로 이번엔 적용하지
않은 항목이다. 적용 시 반드시 latency meta 전후 비교 + RAG 정답 회귀를 함께 본다.

### [RAG 재랭킹] — 조건부 skip / 후보 축소
참조: `app/services/reranker.py:232`, `app/services/chatbot_rag.py:154`(`rerank` 항상 실행).
`rerank_ms` 가 최대 병목이므로, 모든 질의에 LLM 왕복을 강제할 필요가 없는 경우를 거른다.

- **(a) reranker skip 조건 추가** — 쿼리 길이 또는 1차 RRF 점수 분포로 판단.
  예: 1차 RRF top 점수가 충분히 지배적(예: `top 점수 ≥ 9.5` 같은 임계, 또는 top-2 점수
  gap 이 크면)일 때 재랭킹을 건너뛰고 RRF 순서를 그대로 신뢰.
  - 주의: RRF 점수는 `1/(k+rank)` 합산이라 절대 스케일이 작다(k=60 기준 단일 리스트
    1위 ≈ 0.0164). "≥9.5" 같은 임계는 **RRF 원점수 기준이 아니라** reranker 의 0-10
    점수 척도이거나, 별도 정규화 점수를 기준으로 해야 의미가 맞는다. 임계 도입 전
    실제 점수 분포를 latency meta 와 함께 로깅해 캘리브레이션할 것.
- **(b) `rerank_candidates` 축소** — 현재 기본 16(`config.py:90`). 8 로 줄이면 reranker
  프롬프트 후보가 절반이라 토큰/지연이 준다. 단 후보 풀이 좁아져 정답 청크가 16~9위에
  있던 질의에서 누락 위험 → recall 회귀를 RAG 회귀셋으로 확인 후 조정.
- **(c) 고급** — 경량 로컬 cross-encoder(예: bge-reranker) 로 1차 컷 후 상위 소수만
  LLM 재랭킹하는 2단 캐스케이드, 또는 (쿼리, 후보셋) 해시 기반 rerank 결과 캐싱.
  LLM 왕복 자체를 줄이는 가장 큰 레버지만 모델 도입/인프라 비용이 따른다.

### [DB 쿼리 (Index)] — 복합 인덱스 검토
참조: `app/db/models.py:269-273` (현재 `DocumentChunk.document_id` 단일 인덱스만).

- **(a) `(document_id, chunk_index)` 복합 인덱스** — chunk 순서 정렬/조회
  (`fetch_chunk_contents`, 번호 매긴 컨텍스트 빌드)에서 `document_id` 필터 후
  `chunk_index` 정렬을 인덱스만으로 처리해 정렬 비용 제거.
- **(b) lexical 경로 최적화** — `document_id` 의 선택도(scope 가 좁을 때)와 `content_tsv`
  GIN 을 합성 고려. PostgreSQL 에서 단일 GIN 에 스칼라 컬럼을 함께 넣으려면 `btree_gin`
  확장이 필요하고, 보통은 *부분 인덱스*나 플래너의 BitmapAnd(별도 두 인덱스 교집합)가
  더 단순하고 효과적이다. `EXPLAIN ANALYZE` 로 실제 scope 크기 분포를 보고 결정할 것.
  - 주의: 인덱스 추가는 인제스트 쓰기 비용과 디스크를 늘린다. RAG 읽기 빈도 ≫ 쓰기인
    워크로드에서만 순이득. 추가 시 `schema_upgrade.py` 에도 `CREATE INDEX IF NOT EXISTS`
    로 운영 테이블 보장을 같이 넣어야 (C) 처럼 누락 회귀가 안 난다.

---

## 4. 검증

- 백엔드 테스트: `pytest tests/ -q` → **138 passed**.
- 변경 `.py` 4건 `ast.parse` OK (`cache.py` · `chatbot_rag.py` · `ingest.py` · `schema_upgrade.py`).
- `main.py` touch → uvicorn reload → `/health` **200** (`{"ok":true}`).
- 프런트엔드 변경 없음 → tsc/test 생략.
