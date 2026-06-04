# ADR-0007 · Claude Code Mode 분리 & Workspace Persistence Hardening

| | |
|---|---|
| **Status** | Accepted (2026-06-04) |
| **Deciders** | CTO |
| **Layer** | Product + Frontend Architecture |
| **Supersedes (in scope)** | ADR-0006 §2 "자동 slide-out" 의 *일부* — claude code 모드 ON 시점에만 Workspace 마운트 |
| **Related** | [ADR-0002 User-Scoped Platform](0002-user-scoped-ai-execution-platform.md) · [ADR-0005 Claude Code Agent](0005-claude-code-as-coding-agent.md) · [ADR-0006 Progressive AI Workspace](0006-progressive-ai-workspace.md) |

---

## 0. 한 줄 결론

> `/chat` 의 *기본 상태* 는 **단순 채팅** 이다. **Claude Code 모드는 명시적 토글로만 켜지는 고급 모드**.
> Workspace Panel 은 *모드 ON 일 때만 마운트* — OFF 면 Provider 조차 mount 되지 않는다.
> 그리고 *대화별로 workspace 가 다르므로*, `Conversation.workspace_id` 로 N:1 매핑하고 *activeId 변경 시 SET_WORKSPACE_ID → Provider 가 직접 fetch* 한다.

---

## 1. Context — 왜 추가 분리가 필요한가

ADR-0006 적용 후 두 가지 *실사용 버그* 가 노출됨:

1. **단순 채팅에서도 Workspace UI 가 항상 보임**
   - 새 사용자는 "이게 뭐지" — 진입 장벽 0 목표가 무너짐
   - `WorkspaceProvider` 가 *항상* mount → 불필요한 SSE 분기 + 파일 fetch 부담
2. **대화 전환 시 이전 workspace 의 파일이 그대로 남음**
   - `activeId` 가 바뀌어도 `WorkspaceProvider.state.files` 는 *이전 conversation 의 workspace_id 의 파일 그대로*
   - 사용자: *"대화를 바꿨는데 왜 옛날 파일이 보이지?"*

→ ADR-0006 의 자동 slide-out 은 **claude code 가 켜진 대화** 에 한정해야 한다.
→ Workspace 와 Conversation 은 1:1 이 아니라 *N:1* — 한 workspace 를 여러 대화가 공유 가능, 하지만 *한 대화는 정확히 한 workspace*.

---

## 2. Decision — 모드 완전 분리 + Workspace Persistence 강화

### 2-1. `/chat` 두 모드

| 모드 | 트리거 | UI 마운트 | Agent 선택 |
|---|---|---|---|
| **단순 채팅 (default)** | 신규 대화 또는 사용자가 모드 OFF | Chat Panel only. `WorkspaceProvider` *없음* | LLM 직접 응답. tool 호출 없음 |
| **Claude Code Mode (고급)** | 사용자 명시 토글 ON | `<WorkspaceProvider>` + `<WorkspacePanel>` mount | `claude_code` 분기 + 11 type SSE + workspace 영속 |

핵심: **OFF 상태에선 Workspace 관련 React tree 가 *전혀 존재하지 않음*** — DOM, Context, SSE 분기, 파일 fetch *모두* 없음.

```tsx
// frontend/app/(workspace)/chat/page.tsx
{claudeCodeMode ? (
  <WorkspaceProvider conversationId={activeId}>
    <LayoutSwitcher>
      <ChatPanel />
      <WorkspacePanel />
    </LayoutSwitcher>
  </WorkspaceProvider>
) : (
  <ChatPanel />   // 단순 채팅 — Workspace 컨텍스트 0
)}
```

### 2-2. Workspace Persistence — Conversation.workspace_id 직결

```
Conversation (N) ──── workspace_id (FK) ────► Workspace (1)
```

- `conversations.workspace_id` 컬럼은 ADR-0006 §5 에서 이미 추가됨. **본 ADR 은 그 컬럼을 *프런트엔드에서 1급 시민으로*** 끌어올린다.
- `activeId` 변경 시 흐름:
  1. ConversationList 가 `setActiveId(newId)`
  2. 새 conversation row 의 `workspace_id` 가 `WorkspaceProvider` 의 prop 으로 흘러감
  3. Provider `useEffect([workspaceId])` 가 **`SET_WORKSPACE_ID` 디스패치**
  4. reducer 가 `state.workspaceId` 만 바꾸고 `files: []` 로 리셋
  5. **같은 effect 가 `GET /workspaces/{id}/files` 호출** → 응답으로 `REPLACE_FILES` 디스패치
  6. FileExplorer 즉시 새 파일 트리 표시

→ 대화 전환 = workspace fetch. UI mount/unmount 와 무관.

### 2-3. 6-Effort Mode + Anthropic Model Select

Claude Code 모드 ON 시 헤더에 두 개의 셀렉터:

| 컨트롤 | 값 | 의미 |
|---|---|---|
| **Effort** | `minimal / low / medium / high / very_high / max` | `--effort` 플래그 → 사고 토큰 예산 + 도구 재시도 횟수 |
| **Model** | Anthropic 한정 (`claude-opus-4-7`, `claude-sonnet-4-5`, `claude-haiku-4-5` 등) | `--model` 플래그. OpenAI/Google *제외* — claude code 는 Anthropic 전용 |

- Effort 6단계: ADR-0005 의 3단계(`low/medium/high`) → 6단계 확장. `minimal` 은 즉답 우선, `max` 는 깊은 reasoning + 대량 tool loop 허용.
- 모델 셀렉트는 *Anthropic 모델만* — 다른 provider 는 `/chat` 단순 채팅 모드에서 별도 셀렉터로 노출 (본 ADR scope 아님).
- 두 값은 `conversations.effort` / `conversations.model` 에 영속 (대화 단위 기억).

---

## 3. Frontend 구조 변화

### 3-1. WorkspaceProvider — conversationId/workspaceId 분리

```tsx
// features/workspace/WorkspaceProvider.tsx
type Props = { conversationId: string; children: ReactNode };

export function WorkspaceProvider({ conversationId, children }: Props) {
  const [state, dispatch] = useReducer(workspaceReducer, initialState);

  // (1) conversationId → workspaceId 조회 (이미 conversation 객체에 있음)
  const workspaceId = useConversation(conversationId)?.workspace_id;

  // (2) workspaceId 변경 시 SET + fetch + REPLACE_FILES
  useEffect(() => {
    if (!workspaceId) return;
    dispatch({ type: "SET_WORKSPACE_ID", workspaceId });
    fetch(`/api/workspaces/${workspaceId}/files`)
      .then((r) => r.json())
      .then((files) => dispatch({ type: "REPLACE_FILES", files }));
  }, [workspaceId]);

  // (3) SSE 이벤트는 기존대로 file_created / file_updated 누적
  return <Ctx.Provider value={{ state, dispatch }}>{children}</Ctx.Provider>;
}
```

### 3-2. reducer 액션 추가

| 액션 | 페이로드 | 효과 |
|---|---|---|
| `SET_WORKSPACE_ID` | `{ workspaceId }` | `state.workspaceId = workspaceId; state.files = []` (리셋) |
| `REPLACE_FILES` | `{ files: WorkspaceFile[] }` | `state.files = files` (fetch 결과 통째 교체) |

기존 `FILE_CREATED` / `FILE_UPDATED` / `RESET_TASK` 와 *공존*.

### 3-3. Mode Toggle

`useChatbotMode()` hook 이 `conversations.claude_code_enabled` boolean 을 관리.
- ON → `<WorkspaceProvider>` mount + `/chat/stream` 호출 시 `?agent=claude_code` 쿼리
- OFF → 단순 채팅. `/chat/stream` 은 LLM 직접

---

## 4. Backend 변화

기존 ADR-0006 의 `/workspaces/{id}/files` 엔드포인트를 *그대로 사용*. 변경점:

- `chat/router.py` 가 *모드 ON 일 때만* claude_runner 분기 호출. OFF 면 plain LLM stream.
- `conversations` 테이블에 `claude_code_enabled BOOLEAN DEFAULT FALSE`, `effort VARCHAR(16) DEFAULT 'medium'`, `model VARCHAR(64) DEFAULT 'claude-sonnet-4-5'` 컬럼 추가.
- workspace 생성은 *모드 처음 ON 되는 시점* lazy — OFF 상태로 끝나면 workspace 0개 (디스크 절약).

---

## 5. Consequences

### 5-1. 좋아지는 것

- **단순 채팅 진입 장벽 진짜 0** — Workspace UI 가 *DOM 에 없음*. 새 사용자는 *완전한 챗봇 UX*.
- **대화 전환 시 파일 정확히 동기화** — `activeId` 바뀌면 `workspaceId` 따라가고, files 가 새로 fetch 됨. 이전 잔상 0.
- **Workspace 비용 절감** — claude code 안 쓰는 대화는 디렉토리 0, DB row 0, file watcher 0.
- **Mode 토글 = 명확한 사용자 의도 신호** — 분석/실험 측면에서 *언제 사용자가 코드 모드 켰는지* 깔끔하게 추적.
- **6-effort + Anthropic model** — 가벼운 질문엔 `minimal/haiku`, 무거운 작업엔 `max/opus`. 비용 사용자 통제.

### 5-2. Trade-offs / 비용

1. **UI 마운트와 무관한 fetch 비용**
   - `activeId` 변경마다 `GET /workspaces/{id}/files` 1회 호출. 200 대화면 200회 fetch (탐색 시).
   - 완화: SWR/React Query 캐싱 + last-active workspace prefetch. 그러나 *처음 한 번* 은 항상 미스.

2. **모드 토글 비용 — re-mount**
   - ON ↔ OFF 토글 시 `WorkspaceProvider` 가 unmount/mount → state 휘발. *재 fetch* 필요.
   - 사용자가 빠르게 토글하면 깜빡임 + 네트워크 낭비. 완화: 300ms debounce + 상위에 *얇은 캐시*.
   - 의도된 비용 — *완전 분리* 의 대가. 부분 분리(컴포넌트만 숨김) 보다 인지 부담/번들 영향 측면에서 우위.

3. **잔존 `ChatbotVisibility.shared` 와의 관계**
   - ADR-0002 §10 의 user-scope 정신은 *workspace 도 user_id+tenant_id 필터* 로 그대로 적용.
   - 그러나 *공유 챗봇* (visibility=shared) 은 *여러 사용자가 같은 봇 구성 공유*. workspace 는 *대화 단위* 이므로 **공유 챗봇이라도 대화 (=workspace) 는 user-scoped**.
   - 즉 공유는 *챗봇 정의 (system prompt / model / tools)* 까지, workspace/파일은 *각자*. ADR-0002 §10-3 의 "공유 챗봇 ≠ 공유 데이터" 원칙 유지.
   - 향후 *팀 workspace* (여러 사용자가 같은 workspace 공유) 요구가 생기면 별도 ADR.

4. **6-effort 정의의 backend 매핑**
   - `claude` CLI 의 `--effort` 는 현재 *3단계만* 공식 지원. `minimal/very_high/max` 는 우리가 *budget_tokens* + *tool retry caps* 로 매핑하는 가상 단계.
   - 매핑 표는 별도 docs 에 (`docs/claude-code/effort-mapping.md`).

5. **모델 추가/제거 시 UI 업데이트 부담**
   - Anthropic 모델 목록은 *백엔드 endpoint* `GET /models?provider=anthropic` 로 동적 로드. 하드코딩 X.

---

## 6. Alternatives

### A. 모드 분리 X — Workspace UI 항상 mount + 조건부 hide
- 장점: re-mount 비용 없음. fetch 1회로 끝.
- 단점: ADR-0006 직후의 *현재 상태*. 단순 채팅 진입 장벽 보존 못함. 번들/SSE 항상 활성.
- → **기각**. 모드 분리는 *제품 결정*.

### B. Workspace fetch 를 *컴포넌트 mount 시점* 에 (WorkspacePanel 안)
- 장점: Provider 가 가벼움.
- 단점: Provider 가 `state.files` 를 *모르는* 상태가 됨. tool_use 이벤트가 도착해도 *어디에 누적할지* 모름.
- → **기각**. Provider 가 *workspace 의 single source of truth* 여야 함.

### C. activeId 변경마다 *Provider 자체* 를 key-remount
- 장점: 코드 단순. `<WorkspaceProvider key={activeId}>`.
- 단점: 같은 workspace_id 인데 conversation 만 다른 경우(드물지만 가능) 불필요 re-mount.
- → **거의 채택 직전**. 하지만 *명시적 dispatch* 가 디버깅에 유리해 §2-2 안 선택.

### D. effort 3단계 유지 (ADR-0005 그대로)
- 장점: claude CLI 그대로.
- 단점: 사용자가 *체감* 가능한 granularity 부족. `minimal` 과 `max` 가 *실제 비용 차 10배*.
- → **기각**. 6단계 매핑 docs 로 보완.

---

## 7. 관련

- ADR-0002 §10 — user-scope 원칙 (workspace 도 동일 적용)
- ADR-0005 — claude_code agent 분기 (본 ADR 이 *모드 ON 시점만* 활성화로 좁힘)
- ADR-0006 §5 — workspace persistence DB schema (본 ADR 이 *프런트엔드 dispatch flow* 추가)
- ADR-0006 §7-2 — chat/page.tsx 통합 (본 ADR 이 *조건부 mount* 로 재작성)
- docs/claude-code/effort-mapping.md — 6단계 effort 의 백엔드 파라미터 매핑 (별도 문서)

---

## 8. 변경 이력

- **2026-06-04** — `/chat` 의 두 실사용 버그 (단순 채팅에서도 Workspace 보임 / 대화 전환 시 이전 파일 잔존) 확인 후 작성. Workspace 컴포넌트 *조건부 mount* + `SET_WORKSPACE_ID/REPLACE_FILES` dispatch flow + 6-effort + Anthropic-only 모델 셀렉트 결정. 구현은 P0 (1주) 예정.
