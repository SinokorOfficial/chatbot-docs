# 도구 마켓플레이스

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
| `builtin` | `tool_registry.py` 내 파이썬 함수 | `web_search`, `code_interpreter`, `image_generate`, `stock_price`, `stock_history` |
| `http` | 등록된 URL로 POST 호출 (webhook 스타일) | 사내 API, 슬랙 Incoming Webhook |
| `oauth` | 자격증명을 꺼내 외부 API 호출 | Gmail 발송, 캘린더 조회, Microsoft 365 |
| `mcp` | MCP 서버의 도구 호출 | `mcp__claude_ai_Gmail__*` 등 Anthropic MCP 커넥터 |

## 3. 기본 시드 카탈로그

서버 기동 시 `tool_registry.load_builtin_catalog()`가 다음 도구를 `tools` 테이블에 upsert 합니다.

| slug | display_name | category | kind | 기본 장착 대상 |
| ---- | ------------ | -------- | ---- | -------------- |
| `web_search` | 웹 검색 | search | builtin | 모든 챗봇 권장 |
| `code_interpreter` | 코드 실행 | data | builtin | 분석용 챗봇 |
| `image_generate` | 이미지 생성 | productivity | builtin | 마케팅 챗봇 |
| `stock_price` | 실시간 주가 | data | builtin | 금융 챗봇 |
| `stock_history` | 주가 이력 CSV | data | builtin | 금융 챗봇 |
| `gmail.send` | 지메일 발송 | communication | oauth | 요청 시 |
| `gmail.search` | 지메일 검색 | communication | oauth | |
| `kakao.send_message` | 카카오톡 나에게 보내기 | communication | oauth | |
| `kakao.friends_message` | 카카오톡 친구에게 보내기 | communication | oauth | |
| `google_calendar.create_event` | 구글 캘린더 일정 등록 | productivity | oauth | |
| `microsoft365.outlook_search` | 아웃룩 메일 검색 | communication | oauth | |
| `webhook.post` | 사내 Webhook POST | productivity | http | 팀장 구성 |

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
| `POST` | `/admin/tools` | (super_admin 전용) 새 도구 카탈로그 추가 |

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

- `tool_credentials.data`는 민감 정보이므로 다음 중 하나를 적용:
 - **Pgcrypto** `pgp_sym_encrypt`/`decrypt` (서버 시크릿으로)
 - 응용 레벨에서 `cryptography.fernet`로 암호화 후 JSON에 저장
- 로깅에서 자격증명이 출력되지 않도록 `RequestLoggingMiddleware`가 `/tools/credentials` 경로는 body 스킵.
- 권한: 자격증명 CRUD는 본인 계정 대상만. 팀 공용 자격증명은 `team_admin` 이상만 등록/수정 가능.

## 8. MCP 연동 방향

Anthropic의 MCP 프로토콜 기반 커넥터(예: Claude.ai Gmail MCP)를 `kind=mcp` 로 등록하면 동일한 인터페이스로 호출 가능합니다. 이 경우 `tool.parameters_schema`는 MCP 서버가 반환한 스키마를 그대로 저장합니다.

## 9. UI (프론트엔드)

- `/tools`: 카드 그리드. 각 카드: 아이콘, 제목, 카테고리 배지, "연결됨/연결 필요" 상태, "연결하기"/"해제" 버튼.
- `/chatbots/[id]` 편집 화면: "도구" 탭에서 본인이 연결한 도구 목록을 체크박스로 활성화.
- 챗봇 내 도구가 자격증명을 요구하지만 사용자가 연결하지 않은 경우 → 에이전트 실행 시 우아하게 실패하고 "{도구} 연결이 필요합니다" 메시지 반환.
