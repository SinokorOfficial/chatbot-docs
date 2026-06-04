# ADR-0003 · Embedded IDE Workspace

| | |
|---|---|
| **Status** | ⏸ **Deferred** (2026-05-22 · Accepted → Deferred 같은 날 변경) |
| **Deciders** | CTO / Platform Eng |
| **Layer** | Product / UX (ADR-0002 의 *동위* 결정) |
| **Related** | [ADR-0002 · User-Scoped AI Execution Platform](0002-user-scoped-ai-execution-platform.md), [ADR-0001 · Sandbox Runtime & Lifecycle](0001-sandbox-runtime.md), **[ADR-0004 · Self-Modifying Chatbot Tools](0004-self-modifying-chatbot-tools.md) (대체 그림)** |

---

## ⏸ 왜 Deferred 인가 (2026-05-22)

본 ADR 의 *임베드 IDE* 는 **외부 개발자 사용자** 가 우리 sandbox 위에서 *자기 코드*를 작성·실행하는 SaaS 시나리오에 최적화. 그러나:

1. **현 제품 컨텍스트** — `장금상선 그룹 챗봇` 의 주 사용자는 *사내 일반 직원*. 90% job-to-be-done 은 *문서 검색·답변·자동화*. IDE 가 필요한 사용자 0~소수.
2. **사용자 직관** — 오너 직접: *"채팅으로 바로 반영하면 IDE 안 해도 되지 않나"*
3. **자연스러운 진화** — 이미 `agent.py` 의 `schedule_query` 가 *"채팅으로 예약 등록"* 패턴. 이를 *챗봇·워크플로우·문서·스킬·계정* 으로 확장하는 게 우리 사용자에게 필요한 가치.
4. **재활성 조건** — 외부 개발자 사용자 · ISV 파트너 · 코드 작업이 *주 사용 사례* 인 시점. 그때 본 ADR 의 Phase 2 작업을 다시 시작.

본 ADR 의 모든 기술 결정(code-server 임베드 / scoped JWT / reverse proxy / 보안 모델)은 *기록으로 유효*. 코드 작업은 시작하지 않는다.

> 다음 그림: **[ADR-0004 · Self-Modifying Chatbot Tools](0004-self-modifying-chatbot-tools.md)** — 채팅 안에서 챗봇·워크플로우·문서·스킬을 *바로* 수정.

---

## (이하 원안 — 미래 참고용)

---

## 0. 한 줄 결론

> 우리는 IDE 도 직접 만들지 않는다.
> **오픈소스 Cloud IDE engine (`code-server` 우선) 을 sandbox MicroVM 안에 임베드** 하여
> 사용자가 *직접 편집·터미널·디버깅* 할 수 있는 user-scoped IDE 를 제공한다.
> Claude Code CLI 와 *같은 workspace volume* 을 공유하여 **사용자 편집 ⇄ AI 편집** 이 동시 가능하다.
>
> 신뢰 경계는 IDE 가 아니라 sandbox. 우리는 *IDE 의 lifecycle / token / snapshot / audit* 만 책임진다.

---

## 1. Context — 왜 임베드 IDE 인가

### 1-1. 사용자가 그린 그림

> *"완전 IDE 를 연결한다 느낌"*

이는 ADR-0002 §6 의 옵션 A(채팅 안 도구) · 옵션 B(우측 diff 패널) 보다 한 계단 위. *진짜 편집기 · 터미널 · 파일 트리* 가 함께 있어야 한다.

### 1-2. 빌드 vs 사기

| 길 | 비용 | 성숙도 | 위험 |
|---|---|---|---|
| 자체 구현 (Monaco + xterm.js + LSP 직접) | 6~12개월 | 낮음 | 매우 큼 (LSP·debug·extension 생태계) |
| **code-server 임베드** | 1~2개월 | **매우 높음** (VS Code 그대로) | 운영(=세션 라이프사이클 우리 책임) |
| Theia 임베드 | 2~3개월 | 높음 (Gitpod 채택) | 라이선스·튜닝 |
| Gitpod Workspace SaaS 재판매 | 0 | 매우 높음 | 데이터 주권·비용 |

ADR-0002 의 *"코딩 모델은 소비, 플랫폼은 자산"* 과 같은 정신으로 ***"IDE 는 소비, 격리/세션/감사는 자산"*** → **code-server 가 1순위**.

---

## 2. 책임 분리 확장 (ADR-0002 §3 보강)

| 책임 | Claude Code | code-server | Our Platform |
|---|---|---|---|
| 코드 생성/수정 (AI) | ✅ | ❌ | ❌ |
| 사용자 직접 편집 (수동) | ❌ | ✅ | ❌ |
| 파일 트리 / 다중 탭 / IntelliSense | ❌ | ✅ | ❌ |
| 터미널 (xterm.js) | ❌ | ✅ | ❌ |
| LSP / debugger / extension | ❌ | ✅ | ❌ |
| **신뢰 경계** | ❌ | ❌ | ✅ (sandbox) |
| Sandbox allocation | ❌ | ❌ | ✅ |
| Workspace volume 마운트 | ❌ | ❌ | ✅ |
| Scoped token 발급 (`ide:open`) | ❌ | ❌ | ✅ |
| Session lifecycle (idle/suspend) | ❌ | ❌ | ✅ |
| Snapshot / restore | ❌ | ❌ | ✅ |
| Audit (파일 변경/터미널 명령) | ❌ | ❌ | ✅ |
| Cross-user isolation | ❌ | ❌ | ✅ |

> code-server 는 *사용자에게 보이는 IDE*, Claude Code 는 *AI 의 손*, 우리는 *둘을 안전하게 묶는 layer*.

---

## 3. 아키텍처 — Sandbox 내부 구조

```
┌───────────── User Browser ─────────────────────────────────────────────┐
│  채팅 패널 (지금 화면)            │ IDE 패널 (신규)                   │
│  - 대화 / 명령 / 결과 요약        │ <iframe src=".../ide/{token}"/>   │
│  - Claude Code 도구 호출 결과     │  - 파일 트리                      │
│                                  │  - Monaco editor                  │
│                                  │  - terminal (xterm.js)            │
│                                  │  - diff / preview                 │
└──────────┬───────────────────────┴──────────────┬───────────────────┘
           │ SSE (chat)                          │ WebSocket (code-server)
           │                                     │  + scoped JWT 검증
           ▼                                     ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Our Platform (FastAPI)                                              │
│   - /chat/stream  (기존)                                              │
│   - /ide/session  POST  → sandbox allocate + scoped JWT 발급          │
│   - /ide/{sandboxId}/{path}  GET/UPGRADE → code-server 로 proxy       │
│   - /ide/heartbeat  PATCH                                            │
└──────────────────────────────────┬───────────────────────────────────┘
                                   ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Sandbox MicroVM (Kata / Firecracker, Phase 2~3)                    │
│   /workspace                    ← shared volume                       │
│     ├─ <user files>                                                  │
│   /usr/local/bin/code-server    (포트 :8080 listen, host-binding x)  │
│   /usr/local/bin/claude         (Claude Code CLI)                    │
│   /entrypoint.sh                ← supervisord 또는 dumb-init          │
│                                                                      │
│  egress: default deny (npm/pypi proxy 만 allow-list)                │
│  user:   nobody (UID 65534)                                          │
│  caps:   ALL dropped, no-new-privileges                              │
└──────────────────────────────────────────────────────────────────────┘
```

핵심 포인트:
1. **code-server 는 sandbox 안에서만 listen**. 호스트로 바인딩 *금지*. 외부 노출은 *우리 platform 의 reverse proxy* 만.
2. **/workspace 는 단일 volume**. code-server (사용자 편집) 와 Claude Code (AI 편집) 가 *같은 파일*을 본다.
3. **scoped JWT (`ide:open`) 으로만 proxy 통과**. 토큰은 *세션 단위 short-lived*, *user/sandbox 검증* 포함.
4. **WebSocket upgrade** 는 platform 이 가운데서 *audit log 기록* 후 forward.

---

## 4. 사용 흐름 (사용자 입장)

```
1) 사용자가 "이 챗봇 코드 좀 손볼래?" 라고 입력
   → /chat/stream 응답 + 우측 [IDE 열기] 버튼

2) [IDE 열기] 클릭
   → POST /ide/session
      - 우리 platform 이 sandbox lazy-create (Kata/Firecracker)
      - workspace snapshot S3 → restore
      - scoped JWT (`ide:open` + `sandbox:{id}`) 발급, TTL 15분 (idle TTL 과 동일)
   → 응답: { ide_url: "/ide/{sandboxId}/?folder=/workspace", expires_at: ... }

3) 우측 iframe 에 ide_url 로드
   → user 가 직접 파일 열고 편집, 터미널 사용

4) 채팅에서 "ChatMessage.tsx 의 마크다운 sanitize 추가해줘"
   → Claude Code CLI 가 sandbox 안에서 동일 /workspace 에서 작업
   → 변경된 파일이 code-server 에 *자동 reload* (file watcher)
   → 사용자가 IDE 에서 diff 즉시 확인

5) 15분 IDE activity 없으면 idle, 30분 더 지나면 suspend
   → workspace snapshot S3 업로드 + sandbox 종료
   → 사용자 재접속 시 새 sandbox + restore (ADR-0001 §5)
```

---

## 5. Frontend 통합 사양

### 5-1. 라우트 / 컴포넌트

* `frontend/app/(workspace)/chat/page.tsx` — 기존. 우측 영역에 *IDE 패널 toggle 버튼* 추가
* `frontend/features/ide/IdePanel.tsx` (신규) — iframe + 헤더 + lifecycle 상태 표시
* `frontend/features/ide/useIdeSession.ts` (신규) — `POST /ide/session` 호출 + heartbeat 갱신 + expire 처리
* `frontend/features/ide/IdeToolbar.tsx` (신규) — 새 터미널 / preview 열기 / snapshot 강제 저장
* `frontend/shared/layout/AppShell.tsx` — chat 페이지에서 *split layout* 으로 진입 (`?ide=open` 쿼리 토글)

### 5-2. UX 모드

기본 모드는 **Split**:

| Split (기본) | IDE Only | Chat Only |
|---|---|---|
| 채팅 60% / IDE 40% (드래그로 비율 조절) | IDE 100% (채팅은 우상단 mini bubble) | IDE 닫힘. 채팅 100% |

토큰 부족·sandbox 없음·suspend 상태에서는 *IDE 영역에 EmptyState* — "IDE 시작하기" 단일 CTA. ADR-0002 의 *EntryMissionBar 패턴* 과 동일 톤.

### 5-3. iframe 보안

- `sandbox="allow-scripts allow-same-origin allow-forms allow-popups"` (clipboard 는 별도 토글)
- `allow="clipboard-read; clipboard-write"` — 사용자가 IDE 코드 복사 가능해야 함
- `referrerpolicy="strict-origin-when-cross-origin"`
- iframe 의 origin 은 *우리 platform 도메인 하위 path* — third-party cookie 문제 회피
- CSP `frame-src 'self'` — 외부 iframe 임베드 금지

---

## 6. Backend 통합 사양

### 6-1. 신규 라우터 (`backend/app/features/ide/router.py`)

| Method | Path | 동작 |
|---|---|---|
| `POST` | `/ide/session` | sandbox 할당 + scoped JWT 발급 |
| `PATCH` | `/ide/session/{sandboxId}/heartbeat` | idle marker 갱신 (TTL 15m) |
| `DELETE` | `/ide/session/{sandboxId}` | explicit close — snapshot + suspend |
| `GET / WS` | `/ide/{sandboxId}/{path:path}` | code-server 리버스 프록시 (`httpx + WebSocket forward`) |

핵심 검증:
* 매 요청마다 `Authorization: Bearer <scopedJWT>` 의 `scopes: ["ide:open"]` + `sandbox_id == path param` + `user_id == JWT.sub` 일치
* 미일치 → `403 Forbidden`. 로그에 `audit.ide.access_denied` 기록
* WebSocket upgrade 시 origin 검증 — `request.headers.origin` 가 platform 도메인 외면 거부

### 6-2. Scoped JWT (ADR-0002 §6-4 확장)

새 scope:
```
ide:open              — IDE iframe 열기
ide:terminal:exec     — 터미널 명령 (별도 검증)
ide:write             — 파일 저장
```

기본 JWT exp = 15분 (idle TTL). 헬스 PATCH 호출로 갱신. expire 시 frontend 가 *재발급 후 iframe reload*.

### 6-3. Sandbox 이미지 확장 (`backend/sandbox/Dockerfile`)

현 Dockerfile (Python 분석 라이브러리 22개 + 한글 폰트) 위에 *layer 추가*:

```dockerfile
# 1) code-server (VS Code in browser)
ARG CS_VERSION=4.97.2
RUN curl -fsSL https://github.com/coder/code-server/releases/download/v${CS_VERSION}/code-server-${CS_VERSION}-linux-amd64.tar.gz \
    | tar -xz -C /usr/local/lib \
 && ln -s /usr/local/lib/code-server-${CS_VERSION}-linux-amd64/bin/code-server /usr/local/bin/code-server

# 2) Claude Code CLI (Anthropic)
RUN curl -fsSL https://claude.ai/install.sh | sh -s -- --bin-dir /usr/local/bin

# 3) supervisord — sandbox 안에서 code-server + ssh-less terminal 관리
RUN apt-get update && apt-get install -y --no-install-recommends supervisor \
 && rm -rf /var/lib/apt/lists/*

COPY supervisord.conf /etc/supervisor/conf.d/sandbox.conf
ENTRYPOINT ["supervisord", "-n", "-c", "/etc/supervisor/conf.d/sandbox.conf"]
```

> Phase 2 진입 시 분리된 `Dockerfile.ide` 또는 multi-stage 권장 — 가벼운 *exec-only* sandbox 와 *IDE-included* sandbox 두 종류.

---

## 7. 동시 편집 / 충돌

### 7-1. 시나리오: 사용자가 편집 중 Claude Code 가 같은 파일 수정

- code-server 는 *파일 watcher* 기반. 외부에서 파일이 바뀌면 *자동 reload* 또는 *"외부에서 변경됨, reload? "* 모달.
- 우리 platform 의 정책: **AI 가 수정한 파일은 audit 으로 강제 기록** + IDE 우측 패널에 *"Claude Code 가 X 파일을 수정했습니다 [diff 보기] [되돌리기]"* 토스트.
- 사용자의 *미저장 변경*이 있으면 — code-server 가 기본 충돌 모달을 띄움. 그게 정답.

### 7-2. 시나리오: terminal 에서 사용자가 `rm -rf /workspace`

- sandbox `--read-only` rootfs + 마운트된 `/workspace` 만 write 가능
- 사용자가 자기 workspace 를 자기가 지우는 건 *허용* (snapshot 에서 복구 가능)
- 단 *cross-user 영향 0* (별 sandbox)

---

## 8. ADR-0002 갭 추가

[ADR-0002 §10 현재 구현과의 갭](0002-user-scoped-ai-execution-platform.md#10-현재-구현과의-갭-분석-adr-0001-13--claude-code-통합-항목) 표에 다음 추가:

| 결정사항 | 현재 코드 위치 | 갭 | 우선순위 |
|---|---|---|---|
| **임베드 IDE** | 부재 | `features/ide/` frontend·backend 모두 신규. iframe + reverse proxy + WebSocket forward | P2 |
| **scoped JWT `ide:*`** | 부재 | `core/security.py` 에 scope claim 확장 필요 | P2 |
| **Dockerfile.ide (code-server + claude CLI)** | 현재 Dockerfile 은 exec-only | 별도 image 또는 layer 추가 | P2 |
| **WebSocket reverse proxy** | 부재 | httpx + uvicorn WebSocket forwarder | P2 |
| **IDE heartbeat / lifecycle** | 부재 | ADR-0001 §6-1 Redis TTL 패턴 활용 | P2 |
| **iframe sandbox attrs / CSP frame-src** | 부재 | next.config.ts headers 확장 | P2 |

---

## 9. Phase 매핑 (ADR-0001 §14 위에 추가)

### Phase 2 (1~2개월) — 기본 IDE 동작
- [ ] `backend/sandbox/Dockerfile.ide` — code-server + claude CLI 통합 image
- [ ] `backend/app/features/ide/router.py` — session/heartbeat/proxy endpoints
- [ ] `backend/app/core/scoped_token.py` — JWT scope 확장
- [ ] `frontend/features/ide/IdePanel.tsx` + `useIdeSession.ts`
- [ ] `frontend/app/(workspace)/chat/page.tsx` — split layout + IDE toggle
- [ ] CSP / iframe 보안 헤더 (`next.config.ts`)
- [ ] e2e: 새 계정 → 채팅 → IDE 열기 → 파일 편집 → 저장 → 새로고침 시 복원

### Phase 3 (3~6개월) — 운영 성숙
- [ ] code-server *extension marketplace 정책* — 화이트리스트 vs 전체 허용
- [ ] LSP / debugger backend (Python/Node) 미리 설치
- [ ] preview server reverse proxy (3000 등 임의 port 노출)
- [ ] *Live Share* 같은 페어 편집 (선택)
- [ ] IDE 안에서 git 통합 — 우리 platform 의 git proxy (사내 GitLab)

---

## 10. Open Questions

1. **code-server 라이선스** — MIT 와 비상업 제한 검토. 사내 한정이면 보통 OK, SaaS 출시 시 별도 확인.
2. **VS Code official Server (`code tunnel`)** vs **code-server** — Microsoft 의 공식 서버는 *Microsoft 계정 필수*. 자체 호스팅에는 code-server 가 맞음.
3. **extension marketplace** — Microsoft 의 공식은 라이선스로 third-party 사용 불가. *open-vsx.org* 가 대안.
4. **IDE 안 git 인증** — SSH key / PAT / OAuth? 사내 GitLab 연동 방식.
5. **사용자별 settings.json 영속화** — `~/.config/code-server/` 를 user-scoped persistent volume 으로 (workspace 와 별개).
6. **여러 IDE 세션 동시 열기** — 한 사용자가 여러 workspace 동시 편집? per-workspace sandbox 분리.

각각 ADR-0004 ~ ADR-0009 후보.

---

## 11. 1페이지 결정 요약

```
┌──────────────────────────────────────────────────────────────────┐
│  비전                                                              │
│   "우리는 IDE 도 만들지 않는다. code-server 를 임베드한다."          │
│                                                                  │
│  책임 분리                                                         │
│   Claude Code   = AI 의 손 (코드 생성/수정)                        │
│   code-server   = 사용자의 손 (직접 편집/터미널)                    │
│   Our Platform  = 둘을 격리·세션·snapshot·audit 하는 layer         │
│                                                                  │
│  workspace                                                       │
│   /workspace volume 을 code-server 와 Claude Code 가 *공유*       │
│   → 동시 편집 가능 (file watcher 가 reload)                       │
│                                                                  │
│  연결                                                              │
│   frontend iframe ─► our reverse proxy ─► sandbox :8080           │
│   ── scoped JWT (ide:open) ─ user/sandbox 검증                    │
│   ── WebSocket upgrade ─ audit log 기록 후 forward                │
│                                                                  │
│  Lifecycle                                                         │
│   ADR-0001 §4 그대로 — lazy → active → idle(15m) → suspend(30m)   │
│   재접속 시 새 sandbox + workspace snapshot 복구                  │
│                                                                  │
│  보안                                                              │
│   code-server 는 신뢰 경계가 *아니다*                              │
│   sandbox MicroVM 격리가 ultimate boundary                       │
│   default deny network · nobody user · cap-drop=ALL              │
└──────────────────────────────────────────────────────────────────┘
```

---

## 12. 변경 이력

- **2026-05-22** — 초안 Accepted. ADR-0002 의 *"코딩 모델은 소비"* 정신을 *IDE 영역에도* 확장. code-server 1순위 선정 (Theia/Gitpod 차순위). Phase 2 의 1~2개월 작업으로 분류.
