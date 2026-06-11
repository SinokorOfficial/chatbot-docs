# Claude Code CLI 옵션 매핑 · MCP/스킬 연동

클로드 코드 모드(`/chat` direct routing)는 사용자의 요청을 **실제 `claude` CLI** 로 그대로 흘려보냅니다. 이 문서는 2026-06-10 기준 **claude CLI 2.1.170 을 실측**해 정리한 *실제 옵션 ↔ 앱 인자/페이로드* 매핑과, 앱의 MCP 서버·Skill 을 세션 스코프로 결선하는 구조를 정리합니다.

> 배경: 과거 일부 옵션은 *유추* 로 추가돼 있었습니다(예: `--max-turns`). 실 CLI 에 존재하지 않아 무시되거나 오류였고, 이번에 실측값으로 교정했습니다.

---

## 빠른 요약 (무엇이 바뀌었나)

- `--max-turns` 는 **CLI 에 존재하지 않는** 옵션이었음 → 제거. 비용/과도실행 상한은 실존 옵션 **`--max-budget-usd`** 로 대체.
- 추론강도는 **`--effort`** 한 가지이며 유효값은 **`low | medium | high | xhigh | max`** 5단계. UI 의 6단계(`none`/`minimum` 포함)는 가짜 레벨이라 제거(가짜 값은 `medium` 으로 보정).
- 앱의 MCP 서버는 **`--mcp-config <json>`**, 앱의 Skill 은 **`--plugin-dir <dir>`** 로 *요청마다* 임시 물질화해 연동 → 멀티테넌트 격리.

---

## 옵션 매핑표 (실 CLI ↔ 앱)

`run_claude_code_stream` 이 조립하는 argv 와, 그 인자가 프런트엔드 페이로드(`claude_code_options` / `reasoning_effort`)에서 어떻게 채워지는지의 매핑입니다.

| 실제 CLI 옵션 | 앱 함수 인자 | 프런트 페이로드 키 | 유효값 / 비고 |
| --- | --- | --- | --- |
| `--bare` | (고정) | — | 항상 포함 |
| `--print` | (고정) | — | 비대화(스트리밍) 실행 |
| `--output-format stream-json` | (고정) | `output_format` | stream-json 고정 패스스루 시작점; UI 의 `output_format` 은 별도 옵션 노출용 |
| `--verbose` | `verbose` | `verbose` | bool |
| `--permission-mode <mode>` | `permission_mode` | (미노출) | `acceptEdits\|auto\|bypassPermissions\|default\|dontAsk\|plan`. **기본 `bypassPermissions`**. 무효값은 기본으로 보정 |
| `--append-system-prompt <text>` | `append_system_prompt` | `append_system_prompt` | 챗봇 시스템 프롬프트 + 에이전트 행동 지침. sanitize 적용 |
| `--allowedTools=<csv>` | `allowed_tools` | `allowed_tools` | 기본 `Read,Edit,Bash,Write` |
| `--disallowedTools=<csv>` | `disallowed_tools` | `disallowed_tools` | 선택 |
| `--model <id>` | `model` | `model` | 생략 시 CLI 기본(보통 sonnet 계열) |
| **`--effort <level>`** | `effort` | `reasoning_effort` | **`low\|medium\|high\|xhigh\|max`**. None 이면 옵션 생략. 가짜 레벨은 `medium` 보정 |
| **`--max-budget-usd <float>`** | `max_budget_usd` | (미노출) | 비용 상한. `>0` 일 때만 추가. 구 `--max-turns` 의도의 실존 대체 |
| **`--mcp-config <path>`** | `mcp_config_path` | (자동 결선) | 챗봇 enabled MCP Tool 을 임시 JSON 으로 물질화한 경로 |
| **`--plugin-dir <dir>`** | `plugin_dirs[]` | (자동 결선) | team/org/public 활성 Skill 을 물질화한 디렉토리(들) |
| `--add-dir <dir>` | `add_dir` | `add_dir` | path-traversal 검증 후 추가 |
| `--resume <session>` | `resume` | `resume` | 세션 이어가기 |
| `--dangerously-skip-permissions` | `dangerously_skip_permissions` | `dangerously_skip_permissions` | 위험 — UI 빨간 경고. 기본 permission-mode 와 중복이나 허용 확대 |

### 제거된(유추였던) 옵션

| 과거 표기 | 상태 | 대체 |
| --- | --- | --- |
| `--max-turns <n>` | **CLI 에 없음 — 제거** | 비용 상한은 `--max-budget-usd`. `max_turns` 인자는 하위호환 위해 signature 에만 남아 *무시*되며 argv 로 새지 않음 |
| effort `none` / `minimum` | **CLI 에 없음 — 제거** | `--effort` 5단계(`low~max`). 가짜 값 입력 시 `medium` 보정 |

> 검증 회귀: `backend/tests/test_claude_runner_cli_args.py` 가 실제 CLI 호출 없이 argv 만 캡처해 위 계약(특히 `--max-turns` 미포함, `--effort`/`--max-budget-usd`/`--mcp-config` 포함, 기본 permission-mode)을 잠급니다.

---

## 추론강도(`--effort`) 단계 가이드

| 단계 | 권장 용도 |
| --- | --- |
| `low` | 단순 문답·포맷 변환·짧은 스니펫(가장 빠름, 토큰 최소) |
| `medium` | 다중 파일 변경, API 설계 — **기본값** |
| `high` | 시스템 설계, 마이그레이션 계획(깊은 추론) |
| `xhigh` | 난도 높은 리팩터링·디버깅(매우 깊은 추론) |
| `max` | 대규모 아키텍처 변경, 보안 감사, 어려운 코드/수학(최대 thinking budget) |

---

## MCP / Skill 세션 스코프 연동 구조

앱은 **챗봇/사용자 단위로 enabled 된** MCP 서버와 Skill 을 *요청마다 임시로 물질화*해 CLI 에 넘깁니다. 전역 등록(`claude mcp add-json`)을 쓰지 않으므로, 한 사용자의 자격증명/도구가 다른 테넌트로 새지 않습니다(멀티테넌트 격리).

브리지: `backend/app/services/claude_cli_bridge.py`

```
요청 시작
  └─ new_scope_dir()                # 요청 스코프 임시 디렉토리(0o700)
       ├─ build_mcp_config(...)     # 챗봇 enabled MCP Tool → mcp-config-XXXX.json (0o600)
       │     → "--mcp-config <path>"
       └─ materialize_skills(...)   # team/org/public 활성 Skill → plugin-XXXX/skills/<slug>/SKILL.md
             → "--plugin-dir <dir>"
응답 종료(finally)
  └─ cleanup_scope_dir()            # 임시 산출물(자격증명 포함 가능) 일괄 제거 + 검증
```

라우터(`backend/app/features/chat/router.py` · `_stream_claude_code_direct`)가 위 산출물을 `run_claude_code_stream(mcp_config_path=..., plugin_dirs=...)` 로 결선하고, 응답 종료 시 `cleanup_scope_dir` 로 정리합니다. **브리지 실패는 비치명** — MCP/skill 없이 그대로 진행하고 로그만 남깁니다.

### MCP config JSON 스키마 (CLI 가 받는 형태)

```json
{
  "mcpServers": {
    "<sanitized-name>": {
      "type": "http",            // 또는 "sse" / "stdio"
      "url": "https://...",       // http/sse
      "headers": { "Authorization": "..." },
      "command": "node",          // stdio
      "args": ["server.js"],
      "env": { "TOKEN": "..." }
    }
  }
}
```

- 우선순위(뒤가 우선): `Tool.default_config` < `ChatbotTool.config`(바인딩) < `ToolCredential`(자격증명, headers/env 에 마지막 주입).
- transport 정규화: `http|streamable-http|https → http`, `sse → sse`, `stdio|local|subprocess|"" → stdio`.

### Skill plugin-dir 구조 (CLI 가 스캔하는 형태)

```
<plugin_dir>/
  skills/
    <slug>/
      SKILL.md      # YAML frontmatter(name/description/triggers) + 본문
```

이는 `~/.claude/skills/<name>/SKILL.md` 와 동일 구조이므로 `--plugin-dir <plugin_dir>` 로 그 세션에서만 로드됩니다.

---

## 보안 모델

- **자격증명 비로깅**: `ToolCredential.data`(Fernet 가능)는 복호화 시도 후 header/env *값으로만* 주입하고, 값 자체는 어떤 로그에도 남기지 않습니다(서버명/슬러그만 기록).
- **원자적 0o600 생성**: 자격증명이 담기는 MCP config 는 `os.open(O_CREAT|O_EXCL|O_WRONLY, 0o600)` 으로 *처음부터* 소유자 전용 권한으로 생성하고, 생성 후 그룹/기타 비트가 남아 있으면 파일을 지우고 예외를 던집니다(race·심링크 공격 차단).
- **sanitize**: MCP 서버명·skill 슬러그는 `[A-Za-z0-9_-]` 로 정규화 → path-traversal / JSON 키 오염 / argv injection 방지. skill 디렉토리는 `commonpath` 로 루트 하위인지 최종 검증.
- **plugin-dir traversal 검증**: `--plugin-dir` 경로는 `add_dir` 와 동일한 traversal 방어를 재사용하되, 공유 마켓플레이스 경로를 위해 cwd 를 추가 허용 루트로 둡니다.
- **정리 보장**: `cleanup_scope_dir` 는 조용히 실패하지 않습니다. rmtree 후 *실제 제거 여부를 검증*하고 1회 재시도, 그래도 남으면 ERROR 로 로깅(자격증명 누출 위험 알림).
- **전역 등록은 admin 전용**: `admin_mcp_register/remove` 는 list-argv subprocess(shell=False)로만 호출해 injection 을 차단합니다. *권한 검사는 라우터 책임*.
- **첨부 파일 디렉토리 구조 보존 + traversal 게이트** (2026-06-10): 워크스페이스로 들어가는 첨부 파일(`workspace_files` / 라우터 직행 pre-write / `_attachments/` 영속 3곳 모두)은 `app/core/pathsafe.safe_relative_path` 로 상대 경로 구조를 보존합니다(`src/app/page.tsx` 가 `page.tsx` 로 평탄화되던 회귀 금지). 절대경로·`..`·`.`·제어문자·`:`(드라이브) 세그먼트, 깊이 16 단계·512자 초과 경로는 거부되어 **basename 폴백**으로 격하되고, 조립된 최종 경로는 `resolve()` + `commonpath` 로 workdir 격리를 한 번 더 검증합니다(symlink 우회 방어). 회귀 잠금: `backend/tests/test_attachment_structure.py`.

---

## 슬래시 명령(`/`) 동기화표 — 웹 모드 `/` 메뉴 ↔ 실제 CLI 카탈로그

클로드 코드 모드의 채팅 입력창에서 `/` 를 누르면 뜨는 빠른 명령 메뉴는 **실제 `claude` CLI 슬래시 카탈로그**(2026-06-10 전수 재추출 — 총 **87개**: web-action 8 · cc-passthrough 33 · cli-only 46)와 동기화됩니다. 동기화의 단일 출처(SSOT)는 데이터 전용 파일 **`frontend/features/chat/claudeCodeCommands.ts`** 이며, 카탈로그는 **claude CLI 2.1.170 바이너리의 명령 정의(local-jsx name 필드) 직접 스캔**을 **`~/.claude/i18n/trans.py`**(한글 설명 88종)와 교차 대조해 만들었습니다. **유추 금지** — 명령/설명 갱신 시 바이너리 스캔 결과·`trans.py` 와 대조하고, 카탈로그가 단일 출처가 되도록 UI 는 카탈로그를 import 만 합니다(`ccPassthroughCommands()` / `cliOnlyCommands()` / `webActionCommands()`).

> 2026-06-10 전수 재추출에서 기존 카탈로그(34개)의 이름 오기 5건을 교정했습니다: `init-verifier`→`init-verifiers`, `cost`→`usage`(별칭 `cost`/`stats`), `primer`→`powerup`, `privacy`→`privacy-settings`, `heap`→`heapdump`.

각 명령은 세 분류 중 하나로 매핑됩니다.

- **web-action** — 챗봇 웹 UI 액션/라우팅으로 매핑되는 명령(`webAction` 라벨 보유). `/` 메뉴 "빠른 작업" 섹션에 노출.
- **cc-passthrough** — Claude Code 로 그대로 전달되는 명령. 선택 시 입력창에 `/{name} ` 가 prepend 되고, Enter 시 `claude_code` 직행 라우팅으로 넘겨 **헤드리스(비대화형) 실제 실행**됩니다. `/` 메뉴 "Claude Code 명령·스킬" 섹션에 `/{name}` 배지로 노출.
- **cli-only** — 터미널 TUI 전용(웹/헤드리스 불가). `/` 메뉴 하단에 *비활성 정보성* 펼침 안내로만 노출(선택 불가).

> 표는 `claudeCodeCommands.ts`(출처: `trans.py`)에서 생성합니다. "웹 동작" 열은 web-action 인 명령에 한해 매핑되는 웹 UI 액션이며, 나머지는 분류 동작을 따릅니다.

**web-action 전체(8개)** — 웹 UI 액션으로 매핑:

| 명령 | 웹 동작 | 설명 |
| --- | --- | --- |
| `/clear` | 새 채팅 | 빈 컨텍스트로 새 세션 시작; 이전 세션은 디스크에 남음(/resume 재개) |
| `/compact` | 요약 | 지금까지 대화를 요약해 컨텍스트 확보 |
| `/effort` | 추론 강도 | 모델 effort 수준 설정 (`low~max`) |
| `/help` | 가이드 | 도움말·명령어 표시 |
| `/mcp` | 도구·MCP 관리 | MCP 서버 관리 (`/tools` 라우팅) |
| `/model` | 모델 선택 | Claude Code의 AI 모델 설정 |
| `/usage` | 사용량 | 세션 비용·사용량·통계 표시 (별칭 `cost`/`stats`) |
| `/powerup` | 튜토리얼 | 클로드 코드 모드 3단계 튜토리얼 열기 (구 `primer`) |

**cc-passthrough(33개) 대표 항목** — 선택 시 `/{name} ` prepend 후 헤드리스 실제 실행:

| 명령 | 설명 |
| --- | --- |
| `/add-dir` | 작업 디렉터리 추가 |
| `/agents` | 에이전트 구성 관리 |
| `/copy` | Claude의 마지막 응답을 클립보드에 복사 (/copy N=N번째) |
| `/diff` | 커밋 안 된 변경과 턴별 diff 보기 |
| `/export` | 현재 대화를 파일/클립보드로 내보내기 |
| `/fork` | 대화를 잇는 백그라운드 에이전트 생성 |
| `/goal` | 목표 설정 — 조건 충족까지 계속 작업 |
| `/init-verifiers` | 코드 변경 자동 검증용 verifier 스킬 생성 |
| `/insights` | Claude Code 세션 분석 리포트 생성 |
| `/loop` `/loops` | 반복 루프 실행·조회·생성·삭제 |
| `/memory` | Claude 메모리 편집 |
| `/permissions` | 도구 권한 허용/거부 규칙 관리 (별칭 `allowed-tools`) |
| `/plan` | 플랜 모드 켜기/현재 세션 플랜 보기 |
| `/privacy-settings` | 개인정보 설정 보기 및 변경 |
| `/recap` | 지금 한 줄 세션 요약 생성 |
| `/review` `/security-review` | PR 리뷰 / 미반영 변경분 보안 검토 |
| `/status` | 버전·모델·계정·API 연결·도구 상태 표시 |

**cli-only(46개) 대표 항목** — 터미널 TUI 전용, 메뉴 하단 비활성 안내:

| 명령 | 설명 |
| --- | --- |
| `/desktop` | 세션을 Claude Desktop에서 계속 (별칭 `app`) |
| `/doctor` | Claude Code 설치·설정을 진단하고 점검 |
| `/exit` | CLI 종료 |
| `/heapdump` | JS 힙을 ~/Desktop에 덤프 |
| `/ide` | IDE 통합 관리 및 상태 표시 |
| `/keybindings` | 키 바인딩 설정 파일 열기/생성 |
| `/teleport` | claude.ai에서 Claude Code 세션 재개 (별칭 `tp`) |
| `/theme` | 테마 변경 |

> 위 표 중 cc-passthrough/cli-only 는 카탈로그의 *대표 항목*입니다. 전체 87개(web-action 8 · cc-passthrough 33 · cli-only 46)와 스킬은 `claudeCodeCommands.ts` 에서 추적하며, cc-passthrough 분류 명령과 스킬은 헤드리스 실행으로 그대로 흘려보냅니다. 회귀 잠금: `frontend/features/chat/__tests__/SlashCommandMenu.test.tsx`.

---

## 관련 문서

- [클로드 코드 모드](./claude-code-mode.md)
- [도구](./tools.md)
- [스킬 마켓플레이스](./skills-marketplace.md)
- [변경 이력](./changelog.md)
