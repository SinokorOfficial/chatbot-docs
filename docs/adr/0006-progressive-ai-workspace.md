# ADR-0006 · Progressive AI Workspace

| | |
|---|---|
| **Status** | Accepted (2026-05-22) |
| **Deciders** | CTO |
| **Layer** | Product (제품 재정의) |
| **Supersedes (in scope)** | "챗봇" 정의. [ADR-0003 Embedded IDE](0003-embedded-ide-workspace.md) 는 Builder Mode 의 *상위 진화* 로 재배치 |
| **Related** | [ADR-0002 User-Scoped Platform](0002-user-scoped-ai-execution-platform.md) · [ADR-0005 Claude Code Agent](0005-claude-code-as-coding-agent.md) |

---

## 0. 한 줄 결론

> 우리 제품은 **"챗봇"** 이 아니다. **"Progressive AI Workspace"** 다.
> 사용자는 *대화* 로 시작하고, *AI 가 작업을 시작하면* 우측 Workspace 가 *자동으로 열리며*,
> 깊은 작업에는 *IDE 처럼 확장* 된다.
>
> "챗봇에 파일 기능을 붙이는 것" 이 아니라 **"AI Workspace 에 대화형 Agent Console 을 붙이는 것"**.

---

## 1. Context — 왜 이 재정의인가

ADR-0005 적용 후 사용자가 실제 채팅에서 `claude_code` 호출한 결과:

- ✅ Backend 작동: docker 안 claude 가 hello.py 만들고 응답
- ❌ Frontend: *"파일을 만들었습니다"* 텍스트 한 줄. 작업 흔적·다운로드·폴더 *모두 없음*
- 🔥 사용자 인식: *"파일도 클릭하고 내 폴더도 보고…"*

→ "챗봇 + 코드 도구" 모델로는 *작업의 결과* 가 보이지 않음.
→ 우리 사업은 **AI 가 실제로 사는 작업공간**. 챗봇은 *그 안의 한 모드*.

---

## 2. 3 Mode 레이아웃 — Progressive Disclosure

```
┌──────────────────────────────────────────────────────────────────────┐
│  Chat Mode (default)                                                  │
│  ─────────────────────────────────────────────────                  │
│                                                                      │
│                       [ Chat 화면 — full width ]                     │
│                                                                      │
│  • 진입 장벽 0. 새 사용자는 챗봇처럼 인식.                            │
│  • Workspace UI 가 *전혀* 안 보임.                                    │
└──────────────────────────────────────────────────────────────────────┘

                            ↓ AI 가 tool 실행 시 자동 전환

┌──────────────────────────────────────────────────────────────────────┐
│  Assisted Mode (자동 slide-out)                                       │
│  ──────────────────────────────────────────────────────              │
│                                                                      │
│     [ Chat 60% ]              │   [ Workspace Panel 40% ]            │
│      Agent Console            │   ┌─ File Explorer                  │
│                               │   ├─ Tool Execution Log              │
│                               │   ├─ stdout / stderr                 │
│                               │   └─ Cost / Download                 │
│                                                                      │
│  • tool_start / file_created 이벤트 시 우측이 *부드럽게* slide-out    │
│  • 드래그로 비율 조절 (Chat 60→40, Workspace 40→60)                  │
└──────────────────────────────────────────────────────────────────────┘

                            ↓ "확장" 버튼 또는 깊은 작업 진입

┌──────────────────────────────────────────────────────────────────────┐
│  Builder Mode (full power)                                            │
│  ───────────────────────────────────────────────────────────         │
│                                                                      │
│  [ Files 20% ]  │  [ Agent Console / Editor / Logs 50% ]  │ [Preview 30%] │
│   ▾ /workspace  │   ────────────────                       │   ┌──────┐ │
│     ├ hello.py  │   📝 hello.py (Monaco editor)            │   │ live │ │
│     ├ data.csv  │   ────────────                           │   │ web  │ │
│     └ README.md │   $ python hello.py                      │   │ view │ │
│                 │   hi from sandbox                        │   └──────┘ │
│                                                                      │
│  • Chat 은 *하단 dock* 또는 *상단 thin bar* 로 축소                  │
│  • IDE 에 가까운 UX. ADR-0003 의 그림이 여기서 부활                  │
└──────────────────────────────────────────────────────────────────────┘
```

### 2-1. Workspace Panel 4 상태

| 상태 | 트리거 | 폭 | 내용 |
|---|---|---|---|
| **collapsed** | 기본 (Chat Mode) | 0 | 안 보임. *"우측에서 끌어오기"* 핸들만 |
| **small** | 자동 slide-out (tool 실행 시) | 360~400px | File explorer + 최근 tool log 2~3건 |
| **expanded** | 사용자 클릭 ("크게") | 50% | 모든 요소 풀 표시 + preview |
| **fullscreen** | 사용자 "Builder Mode" 진입 | 100% | Chat dock 으로 축소, 3-pane Builder |

상태 전환은 **사용자 explicit 또는 시스템 자동** — 사용자 마지막 선택 *기억* (localStorage / user_pref).

---

## 3. 자동 트리거 — 어떤 이벤트가 우측을 여는가

다음 이벤트가 발생하면 `collapsed → small` 자동 전환:

- `file_created` / `file_updated`
- `tool_start` (claude_code / 향후 다른 작업 도구)
- `preview_ready`
- `stdout` 첫 라인 (긴 출력 예상 시)

사용자가 이미 *닫음* (`collapsed` 명시) 상태면 자동 전환 *X* — UX 강요 금지. 대신 *알림 dot* (📌) 우측 상단에.

---

## 4. Structured SSE 이벤트 카탈로그

기존 `TextDeltaEvent / StepStartEvent / StepEndEvent` 가 부족. 11개 타입으로 확장:

| 이벤트 | 페이로드 | 발사 시점 |
|---|---|---|
| `agent_start` | `{agent: "claude_code", task_id, model}` | run_claude_code 진입 |
| `tool_start` | `{tool_id, name: "Bash"\|"Edit"\|"Read", input: {...}}` | stream-json `tool_use` 블록 |
| `tool_delta` | `{tool_id, chunk: str}` | (Phase 1) Bash stdout 실시간 |
| `tool_end` | `{tool_id, ok, result_preview, duration_ms}` | stream-json `tool_result` |
| `file_created` | `{path, size_bytes, mime, content_b64?}` | workspace 디렉토리 watcher |
| `file_updated` | `{path, size_bytes, mime, content_b64?}` | watcher |
| `stdout` | `{tool_id?, line: str}` | Bash 진행 라인 |
| `stderr` | `{tool_id?, line: str}` | Bash 에러 라인 |
| `cost_update` | `{cost_usd, input_tokens, output_tokens, model}` | stream-json `result` 또는 중간 |
| `agent_done` | `{ok, total_cost_usd, duration_ms, files_changed: int}` | run_claude_code 종료 |
| `error` | `{category, message, retriable: bool}` | 모든 exception 경로 |

이벤트는 **SSE event field** + **JSON payload** — 우리 chat SSE 와 통합. Frontend 가 `event_type` 으로 분기해 *Chat 메시지* 또는 *Workspace Panel* 로 분배.

```
event: tool_start
data: {"tool_id":"t1","name":"Bash","input":{"command":"echo hi > hello.py"}}

event: stdout
data: {"tool_id":"t1","line":""}

event: tool_end
data: {"tool_id":"t1","ok":true,"result_preview":"","duration_ms":42}

event: file_created
data: {"path":"hello.py","size_bytes":21,"mime":"text/x-python"}
```

---

## 5. Workspace Persistence

기존 `tempfile.mkdtemp(prefix="claude-")` 임시 디렉토리 폐기. 새 구조:

```
{repo}/data/workspaces/
  └─ {workspace_id}/                ← workspace_id ≠ user_id (한 사용자가 여러 workspace 가능)
       ├─ files/                    ← claude code 가 /workspace 로 마운트
       │   ├─ hello.py
       │   ├─ data.csv
       │   └─ ...
       ├─ snapshots/                ← Phase B
       │   ├─ 2026-05-22T12-00-00.tar.zst
       │   └─ ...
       └─ .meta.json                ← {workspace_id, user_id, tenant_id, created_at, last_active_at}
```

DB:

```sql
CREATE TABLE workspaces (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL,
  tenant_id UUID,
  name VARCHAR(200),            -- 사용자 표시명
  storage_path TEXT NOT NULL,   -- data/workspaces/{id}/files/
  created_at TIMESTAMPTZ,
  last_active_at TIMESTAMPTZ,
  archived_at TIMESTAMPTZ
);

CREATE TABLE workspace_files (   -- 메타데이터 (성능 최적화 — 대용량 디렉토리 listing 회피)
  workspace_id UUID,
  path TEXT,
  size_bytes BIGINT,
  mime VARCHAR(64),
  created_by VARCHAR(32),       -- 'user' | 'claude_code' | 'agent'
  created_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ,
  PRIMARY KEY (workspace_id, path)
);
```

ADR-0002 §10 의 *user-scope* 정신 유지 — 모든 쿼리에 `(user_id, tenant_id)` 필터.

대화 ↔ workspace 관계:
- `conversations` 테이블에 `workspace_id` 컬럼 추가 (FK, nullable)
- 같은 대화에 *기본 workspace 1개*. 사용자가 *명시적*으로 새 workspace 생성 가능 ("새 작업공간")
- 대화 끝나도 workspace 유지 (retention 7d, ADR-0001 §4-5)

---

## 6. Backend 구조

### 6-1. 신규 모듈
- `backend/app/features/workspace/router.py` — `/workspaces` CRUD + 파일 API
  - `POST /workspaces` (생성)
  - `GET /workspaces` (내 workspace 목록)
  - `GET /workspaces/{id}/files` (디렉토리 트리)
  - `GET /workspaces/{id}/files/{path}` (파일 내용 — text/binary 분기)
  - `DELETE /workspaces/{id}` (soft delete)
- `backend/app/services/workspace_manager.py` — 디렉토리 mkdir / 파일 watcher / 사용량 집계

### 6-2. claude_runner 변경
- `tempdir` → `workspace.storage_path` (mount 경로 동일 `/workspace`)
- 호출 후 *wipe X* — 영속 유지
- *파일 watcher* — claude 호출 전후 diff → `file_created` / `file_updated` 이벤트

### 6-3. SSE 통합
- 기존 `chat/router.py` 의 SSE generator 에 새 이벤트 타입 11개 추가
- agent.py 가 `AgentEvent` enum 확장
- workspace_manager 가 watcher 결과를 *agent 이벤트 큐* 에 push

---

## 7. Frontend 구조

### 7-1. 신규 features/workspace/

```
frontend/features/workspace/
  WorkspacePanel.tsx              ← 우측 4상태 컨테이너
  WorkspaceProvider.tsx           ← 상태 (mode/panel state) Context
  FileExplorer.tsx                ← 좌측 트리
  FileViewer.tsx                  ← Monaco (텍스트) 또는 image preview
  ToolExecutionLog.tsx            ← agent_start / tool_* 스트림 표시
  StdoutStreamView.tsx            ← stdout/stderr 라이브
  CostMeter.tsx                   ← cost_update 누적 표시
  PreviewSurface.tsx              ← Phase B
  useWorkspaceEvents.ts           ← SSE 이벤트 파서 → state 분배
  useWorkspaceLayout.ts           ← collapsed/small/expanded/fullscreen 상태 머신 + persist
```

### 7-2. 기존 chat 페이지 변경

`frontend/app/(workspace)/chat/page.tsx`:
- `<AppShell><ChatPanel /></AppShell>` → `<WorkspaceProvider><LayoutSwitcher><ChatPanel /><WorkspacePanel /></LayoutSwitcher></WorkspaceProvider>`
- mode = `chat | assisted | builder` 상태 머신
- 자동 slide-out: `useWorkspaceEvents` 가 `file_created` / `tool_start` 감지 → `setMode("assisted")` (단 사용자 명시 collapsed 면 skip)

### 7-3. 상태 transitions

```
chat → assisted    : 자동 (tool 이벤트) 또는 사용자 핸들 드래그
assisted → builder : 사용자 "확장" 버튼 (또는 ⌘⇧B)
builder → assisted : 사용자 "축소" 또는 ESC
assisted → chat    : 사용자 "닫기" (workspace 활동 후 30분 idle 자동 collapse 옵션)
```

각 전환에 0.2~0.3s 트랜지션 + `localStorage[ws.last_mode]` 기억.

---

## 8. 현재 코드와 갭

| 영역 | 현재 | 갭 |
|---|---|---|
| Workspace 디렉토리 | `tempfile.mkdtemp` per call | *영속 디렉토리* + DB 메타 + watcher |
| SSE 이벤트 | 5 type (text/step/sandbox/image/done) | 11 type 확장 |
| Frontend layout | Chat 단일 패널 | 4 상태 split layout |
| File explorer | 없음 | 좌측 트리 + 클릭 보기 |
| Tool execution log | toolSteps 단순 리스트 | 스트리밍 + 단계별 카드 |
| Cost meter | claude_code 응답 텍스트 안 | 우측 패널 상단 누적 표시 |
| Preview | 없음 | iframe / image / markdown 렌더 |
| Workspace persistence | 매 호출 wipe | 7d retention + DB |
| Mode 전환 | 없음 | collapsed/small/expanded/fullscreen 머신 |
| File download | 없음 | per-file 다운로드 endpoint + 토큰 |

---

## 9. Phase 매핑 (CTO P0/P1/P2 → 구체 작업)

### P0 — 1~2주 (최우선)

**P0-A. 백엔드 영속 workspace + 메타**
1. `db/models.py` — `Workspace`, `WorkspaceFile` 추가
2. `services/workspace_manager.py` — create / open / list / file watcher (diff)
3. `features/workspace/router.py` — CRUD + 파일 API
4. `claude_runner.py` — tempdir 분기 제거. `workspace_id` 받으면 영속 path 사용
5. `agent.py` claude_code 분기 — workspace 자동 생성/연결 + file watcher 결과 events 발사
6. **acceptance**: 같은 대화에서 두 번째 호출에서 *이전 파일 보임*

**P0-B. Structured SSE 이벤트**
7. `agent.py` `AgentEvent` enum 11 type 확장
8. `chat/router.py` SSE generator — 새 type 발사
9. `tests/test_sse_events.py` 부활 + 11 type 모두 회귀

**P0-C. Frontend Workspace Panel (4 상태)**
10. `features/workspace/` 신규 컴포넌트 7개
11. `chat/page.tsx` — `<LayoutSwitcher>` 3-mode 머신
12. 자동 slide-out (tool 이벤트 시)
13. File explorer + 텍스트 클릭 보기 (Monaco 또는 simple `<pre>`)
14. Cost meter 우측 상단
15. Tool execution log 스트리밍 표시
16. Generated file 다운로드 버튼

**P0-D. Workspace persistence DB-driven**
17. `conversations.workspace_id` FK 추가
18. 대화 새로 시작 시 workspace 자동 생성 + 기본 연결
19. 사용자가 *수동* 으로 "새 workspace" 또는 "기존 workspace 선택" 가능

### P1 — 3~4주

- **terminal stream** (xterm.js + WebSocket) — sandbox 의 bash session 라이브
- **diff viewer** — file_updated 시 before/after 표시 (Monaco diff editor)
- **preview runtime** — `python -m http.server` 또는 Vite dev server 를 sandbox 안에서 띄우고 우측에 iframe
- **workspace snapshots** — suspend/explicit save → S3 tar.zst (ADR-0001 §7 도입)
- **Builder Mode** — 3-pane fullscreen + Monaco editor

### P2 — 1~2개월

- **collaborative workspace** — 같은 workspace 를 2+명이 함께 (CRDT 또는 OT — Yjs)
- **background agent** — 사용자가 "이거 5분 안에 끝내줘" → 채팅 닫아도 sandbox 계속, 끝나면 알림
- **task queue** — 여러 작업 병렬 실행 + per-task progress bar
- **replay/debugging** — 과거 agent 실행을 *step-by-step 재생* (audit log 기반)

---

## 10. ADR-0003 (Embedded IDE) 와의 관계

ADR-0003 은 *Deferred* 상태였으나 본 ADR 의 **Builder Mode** 가 그 그림과 같음:
- code-server 임베드 — 본 ADR §2 의 Builder Mode 의 *고급 옵션*
- iframe + scoped JWT + WebSocket — Phase P1 의 terminal/preview 와 패턴 동일

→ ADR-0003 을 *Builder Mode 의 향후 진화* 로 *재배치*. 본 ADR P1/P2 가 끝난 후 *고급 사용자 토글*로 code-server 임베드 부활 검토.

---

## 11. Open Questions

1. **workspace 생성 시점** — 대화 시작 자동 vs 첫 tool 실행 시 lazy
2. **사용자 1명 동시 workspace 수** — 제한? 5개? tier 별
3. **file_created 이벤트와 base64 페이로드** — 작은 파일만 (~256KB) inline, 큰 파일은 메타만 + 별도 GET
4. **Monaco editor 도입** — 번들 큼 (~3MB). 옵션 lazy load
5. **builder mode 의 키보드 단축키** — VS Code 식 vs 우리 식
6. **모바일 layout** — chat 만 + workspace 는 *별도 페이지 toggle* (split 불가능)

각각 ADR-0007~0012 후보.

---

## 12. 1페이지 요약

```
┌──────────────────────────────────────────────────────────────────┐
│  제품 재정의                                                       │
│   ❌ 챗봇 + 코드 도구                                             │
│   ✅ Progressive AI Workspace                                    │
│                                                                  │
│  3 모드 레이아웃                                                   │
│   Chat → (tool 실행) → Assisted → (확장) → Builder                │
│                                                                  │
│  4 패널 상태                                                       │
│   collapsed / small (360px) / expanded (50%) / fullscreen        │
│                                                                  │
│  Persistent workspace                                            │
│   data/workspaces/{workspace_id}/files/                          │
│   DB: workspaces / workspace_files                               │
│   conversations.workspace_id FK                                  │
│                                                                  │
│  Structured SSE (11 type)                                        │
│   agent_start / tool_start / tool_delta / tool_end               │
│   file_created / file_updated / stdout / stderr                  │
│   cost_update / agent_done / error                               │
│                                                                  │
│  ADR-0003 (IDE) 재배치                                            │
│   → Builder Mode 의 *고급 옵션* (P1+ 이후)                        │
└──────────────────────────────────────────────────────────────────┘
```

---

## 13. 변경 이력

- **2026-05-22** — 사용자 직접 시연(`hello.py` 요청) 결과 *frontend 격차* 노출 → 제품 재정의 결정. ADR-0003 deferred → 본 ADR 의 Builder Mode 로 흡수. P0 구현 1~2주 착수.
- **2026-05-22 (Turn 1)** — Backend P0-A + P0-B 완료. 11 파일 신규/변경. DB schema 적용(workspaces/workspace_files/conversations.workspace_id). workspace_manager 7 단위 테스트 통과. e2e smoke 통과: workspace 생성 → claude_code 호출 → 영속 디스크에 hello.py 38 bytes sha256 동기화 + file_events 발사. 백엔드 :8001 docker mode 정상.
- **2026-05-22 (Turn 2)** — Frontend P0-C 완료. `features/workspace/` 6 신규 파일 (WorkspaceProvider + reducer + dispatchSseEvent / WorkspacePanel 4상태 / FileExplorer / FileViewer / ToolExecutionLog / CostMeter). chat/page.tsx 통합 — Provider wrap + SSE 새 이벤트 dispatch + 자동 slide-out + RESET_TASK on user msg. tsc 통과 + prod 빌드 60s + `/chat` 200.
- **2026-05-22 (Turn 2 확장)** — (1) Claude Code 권한 정책 정리: `--permission-mode acceptEdits` + `Write` 도구 추가 → /workspace 안 파일 생성 정상 작동. (2) `--append-system-prompt` 로 *작업 후 Read 검증 → 증거와 함께 보고* 지침 주입 + 라이브러리 사전 설치 명시. (3) Sandbox runtime metrics — `services/runtime_metrics.py` Protocol + DockerStatsCollector(K8s 자리 marked) + claude_runner background poller (1초 간격) + `RuntimeMetricsEvent` (16-type union) + SSE 매핑 + Frontend `RuntimeMetricsBar` (compact 헤더용 / 4-게이지 expanded). DOCKER/K8S/NOOP 백엔드 라벨 자동 표시. e2e: 7 sample CPU 88%→2.85% Mem 115MB→125MB 정상 수집. K8s 전환 시 frontend/agent 변경 0 — `source` 만 자동 라벨링.
- **2026-05-22 (Stream 화)** — `claude_runner.py` 의 `run_claude_code_stream()` AsyncIterator 신규 (290 LOC) — `proc.stdout` 줄 단위 readline + metrics queue + stderr task 병행 + 각 NDJSON line 도착 즉시 yield. 기존 `run_claude_code` 는 *stream consume → dict* wrapper (backward compat). `agent.py` run_agent 안에서 *claude_code 분기 특수 처리* — `_exec_tool` blocking path 우회하고 stream 의 각 event 를 AgentEvent 로 *즉시* yield. e2e: 15 events 가 0.3~2초 간격으로 진짜 시간차 도착 (이전 batch 12초 일제 도착 → 좀비 의심). FailureBanner + Workspace dock + stream 3 박자로 "생각 중" 멈춤 인상 해소.
