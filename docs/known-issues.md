# 알려진 이슈 · 백로그 (Known Issues)

> 검증 워크플로우(adversarial verify)가 찾았으나 *아직 적용하지 않은* 항목을 추적한다.
> critical/high 가 후속 커밋에서 처리되면 이 표에서 지우고 changelog 로 옮긴다.
> 최종 갱신: 2026-06-09 (라운드2) · 출처 워크플로우: wj1503e5s / wzvn0kjh5 / wj45z8k04 / wwqa8ggcl + 2026-06-09 종합 감사(라운드1·2)

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

전체 코드베이스 대상 종합 감사(보안 / 성능 / 버그 / UX). 직접 코드 확인으로 *검증된* 항목만 자동 적용했고, 제품 결정이 필요하거나 대규모 리팩토링이 얽힌 항목은 보류했다. (라운드2에서 보류분 7건을 추가 해소 — 아래 "라운드2 처리 완료" 표 참고.)

### 라운드1 적용한 수정 (8파일)

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

### 라운드2 처리 완료 (2026-06-09) — 보류 항목 7건 해소

라운드1 에서 "제품 결정/대규모 리팩토링"으로 보류했던 항목 중 신뢰 경계·근본 fix 를
확정해 7건(#1 #2 #3 #6 #7 #8 #9)을 실제로 처리했다. 남는 보류는 #4 #5 2건뿐이다.

| # | 제목 | sev | 파일 | 처리 요약 |
|---|------|-----|------|-----------|
| 1 | `stream_chat()` 스트림 반복 예외 스코프 — 부분응답 유실 | medium | `chat/router.py` | 스트리밍 `while` 루프만 좁게 감싸 `CancelledError` 는 즉시 전파(외곽이 부분응답 저장), 그 외 LLM 예외는 *여기서* 잡아 friendly SSE error + 부분 버퍼 `_save_and_postprocess` 저장 + `record_llm(success=False, error_code=…)`. 이전엔 광역 except 로 흘러 부분응답이 *저장 안 되던* 갭 해소. |
| 2 | 챗봇 경로 `search_for_chatbot()` 타이밍 미측정 | high | `services/chatbot_rag.py` | 근본 fix — `search_for_chatbot(..., timings=dict)` 추가해 `search_ms`(임베딩+하이브리드/스코프 검색)·`rerank_ms`(재랭킹)를 in-place 적재. 챗봇 경로도 `_run_rag_pipeline` 과 동등한 latency meta 기록. 2-tuple 반환 하위호환 유지(기존 호출부 무영향). |
| 3 | `add_dir` 파라미터 Path Traversal 검증 미흡 | high | `services/claude_runner.py` | 신뢰 경계 확정 후 처리 — `_validate_add_dir()` 게이트 추가. 허용 루트=이번 요청 `workdir`(ephemeral 은 tempdir). `Path.resolve()` + `os.path.commonpath()` 로 루트 하위 검증, `..` 토큰·루트 밖 절대경로 거부(skip + warning). `workspace_dir` None 이면 절대경로 전부 거부(보수적). |
| 6 | 스킬/RecoveryTip 키워드 매칭 풀스캔 | medium | `services/learning.py` | JSONB `?|`(jsonb_exists_any) 1차 SQL 필터(`_jsonb_overlaps_any`) — `triggers`/`tags`/`prompt_keywords` GIN 인덱스로 풀스캔 제거. 키워드 단일 `bindparam`(text[] 캐스팅, raw 연결 없음). 키워드 0개 폴백은 `_FALLBACK_LIMIT=200` 상한으로 메모리 보호. |
| 7 | 초기 계정/키 상태 조회 실패 silent | low(UX) | `chat/page.tsx` | 안전 폴백 유지 + *첫 실패 한 번만* 조용한 인라인 힌트(`role=status`) + `console.warn`. 오프라인(`ApiUserError.status===0`)은 전역 `ServerStatusToast` 가 담당하므로 조용히 무시(중복 알림 금지). `keyStatusWarnedRef` 로 반복 실패 스팸 차단. |
| 8 | RAG 조회 실패 vs 비활성 미구분 (graceful degradation) | medium | `prompt_builder.py` + `chat/router.py` | `rag_status`("ok"/"failed"/"disabled") 도입 — 파이프라인이 in-place 적재, router 가 최종 판정. `rag_failed`(=failed && 빈 컨텍스트)면 시스템 프롬프트에 "참고 문서 조회 실패" degradation 한 줄 주입(LLM 이 *문서 없음*과 *조회 실패* 를 구분). `record_llm` meta 에 `rag_status` 기록. 타임아웃류는 info, 그 외는 warning+stacktrace 로 로그 레벨 차등. |
| 9 | 컨텍스트 로더 try/except 중복 | medium | `chat/router.py` | 메모리/스킬/Recovery 로더의 동일 "실패→빈 문자열 폴백+예외 로깅" 패턴을 `_safe_call(coro, rid, label)` 공용 래퍼로 추출(DRY). |

### 남은 보류 — 제품/배포 결정 필요 (2건)

- **[#4] Alembic 마이그레이션 도입** — 현재 스키마 변경은 `Base.metadata.create_all` 류 부트스트랩에 의존해 *기존 테이블 ALTER/인덱스 추가 이력*이 코드로 추적되지 않는다(예: 라운드1 `ix_messages_conv_role_created` 추가도 신규 환경에서만 자동 반영, 기존 DB 는 수동 적용 필요). 정식 마이그레이션 프레임워크(Alembic) 도입은 배포 파이프라인·롤백 정책·기존 DB 백필 전략과 얽혀 있어 단독 코드 수정으로 끝나지 않음 → **보류(배포 결정 필요)**.
- **[#5] 부트스트랩 관리자 기본값** — 최초 부트스트랩 시 기본 관리자 계정/자격이 어떻게 생성·전달되는지(고정 기본값 vs 환경변수 강제 vs 최초 가입자 승격)에 대한 정책이 미확정. 보안(고정 기본 자격 금지)과 운영 편의(초기 접근성) 사이 트레이드오프라 제품 차원 결정 선행 필요 → **보류(제품 결정 필요)**.

*세부 위치·재현 절차는 사내 코드 리포 이슈 트래커 참조(공개판 비공개).*
