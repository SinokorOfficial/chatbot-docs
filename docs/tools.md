# 도구 마켓플레이스

> **개발 전용(운영 미노출)** — 도구 마켓플레이스는 ADR-0009 의 개발 티어 전용 기능입니다. 운영(`APP_ENV=prod` / `NEXT_PUBLIC_FEATURE_SET=rag`)에서는 `FEATURE_CUSTOM_TOOLS=off` 라 `main.py` 가 `/tools` 라우터를 등록하지 않아 모든 `/tools/*` 가 404 로 차단되며, 프론트에서도 메뉴/화면이 숨겨집니다(`shared/lib/features.ts` 의 `DEV_ONLY_PATH_PREFIXES`·`isRagOnly`). 운영에는 RAG 챗봇 4기능만 노출됩니다.

## 1. 목표

- 기본 제공 도구(빌트인)를 클릭 한 번으로 챗봇에 장착.
- 외부 도구(Gmail, 카카오톡, 캘린더 등)를 카탈로그에 등록하고 챗봇에 바인딩.
- 사용자별/팀별 자격증명(OAuth 토큰 등)을 안전하게 저장, 런타임에 주입.

## 2. 데이터 모델

```
tools (카탈로그 — 시스템 전역)
 └ chatbot_tools (챗봇 ↔ 도구 바인딩 + override 설정)
 └ tool_credentials (사용자별/팀별 자격증명)
```

스키마 상세는 [erd.md](./erd.md#tools)을 참고.

### Tool.kind

| kind | 실행 방식 | 예시 |
| ---- | -------- | ---- |
| `builtin` | 에이전트 루프(`agent.py`)에서 직접 실행하는 파이썬 핸들러 | `web_search`, `claude_code`, `image_generate`, `stock_price`, `stock_history`, `schedule_query` |
| `http` | 등록된 URL로 POST 호출 (webhook 스타일) | 사내 API, 슬랙 Incoming Webhook |
| `oauth` | 자격증명을 꺼내 외부 API 호출 | Gmail 발송, 캘린더 조회, Microsoft 365 |
| `mcp` | MCP 서버의 도구 호출 | `mcp__claude_ai_Gmail__*` 등 Anthropic MCP 커넥터 |

## 3. 기본 시드 카탈로그

서버 기동 시 `tool_registry.load_builtin_catalog()`가 다음 도구를 `tools` 테이블에 upsert 합니다.

| slug | display_name | category | kind | 기본 장착 대상 |
| ---- | ------------ | -------- | ---- | -------------- |
| `web_search` | 웹 검색 | search | builtin | 모든 챗봇 권장 |
| `claude_code` | 코드 작업 (Claude Code) | data | builtin | 분석/코딩 챗봇 (개발 전용) |
| `image_generate` | 이미지 생성 | productivity | builtin | 마케팅 챗봇 |
| `stock_price` | 실시간 주가 | data | builtin | 금융 챗봇 |
| `stock_history` | 주가 이력 CSV | data | builtin | 금융 챗봇 |
| `schedule_query` | 예약 등록 미리보기 | productivity | builtin | 내부 전용(마켓 미노출, `show_in_marketplace=false`) |
| `gmail.send` | Gmail 메일 발송 | communication | oauth | 요청 시 |
| `gmail.search` | Gmail 메일 검색 | communication | oauth | |
| `kakao.send_memo` | 카카오톡 '나에게 보내기' | communication | oauth | |
| `google_calendar.create_event` | 구글 캘린더 일정 추가 | productivity | oauth | |
| `slack.post` | Slack 메시지 보내기 | communication | oauth | |
| `webhook.post` | Webhook POST | productivity | http | 팀장 구성 |
| `echo.sample` | Echo (외재화 sample) | productivity | http | |

> `code_interpreter` 빌트인은 ADR-0005(2026-05-22)로 폐기되어 `claude_code` 로 대체되었습니다. 시드 슬러그의 단일 출처는 `backend/config/tools.yaml` 입니다.

외부 연동이 필요한 도구는 `requires_credentials=true`로 표시되며, 사용자는 `/tools` 페이지에서 OAuth 또는 API Key를 등록해야 실행 시 동작합니다.

## 4. 실행 경로

```
agent loop → tool_call { name: "gmail.send", arguments: {...} }
 │
 ▼
tool_registry.dispatch("gmail.send", args, ctx)
 │
 kind == oauth │
 ▼
 ToolCredential.user_id == ctx.user_id AND tool_id == <gmail>
 │
 ▼
 HTTP → Gmail API (SMTP 혹은 REST)
 │
 ▼
 결과 JSON → tool message → LLM
```

`ctx` 객체는 다음 정보를 담습니다:
- `user`: `User` ORM
- `team_id`
- `db`: 현재 세션
- `workspace`: 에이전트 워크스페이스(파일 공유)

## 5. API

| 메서드 | 경로 | 설명 |
| ------ | ---- | ---- |
| `GET` | `/tools` | 활성화된 도구 카탈로그 조회 (카테고리별) |
| `GET` | `/tools/installed` | 내가 자격증명을 등록해둔 도구 목록 |
| `POST` | `/tools/credentials` | 자격증명 저장 (`tool_slug`, `data`) |
| `DELETE` | `/tools/credentials/{tool_id}` | 자격증명 삭제 |
| `POST` | `/tools/test` | 개별 도구를 1회 호출해 연결 상태 확인 (dry-run) |
| `POST` | `/tools/custom` | 사용자 커스텀 HTTP 도구 등록(인증된 누구나). endpoint 는 등록·디스패치 양 시점 모두 SSRF 가드(`check_url_safe`) 통과 필요 |
| `PATCH` | `/tools/{tool_id}` | 본인/관리자 도구 수정. endpoint 변경 시 SSRF 재검증 |
| `DELETE` | `/tools/{tool_id}` | 도구 삭제 |
| `POST` | `/tools/{tool_id}/upvote` · `favorite` · `fork` | 좋아요 · 보관 · 포크(복제) |

> 시스템 카탈로그(시드) 추가는 `backend/config/tools.yaml` 시드(부팅 시 빌트인 카탈로그 upsert)로만 수행합니다. `POST /admin/tools` 같은 라우트는 없습니다.

### 챗봇 바인딩은 `/chatbots/{id}/tools` PUT으로:
```json
{
 "tool_bindings": [
 { "tool_slug": "web_search", "enabled": true },
 { "tool_slug": "gmail.send", "enabled": true, "config": { "from_label": "장금상선" } }
 ]
}
```

## 6. OAuth 플로우 개요

1. 사용자가 `/tools`에서 Gmail "연결" 클릭
2. 서버는 Google OAuth consent URL로 리다이렉트
3. 승인 후 콜백: `/tools/oauth/gmail/callback?code=...`
4. 토큰 교환 → `tool_credentials.data = {access_token, refresh_token, expiry}`
5. 이후 런타임은 만료 시 refresh_token으로 갱신

(실제 Google/Kakao 연동은 별도 후속 작업. 현재 리포에는 카탈로그 뼈대와 빌트인 도구만 구현되어 있습니다.)

## 7. 보안

- **SSRF 가드 (커스텀 HTTP 도구)**: 사용자 등록(`source=user`) 도구의 endpoint 와 webhook URL 은 등록·수정(`POST /tools/custom`, `PATCH /tools/{id}`)과 실제 디스패치 시점(`tool_registry._dispatch_http`/`_dispatch_oauth`) 모두 `app/core/net_guard.check_url_safe` 를 통과해야 합니다. 호스트가 해석되는 **모든 IP** 를 검사해 사설/루프백/링크로컬/예약/멀티캐스트(메타데이터 `169.254.169.254` 포함)를 차단하고(DNS 리바인딩 우회 방어), `http/https` 외 스킴은 거부합니다. 비신뢰 endpoint 에는 사내 tool-server 공유 비밀(`TOOL_SERVER_TOKEN`)을 전송하지 않습니다(시스템 시드 도구에만 동봉).
- `tool_credentials.data`는 민감 정보이므로 다음 중 하나를 적용:
 - **Pgcrypto** `pgp_sym_encrypt`/`decrypt` (서버 시크릿으로)
 - 응용 레벨에서 `cryptography.fernet`로 암호화 후 JSON에 저장
- 로깅에서 자격증명이 출력되지 않도록 `RequestLoggingMiddleware`가 `/tools/credentials` 경로는 body 스킵.
- 권한: 자격증명 CRUD는 본인 계정 대상만. 팀 공용 자격증명은 `team_admin` 이상만 등록/수정 가능.

## 7-1. 자격증명 입력 폼 / 발급 가이드 작성법 (tools.yaml 데이터기반)

`requires_credentials=true` 도구의 자격증명 등록 화면은 **`backend/config/tools.yaml` 의 데이터** 로 동적 렌더됩니다. 프론트엔드(`CredentialForm`)가 라벨드 입력 폼과 발급 가이드를 그려주므로, 사용자는 raw JSON 을 직접 작성할 필요가 없습니다.

핵심: **가이드/URL/필드는 데이터(YAML)에 둡니다. 갱신은 `tools.yaml` 수정만으로 끝나며 코드 변경이 없습니다.** (`catalog_loader.tool_credential_meta()` 가 slug 로 매핑 → `/tools` 라우터가 `ToolOut` 에 직렬화 → 프론트엔드가 폼/가이드 렌더.)

### `credential_schema` — 입력 폼 필드 정의

도구 항목에 `credential_schema` 리스트를 추가하면, 각 항목이 라벨드 입력 필드 한 칸이 됩니다.

```yaml
- slug: slack.post
  display_name: Slack 메시지 보내기
  requires_credentials: true
  credential_schema:
    - key: bot_token          # 저장될 자격증명 키 (백엔드 data 객체의 키)
      label: 봇 토큰 (Bot User OAuth Token)   # 화면 라벨
      type: password          # text | password | url  (password 는 표시/숨김 토글 + 마스킹)
      required: true          # 기본 false. true 면 미입력 시 제출 차단
      placeholder: xoxb-...    # 입력칸 placeholder (선택)
```

- `key` 만 필수, 나머지는 선택(누락 시 `label=key`·`type=text`·`required=false`).
- 비밀값은 `type: password` 로 두면 마스킹 + [표시/숨김] 토글이 붙습니다.
- 빈 선택(`required: false`) 필드는 제출 시 자동 제외됩니다.
- ★ `credential_schema` 를 통째로 **생략하면** 프론트엔드는 기존 raw JSON textarea 로 자동 폴백합니다(하위호환). 사용자가 직접 등록한 커스텀 도구도 메타가 없으므로 JSON 폴백.

### `credential_guide` — 발급 안내 (선택)

`credential_guide` 를 추가하면 폼 안에 접이식 [자격증명 발급 방법 보기] 패널이 생깁니다.

```yaml
  credential_guide:
    steps:                    # 한국어 단계 — 번호목록으로 렌더
      - "Slack API(api.slack.com/apps)에서 [Create New App]으로 앱을 생성합니다."
      - "[OAuth & Permissions > Bot Token Scopes]에 chat:write 를 추가합니다."
      - "[Install to Workspace]로 설치하고, 표시되는 xoxb- 토큰을 복사합니다."
    doc_url: https://api.slack.com/messaging/sending   # [공식 문서 열기 ↗] 링크 (선택)
    note: "토큰은 xoxb- 로 시작하는 Bot User OAuth Token 이어야 합니다."  # 주의 박스(호박색)로 강조 (선택)
```

- `steps`·`doc_url`·`note` 모두 선택이며, 셋 중 하나라도 있으면 가이드 토글이 노출됩니다.
- 발급 절차가 바뀌면(예: 콘솔 메뉴 위치 변경, 새 scope 요구) **이 YAML 만 고치면** 곧바로 반영됩니다.

### 보안 불변식

- `credential_schema`/`credential_guide` 는 **폼 정의·발급 안내 메타데이터일 뿐**, 자격증명 *값* 은 절대 포함하지 않습니다. `/tools` 응답에도 값이 직렬화되지 않습니다.
- 입력된 값은 기존 계약 그대로 `POST /tools/credentials` 의 `{tool_slug, data}` 로 전송되어 저장됩니다(7장 보안 규약 적용).

## 8. MCP 연동 방향

Anthropic의 MCP 프로토콜 기반 커넥터(예: Claude.ai Gmail MCP)를 `kind=mcp` 로 등록하면 동일한 인터페이스로 호출 가능합니다. 이 경우 `tool.parameters_schema`는 MCP 서버가 반환한 스키마를 그대로 저장합니다.

## 9. UI (프론트엔드)

- `/tools`: 카드 그리드. 각 카드: 아이콘, 제목, 카테고리 배지, "연결됨/연결 필요" 상태, "연결하기"/"해제" 버튼.
- `/chatbots/[id]` 편집 화면: "도구" 탭에서 본인이 연결한 도구 목록을 체크박스로 활성화.
- 챗봇 내 도구가 자격증명을 요구하지만 사용자가 연결하지 않은 경우 → 에이전트 실행 시 우아하게 실패하고 "{도구} 연결이 필요합니다" 메시지 반환.
