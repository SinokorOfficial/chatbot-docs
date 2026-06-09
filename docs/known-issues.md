# 알려진 이슈 · 백로그 (Known Issues)

> 검증 워크플로우(adversarial verify)가 찾았으나 *아직 적용하지 않은* 항목을 추적한다.
> critical/high 가 후속 커밋에서 처리되면 이 표에서 지우고 changelog 로 옮긴다.
> 최종 갱신: 2026-06-09 · 출처 워크플로우: wj1503e5s / wzvn0kjh5 / wj45z8k04 / wwqa8ggcl + 2026-06-09 종합 감사

## 미처리 (Open)

총 3건 — **high 1, medium 1, low 1** (critical 0) · 2026-06-08 백로그 일괄 처리로 9건 → 처리 완료 이동

| # | 제목 | sev | 파일 | 요약 | 상태 |
|---|------|-----|------|------|------|
| 2 | 기존 대화 워크스페이스 마이그레이션 부재 | high | `chat/router.py:127` | 신규 대화만 분리(forward-fix). 기존 `workspace_id!=NULL` 대화는 default ws 계속 공유 → 중첩 잔존. `UPDATE…SET workspace_id=NULL` 배치 또는 "새 ws 로 분리" UI 버튼. **제품 결정 필요** | open |
| 6 | "모든 워크스페이스 보기" 페이지 / orphan cleanup | medium | frontend (미구현) | 대화별 ws 수십 개 누적되나 전체 목록 UI 없음. 대화 삭제 시 ws row 잔존. `/workspace/list` + orphan(7일) 자동삭제. **제품 결정 필요** | open |
| 12 | 직행 모드 짧은 잡담 → 불필요한 claude_code 호출 | low | `chat/router.py` | 사용자 결정 "잡담도 직행" 이라 의도된 동작. 단 빈 상태 카드 안내(`adaeba4`)로 완화. 최소 길이 가드는 보류 | wontfix(의도) |

## 처리 완료 (후속 커밋에서 해결, 참고용)

### 2026-06-08 백로그 일괄 처리 (9건) — `aca596f` (배치 커밋)

| # | 제목 | sev | 처리 요약 |
|---|------|-----|-----------|
| 1 | 첨부 PDF 텍스트 추출 실패 시 silent failure | high | `chat/router.py` direct-routing 포함 모든 경로에서 `txt.strip()==""` 시 LLM·사용자에게 명시적 안내 문구 주입(silent failure 재발 방지 recheck). |
| 3 | 대용량 PDF 업로드 진행률 실시간 표시 부재 | medium | `documents/router.py` `/documents/{id}/status` (처리 페이지 수/전체) + `documents/page.tsx` 진행률 UI·폴링 단축. |
| 4 | 단계별 latency record_llm meta 미포함 | medium | `chat/router.py` `meta={search_ms,rerank_ms,generation_ms}` 분해 기록 — RAG vs LLM latency 분리 가능. |
| 5 | RAG score floor 동적 조정 | medium | `config.py` + `reranker.py` 쿼리 길이/모호성 기반 동적 floor (고정 2.0 → 동적). |
| 7 | per-step LLM timeout inner/outer CancelledError 미구분 | medium | `agent.py` 외부 `asyncio.timeout` 범위와 내부 `wait_for` 의 CancelledError↔TimeoutError 별도 except + 구분 로깅. |
| 8 | tool_call_counter contextvars 방어 | medium | `agent.py` `ContextVar` 가드로 request-local 강제 — gen() 재사용 시 공유 위험 차단. |
| 9 | docx 페이지 분리 / 이미지 OCR | medium | `ocr.py` + `document_parser.py` magic-byte 검사 + docx 페이지 분리 (suffix-only 한계 제거). |
| 10 | `_OCR_CONCURRENCY` rate-limit 헤드룸 | medium | `ocr.py` 동시성 설정화 + 429 지수 백오프 재시도. |
| 11 | WorkspaceProvider 파일트리 GET race | medium | `WorkspaceProvider.tsx` 신규 ws 생성 후 SSE agent_start race 가드 — 빈 패널 표시 제거. |

### 이전 후속 커밋

| 항목 | 커밋 |
|------|------|
| NameError (run_code/format_code_result import) | `0da45f6`→`1ecb9d3` |
| asyncio.timeout(None) graceful | `0da45f6` |
| /regenerate chatbot_system_prompt + files 누락 | `0da45f6` / `adaeba4` |
| keepalive task 누수 / 가드 메시지 SSE 노출 / 배지 false-positive | `3435979` / `0da45f6` |
| path traversal(realpath) / CLI injection / system_prompt sanitize | `adaeba4` |
| mkdir 안전성 / claude CLI error handling | `adaeba4` |
| 워크스페이스 race condition (SELECT FOR UPDATE) | `166d3d9` |
| claude_code 결과 LLM 환원 | `1ecb9d3` |
| citation dedup wiring | `1ecb9d3` |

## 2026-06-09 종합 감사

전체 코드베이스 대상 종합 감사(보안 / 성능 / 버그 / UX). 직접 코드 확인으로 *검증된* 항목만 자동 적용했고, 제품 결정이 필요하거나 대규모 리팩토링이 얽힌 항목은 보류했다.

### 이번에 적용한 수정 (8파일)

| 카테고리 | 파일 | 수정 요약 |
|----------|------|-----------|
| 보안 (CORS 하드닝) | `main.py` | 하드코딩된 사내 내부 IP 출처 제거 → `BACKEND_ALLOWED_ORIGINS`(쉼표 구분) env 기반 출처, 미설정 시 로컬 dev 기본값. `allow_methods=["*"]` → 명시적 화이트리스트(`GET/POST/PUT/DELETE/OPTIONS`). |
| 보안 (로그 마스킹) | `request_logging.py` | 요청 쿼리스트링 로깅 시 민감 파라미터(`password/token/api_key/secret/auth/apikey/access_token/refresh_token`)를 `<REDACTED>` 로 마스킹. 파싱 실패 시 길이만 표시. |
| 보안 (Path traversal) | `documents/router.py` | `clean_folder` 절대경로(`/`,`\`) 명시 거부 + 업로드 저장 직후 `resolve()` 가 `upload_root` 하위인지 검증(심볼릭/탈출 시 unlink+500) + render/download 엔드포인트에 `is_relative_to(upload_root)` 검증(`ValueError` → 403). |
| 성능 (인덱스) | `db/models.py` | regenerate hot path 용 복합 인덱스 `ix_messages_conv_role_created(conversation_id, role, created_at)` 추가 — role 필터 오버헤드 제거. |
| 성능/안정 (타임아웃) | `embeddings.py` | `client.embeddings.create` 를 `asyncio.wait_for(timeout=EMBEDDING_API_TIMEOUT_S, 기본 10s)` 로 감싸 무한 hang 차단, 초과 시 `EmbeddingTimeoutError`(model/batch 컨텍스트 포함). |
| 성능/안정 (재시도) | `reranker.py` | 재랭커 `acompletion` 에 429/Timeout/5xx 지수 백오프 재시도(3회, 1s→2s→4s) 추가 — `ocr.py` 의 `_is_rate_limit_error` 패턴 재사용. BadRequest/Auth 등 비재시도 계열은 즉시 실패. |
| 명료성 (문서화) | `config.py` | `max_attachment_parse_chars` 가 *파싱된 텍스트* 상한이며 업로드 바이트 상한과 별개임을 주석 명시(혼동 방지). |
| UX (클립보드) | `ChatMessage.tsx` | `CodeBlock`/`ChatMessage` 의 중복 복사 로직을 `useCopyToClipboard` 공용 훅으로 통합 + `.catch()` 로 권한 거부 시 silent unhandled rejection 방지(`copied=false` 복원 + 경고). |

### 제품 결정 / 대규모라 보류한 항목

- **[medium] Async/await — `stream_chat()` 스트림 반복 예외 스코프** (`chat/router.py`) — 확인 결과 inner try-except(1438–1445)가 `asyncio.CancelledError` 를 이미 명시적으로 처리하므로 원 finding 의 "CancelledError 전파 안 됨" 진술은 *부정확*. 다만 비-StopAsyncIteration·비-CancelledError 예외(LiteLLM APIError, stream_chat RuntimeError 등)는 inner 에서 잡히지 않고 outer 핸들러(1644)로 전파되어 로깅 후 에러 SSE 전송(re-raise 없음). 이는 의도된 설계(상위에서 처리·클라이언트 통지)로 동작상 결함은 아니며, 예외 처리가 두 레이어로 분산된 *코드 명료성* 이슈에 가깝다. 클라이언트 통지 vs 로깅 vs re-raise 중 무엇이 옳은지는 비즈니스 로직 판단이라 기계적 자동수정 부적합 → **보류(제품 결정)**.
- **[high] 상태 초기화 — `rag_timings` dict 에러 경로** (`chat/router.py`) — 챗봇 경로(1289–1304)의 `search_for_chatbot()` 이 timings 를 반환하지 않아 `rag_timings` 가 비는 점은 *확인됨*. 그러나 line 1473 이 이미 `"use_rag": bool(use_rag_effective)` 로 RAG 실행 여부를 명시 기록하므로 제안된 `"rag_executed": bool(rag_timings)` 추가는 *중복 정보*. 더 근본 문제는 `search_for_chatbot()` 에 타이밍 측정 기능 자체가 없어(=`_run_rag_pipeline()` 와 달리) 챗봇 경로가 정확한 `search_ms`/`rerank_ms` 를 기록 못 한다는 점. 올바른 fix 는 (1) `search_for_chatbot()` 에 timing 측정 추가, (2) 챗봇 경로도 `timings` dict 를 받아 populate — 제안 fix 가 근본 문제를 못 풀어 자동 적용은 불완전 → **보류(대규모 리팩토링)**.
- **[high] `add_dir` 파라미터 Path Traversal 검증 미흡** (`services/claude_runner.py`) — `add_dir` 엔트리에 단순 문자열 정규화만 적용하고 상대경로 탈출·절대경로·홈 디렉터리 확장에 대한 정규화 검사 없이 CLI 인자로 전달하는 갭 *확인됨*. 동일 코드베이스에 선례(첨부 처리, `adaeba4`)가 `Path.resolve()` + `os.path.commonpath()` 로 방어한다. 다만 `add_dir` 의 신뢰 경계(누가 값을 채우는지·허용 루트가 무엇인지)와 claude_code 의 합법적 다중 디렉터리 접근 요구가 얽혀 있어, 단순 거부가 정당한 사용을 깨뜨릴 위험 → **보류(제품 결정: 허용 루트/신뢰 경계 정의 선행 필요)**. *세부 위치·재현 절차는 사내 코드 리포 이슈 트래커 참조(공개판 비공개).*
