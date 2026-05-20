# 코드 워크스루 — 모듈별 상세

서비스가 어떻게 동작하는지 코드를 따라가며 한 줄 한 줄 뜯어보는 가이드입니다. 새로 합류한 동료가 30분 안에 머릿속 지도를 만들 수 있도록 구성했습니다.

읽는 순서 — 위에서 아래로 따라가면 큰 그림 → 세부 흐름 → 도메인 깊이 순으로 이해됩니다.

## 1. 큰 그림

| 문서 | 다루는 코드 | 한 줄 |
|---|---|---|
| [main · 부팅 흐름](backend-main.md) | `backend/app/main.py` | FastAPI lifespan, 라우터 등록, CORS, 미들웨어 |
| [DB 모델 지도](backend-models.md) | `backend/app/db/models.py` | ORM 테이블 13종 + Enum + 관계 |
| [공통 의존성](backend-deps.md) | `backend/app/core/`, `backend/app/db/database.py` | 설정·세션·인증 가드·RBAC |

## 2. 인증·권한

| 문서 | 다루는 코드 |
|---|---|
| [auth — 로그인/가입/승인](backend-auth.md) | `features/auth/` + `services/bootstrap_admin.py` |
| [나머지 백엔드 모듈 (admin · team · …)](backend-misc.md) | `features/admin/`, `features/team/` |

## 3. 채팅·에이전트 루프

| 문서 | 다루는 코드 |
|---|---|
| [chat — SSE 스트리밍 + tool_use 루프](backend-chat.md) | `features/chat/`, `services/agent.py`, `services/llm_runtime.py` |
| [나머지 백엔드 모듈 (chatbots · conversations)](backend-misc.md) | `features/chatbots/`, `features/conversations/` |

## 4. 지식·RAG

| 문서 | 다루는 코드 |
|---|---|
| [documents — 업로드부터 임베딩까지](backend-documents.md) | `features/documents/`, `services/ingest.py`, `services/chunking.py`, `services/embeddings.py` |

## 5. 도구·스킬·워크플로

| 문서 | 다루는 코드 |
|---|---|
| [workflows — 노드/엣지 + 실행 엔진 + NL 생성](backend-workflows.md) | `features/workflows/`, `services/workflow_engine.py`, `services/workflow_nl.py` |
| [나머지 백엔드 모듈 (tools · skills)](backend-misc.md) | `features/tools/`, `features/skills/` |

## 6. 게시판·공지·사용량

| 문서 | 다루는 코드 |
|---|---|
| [faqs — 챗봇 FAQ + 기능 요청 + AI 답글](backend-faqs.md) | `features/faqs/`, `services/faq_ai.py` |
| [나머지 백엔드 모듈 (notices · usage · schedules · images · themes · metrics)](backend-misc.md) | 작은 라우터 묶음 |

## 7. 프론트엔드

| 문서 | 다루는 코드 |
|---|---|
| [프론트 부팅 + AppShell](frontend-shell.md) | `frontend/app/layout.tsx`, `frontend/shared/layout/AppShell.tsx` |
| [API 클라이언트 + 토큰](frontend-api.md) | `frontend/shared/lib/api.ts`, `userError.ts` |
| [테마 시스템 (16 브랜드)](frontend-theme.md) | `frontend/shared/theme/` |
| [채팅 화면 — SSE 수신 흐름](frontend-chat.md) | `frontend/app/(workspace)/chat/page.tsx` |
| [워크플로 캔버스 — 2D · 3D](frontend-workflow-canvas.md) | `frontend/features/workflows/` |
| [디자인 시스템 — Stitch 토큰](frontend-design.md) | `frontend/app/globals.css` |

각 워크스루는 다음 구조를 따릅니다.

1. **무엇을 하는 모듈인가** — 한 문단 요약.
2. **외부에서 보이는 모양** — public API (라우트·함수·컴포넌트).
3. **내부 흐름** — 호출 순서 + Mermaid 다이어그램(필요 시).
4. **핵심 코드 발췌** — 줄 번호와 함께, 왜 이 줄이 중요한지 코멘트.
5. **함정·결정 기록** — "왜 X 가 아니라 Y 인지" 명시.

코드는 항상 현재 main(원본)/cleanup(개발) 브랜치 기준이며, 인용 줄 번호가 바뀌면 PR 에 워크스루도 같이 갱신합니다 (`CONTRIBUTING.md` (리포 루트) 참조).
