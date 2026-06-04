# ADR-0002 · User-Scoped AI Execution Platform

| | |
|---|---|
| **Status** | Accepted (2026-05-22) |
| **Deciders** | CTO / Platform Eng |
| **Layer** | Product/Platform vision (ADR-0001 의 *상위* 결정) |
| **Related** | [ADR-0001 · Sandbox Runtime & Lifecycle](0001-sandbox-runtime.md) — 본 ADR 의 구현 사양 |
| **Refines** | `docs/sandbox.md` 의 단일 host docker 가정 |

---

## 0. 한 줄 결론

> 우리는 **코딩 모델**을 만들지 않는다.
> 우리는 **유저별로 완전히 격리된 AI Execution Platform** 을 만든다.
>
> Claude Code 는 *coding intelligence engine* 으로 통합 소비한다.
> 우리가 제공하는 가치는 *sandbox · lifecycle · isolation · audit · policy · quota · orchestration* 이다.

---

## 1. Context — 왜 이 방향인가

### 1-1. 시장에서 우리의 자리

| 영역 | 누가 잘 만드는가 | 우리 전략 |
|---|---|---|
| Coding intelligence (LLM, IDE 패턴) | Anthropic / OpenAI / 오픈소스 | **소비** (Claude Code · LiteLLM) |
| Multi-tenant secure execution | 시장 빈 자리 (Replit/StackBlitz/Daytona 일부) | **자체 구축** ← *여기가 우리 사업* |
| Compliance / audit / billing | 엔터프라이즈가 요구하는데 SaaS 가 약함 | **자체 구축** |

코딩 모델 자체는 *경쟁 비교*가 어려운 영역(천억 단위 학습 비용·데이터). 그러나 *조직 안에 안전하게
들여와 유저 단위로 격리·감사·정책 적용하는 layer* 는 *지금까지 아무도 깨끗하게 만들지 않은* 영역.

### 1-2. 우리의 차별점

```
Claude Code  =  Coding Intelligence (소비)
Our Platform =  User-Scoped Secure Execution Infrastructure (자산)
```

* AI 가 코드를 *잘 만드는 것*이 목적이 아니다.
* AI 가 *조직 내부에서 유저 단위로 안전·격리·감사·비용 통제된 방식으로* 실행되는 것이 목적이다.

---

## 2. 핵심 철학 — "User 가 보안 경계"

모든 실행은 다음 4단 격리 안에서만 발생한다:

```
/sandbox
  /tenant/{tenantId}
    /user/{userId}
      /workspace/{workspaceId}
        /session/{sessionId}
          /task/{taskId}
```

각 task 는 *독립*:

* filesystem
* process namespace
* network policy
* secret scope
* quota
* audit trail
* workspace lifecycle

**한 유저의 코드 · 파일 · artifact · 로그 · secret · process 는 다른 유저에게 절대 노출 / 공유되지 않는다.**

> Shared coding agent 는 운영하지 않는다.

---

## 3. 책임 분리 매트릭스 — Claude Code vs Our Platform

| 책임 | Claude Code | Our Platform |
|---|---|---|
| 코드 생성 / 수정 / refactor / debug | ✅ | ❌ |
| diff / patch 생성 | ✅ | ❌ |
| bash reasoning / 파일 편집 | ✅ | ❌ |
| **신뢰 경계** | ❌ (신뢰하지 않음) | ✅ (sandbox isolation) |
| sandbox allocation | ❌ | ✅ |
| runtime isolation (Kata/Firecracker) | ❌ | ✅ |
| workspace lifecycle (create/idle/suspend) | ❌ | ✅ |
| auth · authz · token scoping | ❌ | ✅ |
| snapshot · restore | ❌ | ✅ |
| audit log (immutable) | ❌ | ✅ |
| retention · billing · quota | ❌ | ✅ |
| network policy · egress allowlist | ❌ | ✅ |
| secret scope · masking · revoke | ❌ | ✅ |
| observability (OTel · Prometheus · SIEM) | ❌ | ✅ |
| orchestration (node pool · scheduler) | ❌ | ✅ |

> **신뢰 경계는 Claude Code 가 아니다. Sandbox isolation 이다.**

---

## 4. 격리 단위별 책임

| 단위 | 분리되는 리소스 | 만료 시 처리 |
|---|---|---|
| **Tenant** | 노드 풀 / 네트워크 zone / DB schema / 키 관리 | 계약 종료 시 데이터 wipe |
| **User** | 개인 workspace · personal secret · quota 한계 · audit ID | 계정 비활성 시 retention 정책 적용 |
| **Workspace** | filesystem (snapshot 단위) · dependency cache · build state | 7d retention |
| **Session** | 연결 lease · websocket · preview URL · temp env | idle 15m / suspend 30m |
| **Task** | process namespace · runtime · per-task quota · per-task audit | 완료 즉시 종료 |

---

## 5. 실행 흐름 (End-to-End)

```
┌────────────────────────────────────────────────────────────────────────┐
│ 사용자 요청 (코드 생성/build/test/refactor/패키지 설치/...)           │
└──────────────────────────────┬─────────────────────────────────────────┘
                               │
                               ▼
        ┌──────────────────────────────────────────────────┐
        │ 1) Auth & Authz                                   │
        │  - user JWT 검증 + tenant 결정                     │
        │  - scoped token 발급 (workspace:rw / terminal:exec │
        │    / preview:open / artifact:download)            │
        └──────────────────────────────┬───────────────────┘
                                       ▼
        ┌──────────────────────────────────────────────────┐
        │ 2) Sandbox Allocation                            │
        │  - 위험도 평가 → 노드 풀 선택                     │
        │    - arbitrary code   → secure pool (Kata/Firec) │
        │    - internal low-risk → shared pool (hard runc) │
        │  - 새 MicroVM lazy create                        │
        │  - workspace snapshot S3 → restore               │
        │  - dependency cache mount (선택)                 │
        │  - secret 주입 (user/task-scoped, short-lived)   │
        └──────────────────────────────┬───────────────────┘
                                       ▼
        ┌──────────────────────────────────────────────────┐
        │ 3) Claude Code 실행 (sandbox 내 단일 process)      │
        │  claude -p "<intent>" \                          │
        │    --allowedTools "Read,Edit,Bash"               │
        │    --output-format stream-json                   │
        │  - SSE 로 우리 chat router 에 forward             │
        │  - 권한은 sandbox 가 ultimate boundary            │
        └──────────────────────────────┬───────────────────┘
                                       ▼
        ┌──────────────────────────────────────────────────┐
        │ 4) 결과 수집                                       │
        │  - diff / artifact / stdout-stderr / generated   │
        │  - audit event (action · user · resource · ts)   │
        │  - resource usage (CPU sec · mem peak · egress)  │
        │  - workspace 변경 → snapshot trigger             │
        └──────────────────────────────┬───────────────────┘
                                       ▼
        ┌──────────────────────────────────────────────────┐
        │ 5) Lifecycle 평가                                 │
        │  - heartbeat 갱신 → active                       │
        │  - 15m idle → suspend 후보                       │
        │  - 30m idle → snapshot · compute 회수            │
        └──────────────────────────────────────────────────┘
```

---

## 6. Claude Code 통합 사양

### 6-1. 호출 인터페이스

Claude Code 는 **sandbox 안에서 *하나의 프로세스***로만 실행한다. 호스트에서 직접 실행 금지.

기본 invoke:

```bash
claude -p "<사용자 요청 또는 우리가 빌드한 prompt>" \
  --allowedTools "Read,Edit,Bash" \
  --output-format stream-json \
  --permission-mode acceptEdits
```

* `--allowedTools` — *task 단위* 로 결정. 빌드 task 면 `Bash` 만, refactor task 면 `Read,Edit`.
* `--output-format stream-json` — 우리 SSE pipeline (`backend/app/features/chat/router.py` `chat_stream`) 으로 forward 가능한 NDJSON.
* `--permission-mode` — `acceptEdits` (자동) vs `default` (확인) — task type 으로 결정.
* `--cwd /workspace` — sandbox 내부의 workspace mount 지점만.

### 6-2. 신뢰 모델

> **Claude Code 는 신뢰 boundary 가 아니다.** Claude Code 가 무엇을 출력해도 sandbox 가 막는다.

* Claude Code 가 `Bash` 도구로 `rm -rf /` 시도 → sandbox `--read-only` rootfs + nobody user + cap-drop 으로 차단
* Claude Code 가 `curl http://169.254.169.254/...` 시도 → sandbox `--network none` 또는 egress allow-list 로 차단
* Claude Code 가 무한 루프 → sandbox `--memory` `--cpus` `--pids-limit` 으로 제한

즉 *Claude Code 의 권한 정책은 UX 가이드*고, *sandbox 정책이 최종 보안*이다.

### 6-3. 출력 처리

`stream-json` 출력을 NDJSON 으로 받아 우리 schema 로 매핑:

| Claude Code 이벤트 | 우리 schema |
|---|---|
| `tool_use` (`Read`/`Edit`/`Bash`) | `StepStartEvent` (`agent.py` 의 `_exec_tool` 가 발사하던 것과 동형) |
| `tool_result` | `StepEndEvent` |
| `text_delta` | `TextDeltaEvent` (chat SSE 그대로) |
| `final` | `DoneEvent` |
| `error` | `ErrorEvent` |

### 6-4. 권한 / 토큰

* Claude Code 가 LLM API 호출에 쓰는 *모델 키* 는 sandbox 안에서만 유효한 *short-lived proxy 토큰* 으로 주입 (직접 API key 노출 금지)
* `~/.claude/` 같은 사용자 설정은 *user-scoped persistent volume* (workspace 와 별개의 user-level snapshot)

---

## 7. State 외재화 — ADR-0001 참조 + user-scope 추가

[ADR-0001 §6](0001-sandbox-runtime.md#6-상태관리-redis-db-object-storage-3-tier) 의 3-tier 를 그대로 따른다. 본 ADR 은 *user-scope* 추가만:

* 모든 Redis 키에 `{userId}` 또는 `{tenantId}:{userId}` 접두 — cross-user 읽기 방지 (Redis ACL 또는 Lua script 검증)
* DB 의 모든 테이블에 `user_id` / `tenant_id` 컬럼 + RLS (Row Level Security) 또는 application-level filter (mandatory in repo layer)
* Object Storage 경로 `s3://workspace/{tenant}/{user}/{workspace}/...` — IAM policy 로 다른 user prefix 차단

---

## 8. 격리 모델 매트릭스 — 어디가 무엇을 분리하는가

| 자원 | 격리 메커니즘 |
|---|---|
| CPU / Memory | cgroups (runc) → MicroVM kernel (Kata/Firecracker) |
| Filesystem | per-workspace volume + `--read-only` rootfs + tmpfs `/tmp` |
| Process / PID | `--pid` namespace (runc) + 별도 VM kernel (Kata) |
| Network | `--network none` 기본, Phase 2 에서 egress allow-list |
| User credential | secret broker (Vault/SSM) → user-scoped short-lived token only |
| Build cache / dep cache | per-user cache + Object Storage layer (cross-user 차단) |
| Audit | per-user audit stream (immutable append-only) |
| Quota | per-user counter (Redis + DB billing) |

---

## 9. ADR-0001 과의 관계

| ADR | Layer | 범위 |
|---|---|---|
| **ADR-0002 (본 문서)** | Vision / Product | *왜* User-scoped 인가, *Claude Code 와의 책임 분리*, *4단 격리 트리* |
| **ADR-0001** | Implementation | *어떻게* sandbox 를 만드는가 — Kata/Firecracker · Redis/DB/S3 · lifecycle · snapshot · network · audit · quota 의 *기술 사양* |

ADR-0001 의 모든 결정은 본 ADR 의 비전 하위에서 유효하다. 두 문서 간 충돌이 생기면 **본 ADR 우선**.

---

## 10. 현재 구현과의 갭 분석 (ADR-0001 §13 + Claude Code 통합 항목)

ADR-0001 §13 의 14개 갭을 그대로 상속 + 본 ADR 의 신규 갭:

| 결정사항 | 현재 코드 위치 | 갭 | 우선순위 |
|---|---|---|---|
| **User-scope 4단 격리 트리** | 부재 | `tenant_id` 컬럼 존재(`db/models.py` Team), `user_id` ✅. 그러나 `workspace_id` / `session_id` / `task_id` 분리 *없음* | P1 |
| **scoped token** | `core/security.py` 단일 JWT | scope 분리 없음 — `workspace:read` `terminal:exec` 같은 권한 분해 부재 | P2 |
| **Claude Code 통합** | 현재 agent.py 가 OpenAI/Claude **API 직접 호출** (코드 생성·tool 디스패치). Claude Code CLI 호출 *없음* | 신규 통합 필요 — sandbox image 에 `claude` CLI + stream-json 파서 + SSE 매핑 | P2 |
| **per-task quota** | `services/quota.py` 팀 storage 만 | task-level CPU/memory/egress 카운터 부재 | P2 |
| **Per-user audit stream** | `usage_events` 일부 | immutable 보장·sandbox-specific audit 분리 없음 | P2 |
| **secret broker** | `tool_credentials` DB column (`features/tools/router.py`) | broker 인터페이스 없음 — DB 에서 직접 읽음. revoke / masking 분리 없음 | P2 |

> ADR-0001 §13 의 다른 항목 (MicroVM 기본 / native fallback 금지 / disposable compute / Redis-DB-S3 외재화 / lifecycle / snapshot / node pool 분리 / network deny / OTel / mTLS) 는 본 ADR 에서도 *동일하게* 유효.

---

## 11. Phase 0 즉시 결정 — Docker 미가용 환경 처리

### 11-1. 추천 결정: **A + B 병행**

**A. 지금 당장**: `.env` 에 `SANDBOX_REQUIRE_DOCKER=false` 추가 + 백엔드 재기동.
- 단, **DEV 전용 토글** 임을 코드 경고 로그가 명시
- 사용자에게 보이는 메시지는 *기존 동작 그대로* — 일선 사용자 영향 0
- *명시적 의사결정 기록* — `.env` 가 git 추적 안 되므로 별도 운영 노트 (`docs/operations/runbook.md` 또는 ADR 변경 이력) 에 기록

**B. 1~2주 내**: Docker 설치 (또는 인프라 정비) 후 `SANDBOX_REQUIRE_DOCKER=true` 로 복원.
- 사내 호스트에 docker 설치 (예상 1시간)
- `chatbot-sandbox` 이미지 빌드 (가벼운 Python + 필수 라이브러리)
- 빌드된 이미지로 `tests/test_sandbox_isolation.py` 의 통합 3개 (`test_docker_integration_*`) 통과 확인
- 그 후 require=true 로 복귀

### 11-2. 이렇게 가는 이유

- 본 ADR 의 *User 가 보안 경계* 원칙은 **단일 사용자 환경** 에서도 *처음부터* 적용되어야 함
- 그러나 *현재 운영 영향 0 + 향후 진짜 보안 적용* 두 목표를 동시에 달성하려면 *명시 토글*이 합리적
- `.env` 의 `SANDBOX_REQUIRE_DOCKER=false` 는 *DEV 토글* 이라는 *코드 경고 로그*가 매번 출력되어 *놓칠 수 없음*
- 토글이 *어딘가에 명시*되어야 PR 리뷰·운영 핸드오프에서 *위험을 인지*함 — 본 ADR §변경 이력에 기록

### 11-3. 안 가는 길

- **옵션 C (코드 되돌림)** 은 *결정 자체를 미루는 것*. ADR 정신과 어긋남. *Phase 0 차단은 코드에 들어가 있어야* 다음 단계가 의미를 가짐.
- **Docker 즉시 설치** 는 *옳지만 부담 시점이 큼* — 이번 sprint 안에서 가능하면 B 단독, 아니면 A 먼저.

---

## 12. 마이그레이션 로드맵 (ADR-0001 §14 위에 Claude Code 통합 + user-scope 추가)

### Phase 0 — *이미 코드 들어감* (오늘)
- ✅ `Settings.sandbox_require_docker / runtime_class / max_timeout / memory / cpus / pids_limit`
- ✅ `code_sandbox.run_code` native fallback 차단 + Docker hardening flags
- ✅ `agent._exec_tool` code_interpreter disabled 응답 매핑
- ✅ `tests/test_sandbox_isolation.py` 4 passed / 3 skipped
- 🟡 **본 ADR 의 §11 결정** — `.env` 토글 + 백엔드 재기동 *대기 중*

### Phase 1 — State 외재화 + user-scope 진입 (다음 2주)
* ADR-0001 §14 Phase 1 + ↓ 본 ADR 추가:
* `db/models/sandbox_session.py` `sandbox_workspace.py` `sandbox_task.py` 신규 — 모든 테이블에 `(tenant_id, user_id)` 컬럼 + index
* `services/redis_client.py` 키 네임스페이스에 `{tenantId}:{userId}` 접두 강제 (helper 제공)
* `core/scoped_token.py` 신규 — `workspace:read/write/terminal:exec/preview:open/artifact:download` scope enum + JWT 안에 `scopes: list[str]` 클레임
* SSE/WebSocket 핸드셰이크 시 *scope 검증* — 잘못된 scope 면 403

### Phase 2 — Kata + Claude Code 통합 + snapshot (1~2개월)
* ADR-0001 §14 Phase 2 + ↓ 본 ADR 추가:
* `chatbot-sandbox` 이미지에 Claude Code CLI 포함 (`npm i -g @anthropic-ai/claude-code` 또는 공식 설치 방법)
* `services/claude_runner.py` 신규 — `claude -p ... --output-format stream-json` invoke + NDJSON parser + 우리 `AgentEvent` 로 매핑
* `agent.py` 의 `_exec_tool` 에 `claude_code` 도구 추가 — `agent.py` 의 거대 if-elif 를 dispatch dict + tool_handlers/ 폴더로 분리 (full-audit P2)
* WebSocket terminal 엔드포인트에 `scoped_token` 검증 + sandbox lease 갱신
* Workspace snapshot 메커니즘 → S3 (또는 minio) upload + restore on reconnect

### Phase 3 — Firecracker + 멀티테넌시 본격 + 관측성 (3~6개월)
* ADR-0001 §14 Phase 3 + ↓ 본 ADR 추가:
* `RuntimeClass: firecracker` (k8s) 또는 nomad + firecracker 검토
* per-tenant network zone (calico / cilium NetworkPolicy 또는 firewall rules)
* SIEM 연동 — sandbox audit 을 별도 SIEM bucket 에 forward
* per-user / tier-based quota 적용 + billing 집계 + 사용자 대시보드
* Acceptance: 동시 100 사용자 + 각 사용자 평균 3 동시 sandbox + cross-user leak 0

---

## 13. 운영 체크리스트 (Production 진입 전 필수)

본 ADR 의 비전이 *실제 운영*에서 무너지지 않으려면 다음이 *동시에* 만족:

- [ ] **User 4단 격리** — DB 모든 쿼리에 `user_id` 또는 `tenant_id` 필터 (어플리케이션 레이어 또는 RLS)
- [ ] **Sandbox runtime** — `SANDBOX_REQUIRE_DOCKER=true` (또는 Kata) — *DEV 토글 명시 종료*
- [ ] **Scoped token** — Bearer JWT 가 *최소 권한* 만. ⌘K admin endpoint 호출엔 admin scope 명시
- [ ] **Network default deny** — egress allow-list 도입 (SSRF 가드와 통합 — full-audit TOP 1)
- [ ] **Audit immutable** — append-only bucket + retention policy
- [ ] **Per-user quota** — Redis counter + DB billing — 초과 시 graceful 거부
- [ ] **Claude Code 통합** — sandbox 내부 process only · 호스트 invoke 금지
- [ ] **Snapshot restore E2E** — suspend → reconnect → workspace 복구 통과
- [ ] **Cross-user leak test** — 사용자 A 의 workspace/secret/audit 이 사용자 B 에게 *모든* 채널(API/SSE/WebSocket/log)로 노출되지 않음을 e2e 검증

---

## 14. Open Questions (후속 ADR 필요)

1. **ADR-0003 · Claude Code Integration spec** — invoke 옵션의 *task-type → allowedTools* 매핑 룰, stream-json schema 매핑, 권한 모드 결정
2. **ADR-0004 · Tenant Isolation depth** — 노드 풀까지 분리(strict) vs cgroup 분리(soft). 비용/보안 trade-off
3. **ADR-0005 · Secret Broker** — Vault vs AWS SSM vs 자체 — short-lived token 발급 / revoke / audit 인터페이스
4. **ADR-0006 · Billing & Quota** — 측정 단위(CPU sec / memory peak / egress GB / build min) + tier 정의 + 한도 초과 시 graceful 거부 UX
5. **ADR-0007 · Multi-Region / Disaster Recovery** — workspace snapshot region 분리 / RPO·RTO 목표

---

## 15. 결정 요약 (한 페이지)

```
┌──────────────────────────────────────────────────────────────────┐
│  비전                                                              │
│   "User-Scoped AI Execution Platform"                            │
│                                                                  │
│  책임 분리                                                         │
│   Claude Code   = Coding Intelligence (소비)                     │
│   Our Platform  = Secure Execution Infrastructure (자산)         │
│                                                                  │
│  격리 단위                                                         │
│   tenant > user > workspace > session > task                     │
│                                                                  │
│  Runtime                                                          │
│   Production user code  →  Kata + Firecracker (MicroVM)          │
│   Internal low-risk     →  hardened runc (현재)                   │
│   User arbitrary code   →  runc *금지* (Phase 0 차단 완료)         │
│                                                                  │
│  State                                                             │
│   Redis  (runtime / TTL / queue / lease)                         │
│   PostgreSQL (metadata / billing / audit)                        │
│   Object Storage (workspace / artifact / log)                    │
│                                                                  │
│  Lifecycle                                                         │
│   lazy → active → idle(15m) → suspend(30m) → archive → delete    │
│                                                                  │
│  Recovery                                                          │
│   파일 복구 ✅  · 프로세스 복구 ❌  · 새 sandbox 생성              │
│                                                                  │
│  Network · Secret · Audit · Quota                                │
│   default deny  ·  user-scoped short-lived  ·  immutable         │
│   ·  per-user counter                                            │
└──────────────────────────────────────────────────────────────────┘
```

---

## 16. 변경 이력

- **2026-05-22** — 초안 Accepted. ADR-0001 의 비전 상위 결정으로 분리. Phase 0 운영 결정(§11)은 사용자 confirm 대기.
