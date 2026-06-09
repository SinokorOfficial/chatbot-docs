# Claude Code CLI 옵션 매핑 · MCP/스킬 연동

클로드 코드 모드(`/chat` direct routing)는 사용자의 요청을 **실제 `claude` CLI** 로 그대로 흘려보냅니다. 이 문서는 2026-06-09 기준 **claude CLI 2.1.169 를 실측**해 정리한 *실제 옵션 ↔ 앱 인자/페이로드* 매핑과, 앱의 MCP 서버·Skill 을 세션 스코프로 결선하는 구조를 정리합니다.

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

---

## 관련 문서

- [클로드 코드 모드](./claude-code-mode.md)
- [도구](./tools.md)
- [스킬 마켓플레이스](./skills-marketplace.md)
- [변경 이력](./changelog.md)
