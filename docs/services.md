# 서비스 · 기능 카탈로그

> "이 챗봇은 무엇을 할 수 있는가"를 한 페이지로 요약합니다. 상세는 각 참조 문서로 연결.

## 1. 서비스 지도 (Feature Map)

```mermaid
mindmap
 root((장금상선 챗봇))
 대화
 실시간 SSE 스트리밍
 이미지 첨부
 파일 첨부(문서 자동 RAG)
 대화 제목 자동 생성
 사용자 메모리(장기 선호도)
 문서 RAG
 업로드·파싱(PDF/DOCX/PPTX/XLSX/CSV/HWP/HWPX/…)
 pgvector HNSW + tsvector 하이브리드
 팀 문서·개인 문서·공유 문서
 자동 청크/임베딩(text-embedding-3-large)
 챗봇
 Private / Public(팀 내)
 Shared(팀 간 공유) - 계획
 RAG 스코프 linked_only / owner_visible / team_all
 도구 바인딩
 도구
 builtin: web_search, code_interpreter, image_generate, stock_price/history
 oauth: Gmail, Kakao, Google Calendar, M365
 http: 사내 Webhook
 mcp: MCP 서버
 관리
 3단계 RBAC
 가입 승인 / 비활성화
 팀 생성·초대코드
 대화 감사(auditor)
 인프라
 PostgreSQL + pgvector
 Docker 샌드박스
 Azure 배포(계획)
 CI/CD(계획)
```

## 2. 제공 기능 요약표

| 영역 | 기능 | 참조 |
| ---- | ---- | ---- |
| 인증 | 이메일/비밀번호, JWT, 초대코드 가입, 승인/비활성화 | 관리자 문서(내부) · [rbac.md](./rbac.md) |
| 대화 | SSE 스트리밍, 멀티모달(이미지·파일), 모델 선택, 제목 자동 생성 | [api.md](./api.md#chat-chat) |
| 문서 RAG | 업로드→파싱→청크→임베딩, 삭제 cascade, 개인/팀/공유 스코프 | [document_rag.md](./document_rag.md) |
| 챗봇 | 시스템 프롬프트·모델·문서·도구 번들링, Public/Private | [chatbot.md](./chatbot.md) |
| 교차팀 공유 | 특정 팀이 만든 RAG 챗봇을 다른 팀이 사용 | [chatbot_sharing.md](./chatbot_sharing.md) |
| 도구 | builtin/oauth/http/mcp 4종, 자격증명 안전 저장 | [tools.md](./tools.md) |
| 코드 실행 | Docker 기반 Python 샌드박스, 네트워크 차단 | [sandbox.md](./sandbox.md) |
| 감사 | 팀원 대화 열람(team_auditor+) | [rbac.md](./rbac.md) |
| 배포 | Azure + GitHub Actions CI/CD | 배포 문서(내부) |

## 3. 사용자 관점 시나리오

### (a) 팀원이 "인사 규정" 챗봇에 질문
1. 팀장이 `인사 규정 봇` 챗봇 생성(`visibility=public`, `rag_scope=linked_only`).
2. 문서(HWPX) 업로드 → 자동 청크·임베딩.
3. 팀원이 `/chat`에서 챗봇 선택 후 질문 → 팀장이 연결한 문서만 참조.

### (b) 팀장이 직원 가입을 승인
1. 직원이 초대코드로 회원가입 → `approval_status=pending`.
2. 팀장 `/admin` 진입 → "승인 대기" 섹션에 직원 표시.
3. [ 승인 ] 클릭 → `approval_status=approved` → 직원 로그인 가능.
4. 이직 시 [ 비활성화 ] → 로그인 즉시 차단, 이력은 보존.

### (c) 타 팀에서 제작한 챗봇 사용 (교차팀 공유, 계획)
1. A팀 팀장이 챗봇을 `shared` 가시성으로 전환 → 공유 대상 팀 선택.
2. B팀 팀원 `/chatbots`에서 해당 챗봇 표시(공유 뱃지).
3. B팀원이 질문 → 응답은 A팀 소유 문서 기반. (문서 직접 열람 권한은 없음 — 챗봇 경유만 허용)

## 4. 백엔드 서비스 모듈 ↔ 기능 맵

| 서비스 (`backend/app/services/`) | 담당 기능 |
| ---- | ---- |
| `ingest.py` | 업로드 문서 파이프라인(파싱→청크→임베딩) |
| `document_parser.py` | PDF/DOCX/HWPX 등 평문 추출 |
| `chunking.py` | 슬라이딩 윈도우 분할 |
| `embeddings.py` | OpenAI `text-embedding-3-large` 배치 호출 |
| `rag.py` | 하이브리드 검색(RRF) — 일반 |
| `chatbot_rag.py` | 챗봇 스코프 적용된 RAG |
| `chatbot_service.py` | 챗봇 CRUD·가시성·권한 체크 |
| `tool_registry.py` | 도구 카탈로그 + `dispatch()` |
| `agent.py` | 에이전트 루프(툴 콜/응답) |
| `code_sandbox.py` | Docker 샌드박스 실행 |
| `web_search.py` | DuckDuckGo 검색 |
| `stock.py` | 네이버 시세/이력 |
| `llm_runtime.py` | LiteLLM 래퍼(모델 프로바이더 추상화) |
| `user_memory.py` | 사용자별 요약 메모리 |
| `title_gen.py` | 대화 제목 자동 생성 |
| `bootstrap_admin.py` | 최초 기동 시 super_admin 시드 |
| `schema_upgrade.py` | 기동 시 enum/컬럼 누락 보정 |

## 5. 프론트엔드 페이지 ↔ 기능 맵

| 라우트 | 기능 |
| ---- | ---- |
| `/login` · `/register` | 로그인·가입(초대코드) |
| `/chat` | 실시간 대화, 챗봇 선택, 파일/이미지 첨부 |
| `/documents` | 업로드·삭제·공유 |
| `/chatbots` | 내 챗봇 + 팀 공개 챗봇 + 공유 받은 챗봇(계획) |
| `/chatbots/[id]` | 편집: 기본/문서/도구 3 탭 |
| `/tools` | 도구 마켓, 자격증명 연결 |
| `/admin` | 승인 대기 / 팀원 관리(비활성화) / 팀 관리 |
| `/team` | 멤버 목록 · 감사 |
| `/studio` | 이미지/코드 실험실 |

## 6. 품질 · SLO 초안 (합의 대상)

| 항목 | 목표(잠정) | 측정 포인트 |
| ---- | ---- | ---- |
| SSE 첫 토큰 지연 | p50 ≤ 1.5s, p95 ≤ 3.5s | `/chat/stream` 첫 이벤트 |
| 문서 업로드 처리 | 5 MB ≤ 30s로 `status=ready` | `process_document_job` 로그 |
| RAG 검색 지연 | p95 ≤ 400 ms (청크 1M) | `rag.py` 타이머 |
| 가용성 | 99.5%/월 | Azure App Service / Container Apps 지표 |
| 샌드박스 실패율 | < 2% (비-사용자 오류) | `exit_code=-1` 비율 |
