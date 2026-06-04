# ADR-0001 · Sandbox Runtime & Lifecycle Architecture

| | |
|---|---|
| **Status** | Accepted (2026-05-22) |
| **Deciders** | CTO / Platform Eng |
| **Supersedes** | `backend/app/services/code_sandbox.py` 의 *native subprocess fallback* 정책 |
| **Related** | [docs/full-audit.md](../full-audit.md) §2 TOP 2 (P0 — sandbox RCE 경로), [docs/sandbox.md](../sandbox.md), **[ADR-0002 · User-Scoped AI Execution Platform](0002-user-scoped-ai-execution-platform.md)** (본 ADR 의 *상위* 비전 결정) |
| **Implementation tracking** | Phase 0~3 (본 문서 §17) |

> **결론**: AI 코딩 에이전트 샌드박스는 일반 runc 컨테이너 기준으로 운영하지 않는다.
> 사용자 임의 코드 실행 환경에서는 **runc 를 최종 보안 경계로 신뢰하지 않는다**.
> Kata Containers / Firecracker 기반 **MicroVM** 으로 전환한다.

---

## 1. Context — 왜 이 결정이 필요한가

[`docs/full-audit.md`](../full-audit.md) §2 TOP 2 에서 확인된 **P0**: 현재
`backend/app/services/code_sandbox.py:117-122` 의 *native subprocess fallback* 분기는
Docker 가 없을 때 **백엔드 venv 의 Python 으로 사용자 코드를 그대로 실행**한다.
`agent._exec_tool` (`backend/app/services/agent.py:128-135`) 의 `code_interpreter`
도구가 LLM 응답 코드를 그대로 `run_code(code)` 로 전달하므로:

```
사용자 프롬프트 → LLM 이 "import os; os.system(...)" 코드 생성
 → agent.code_interpreter → run_code → host venv subprocess
```

*프롬프트 1줄로 호스트 RCE* 가 성립한다. Docker 격리 모드(`--network none --memory
512m --cpus 1 --pids-limit 64`)는 일정 수준 안전하나 **runc 의 host kernel 공유 면**이
여전히 공격 표면이다. 따라서 본 ADR 은 다음을 동시에 해결한다:

1. native fallback 경로 차단 (즉시 — Phase 0)
2. runc → MicroVM 전환 (중기 — Phase 2~3)
3. *오래 살아있는 컨테이너* → *disposable compute + externalized state* 운영 모델로 전환 (전 phase 연속)

---

## 2. 운영 철학 — 한 문장

> 우리는 "오래 살아있는 Docker 컨테이너"를 운영하는 것이 아니다.
> 우리는 **ephemeral compute · externalized state · MicroVM isolation ·
> policy-driven lifecycle · auditability · cost-controlled sandboxing** 을 기반으로 하는
> "멀티테넌트 AI 코드 실행 플랫폼"을 운영한다.

---

## 3. Runtime 결정

### 3-1. 런타임 클래스

| Workload | 런타임 | 비고 |
|---|---|---|
| **Production AI code execution** (사용자 임의 코드) | **Kata Containers + Firecracker** MicroVM | 기본값. runc 금지. |
| **Low-risk / internal workload** (시드 데이터 적재·내부 batch) | hardened runc 허용 | seccomp/apparmor/no-new-privileges/read-only rootfs 필수 |
| **High-risk / user arbitrary code** | runc **금지**. Kata/Firecracker/MicroVM **필수** | LLM 이 생성한 모든 코드 포함 |

### 3-2. 샌드박스는 disposable compute

- 세션별 disposable compute 로 본다.
- *영속 상태*는 컨테이너 내부가 아니라 **Redis + DB + Object Storage** 에서 관리.
- idle → CPU/Memory 회수, workspace/artifact 만 정책적으로 보존.
- 재접속 시 *기존 컨테이너를 살리지 않음*. 새 sandbox 를 만들고 상태를 restore.

---

## 4. 세션 생명주기

### 4-1. 생성 (Lazy)

사용자가 화면에 진입했다고 *바로* 만들지 않는다. 생성 시점:

- 코드 실행 / 터미널 실행 / 패키지 설치 / 파일 빌드 / preview 서버 실행

### 4-2. Active

최근 **15분 이내** 코드 실행 / 터미널 입력 / 파일 변경이 있는 세션.
- sandbox running
- CPU / Memory 점유
- 터미널 / preview server 유지

### 4-3. Idle

**15분간 activity 없음** → idle 전환.
- 새 작업은 받음
- 일정 시간 후 suspend 후보

### 4-4. Suspend

**30분 이상 idle** → sandbox compute 종료.
- CPU / Memory 회수, 프로세스 종료
- workspace snapshot 저장
- artifact 저장
- metadata 저장
- *컨테이너는 죽이고 스토리지만 남긴다*

### 4-5. Retention 정책

| 항목 | 기본값 |
|---|---|
| Running session | 최대 **6 시간** |
| Idle timeout | **15 분** |
| Suspend after idle | **30 분** |
| Workspace retention | **7 일** |
| Artifact retention | **30 일** |
| Log retention | **30 일** |
| Failed task temp files | **24 시간** |
| Anonymous / free user workspace | **24 시간 ~ 3 일** |
| Enterprise user workspace | 계약 정책 기반 |

---

## 5. 재접속 / 재부팅 정책

기존 sandbox 를 *살리려 하지 않는다*.

- 새 sandbox 생성
- Redis / DB 에서 session 상태 확인
- Object Storage 또는 Volume Snapshot 에서 workspace 복구
- 필요한 dependency cache 연결
- preview server 는 **재시작**
- terminal process 는 **복구하지 않음**

> **파일은 복구한다. 프로세스는 복구하지 않는다.**
> 프로세스 복구는 운영 복잡도가 너무 크다. 초기 버전에서는 하지 않는다.

---

## 6. 상태관리 (Redis · DB · Object Storage 3-tier)

Redis 하나만으로는 부족하다. 역할을 분리한다.

### 6-1. Redis — 빠르게 변하는 runtime state

저장 대상:
- session heartbeat
- sandbox state (`running | idle | suspending | suspended | expired`)
- active / idle / suspended 플래그
- task queue
- worker lock / distributed lock
- rate limit
- temporary execution status
- websocket 연결 상태
- sandbox lease

키 스킴:
```
session:{sessionId}:state           = running | idle | suspending | suspended | expired
session:{sessionId}:last_activity   = <unix ts>
sandbox:{sandboxId}:node            = node-3
task:{taskId}:status                = queued | running | success | failed | timeout
lock:workspace:{workspaceId}        = <holder>
```

**TTL 표준**:

| 키 | TTL |
|---|---|
| `session:{id}:heartbeat` | **60 s** |
| `sandbox:{id}:lease` | **120 s** |
| `session:{id}:idle_marker` | **15 m** |
| `lock:cleanup:*` | **10 m** |

> Redis 는 *source of truth 가 아니다*. 캐시일 뿐.

### 6-2. PostgreSQL — Persistent Metadata Layer

영속 상태는 반드시 DB. 권장:
- PostgreSQL 우선 (metadata consistency · auditability · transactional update)

#### 저장 대상

**Identity / Ownership**
- `user_id` `org_id` `tenant_id` `workspace_id` `session_id` `task_id` `sandbox_id`

**Session Metadata**
- `runtime_type` `sandbox_version` `node_id` `image_digest`
- `created_at` `last_activity_at` `suspended_at` `expired_at`
- `restored_from_snapshot_id`

**Workspace Metadata**
- `workspace_path` `storage_bucket` `snapshot_version`
- `encryption_key_id` `retention_policy_id` `storage_usage_bytes`

**Task Metadata**
- `task_type` `queued_at` `started_at` `finished_at`
- `exit_code` `timeout_flag` `retry_count` `failure_reason`

**Security / Audit**
- `auth_context` `policy_version` `secret_scope`
- `network_policy_id` `security_event_reference` `sandbox_runtime_class`

**Billing / Quota**
- `cpu_seconds` `memory_peak_mb` `disk_usage_bytes`
- `egress_bytes` `storage_bytes` `billable_minutes`

#### DB 의 역할

- session recovery source
- billing source
- audit source
- retention scheduler source
- policy evaluation source

> **Redis 데이터 유실 시에도 시스템은 DB 기준으로 복구 가능해야 한다.**

### 6-3. Object Storage — 영속 파일 저장소

Sandbox 내부 filesystem 은 신뢰하지 않는다. 권장:
- S3 compatible
- versioning enabled
- lifecycle policy enabled
- SSE encryption enabled

#### Workspace Snapshot
```
s3://workspace/{tenant}/{workspace}/{snapshotId}.tar.zst.enc
```
포함: source code · uploaded files · build outputs · dependency manifest · editor metadata

#### Artifact
```
s3://artifact/{tenant}/{taskId}/
```
포함: generated binaries · preview assets · export zip · generated docs · test report

#### Log Archive
```
s3://logs/{date}/{sandboxId}.zst
```
포함: stdout · stderr · security events · audit events

> Sandbox 내부 fs 는 **disposable**. Object Storage 만 영속.

---

## 7. Workspace Snapshot 메커니즘

**컨테이너 전체 dump 가 아니다.** Process restore 보다 **Workspace restore** 우선.

### 7-1. Snapshot 트리거

- suspend 직전
- explicit save
- session 종료 직전
- 장시간 idle 진입 시
- graceful shutdown 시
- node drain 시
- (옵션) periodic autosave 5~15 분

### 7-2. Snapshot 범위

**포함**
- workspace files
- uploaded assets
- editor state
- dependency lockfile
- build metadata

**제외**
- running process / PID state / socket state
- terminal process
- kernel state
- `/tmp` cache
- package cache
- `node_modules` cache (선택적 — 옵션 토글)

### 7-3. 포맷

- `tar + zstd`
- (옵션) dedupe / incremental snapshot

Enterprise:
- AES-GCM encryption
- per-tenant key management (KMS)

### 7-4. 보관 전략

- latest full snapshot
- 최근 N개 incremental snapshot (기본 5)

---

## 8. Node Scheduling · Multi-Tenancy · Isolation

샌드박스는 단순 container scheduling 문제가 아니다. **멀티테넌트 보안 시스템**이다.

### 8-1. Scheduling 정책

필수 고려: tenant isolation · runtime class · GPU affinity · memory pressure ·
noisy neighbor 방지 · workload risk level.

| 위험도 | 노드 풀 |
|---|---|
| Low-risk | shared hardened node pool (runc + hardening) |
| High-risk | kata / firecracker isolated node pool |
| Enterprise | tenant dedicated node pool |

### 8-2. Node Pool

#### Shared Pool
- low risk
- runc hardened
- low cost

#### Secure Pool
- kata / firecracker
- stronger isolation
- restricted networking

#### Dedicated Pool
- enterprise / compliance / regulated workloads

### 8-3. Network Isolation

**기본: sandbox 간 east-west traffic 금지.**

필수:
- per-sandbox network namespace
- egress policy (default deny + allow-list)
- metadata IP block (`169.254.169.254`)
- RFC1918 block (`10/8 · 172.16/12 · 192.168/16`)
- DNS control (resolver 강제)
- internal service isolation (mTLS 강제)

> 본 ADR 의 **default-deny egress** 는 `docs/full-audit.md` §2 TOP 1 의 **SSRF 가드 부재** 와 직결된다. 두 작업을 같은 PR 에서 정리한다.

---

## 9. 관측성 (Observability)

권장 스택: OpenTelemetry · Prometheus · Loki · Tempo/Jaeger · SIEM integration.

### 9-1. Metrics

**Runtime**
- sandbox boot latency
- snapshot restore latency
- task runtime
- cold start vs warm start

**Resource**
- CPU · memory · disk · network egress · inode usage

**Stability**
- crash rate · timeout rate · OOM rate
- restore failure · cleanup failure

**Security**
- denied syscall count
- blocked egress count
- policy violation count
- suspicious process execution

### 9-2. Audit Log

**Immutable** audit trail 유지:
- sandbox created / runtime selected
- secret injected
- network denied
- snapshot created / restored
- artifact exported
- cleanup completed

특성: **tamper resistant · append only · retention policy 적용**.

---

## 10. 인증 / 인가

### 10-1. Sandbox Access Token

Scope 예시:
- `workspace:read` `workspace:write`
- `terminal:exec`
- `preview:open`
- `artifact:download`

특성: short-lived · revocable · session-scoped · tenant-scoped.

### 10-2. WebSocket 인증

Terminal / WebSocket 은 **반드시**:
- JWT validation
- session ownership validation
- sandbox ownership validation
- origin validation

### 10-3. Internal Service Auth

Sandbox ↔ internal service:
- **mTLS**
- workload identity
- SPIFFE / SPIRE (검토 대상)

---

## 11. 비용 · Quota 정책

Quota 없으면 abuse 발생한다.

### 11-1. User Quota (필수)

- max concurrent sandbox
- max CPU / max memory / max disk
- max build time
- max artifact size
- max network egress
- max monthly runtime

### 11-2. Tier 기반

| Tier | 정책 |
|---|---|
| **Free** | short timeout · small disk · suspend aggressive · limited artifact retention |
| **Pro** | longer session · higher memory · snapshot retention 증가 |
| **Enterprise** | dedicated pool · custom retention · private networking |

### 11-3. Cost Control

- idle compute 제거
- snapshot compression (zstd)
- object lifecycle cleanup
- auto archive
- warm pool size 제한
- aggressive cleanup

---

## 12. 최종 아키텍처 결정 (요약)

### Runtime
- 기본: **Kata + Firecracker** 기반 MicroVM
- 내부 저위험 workload 만 hardened runc 허용

### State
- **Redis** → runtime state / TTL / queue / lease
- **PostgreSQL** → persistent metadata / billing / audit
- **Object Storage** → workspace snapshot / artifact / logs

### Compute Model
- sandbox 는 disposable
- process 는 disposable
- state 는 externalized

### Recovery Model
- 프로세스 restore 안 함
- workspace restore 만 지원
- reconnect 시 새 sandbox 생성

### Lifecycle
```
lazy create → active → idle → suspend → archive → delete
```

### Security Model
- zero trust sandbox
- default deny networking
- no host kernel trust
- MicroVM isolation
- least privilege
- per-session isolation
- per-tenant namespace

---

## 13. 현재 구현과의 갭 분석

| 결정사항 | 현재 코드 위치 | 갭 | 우선순위 |
|---|---|---|---|
| **MicroVM 기본** | `backend/app/services/code_sandbox.py:88` `_docker_available()` | runc 만, MicroVM 런타임 클래스 *없음* | P0~P1 |
| **native fallback 금지** | `code_sandbox.py:117-122` | Docker 없으면 host venv 실행 — 즉시 차단 필요 | **P0** |
| **Disposable compute** | `code_sandbox.py:67` `tmpdir = tempfile.mkdtemp(...)` | 단일 실행마다 tmpdir — *세션 개념 부재* | P1 |
| **Externalized state — Redis** | 부재 | Redis 의존성 *없음*. memory state 만 사용 | P1 |
| **Externalized state — DB** | `backend/app/db/models.py` (1,330 LOC) | session/sandbox/workspace 모델 *부재* | P1 |
| **Externalized state — Object Storage** | `backend/app/features/documents/router.py:67` 로컬 `data/uploads` | S3 호환 부재. 디스크만. | P1~P2 |
| **Lifecycle (lazy → active → idle → suspend)** | 부재 | 매 도구 호출이 *원샷*. 세션 상태 없음 | P1 |
| **Snapshot** | 부재 | 없음 | P2 |
| **Node scheduling / pool 분리** | 부재 | 단일 호스트 단일 docker | P2 |
| **Network default deny** | `code_sandbox.py:97` `"--network", "none"` ✅ | Docker 모드 한정. native 모드는 호스트 네트워크 풀권한 (위 P0 와 결합) | P0 |
| **Audit log** | `services/metrics.py` 의 `usage_events` 일부 | sandbox-specific audit 분리 *없음*. immutable 보장 X | P2 |
| **Token scope (workspace/terminal/preview/artifact)** | `core/security.py` JWT 단일 토큰 | scope 분리 *없음* | P2 |
| **mTLS 내부 서비스** | `worker/tool_server/base.py:75-80` Bearer 토큰 옵션 | mTLS 부재. 토큰 미설정 시 통과 (full-audit Top 13 참조) | P2 |
| **Per-user quota** | `services/quota.py` storage 만 | CPU/memory/build time/egress quota 부재 | P2 |
| **OpenTelemetry** | `services/metrics.py` (raw) | OTel 미도입 | P2 |

---

## 14. 마이그레이션 로드맵

### Phase 0 — 즉시 차단 (이번 주, P0)

목표: native fallback 경로를 *런타임에 막는다*. ADR 의 *high-risk runc 금지* 정책을 지금 코드로 강제.

- [ ] `Settings.sandbox_require_docker: bool = True` 추가 (env 토글)
- [ ] `code_sandbox.run_code()` 의 `use_docker = _docker_available()` 분기: `False` 이고 `sandbox_require_docker=True` 면 *즉시 `RuntimeError("sandbox runtime unavailable")`* 또는 `{"ok": False, "category": "sandbox_disabled", "error": "관리자가 코드 실행 환경을 비활성화했습니다."}` 반환
- [ ] `services/agent.py` `_exec_tool` 의 `code_interpreter` 분기에서 sandbox 비활성 응답을 사용자 친화 텍스트로 매핑
- [ ] Docker 호출 시 `--read-only --cap-drop=ALL --security-opt=no-new-privileges --user 65534:65534` 추가 (runc 강화)
- [ ] **Acceptance**: `tests/test_sandbox_isolation.py` 에서 native 모드 강제 활성화 시 거부 응답 + Docker 모드에서 hostname/uid/env 격리 확인

### Phase 1 — State 외재화 (다음 2주)

목표: *세션 개념* 도입 + Redis/DB 분리.

- [ ] `db/models/sandbox_session.py` 신규 — 본 ADR §6-2 컬럼들
- [ ] `db/models/sandbox_workspace.py` 신규 — workspace 메타
- [ ] `services/redis_client.py` 신규 — `aioredis` 어댑터
- [ ] Redis TTL 표준(§6-1) 키 스키마 적용
- [ ] `code_sandbox.run_code()` 시그니처: `run_code(code, *, session_id, workspace_id)` — 세션 컨텍스트 받음
- [ ] Lifecycle state machine 구현 (`services/sandbox_lifecycle.py`) — active/idle/suspend 전환 + heartbeat
- [ ] **Acceptance**: heartbeat 만료 시 idle 전환, idle 30분 후 suspend, Redis 유실 후 DB 기반 복구 통과

### Phase 2 — Kata Runtime 도입 (다음 1~2개월)

목표: high-risk 코드 실행을 Kata MicroVM 으로 격리.

- [ ] 인프라 PoC — k8s + `RuntimeClass: kata` 또는 docker `--runtime=kata-runtime`
- [ ] 노드 풀 분리: shared(runc) / secure(kata)
- [ ] `Settings.sandbox_runtime_class: Literal["runc", "kata"]` + 분기
- [ ] Workspace snapshot 메커니즘 (§7) — tar+zstd, S3 업로드, suspend trigger
- [ ] Object Storage 클라이언트 (`services/object_storage.py`) — boto3 또는 minio
- [ ] WebSocket/terminal 토큰 scope 분리 (§10)
- [ ] **Acceptance**: 동일 코드가 secure pool 에서 동작 + 5분간 idle → snapshot S3 업로드 → 30분 후 suspend → reconnect 시 새 sandbox + workspace restore 성공

### Phase 3 — Firecracker / Multi-Tenancy / 운영 성숙 (3~6개월)

목표: Production 등급의 멀티테넌트 운영.

- [ ] Firecracker (또는 Kata + Firecracker runtime) — 더 가벼운 boot
- [ ] Network default deny + egress allow-list per workspace
- [ ] Per-user/tier quota (§11) + billing 집계
- [ ] OTel + Prometheus + Loki + Tempo + SIEM
- [ ] Audit log immutable 저장소 (append-only, 별도 bucket + Object Lock)
- [ ] mTLS / SPIFFE workload identity
- [ ] Enterprise dedicated pool 옵션
- [ ] **Acceptance**: 동시 100 세션 부하 + tenant cross-talk 0건 + 평균 cold start < 1.5s

---

## 15. 운영 체크리스트 (Phase 0~3 공통)

매 배포 전:

- [ ] **Native fallback 금지** 활성? — `Settings.sandbox_require_docker=True` (prod)
- [ ] Docker run flags: `--network none --memory ... --cpus ... --pids-limit ... --read-only --cap-drop=ALL --security-opt=no-new-privileges --user 65534:65534`
- [ ] Workspace 임시 디렉토리 권한 0700, 종료 시 wipe
- [ ] 토큰 스코프: WebSocket 은 `terminal:exec` 만, 미디어 다운로드는 `artifact:download` 만
- [ ] Redis 미가용 시 graceful degrade (in-memory 캐시 + 경고 로그)
- [ ] Audit log append-only 검증 (prev test 통과)
- [ ] Quota: per-user max concurrent sandbox 검증

---

## 16. Open Questions (후속 결정 필요)

1. **K8s 채택 여부** — 현재 docker-compose 만 사용. Phase 2 의 secure pool 구성에 k8s 가 자연스러우나 운영 부담 큼. 대안: nomad + firecracker 직접.
2. **Object Storage provider** — 자체 호스트 minio vs AWS S3 vs Cloudflare R2. 현재 사내 환경이라 minio 가 0차 후보.
3. **periodic autosave 주기** — 5분 vs 15분. 비용/사용자 경험 trade-off 측정 후 결정.
4. **node_modules cache** snapshot 포함 여부 — 옵션 토글로 두되 기본값을 *제외* 또는 *별도 layer* 로 둘지.
5. **WebSocket origin allow-list** vs **CORS 와 분리** — 토큰 scope 도입 후 재검토.

---

## 17. 변경 이력

- **2026-05-22** — 초안 Accepted. Phase 0 (native fallback 차단) 을 *이번 주 P0 PR* 에 포함. Phase 1~3 은 별도 ADR-0002~0004 로 분기 가능.

---

## 18. 참조

- [`docs/full-audit.md`](../full-audit.md) §2 TOP 2 (sandbox RCE), §2 TOP 1 (SSRF — 본 ADR §8-3 default-deny 와 결합)
- [`docs/sandbox.md`](../sandbox.md) (기존 sandbox 사양 — 본 ADR 로 갱신 대상)
- [`backend/app/services/code_sandbox.py`](../../backend/app/services/code_sandbox.py) — Phase 0 수정 대상
- [`backend/app/services/agent.py`](../../backend/app/services/agent.py) `_exec_tool` — code_interpreter 분기
- Kata Containers — https://katacontainers.io
- Firecracker — https://firecracker-microvm.github.io
