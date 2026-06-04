# ADR-0005 · Claude Code as Coding Agent (replacing `code_interpreter`)

| | |
|---|---|
| **Status** | Accepted (2026-05-22) |
| **Deciders** | CTO / Platform Eng |
| **Layer** | Tool design |
| **Replaces** | `code_interpreter` 도구 (Python 단일 실행) |
| **Related** | [ADR-0001 Sandbox Runtime](0001-sandbox-runtime.md) · [ADR-0002 User-Scoped Platform](0002-user-scoped-ai-execution-platform.md) · [ADR-0004 Self-Modifying Tools](0004-self-modifying-chatbot-tools.md) |

---

## 0. 한 줄 결론

> 코딩 도구는 우리가 만들지 않는다. **Anthropic Claude Code CLI** 를 sandbox 안에서 호출하는 어댑터(`claude_runner.py`)만 만들고, 기존 `code_interpreter` 는 폐기한다.
> 사용자가 **자기 ANTHROPIC_API_KEY** 를 등록하면 그 키로 호출되어 *비용도 사용자 청구* (BYOK).

---

## 1. Context — 왜 교체인가

`code_interpreter` 는 Docker/native 에서 *단순 Python 코드 실행* 만. 한계:

- **Read/Edit 분리 안 됨** — 코드 한 덩어리로 `open(f).read() + modify + write()` 짜야 함
- **세션 메모리 없음** — 매 호출이 독립 (LLM 이 매번 같은 컨텍스트 재구성)
- **에러 복구를 outer agent 가 함** — `_detect_code_error` + `pip install` 휴리스틱 (agent.py:442-475). 깨지기 쉬움
- ***우리가 유지보수* 부담** — sandbox 강화 / 폰트 / 라이브러리 / 에러 분류는 *결국 우리 일감*

Claude Code CLI 는:
- ✅ Read / Edit / Bash 도구 분리 + LLM 이 *자체적으로 어떤 도구 쓸지 결정*
- ✅ Extended thinking + stream-json 출력 → 우리 SSE 와 직접 매핑 (ADR-0002 §6 검증)
- ✅ Anthropic 공식 — 모델 업그레이드/버그픽스가 자동
- ✅ 작업 디렉토리 화이트리스트 내장 → sandbox 격리와 **2중 보호**

거래: *우리 자유도가 줄어든다*. 단, 우리 사업은 ADR-0002 §1-2 의 *플랫폼 layer* — 코딩 도구는 *소비* 가 정답.

---

## 2. 결정사항

### 2-1. 도구 카탈로그 변경 (`backend/config/tools.yaml`)

- ❌ **제거**: `code_interpreter` (slug + parameters)
- ✅ **추가**: `claude_code` — `{prompt: str, allowed_tools: ["Read","Edit","Bash"]}`

### 2-2. 실행 어댑터 (`backend/app/services/claude_runner.py`)

```
run_claude_code(prompt, *, workspace_files, allowed_tools, max_turns, timeout,
                user_id, user_api_key)
  → dict { ok, session_id, model, text, tool_steps, cost_usd,
           input_tokens, output_tokens, permission_denials, error,
           disabled, category }
```

- *호스트 native* (Settings.sandbox_require_docker=false) 또는 *Docker* (true) 모드 분기
- Docker 모드 cmd:
  ```
  docker run --rm --network bridge --memory 512m --cpus 1 --pids-limit 64
             --cap-drop ALL --security-opt no-new-privileges
             -e ANTHROPIC_API_KEY=<user/global>
             -v {workspace}:/workspace:rw -w /workspace
             chatbot-sandbox claude --bare --print
             --output-format stream-json --verbose
             --max-turns 5 --allowedTools <comma>
             "<prompt>"
  ```
- stream-json NDJSON 파서 — `system / assistant / user / result` 4 type 매핑 (ADR-0002 §6-3 schema 검증됨)

### 2-3. BYOK (Bring Your Own Key)

- DB: `user_api_keys` 테이블 — `user_id, provider, encrypted_key (Fernet), key_fingerprint, last_used_at, revoked_at`
- 암호화: `Settings.master_encryption_key` (Fernet, env `MASTER_ENCRYPTION_KEY`). 운영은 KMS/Vault 권장
- Vault 서비스: `backend/app/services/key_vault.py` — register / get / list / revoke
- 정책: `Settings.anthropic_key_policy = "fallback_global" | "require_byok"`
  - `fallback_global` (현재 기본): 사용자 키 없으면 전사 키 사용
  - `require_byok`: 사용자 키 없으면 도구 비활성 (예: 우리 비용 제로)
- 보안 룰:
  - 평문 키는 *어디에도 로깅 X* — fingerprint(`sk-ant-...XYZA(len=108)`) 만
  - 응답에 평문 절대 X — UI 는 fingerprint 만 표시
  - 매 호출 후 평문 변수 즉시 폐기 (Python GC 한계 내)
  - 복호화 실패 시 silent fallback — *master key 회전* 시나리오 안전

### 2-4. 사용자 UI

- `/mypage` 페이지 안 `<ApiKeysSection />` 신규 — 등록·조회·해제
- backend `/auth/me/api-keys` GET / POST / DELETE
- 키 입력은 `<input type="password" autocomplete="off">` — 브라우저 캐시 X

### 2-5. 도구 디스패처 (`backend/app/services/agent.py`)

```python
elif fn_name == "claude_code":
    user_api_key = None
    if workspace.user and workspace.db:
        from app.services.key_vault import get_user_api_key, is_vault_available
        if is_vault_available():
            user_api_key = await get_user_api_key(
                workspace.db, user_id=workspace.user.id, provider="anthropic",
            )
    result = await run_claude_code(prompt, ..., user_api_key=user_api_key)
    content = format_claude_result(result)
    images.append({"_claude_code": True, **result, "prompt": prompt})
```

---

## 3. Phase 매핑

### Phase 0 (오늘 적용) — 기능 동작
- ✅ tools.yaml `code_interpreter` 제거, `claude_code` 추가
- ✅ claude_runner.py (host-native + Docker 분기)
- ✅ agent.py `_exec_tool` 분기 교체 + BYOK wiring
- ✅ DB schema (`user_api_keys`) — backend 부팅 시 자동 생성
- ✅ key_vault.py + Fernet 암호화
- ✅ backend /auth/me/api-keys CRUD
- ✅ frontend `ApiKeysSection` 컴포넌트 + `/mypage` 통합
- ✅ `MASTER_ENCRYPTION_KEY` `.env` 주입
- 🟡 Docker 이미지 (`chatbot-sandbox`) + claude CLI 통합 — npm install 기반 빌드. 빌드는 docker group 권한이 *현 셸* 에 적용되어야 (사용자가 새 셸 또는 `newgrp docker`)
- 🟡 운영 토글 `SANDBOX_REQUIRE_DOCKER=false` 유지 (DEV) — 옵션 B (Docker) 정비 후 true 복원

### Phase 1 (다음 주) — 보안·관측
- [ ] `audit usage_events.kind=claude_code` 통합 — 비용·tool_steps·fingerprint 기록
- [ ] `permission_denials` 사용자 UI 토스트
- [ ] `anthropic_key_policy=require_byok` 실험 — 전사 키 fallback 끄기
- [ ] tests: `test_claude_runner.py` (stream-json 파서 회귀 + mock subprocess)
- [ ] Docker mode 통합 — `chatbot-sandbox` 이미지 build → 빌드 권한 정비 → `SANDBOX_REQUIRE_DOCKER=true` 복원

### Phase 2 (2~4주) — 본격 sandbox
- [ ] ADR-0002 §4 의 4단 격리 (tenant/user/workspace/session/task) — workspace 영속
- [ ] workspace snapshot S3 (ADR-0001 §7)
- [ ] Kata/Firecracker runtime 도입
- [ ] LiveLog terminal (xterm.js) — Claude Code 진행 라이브 표시

---

## 4. 갭 / 위험

| 항목 | 위험 | 완화 |
|---|---|---|
| 호스트 native invoke (현재) | claude CLI 가 호스트 fs 일부 접근 가능 | CLI 내장 화이트리스트 + 임시 디렉토리. 단 RCE 가능성 여전 → Phase 1 Docker 강제 |
| BYOK 키 평문이 메모리 거침 | 메모리 덤프 시 노출 | 호출 직후 변수 폐기, 로깅 X, audit fingerprint 만 |
| `master_encryption_key` 회전 | 기존 DB 행 복호화 불가 | 회전 절차: 새 키로 재암호화 마이그레이션 또는 모든 키 revoke 후 사용자 재등록 |
| Claude Code CLI 버전 변경 | stream-json schema 변경 가능 | 파서가 *알 수 없는 필드 무시*. 버전 CI 추적 (`claude --version` 로그) |
| 비용 폭주 | 사용자가 무제한 호출 | `--max-turns 5` 하드 캡 + `Settings.sandbox_max_timeout_seconds=30` + 향후 per-user quota |
| 키 도용 | 사용자가 본인 키 입력 후 다른 사람이 본인 계정 탈취 | `revoke` UI + `last_used_at` 표시로 이상 탐지 |

---

## 5. 운영 체크리스트 (Phase 1 진입 전)

- [ ] `SANDBOX_REQUIRE_DOCKER=true` 복원 (Docker 이미지 정상 빌드 + claude CLI 검증)
- [ ] `MASTER_ENCRYPTION_KEY` 백업 (KMS/Vault 마이그레이션 전제)
- [ ] `anthropic_key_policy` 결정 (fallback_global / require_byok)
- [ ] tests: stream-json parser / docker cmd hardening / BYOK key roundtrip
- [ ] 운영 문서: `MASTER_ENCRYPTION_KEY` 회전 절차 / 마스킹 룰 / audit 보존 기간

---

## 6. 1페이지 요약

```
┌──────────────────────────────────────────────────────────────────┐
│  코딩 도구 = Anthropic Claude Code CLI (소비)                     │
│  우리는 어댑터 + sandbox + BYOK + audit + quota 만 책임           │
│                                                                  │
│  code_interpreter   ❌ 폐기                                       │
│  claude_code        ✅ 신규 (Read / Edit / Bash)                  │
│                                                                  │
│  실행 모드                                                         │
│   - SANDBOX_REQUIRE_DOCKER=true   → docker run chatbot-sandbox    │
│   - false (DEV)                  → host native subprocess         │
│                                                                  │
│  BYOK                                                              │
│   - user_api_keys 테이블 (Fernet AES-128-CBC + HMAC-SHA256)       │
│   - /auth/me/api-keys GET/POST/DELETE                            │
│   - /mypage 안 ApiKeysSection UI                                  │
│   - 평문 절대 노출 X — fingerprint 만                              │
│                                                                  │
│  정책                                                              │
│   - fallback_global (현재): 사용자 키 없으면 전사 키              │
│   - require_byok:           사용자 키 없으면 비활성                │
└──────────────────────────────────────────────────────────────────┘
```

---

## 7. 변경 이력

- **2026-05-22** — Phase 0 코드 머지: tools.yaml / claude_runner / agent / DB schema / key_vault / CRUD / UI / `.env` MASTER_ENCRYPTION_KEY 주입.
- **2026-05-22 (오후)** — Phase 1 도 완료: `chatbot-sandbox` 이미지에 Node.js + Claude Code CLI(`2.1.148`) npm 통합. `.env` `SANDBOX_REQUIRE_DOCKER=true` 복원. backend uvicorn 을 `sg docker` 로 docker group 활성 셸에서 기동. e2e smoke 통과:
  ```
  text: 'The year is 2026 now.'
  cost: $0.0107  model: claude-opus-4-7[1m]
  ```
  `--allowedTools=A,B,C` nargs='+' 가 prompt 까지 흡수하는 이슈를 `=` 문법으로 회피 (claude_runner.py:cli_args).
