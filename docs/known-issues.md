# 알려진 이슈 · 백로그 (Known Issues)

> 검증 워크플로우(adversarial verify)가 찾았으나 *아직 적용하지 않은* 항목을 추적한다.
> critical/high 가 후속 커밋에서 처리되면 이 표에서 지우고 changelog 로 옮긴다.
> 최종 갱신: 2026-06-08 · 출처 워크플로우: wj1503e5s / wzvn0kjh5 / wj45z8k04 / wwqa8ggcl

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
