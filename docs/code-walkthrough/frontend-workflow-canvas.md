# 워크플로 캔버스 — 2D · 3D

같은 `{nodes, edges, variables}` 데이터를 두 가지 캔버스로 렌더. 사용자는 탭으로 전환.

## 2D — Vercel Workflow Builder 풍 (`features/workflows/WorkflowCanvas.tsx`)

454줄. 외부 라이브러리 없이 SVG + div 로 자체 구현.

### 카드

```tsx
<div className="wf-card absolute select-none rounded-2xl px-4 py-3"
  style={{
    background: `linear-gradient(180deg, color-mix(in oklch, ${style.hue} 6%, var(--panel)) 0%, var(--panel) 100%)`,
    border: `1px solid ${selected ? "var(--accent)" : "var(--stroke)"}`,
    boxShadow: selected ? `0 0 0 3px color-mix(...) ...` : "0 4px 12px -8px rgba(0,0,0,0.10)",
  }}
>
```

- 듀얼톤 그라데이션 (kind hue 6% + panel) — 노드 종류를 색으로 식별.
- selected 일 때 accent glow box-shadow.
- hover 시 lift (translateY -1px).

### 엣지

```tsx
<svg>
  <path d={pathFor(e)} className="wf-edge" stroke={color} ...>
</svg>
<style>{`
  @keyframes wf-flow { to { stroke-dashoffset: -24; } }
  .wf-edge { stroke-dasharray: 6 6; animation: wf-flow 1.4s linear infinite; }
`}</style>
```

`stroke-dashoffset` 가 음수 방향으로 흐르면서 점선이 진행 방향으로 흐르는 시각 효과 — "데이터가 흐른다" 느낌.

엣지의 베지에 곡률은 가로 차이에 비례:

```tsx
const dx = Math.max(60, Math.abs(tx - sx) * 0.5);
return `M ${sx},${sy} C ${sx + dx},${sy} ${tx - dx},${ty} ${tx},${ty}`;
```

### 드래그 / 연결

- 노드 카드 mousedown → `dragRef.current = {id, dx, dy}` 저장
- 전역 mousemove 가 position 갱신 + onChange 호출
- output handle mousedown → connecting state 시작
- 다른 노드의 input handle 영역에서 mouseup → 엣지 추가

### autoLayout

```tsx
export function autoLayout(nodes, edges) {
  // 1) incoming edge 없는 노드 = depth 0
  // 2) BFS 로 depth 전파
  // 3) 같은 depth 끼리는 row 으로 적층
  // 4) (60 + d*260, 80 + row*170) 좌표
}
```

워크플로 페이지의 "자동 정렬" 버튼이 이 함수 호출.

## 3D — Three.js via CDN (`features/workflows/WorkflowCanvas3D.tsx`)

400줄+. npm 으로 three 를 설치할 수 없어 **esm.sh CDN 에서 동적 import**:

```tsx
const THREE = await import(/* webpackIgnore: true */ "https://esm.sh/three@0.169.0");
const { OrbitControls } = await import(/* webpackIgnore: true */ "https://esm.sh/three@0.169.0/examples/jsm/controls/OrbitControls.js");
const { CSS2DRenderer, CSS2DObject } = await import(/* webpackIgnore: true */ "https://esm.sh/three@0.169.0/examples/jsm/renderers/CSS2DRenderer.js");
```

`webpackIgnore: true` 가 핵심 — Next 빌드가 이 import 를 정적 분석 안 함.

### 씬 구성

- 1400개 점으로 만든 별(`Points`)
- `FogExp2(0x0b0f1e, 0.0008)` — 안개로 깊이감
- 노드는 PlaneGeometry billboard (항상 카메라 바라봄) + emissive + halo + ring
- 엣지는 `TubeGeometry` — 3D 곡선
- 엣지를 따라 3개의 traveling sphere (`Mesh` 가 t 매개변수로 곡선 위 이동)
- 노드별 bob 애니메이션: `obj.position.y = baseY + Math.sin(t * 1.4 + seed) * 5`

### 인터랙션

- OrbitControls — 회전 / 줌 / 패닝 + autoRotate
- Raycaster 로 노드 클릭 감지
- 드래그 시 카메라를 마주보는 평면(drag plane)에서 좌표 계산

### CSS2DObject 라벨

```tsx
const labelDiv = document.createElement("div");
labelDiv.className = "wf3d-label";
labelDiv.textContent = node.label;
const labelObj = new CSS2DObject(labelDiv);
group.add(labelObj);
```

3D 씬 위에 HTML 텍스트를 띄움 — 폰트가 깨끗하고 항상 카메라 정면.

## 함정·결정

- **2D 의 SVG 점선 흐름** — `stroke-dashoffset` 애니메이션이 모든 브라우저에서 부드럽게 동작. hover 시엔 정지.
- **3D 의 CDN 의존** — esm.sh 가 죽으면 3D 캔버스가 안 뜸. fallback 으로 2D 만 표시 + 토스트.
- **3D 의 webpackIgnore** — 빌드가 이 import 를 번들에 포함시키려 하면 실패. 주석을 손대지 말 것.
- **두 캔버스의 데이터 동기화** — 부모 페이지가 `nodes/edges` state 를 단일 source of truth 로 들고, 두 캔버스는 같은 props 받음. 한쪽에서 변경하면 다른 쪽도 자동 갱신.

## 관련

- 실행 엔진 — [backend-workflows](backend-workflows.md)
- /workflows/[id] 페이지 — 3D · 2D · 리스트 탭 + 노드 args 편집기
