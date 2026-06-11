# 챗봇(에이전트) 설계

## 1. 개념

"챗봇"은 재사용 가능한 에이전트 프로필입니다. 하나의 챗봇은 다음을 묶어서 보관합니다.

- 이름/설명/아이콘
- 모델 ID (`openai/gpt-4o` 등)
- 시스템 프롬프트
- 가시성 (`private` · `public` · `shared`) — private=소유자만, public=소유자 팀 전체, shared=소유자 팀 + 승인된 추가 팀(교차팀). 자세한 내용은 [chatbot_sharing.md](./chatbot_sharing.md)
- 참조 문서 집합 (RAG 스코프)
- 허용된 도구 집합 (마켓에서 선택)

사용자는 대화를 시작할 때 어떤 챗봇으로 대화할지 고릅니다. 챗봇을 선택하지 않으면 기본 "일반" 챗봇(= 기존 동작)이 적용됩니다.

## 2. 가시성 (Visibility)

| 값 | 설명 |
| --- | --- |
| `private` | **오너만** 대화할 수 있음. `conversations` 테이블에서도 `user_id = owner`인 대화만 생성. |
| `public` | **같은 팀의 모든 사용자**가 대화 가능(교차팀 아님 — 오너 팀 한정). 단, 문서는 챗봇에 연결된 것만 바라봄(오너 관점에서 "내가 넣은 문서"). |
| `shared` | **소유자 팀 전체 + `chatbot_team_access` 에 등록된 추가 팀**의 구성원이 대화 가능(교차팀 공유). 단, 타 팀 사용자에게는 RAG 가 챗봇에 연결된 문서(linked-only)로 강제 격리됨(`resolve_rag_scope_doc_ids`). 자세한 내용은 [chatbot_sharing.md](./chatbot_sharing.md). |

Public 챗봇은 예를 들어 "인사규정 Q&A 봇", "기술지원 봇"처럼 팀 공용 어시스턴트를 만들 때 사용합니다. 다른 팀원은 문서에 직접 접근 권한이 없더라도 챗봇을 통해 질의응답이 가능합니다.

## 3. RAG 스코프

| 스코프 | 설명 | 권장 사용처 |
| --- | --- | --- |
| `linked_only` | `chatbot_documents`로 연결된 문서만 | Public 챗봇 기본값 (사용자 요청) |
| `owner_visible` | 오너가 조회 가능한 모든 문서 (팀 공용 + 오너 개인 + 오너에게 공유된 문서) | 오너 개인 비서형 private 챗봇 |
| `team_all` | 팀 전체 문서 | 범용 팀 어시스턴트 |

## 4. 데이터 모델 (신규 테이블)

자세한 스키마는 [erd.md](./erd.md)의 `chatbots`, `chatbot_documents`, `chatbot_tools`를 참고하세요.

```
chatbots (id, team_id, owner_user_id, name, description,
 visibility, system_prompt, model_id,
 use_rag, rag_scope, vectorstore_partition,
 created_at, updated_at)

chatbot_documents (id, chatbot_id, document_id, created_at)

chatbot_tools (id, chatbot_id, tool_id, enabled, config, created_at)
```

## 5. API

| 메서드 | 경로 | 설명 |
| ------ | ---- | ---- |
| `GET` | `/chatbots` | 내가 접근 가능한 챗봇 목록(내 private + 팀 public) |
| `POST` | `/chatbots` | 챗봇 생성 |
| `GET` | `/chatbots/{id}` | 단건 상세 |
| `PATCH`| `/chatbots/{id}` | 이름/프롬프트/스코프/가시성/`extra_team_ids`(공유 팀 인라인 지정) 등 수정. 별도 `/shares` 엔드포인트는 없음 — 공유 팀은 POST·PATCH 의 `extra_team_ids` 로 처리(`visibility != shared` 면 백엔드가 자동 비움) |
| `DELETE`| `/chatbots/{id}` | 삭제 (오너 또는 team_admin+) |
| `PUT` | `/chatbots/{id}/documents` | 연결된 문서 목록 교체 |
| `PUT` | `/chatbots/{id}/tools` | 연결된 도구 목록 교체 |

### 챗봇 요청 예
```http
POST /chatbots
Authorization: Bearer <token>
Content-Type: application/json

{
 "name": "인사규정 봇",
 "description": "인사 정책 FAQ",
 "visibility": "public",
 "model_id": "openai/gpt-4o",
 "system_prompt": "당신은 장금상선 인사팀의 공식 어시스턴트입니다...",
 "use_rag": true,
 "rag_scope": "linked_only",
 "document_ids": ["<doc-uuid-1>", "<doc-uuid-2>"],
 "tool_slugs": ["web_search", "gmail.send"]
}
```

## 6. 챗봇 ↔ 대화 통합

`POST /chat/stream` 입력에 `chatbot_id`(nullable) 필드 추가. 서버 측:

1. `chatbot_id`가 있으면 `chatbot_service.get_for_use()` 로 로드 & 권한 체크(`_user_can_access`)
 - 소유자는 항상 허용
 - 같은 팀이면 `private` 만 소유자 전용, `public`/`shared` 는 팀원 허용
 - 다른 팀이면 `shared` + `chatbot_team_access` 에 내 팀이 등록된 경우만 허용
2. 시스템 프롬프트를 챗봇 값으로 교체 (기본 페르소나와 병합)
3. `use_rag && rag_scope`에 따라 `services.chatbot_rag.search_for_chatbot()` 호출
4. 도구는 `chatbot_tools`에 등록된 것만 `TOOL_DEFS`로 전달
5. `Conversation.chatbot_id`를 기록 (감사, 재사용)

## 7. 권한 요약

| 작업 | 허용 역할 |
| ---- | --------- |
| 본인 챗봇 생성 | 모든 사용자 |
| 다른 팀원의 public 챗봇 사용 | 같은 팀 소속 사용자 |
| 다른 팀의 챗봇 사용 | ❌ (팀 격리) |
| 본인 챗봇 수정/삭제 | 오너 |
| 팀 내 다른 오너의 챗봇 수정/삭제 | `team_admin` 이상 |
| 챗봇에 문서 연결 | 연결할 문서가 **사용자에게 보이는** 문서여야 함 (팀 공용 or 본인 개인 or 본인에게 공유된 문서) |

## 8. 성능 고려 사항

- `rag_scope=linked_only`일수록 검색 대상 청크 수가 작아 빠릅니다.
- 연결된 문서가 많은 챗봇은 `vectorstore_partition`을 지정하여 향후 전용 파티션으로 이관 가능.
- 챗봇별 캐시(`chatbot_service` 모듈에 TTLCache) — 자주 쓰이는 챗봇의 시스템 프롬프트/스코프 id 목록을 메모리에 1분 유지.

## 9. UX (프론트엔드)

- `/chatbots` 목록 페이지: 내 챗봇 + 팀 public 챗봇. "새 챗봇" 버튼.
- `/chatbots/[id]` 편집 페이지: 기본 정보, 문서 선택, 도구 선택, 시스템 프롬프트.
- `/chat` 사이드바: 상단에 챗봇 선택 드롭다운 → 선택 시 `chatbot_id` 세트 후 새 대화 시작.
- 로그아웃 버튼은 전 페이지 공통(`AppShell` 헤더 + `/chat` 사이드바).
