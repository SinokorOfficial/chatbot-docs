# API 레퍼런스 요약

> 기본 경로: `http://localhost:8000` (개발), CORS는 `localhost:3000`, `localhost:3001`, `10.x.x.x:3001` 허용.
> 인증: `Authorization: Bearer <jwt>`

## Auth (`/auth`)

| 메서드 | 경로 | Body | 응답 | 설명 |
| ------ | ---- | ---- | ---- | ---- |
| POST | `/auth/register` | `{email, password, full_name, invite_code?, create_team_name?}` | `{access_token}` | 가입. 팀장 초대코드면 즉시 승인, 그 외엔 `pending` |
| POST | `/auth/login` | `{email, password}` | `{access_token}` | 로그인. 승인 대기 시 403 |
| GET | `/auth/me` | — | `UserOut` | 현재 사용자 |
| GET | `/auth/team` | — | `TeamOut` | 내 팀 |

## Admin (`/admin`)

| 메서드 | 경로 | 권한 | 설명 |
| ------ | ---- | ---- | ---- |
| GET | `/admin/me_role` | 로그인 | 내 역할 / 관리 권한 요약 |
| GET | `/admin/approvals` | team_admin+ | 승인 대기 사용자 목록 |
| POST | `/admin/approvals/{user_id}/approve` | team_admin+ | 승인 |
| POST | `/admin/approvals/{user_id}/reject` | team_admin+ | 반려 |
| GET | `/admin/users` | team_admin+ | 팀 전체 사용자(role·is_active 포함) |
| POST | `/admin/users/{user_id}/deactivate` | team_admin+ | **비활성화** (로그인 차단) |
| POST | `/admin/users/{user_id}/activate` | team_admin+ | **재활성화** |
| GET | `/admin/teams` | super_admin | 팀 목록 |
| POST | `/admin/teams` | super_admin | 팀 신규 생성 |
| POST | `/admin/teams/{team_id}/rotate_invite` | super_admin | 초대코드 재발급 |

팀장이 다른 팀 사용자를 조작하거나 본인 계정을 비활성화하면 400/403이 반환됩니다. 자세한 플로우는 관리자 문서 (사내 전용).

## Team (`/team`)

기존 유지. `POST /team/members`는 `team_admin+` 필요.

## Documents (`/documents`)

| 메서드 | 경로 | 설명 |
| ------ | ---- | ---- |
| POST | `/documents/upload` | 멀티파트. `scope=team\|personal` + 파일 N개 |
| GET | `/documents` | 팀 문서 목록 |
| DELETE | `/documents/{id}` | 삭제 → 청크/공유/챗봇링크 cascade 삭제 + 파일 unlink |
| POST | `/documents/{id}/share` | 개인 문서를 팀원에게 공유 |

## Chatbots (`/chatbots`, 신규)

자세한 내용은 [chatbot.md](./chatbot.md) 참조.

## Tools (`/tools`, 신규)

자세한 내용은 [tools.md](./tools.md) 참조.

## Conversations (`/conversations`)

| 메서드 | 경로 | 설명 |
| ------ | ---- | ---- |
| GET | `/conversations?q=...` | 내 대화 목록 |
| PATCH | `/conversations/{id}` | 제목 변경 |
| DELETE | `/conversations/{id}` | 삭제 (메시지 cascade) |
| GET | `/conversations/{id}/messages` | 메시지 목록 |

## Chat (`/chat`)

| 메서드 | 경로 | 설명 |
| ------ | ---- | ---- |
| POST | `/chat/stream` | SSE. Body: `{conversation_id?, chatbot_id?, model, message, use_rag, image_base64?, files?}` |
| POST | `/chat/regenerate` | 마지막 응답 재생성 |

### SSE 이벤트 스키마
- `data: {"t": "토큰"}` — 텍스트 델타
- `data: {"step_start": {"n":1,"tools":[...]}}`
- `data: {"step_end": {...}}`
- `data: {"sandbox": {"code":"...","stdout":"...","exit_code":0,"chart_images":[...]}}`
- `data: {"image": {"url":"...","media_type":"image/png"}}`
- `data: {"error": "메시지"}`
- `data: {"done": true}`

## Models (`/models`)

| 메서드 | 경로 | 설명 |
| ------ | ---- | ---- |
| GET | `/models` | 사용 가능한 LLM 카탈로그 |

## Notices (`/notices`)

| 메서드 | 경로 | 권한 | 설명 |
| ------ | ---- | ---- | ---- |
| GET | `/notices/latest?limit=3` | 로그인 | 채팅 사이드바용 최신 공지 요약 |
| GET | `/notices?include_inactive=false` | 로그인 | 공지사항 목록 |
| POST | `/notices` | team_admin+ | 공지 생성. `title`, `summary`, `body`, `is_pinned`, `is_active` |
| PATCH | `/notices/{notice_id}` | team_admin+ | 공지 수정/비활성화 |
| DELETE | `/notices/{notice_id}` | team_admin+ | 공지 삭제 |

공지는 `notices` 테이블에 저장되며, 팀 공지와 전역 공지를 함께 조회합니다.

## FAQ / Feature Requests (`/faq/posts`)

| 메서드 | 경로 | 권한 | 설명 |
| ------ | ---- | ---- | ---- |
| GET | `/faq/posts?status=&mine=false` | 로그인 | FAQ/기능 요청 게시글 목록 |
| POST | `/faq/posts` | 로그인 | 질문 또는 기능 요구사항 작성 |
| PATCH | `/faq/posts/{post_id}` | 작성자 또는 team_admin+ | 본문 수정, 관리자 답변, 상태 변경 |
| DELETE | `/faq/posts/{post_id}` | 작성자 또는 team_admin+ | 게시글 삭제 |

상태값은 `open`, `planned`, `answered`, `closed`입니다. 기존 챗봇별 FAQ(`/chatbots/{id}/faqs`)와 별도의 운영/요구사항 게시판입니다.

## Usage (`/usage`)

| 메서드 | 경로 | 설명 |
| ------ | ---- | ---- |
| GET | `/usage/me/summary?window=24h\|7d\|30d\|month` | 내 LLM/임베딩/OCR 사용량, 토큰, USD 비용, KRW 환산액 요약 |

KRW 환산은 USD 환율 조회 성공 시 최신 값을 쓰고, 실패하면 `USD_KRW_FALLBACK_RATE` 환경값을 사용합니다.

## 공통 오류

| 상태 | 상황 |
| ---- | ---- |
| 400 | 검증 실패 / 잘못된 입력 |
| 401 | 토큰 누락/만료/위조 |
| 403 | 역할 불일치 / 승인 대기 / 비활성 계정 |
| 404 | 리소스 없음 (대화/문서/챗봇) |
| 429 | 레이트리밋(운영 추가 시) |
| 500 | 서버 내부 오류 (`expose_internal_errors=true`인 경우 상세 메시지 포함) |

## 인증 메모

- 토큰 TTL: 기본 7일 (`ACCESS_TOKEN_EXPIRE_MINUTES`).
- 리프레시 토큰 미구현 — 만료 시 재로그인 필요.
- 로그아웃은 서버 상태 없음(클라이언트에서 토큰 삭제). 필요하면 `/auth/logout`을 추가해 JWT deny list를 운영 가능.
