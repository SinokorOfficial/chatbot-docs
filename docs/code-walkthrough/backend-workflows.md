# workflows — 노드/엣지 + 실행 엔진 + NL 생성

워크플로는 **노드 그래프** 형태의 자동화. 트리거 → 도구/LLM/HTTP/변환 노드를 엣지로 연결하면, 한 번 실행할 때마다 WorkflowRun 한 row 가 생성되며 각 노드의 입출력이 JSONB 로 기록됩니다.

## 1. 라우터 (`features/workflows/router.py`)

| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/workflows` | 본인/팀/public 가시 |
| POST | `/workflows` | 새 워크플로 |
| GET / PATCH / DELETE | `/workflows/{id}` | 개별 CRUD |
| POST | `/workflows/{id}/run` | 실행 → WorkflowRun row 반환 |
| GET | `/workflows/{id}/runs` | 실행 이력 |
| POST | `/workflows/from-prompt` | 자연어 → 워크플로 자동 생성 |

`POST /workflows/{id}/run` 은 `input_args` 받아 `workflow_engine.execute_workflow` 호출, 결과는 동기 반환(짧은 워크플로 가정). 긴 워크플로면 백그라운드 + 폴링으로 전환 가능.

## 2. 실행 엔진 (`services/workflow_engine.py`)

### 2-1. 토폴로지 정렬 (`workflow_engine.py:54-79`)

```python
def _topo_sort(nodes: list[dict], edges: list[dict]) -> list[str]:
    incoming: dict[str, set[str]] = {nid: set() for nid in by_id}
    children: dict[str, list[dict]] = {nid: [] for nid in by_id}
    for e in edges:
        incoming[b].add(a)
        children[a].append(e)
    # Kahn 알고리즘
    roots = [nid for nid, ins in incoming.items() if not ins]
    while roots:
        n = roots.pop(0)
        order.append(n)
        for e in children[n]:
            incoming[t].discard(n)
            if not incoming[t]:
                roots.append(t)
    if len(order) != len(by_id):
        raise ValueError("워크플로에 순환 의존성이 있어요.")
```

Kahn 알고리즘 — cycle 이면 ValueError. UI 가 노드 드래그 시 cycle 만들지 못하게 frontend 도 가드하지만, 백엔드가 신뢰 경계.

### 2-2. 변수 치환 (`workflow_engine.py` `_resolve` + `VAR_RE`)

```python
VAR_RE = re.compile(r"\{\{\s*([a-zA-Z0-9_\-.]+)\s*\}\}")

def _resolve(value, ctx):
    if isinstance(value, str):
        # {{node_id.field.subfield}} → ctx["node_id"]["field"]["subfield"]
        ...
    if isinstance(value, dict):
        return {k: _resolve(v, ctx) for k, v in value.items()}
    if isinstance(value, list):
        return [_resolve(v, ctx) for v in value]
    return value
```

dict / list 재귀라 nested args 안의 변수도 치환됨. `{{run.input.X}}` 으로 실행 시 input_args 접근.

### 2-3. 노드 디스패치

```python
async def _exec_node(node, ctx, ws):
    kind = node["kind"]
    args = _resolve(node.get("args") or {}, ctx)
    if kind == "trigger":
        return ctx.get("run", {}).get("input", {})
    if kind == "tool":
        return await tool_dispatch(args["slug"], args.get("input") or {}, tool_ctx)
    if kind == "llm":
        # LiteLLM acompletion + 메시지 조립
        ...
    if kind == "http":
        async with httpx.AsyncClient() as c:
            r = await c.request(args["method"], args["url"], json=args.get("body"), headers=args.get("headers"))
            return {"status": r.status_code, "body": r.json() if r.headers.get("content-type","").startswith("application/json") else r.text}
    if kind == "transform":
        # 매우 제한된 eval — 안전 컨텍스트 (numbers/strings/dict/list 만)
        return _safe_eval(args["expr"], ctx)
```

### 2-4. on_error 분기

노드가 예외를 던지면 status='err' 로 기록되고, 그 노드를 부모로 두는 엣지 중 `when='on_error'` 만 활성. `when='on_success'` 와 `when='always'` 는 자식 자손까지 skip 처리.

이 덕분에 "도구 실패 시 대체 경로" 같은 패턴이 단일 워크플로 안에서 표현됨.

## 3. 자연어 → 워크플로 (`services/workflow_nl.py`)

```python
async def generate_workflow_from_prompt(prompt: str, available_tool_slugs: list[str]) -> dict:
    system = SYSTEM_PROMPT_TEMPLATE.format(
        tools="\n".join(f"- {s}" for s in available_tool_slugs),
    )
    resp = await litellm.acompletion(
        model="openai/gpt-4o-mini",
        messages=[{"role": "system", "content": system}, {"role": "user", "content": prompt}],
        response_format={"type": "json_object"},  # strict JSON
    )
    data = json.loads(resp.choices[0].message.content)
    _validate_shape(data)
    return data
```

핵심 트릭:
1. **available_tool_slugs 를 DB 에서 동적으로 주입** — 시스템 프롬프트가 "현재 등록된 도구만" 사용하도록 가이드
2. **`response_format={"type": "json_object"}`** — OpenAI structured outputs 로 JSON 강제
3. `_validate_shape` 가 nodes/edges 형식 검증 후 라우터가 그대로 `Workflow.create`

UI 의 "AI 로 만들기" 버튼 (`frontend/app/(workspace)/workflows/page.tsx`) 가 이 엔드포인트 호출.

## 4. 프론트 캔버스와의 관계

워크플로 데이터는 `{nodes, edges, variables}` 형태로 100% 동일하게 프론트도 들고 있음. 프론트의 `WorkflowCanvas` (2D) 와 `WorkflowCanvas3D` 는 같은 객체를 다른 방식으로 렌더 — 자세히는 [frontend-workflow-canvas.md](frontend-workflow-canvas.md).

## 5. 함정·결정

- **WorkflowRun.node_outputs 의 누적 크기 주의** — 큰 페이로드(예: LLM 응답 전체)를 노드 출력으로 두면 JSONB row 가 커짐. UI 표시용은 truncate 권장.
- **transform 의 eval** — 안전 컨텍스트로 좁혀 있지만 `__import__` 같은 우회를 막기 위해 dict literal + 산술/문자열 메서드만 허용. 새 키워드 추가는 보안 리뷰 필수.
- **자동 생성 워크플로의 검증** — `_validate_shape` 가 형식만 체크, **의미는 검증 안 함**. LLM 이 존재하지 않는 도구 슬러그를 만들면 실행 시 실패. UI 가 만들고 나서 사용자가 한 번 확인할 수 있게 자동 실행 X, 편집 화면으로 진입.

## 관련 문서

- [backend-misc](backend-misc.md) — `tool_dispatch` 가 실제로 무엇을 호출하는지
- [frontend-workflow-canvas](frontend-workflow-canvas.md) — 2D/3D 캔버스
