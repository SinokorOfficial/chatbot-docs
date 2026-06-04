# API 레퍼런스 요약

> 기본 경로: `http://localhost:8000` (개발), CORS는 `localhost:3000`, `localhost:3001`, `10.x.x.x:3001` 허용.
> 운영 도메인은 `BACKEND_ALLOWED_ORIGINS` env(콤마 구분)로 외부화 가능.
> 인증: `Authorization: Bearer <jwt>` 또는 `HttpOnly` 쿠키 (둘 다 받음 — Bearer 우선).

## 헬스 (`/health` · `/healthz` · `/readyz`)

| 메서드 | 경로 | 설명 |
| ------ | ---- | ---- |
| GET | `/health` | 호환 — `{"ok": true}` |
| GET | `/healthz` | Liveness — 외부 의존성 검사 없음. K8s livenessProbe 권장 |
| GET | `/readyz` | Readiness — DB `SELECT 1` 핑. 실패 시 503 |

## Auth (`/auth`)

| 메서드 | 경로 | Body | 응답 | 설명 |
| ------ | ---- | ---- | ---- | ---- |
| POST | `/auth/register` | `{email, password, full_name, invite_code?, create_team_name?}` | `{access_token}` + `Set-Cookie: access_token` | 가입. 팀장 초대코드면 즉시 승인, 그 외엔 `pending` |
| POST | `/auth/login` | `{email, password}` | `{access_token}` + `Set-Cookie: access_token` | 로그인. HttpOnly·SameSite=Lax 쿠키 동시 발급. 승인 대기 시 403 |
| POST | `/auth/logout` | — | 204 | `access_token` 쿠키 즉시 폐기 |
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

팀장이 다른 팀 사용자를 조작하거나 본인 계정을 비활성화하면 400/403이 반환됩니다. 자세한 플로우는 [admin.md](./admin.md).

## Team (`/team`)

기존 유지. `POST /team/members`는 `team_admin+` 필요.

대화 감사(`team_auditor+`) — 수백 명 대비 *디렉토리 드릴다운*:
- `GET /team/audit/summary` → 대화한 구성원별 `{user_id,user_name,user_email,conversation_count,last_at}` (그룹 집계, 가벼움).
- `GET /team/audit/conversations?user_id=&offset=&limit=` → 특정 구성원의 대화만 lazy 로딩(`user_id` 미지정 시 팀 전체).
- `GET /team/audit/conversations/{id}/messages` → 대화 상세(기존).

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

**보안 가드 (2026-05-20 추가)**:
- `POST /tools/custom` 와 `PATCH /tools/{id}` 의 `endpoint` URL 은 SSRF 검사를 통과해야 함. 사설 IP(10/172.16-31/192.168), loopback(127/::1), link-local(169.254 — AWS/GCP/Azure 메타데이터), 메타데이터 호스트명(`metadata.google.internal` 등) 은 자동 차단. 사내 tool-server 가 사설망에 있다면 `SSRF_ALLOWED_HOSTS=host1,host2` env 로 화이트리스트.
- `visibility=team` 또는 `public` 으로 등록·승격하려면 `team_admin` 이상 권한 필요. `private` 가시성은 일반 사용자도 등록 가능 (자기 자신만 호출).
- dispatch 직전에도 한 번 더 가드 — DNS rebinding 1차 방어.

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

`backend/app/schemas_sse.py` 가 단일 진실원. 라우터 · 프론트 · 본 문서가 모두 같은 wire-format 을 참조합니다. 한 메시지는 정확히 하나의 top-level key 를 갖습니다.

| key | payload | 발생 시점 |
| --- | --- | --- |
| `t` | `"…토큰…"` | LLM 토큰이 흘러올 때 |
| `step_start` | `{n, tools}` | 에이전트가 한 step(도구 호출 묶음) 진입 시 |
| `step_end` | `{n, tools}` | step 종료 |
| `sandbox` | `{stdout, stderr, exit_code, image_b64?, files?}` | `code_interpreter` 결과 |
| `image` | `{url, prompt?, model?}` | `image_generate` 결과 |
| `pending_schedule` | `{options:[…]}` | `schedule_query` 도구가 확인 카드 제안 (사용자가 [허용] 클릭 시 등록) |
| `citations` | `[{idx, chunk_id, filename, content, score}]` | RAG 검색이 끝난 직후 1회 |
| `error` | `"사용자 친화 메시지"` | LLM/내부 오류 |
| `done` | `true` | 스트림 종료 |

예시:
```
data: {"t": "안녕하세요"}
data: {"step_start": {"n": 1, "tools": [{"name": "web_search", "args": {"q": "오늘 환율"}}]}}
data: {"sandbox": {"stdout": "1350.21\n", "exit_code": 0}}
data: {"citations": [{"idx": 1, "chunk_id": "abc", "filename": "policy.pdf", "content": "발췌…", "score": 8.4}]}
data: {"error": "외부 도구 응답 시간 초과"}
data: {"done": true}
```

새 이벤트를 추가할 때는 (1) `schemas_sse.py` 에 모델·`serialize_*` 추가, (2) `chat/router.py:_event_to_sse` 에 분기 추가, (3) 본 표 업데이트, (4) `tests/test_sse_events.py` 에 케이스 추가.

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

## Stats (`/stats`, 신규)

| 메서드 | 경로 | 인증 | 설명 |
| ------ | ---- | ---- | ---- |
| GET | `/stats/public` | **없음** | 로그인 화면용 누적 통계: `total_chats`(=user 메시지 수)·`total_users`(활성)·`total_documents` + `top_contributors`(이름·대화수 Top 5). 프로세스 내 30초 TTL 캐시. 실패 시 0 폴백. |
| GET | `/stats/highlights` | 로그인 | 공지/대시보드 하이라이트: `recent_registrations`(최근 등록 도구/스킬/**챗봇** — kind·name·by·at, 최신 12) + `today_top_chatter`(오늘 KST 가장 활발한 사용자 또는 null). |
| GET | `/stats/leaderboard` | 로그인 | 이번 주(월 09:00 KST~) 리더보드. `week_start` + `categories[]`(대화왕/열정왕(비용)/자료왕) 각 Top 3(`rank·name·value·display`). 60초 캐시. |
| GET | `/stats/me/level` | 로그인 | 내 활동량(XP) 기반 40레벨: `tier`(초급/중급/고급/전문가)·`sublevel`(1~10)·`overall`(1~40)·`label`("고급 Lv.3")·`emoji`·`xp`·`chats`·`next_label`·`next_at`·`progress`(0~1). XP = 대화 + 제작물×15 + 받은 좋아요×8 + 문서×3. |

`/stats/highlights` 응답에 `achievements`(좋아요 받은 제작물·최고 활동가 레벨 등 사전 포맷 문구) 가 추가됩니다.

문서 목록 검색: `GET /documents?q=` 가 파일명·폴더·설명 부분일치(ILIKE)로 필터합니다(폴더 picker·자료 페이지 공용).

`/stats/public` 은 *읽기 전용 집계*만 노출하며 DB 스키마 변경이 없습니다. 사내 도구라 우수 기여자 이름을 노출하지만, 민감 데이터(이메일·내용)는 포함하지 않습니다.

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
