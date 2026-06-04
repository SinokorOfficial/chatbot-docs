# ADR-0004 · Self-Modifying Chatbot Tools

| | |
|---|---|
| **Status** | Accepted (2026-05-22) |
| **Deciders** | CTO / Platform Eng |
| **Layer** | Product / Tool design |
| **Replaces (in scope)** | [ADR-0003 · Embedded IDE Workspace](0003-embedded-ide-workspace.md) — 동일 사용자 직관에 대한 *제품 컨텍스트 맞춤* 해석 |
| **Related** | [ADR-0002 · User-Scoped AI Execution Platform](0002-user-scoped-ai-execution-platform.md) |

---

## 0. 한 줄 결론

> **사용자가 채팅 안에서 자기 챗봇·워크플로우·문서·스킬·계정을 직접 수정**한다.
> IDE 도, 별도 관리 화면도 필요 없다.
> AI 가 사용자 입력을 받아 *user-scoped tool* 을 호출 → *변경 미리보기* → 사용자 확인 → 적용.

---

## 1. 직관 — 이미 작동 중인 패턴

`backend/app/services/agent.py:160-200` 의 `schedule_query` 도구가 정확히 이 모델:

```
사용자: "매일 아침 9시에 매출 정리해줘"
AI:    [예약 등록 미리보기]
       제목: 매출 정리
       쿼리: 매출 정리해줘
       옵션: ① 매일 오전 9시 (다음: 2026-05-23 09:00)
       — 확인 카드에서 '등록' 누르세요
사용자: [등록]
시스템: ScheduledQuery 행 생성
```

이 *"미리보기 → 확인 → 적용"* 3단계가 우리의 self-modifying 패턴.

본 ADR 은 같은 패턴을 **챗봇 · 워크플로우 · 문서 · 스킬 · 계정** 으로 확장한다.

---

## 2. 추가할 도구 카탈로그 (5개)

각 도구는 *user-scoped* — `workspace.user.id` 로 *본인 소유* 또는 *변경 권한 있음* 만 허용. 권한 없으면 *친화 거절 메시지* (다른 사용자 자원 무단 변경 불가).

### 2-1. `manage_my_chatbot`

내가 소유한 챗봇의 *이름 / 설명 / 시스템 프롬프트 / 모델 / 가시성 / RAG 스코프 / 도구 / 문서* 를 수정·생성·삭제.

```
사용자: "내 '상담봇' 의 시스템 프롬프트를 더 친근하게 바꿔줘"
AI:    [챗봇 수정 미리보기 — 상담봇 (id: ...)]
       Before: 당신은 장금상선 인사팀 어시스턴트입니다.
       After:  당신은 장금상선 인사팀의 친근한 도우미입니다.
               존댓말로 자연스럽게 답하세요.
       — [적용] [수정] [취소]
```

내부: `PATCH /chatbots/{id}` (이미 존재) 를 도구 디스패처가 호출. 권한은 `owner_user_id == user.id` 확인.

### 2-2. `manage_my_workflow`

내 워크플로우 *트리거 시간 / 노드 추가·삭제 / 도구 바꾸기 / fork* 수정.

```
사용자: "그 매주 월요일 워크플로우, 매일로 바꿔줘"
AI:    [워크플로 수정 미리보기 — 매주 월요일 매출 정리]
       Before trigger: cron 0 9 * * 1
       After  trigger: cron 0 9 * * *
       — [적용]
```

내부: `PATCH /workflows/{id}` + `POST /workflows/from-prompt` 활용.

### 2-3. `manage_my_documents`

내가 업로드한 문서 *공유 범위 변경 / 카테고리 / 삭제 / 재인덱싱*.

```
사용자: "어제 올린 '2026 인사 정책' 을 팀 전체에 공유해줘"
AI:    [문서 변경 미리보기]
       파일: 2026 인사 정책.pdf (...)
       Before scope: personal
       After  scope: team
       — [적용]
```

내부: `POST /documents/{id}/share` (이미 존재).

### 2-4. `manage_my_skill_subscription`

마켓에서 스킬 *즐겨찾기 / 가져오기 / 좋아요 / 내 챗봇에 부착*.

```
사용자: "마켓에서 한국어 요약 스킬 가져와서 내 회의록 봇에 붙여줘"
AI:    [스킬 부착 미리보기]
       Skill:    "한국어 요약" (작성자: 홍길동, 좋아요 12)
       Chatbot:  내 회의록 봇
       동작:     skill 을 chatbot 의 도구 슬롯에 추가
       — [적용]
```

### 2-5. `manage_my_account`

내 *테마 / 알림 / 비밀번호 변경 안내 / 사용량 조회 / 가입 정보 수정*.

```
사용자: "오늘 토큰 얼마 썼는지 + 다크 테마로 바꿔줘"
AI:    이번 달 사용량: ... / 다크 테마로 변경하시려면 [적용]
```

비밀번호 변경 *직접 수행* 은 보안상 도구로 안 함 — `/mypage` deep-link 만 안내.

---

## 3. 권한 모델 — User-Scope 강제

모든 도구는 `ToolContext.user` 기반으로 다음을 *서버 레이어에서* 검증:

| 자원 | 허용 조건 |
|---|---|
| Chatbot | `owner_user_id == user.id` 또는 `user.role in (team_admin, super_admin)` |
| Workflow | `owner_user_id == user.id` (Phase 1 에 share 매트릭스 추가) |
| Document | `owner_user_id == user.id` 또는 *팀 admin* |
| Skill (소비) | 누구나 *favorite/upvote/install*. *수정* 은 owner 만 |
| Account | `user_id == user.id` (자기 자신만) |

ADR-0002 §10 의 *user-scope* 정신을 도구 디스패처 진입점에서 다시 한 번 강제. 권한 누락 시 도구가 *"내가 수정 권한이 있는 자원만 다룰 수 있어요"* 친화 응답.

---

## 4. UI/UX — Pending Card 패턴 재사용

이미 `PostAnswerMissionBand` 와 `PendingScheduleCard` 가 있음. 각 도구별 *pending card* 추가:

| 도구 | Card 컴포넌트 |
|---|---|
| `manage_my_chatbot` | `frontend/features/chat/PendingChatbotUpdateCard.tsx` (신규) |
| `manage_my_workflow` | `frontend/features/chat/PendingWorkflowUpdateCard.tsx` (신규) |
| `manage_my_documents` | `frontend/features/chat/PendingDocumentUpdateCard.tsx` (신규) |
| `manage_my_skill_subscription` | `frontend/features/chat/PendingSkillAttachCard.tsx` (신규) |
| `manage_my_account` | 단순 inline diff + `/mypage` 링크 |

각 card 의 공통 props: `before`, `after`, `onConfirm()`, `onCancel()`, `onEdit()` — 변경 적용 후 *해당 도메인 페이지로 빠른 deep-link*.

---

## 5. 구현 사양

### 5-1. `agent.py` 도구 추가

현재 `_exec_tool` (line 116-240) 의 if-elif dispatch 에 5개 분기 추가. 단 *이미 거대 함수* 라 ADR-0001 §13 + full-audit TOP 10 의 *분리 권고*를 같이 처리:

```
backend/app/services/agent/
  __init__.py
  router.py            # 기존 run_agent (Workspace, AgentEvent)
  tool_handlers/
    __init__.py        # dispatch dict
    web_search.py
    code_interpreter.py
    image_generate.py
    schedule_query.py
    stock.py
    manage_chatbot.py       (신규)
    manage_workflow.py      (신규)
    manage_documents.py     (신규)
    manage_skill.py         (신규)
    manage_account.py       (신규)
```

각 handler 시그니처 통일:

```python
async def handle(args: dict, ctx: ToolContext) -> tuple[str, list[dict]]:
    """반환: (LLM 에게 줄 text, frontend pending card 데이터 리스트)"""
```

### 5-2. LLM 도구 카탈로그 (`TOOL_DEFS`)

`agent.py:TOOL_DEFS` 에 5개 추가. 예 (`manage_my_chatbot`):

```python
{
  "type": "function",
  "function": {
    "name": "manage_my_chatbot",
    "description": "내가 소유한 챗봇의 이름·설명·시스템 프롬프트·모델·"
                   "가시성·RAG 스코프를 수정한다. 사용자 확인 후 적용된다.",
    "parameters": {
      "type": "object",
      "properties": {
        "chatbot_id": {"type": "string", "description": "수정할 챗봇 ID 또는 이름"},
        "field": {"type": "string", "enum": ["name", "description", "system_prompt",
                                              "model_id", "visibility", "rag_scope"]},
        "new_value": {"type": "string"},
      },
      "required": ["chatbot_id", "field", "new_value"],
    },
  },
}
```

### 5-3. Pending 확인 흐름

`schedule_query` 가 이미 사용하는 패턴(`_pending_schedule` 이미지 마커)을 일반화:

```python
images.append({
    "_pending": {
        "kind": "chatbot_update",  # or "workflow_update", "document_scope", ...
        "before": {...},
        "after": {...},
        "endpoint": "/chatbots/{id}",
        "method": "PATCH",
        "body": {...},
    }
})
```

Frontend 의 chat 메시지 렌더러에서 `m.pending.kind` 로 분기하여 알맞은 Pending Card 컴포넌트 표시.

### 5-4. 보안

* 권한 검증은 *handler 진입부* 와 *실제 API 호출 시* **이중 확인** (서버는 client 신뢰 X)
* 모든 self-modifying 도구 호출은 `usage_events` 에 `kind=self_modify` 로 기록 — audit
* *위험한 변경* (가시성 public 으로, 챗봇 삭제, 문서 삭제) 은 *추가 확인 단계* (double confirm)
* prompt injection 방어: 사용자 *prompt 안의* "다른 사용자 챗봇 수정해줘" 같은 시도는 *권한 검증으로 자동 거절* (ADR-0002 §2 user 격리)

---

## 6. 갭 분석

| 결정사항 | 현재 코드 | 갭 |
|---|---|---|
| `agent.py` 도구 분리 | `_exec_tool` 124줄 if-elif | 폴더 분리 + dispatch dict (full-audit TOP 10 와 같이) |
| pending card 패턴 일반화 | `_pending_schedule` 단일 | `_pending.kind` 멀티 |
| 권한 검증 도구 진입부 | 부재 | `ToolContext` 에 `assert_owns(resource)` 추가 |
| audit `self_modify` | 부재 | `usage_events.kind` enum 확장 |
| 위험 변경 double confirm | 부재 | Pending Card 안에 *확인 체크박스* + 30초 cooldown |

---

## 7. Phase 매핑

### Phase A — 즉시 (1주)
- [ ] `agent/tool_handlers/` 폴더 분리 + 기존 6개 도구 마이그레이션
- [ ] `manage_my_chatbot` 1개 도구 + `PendingChatbotUpdateCard.tsx` 구현
- [ ] e2e: 새 계정 → 챗봇 1개 만들기 → 채팅으로 시스템 프롬프트 수정 → DB 반영 확인

### Phase B — 1~2주
- [ ] `manage_my_workflow` + 카드
- [ ] `manage_my_documents` + 카드
- [ ] `audit usage_events.kind=self_modify` 통합

### Phase C — 2~4주
- [ ] `manage_my_skill_subscription` + 카드
- [ ] `manage_my_account` (테마/사용량/알림)
- [ ] 위험 변경 double-confirm UX
- [ ] 사용자 사용 패턴 트래킹 — *어떤 self-modify 가 자주 쓰이는지* 데이터 수집 후 우선순위 재조정

---

## 8. Open Questions

1. **자연어 ID 매칭** — 사용자가 "내 상담봇" 이라 말하면 *fuzzy 매칭* (이름 substring) — 여러 개 매칭 시 *선택 카드*. 매칭 알고리즘은 LLM 에 위임할지 backend 가 할지.
2. **위험도 등급** — 어떤 변경이 *double confirm* 인지 *single click* 인지의 정책 (delete / public 화 / cron 변경 / system prompt 변경 / ...).
3. **bulk operation** — "이번 주 올린 문서 전부 팀 공유로" 같은 다중 자원 변경. 트랜잭션 처리 어떻게.
4. **roll-back** — 적용 후 사용자가 "방금 거 취소해줘" 시 *N분 안* roll-back 지원? 또는 history 만.
5. **다른 사용자 챗봇 *조회* (read-only)** — *수정* 은 금지지만 *공개 챗봇 fork* 는 허용. 도구 권한 매트릭스에 read/write 분리.

---

## 9. 1페이지 결정 요약

```
┌──────────────────────────────────────────────────────────────────┐
│  비전                                                              │
│   "사용자가 채팅 안에서 자기 챗봇·워크플로우·문서·스킬을 *직접*"   │
│   "수정한다. IDE 도, 별도 관리 화면도 필요 없다."                  │
│                                                                  │
│  도구 5개                                                          │
│   manage_my_chatbot                                              │
│   manage_my_workflow                                             │
│   manage_my_documents                                            │
│   manage_my_skill_subscription                                   │
│   manage_my_account                                              │
│                                                                  │
│  패턴                                                              │
│   schedule_query 의 [미리보기 → 확인 → 적용] 3단계 일반화         │
│                                                                  │
│  보안                                                              │
│   user-scoped 권한 *2중 검증* (handler 진입 + API)                │
│   위험 변경 (delete / 공개화) 은 *추가 확인*                       │
│   모든 호출 audit (usage_events.kind=self_modify)                │
│                                                                  │
│  코드 분리                                                         │
│   agent.py `_exec_tool` 124줄 → `agent/tool_handlers/` 폴더       │
│   pending card `_pending.kind` 로 멀티 타입                       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 10. 변경 이력

- **2026-05-22** — 초안 Accepted. 같은 날 *오너 직관* ("채팅으로 바로 반영하면 IDE 안 해도 되지 않나") 에서 출발하여 정식 결정으로 정리. ADR-0003 은 Deferred. ADR-0002 의 user-scope 정신을 *도구 디스패처* 진입부로 확장.
