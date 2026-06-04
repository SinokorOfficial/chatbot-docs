# Design Tokens & 변화 내성 룰

> **철학**: *"색은 바뀔 수 있지만 UX는 절대 무너지지 않는다."*
> 컴포넌트는 어떤 accent 색(밝은 노란색 / 어두운 남색 / 회색 / 채도 0)이 들어와도
> **텍스트 가독성 · 버튼 인지성 · hover/active/disabled/focus 구분**이 유지되어야 한다.

본 문서는 *추가하지 말아야 할 것*을 정의한다. 즉 안티패턴 카탈로그 + 토큰 매트릭스 +
컴포넌트 contract 다.

---

## 1. 토큰 매트릭스 (frontend/app/globals.css)

토큰은 **3 레이어**로 정의된다.

### Layer 1 — 브랜드/표면 토큰 (테마별 다름, 16개 테마)

| 토큰 | 의미 | 예시 (light / janggeum / mono) |
|---|---|---|
| `--bg` | 페이지 최하단 배경 | `#F9FAFB` / `#071224` / `#09090b` |
| `--fg` | 본문 텍스트 | `#18181B` / `#edf2f8` / `#fafafa` |
| `--muted` | 보조 텍스트 | rgba(fg, 0.52) |
| `--surface` | 카드/패널 배경 | `#ffffff` / rgba(4,12,30,0.5) / rgba(0,0,0,0.42) |
| `--surface-muted` | 옅은 강조면 | `#F4F4F5` |
| `--stroke` | 보더/구분선 | rgba(...) |
| `--accent` | 브랜드 강조색 | `#10B981` / `#f0a030` / **`#e4e4e7`** |
| `--accent-fg` | accent 위 텍스트 | `#FFFFFF` / `#0c1020` / `#09090b` |
| `--accent-soft` | accent 12~18% 알파 | 자동 |
| `--accent-2` / `-rgb` | 보조 강조색 | |

> ⚠️ **mono 테마 `--accent: #e4e4e7`** (밝은 회색) 가 *변화 내성 테스트 케이스*다.
> 이 테마에서 `text-white` 를 쓰면 텍스트가 *전혀 보이지 않는다*. 새 컴포넌트는
> 반드시 `var(--accent-fg)` 를 사용 — `text-white` 금지.

### Layer 2 — Interaction state 토큰 (테마 공유, 대비 기반)

```css
--state-hover-bg     : color-mix(in oklch, var(--fg) 6%,  transparent);
--state-pressed-bg   : color-mix(in oklch, var(--fg) 10%, transparent);
--state-active-bg    : color-mix(in oklch, var(--accent) 14%, transparent);
--state-selected-bg  : color-mix(in oklch, var(--accent) 18%, transparent);
--state-disabled-bg  : color-mix(in oklch, var(--fg) 4%,  transparent);
--state-disabled-fg  : color-mix(in oklch, var(--fg) 38%, transparent);

--state-active-indicator   : var(--accent);
--state-active-indicator-w : 3px;

--focus-ring-color  : color-mix(in oklch, var(--accent) 70%, var(--fg) 30%);
--focus-ring-width  : 2px;
--focus-ring-offset : 2px;
```

이 토큰들은 *accent 색에 의존하지 않는다*. hover/pressed/disabled 는 `--fg`(텍스트색)
기준이라 *어떤 테마, 어떤 명도, 어떤 채도* 에서도 동일한 효과를 낸다.

### Layer 3 — Semantic status 토큰 (테마 공유, accent 와 *완전 독립*)

```css
--success / --success-fg / --success-soft / --success-border
--warning / --warning-fg / --warning-soft / --warning-border
--danger  / --danger-fg  / --danger-soft  / --danger-border
--info    / --info-fg    / --info-soft    / --info-border
```

브랜드 색이 바뀌어도 **의미 색은 안정**. 즉 "삭제 버튼은 빨강" 이라는 *학습된 의미*가
새 테마로 전환되어도 유지된다.

---

## 2. 컴포넌트 사용 룰

### Button — `frontend/shared/ui/Button.tsx`

| variant | 의미 | 토큰 |
|---|---|---|
| `primary` | 페이지의 **하나뿐인** 주요 CTA | `bg: var(--accent)` + `color: var(--accent-fg)` |
| `ghost` | 보조 | 배경 transparent + border `var(--stroke)` |
| `soft` | 강조 약함 | `bg: var(--accent-soft)` + `color: var(--accent)` |
| `destructive` | 위험한 액션 | `var(--danger)` 계열 |

룰:
- 한 페이지에 `primary` 는 **1개 또는 0개**. 페이지에 2개면 사용자 시선 분산.
- size 는 페이지 단위 통일 — 폼 `md`, 카드 내 `sm`, 랜딩 `lg`.

### Card — `frontend/shared/ui/Card.tsx`

3 density: `default(40px radius, big padding)` / `compact(small padding)` / `micro`.
**같은 그리드 안에 density 혼합 금지**. 그리드별로 하나만.

### Badge — `frontend/shared/ui/Badge.tsx`

Tone 6개: `neutral / accent / success / warning / danger / info`.

**금지**: `bg-emerald-*`, `text-rose-*` 같은 tailwind 직접 색.
**필수**: `<Badge tone="success">활성</Badge>` 형태.

dot prop: 색 외 *시각 시그널*. 색맹/색약 사용자 대응. status 표시에 적극 사용.

### NavMenu / CommandPalette — active/selected

active 상태는 **4개 시그널 조합**으로 표현 (한 가지로 표현 금지):

1. 배경 `var(--state-active-bg)` 또는 `var(--state-selected-bg)`
2. 텍스트색 `var(--accent)`
3. 폰트 굵기 `font-semibold`
4. 좌측 막대 또는 작은 dot (`u-active-indicator` 클래스)
5. ARIA — `aria-current="page"` (NavMenu) / `aria-selected="true"` (CommandPalette)

→ accent 가 회색에 가까워도 *굵기 + 좌측 막대 + ARIA* 가 active 를 알린다.

---

## 3. 안티패턴 카탈로그 (PR에서 즉시 거절)

| 안티패턴 | 왜 나쁜가 | 대체 |
|---|---|---|
| `text-white` 고정 | mono 테마(accent=#e4e4e7)에서 안 보임 | `text-[var(--accent-fg)]` |
| `bg-emerald-500`, `text-rose-300` 등 tailwind 직접 색 | semantic 의미 안 통함, 테마 안정성 X | `var(--success)` / `<Badge tone>` |
| 색만으로 상태 표현 (예: 빨간 글자로 에러) | 색맹/색약·저대비 사용자 차단 | 색 + 아이콘 + 굵기 |
| hover 색 임의 지정 (예: `hover:bg-emerald-500/20`) | accent 와 무관해짐 | `hover:bg-[var(--state-hover-bg)]` |
| `dark:` 별도 분기 땜질 | 16 테마라 곱셈 폭발 | 토큰 시스템에 위임, `dark:` 사용 금지 |
| disabled 를 `opacity-50` 만으로 | 텍스트 가독성 더 망가짐 | `var(--state-disabled-bg)` + `var(--state-disabled-fg)` |
| inline hex 컬러 (`color: '#10B981'`) | 그레핑 어려움, 테마 무관 | CSS 변수 |
| `focus:outline-none` | 키보드 사용자 차단 | 전역 `*:focus-visible` 토큰에 위임 |

---

## 4. Sidebar / Accordion 품질 기준

체크리스트 — 모든 사이드바/아코디언이 통과해야 한다:

- [ ] 긴 메뉴명: `truncate` + `title=` 속성 (또는 tooltip)
- [ ] collapsed 상태: 아이콘만 + tooltip 제공
- [ ] active 메뉴: **색만 의존 금지** — 좌측 막대 또는 굵기 동반 (`u-active-indicator`)
- [ ] accordion open/close: layout shift 최소화 (`grid-template-rows: 0fr → 1fr` 트랜지션 권장)
- [ ] 사이드바 자체 스크롤: `overflow-y: auto; max-h: 100dvh`
- [ ] viewport 보다 길어도 page content 가 *밀리지 않음* (position: sticky + 자체 스크롤)
- [ ] 모바일 drawer: backdrop + ESC close + focus trap (`<dialog>` 또는 `react-aria` 활용)
- [ ] 키보드: Tab/Shift+Tab 으로 모든 항목 도달, Enter/Space 로 toggle

---

## 5. 반응형 검증 매트릭스

| width | 검증 항목 |
|---|---|
| 320px | 가로 스크롤 0, 카드 1열, 메뉴 hamburger |
| 375px | NavMenu hamburger, ⌘K 풀스크린, 챗 입력 가능 |
| 768px | 카드 2열, NavMenu 인라인, ⌘K 모달 560px |
| 1024px | 카드 3열, 사이드바 240px 고정 |
| 1440px | 사이드바 280px, 메인 max-w 7xl |
| 1920px+ | 메인 중앙 정렬, 좌우 여백 확장 |

zoom 200%: 320×2 = 640px 와 같은 거동. 즉 320 통과하면 200% 도 안전.

---

## 6. 비정상 상태 UX (반드시 모두 구현)

| 상태 | 구현 위치 | 토큰/컴포넌트 |
|---|---|---|
| loading | `stitch-skeleton` 클래스 + spinner | `--state-disabled-bg` 위에 shimmer |
| skeleton | 카드 그리드 첫 진입 시 3개 placeholder | `stitch-skeleton` |
| empty | `EmptyState` / `EntryMissionBar` | dashed `--stroke` + 액션 CTA |
| error | `Alert variant="error"` | `var(--danger-soft)` + `var(--danger)` 텍스트 + 아이콘 |
| retry | error 안에 "다시 시도" 버튼 | `variant="ghost"` |
| permission denied | *항목 숨김* — 클릭 후 403 모달 금지 | AvatarMenu canManageTeam 분기 |
| long text | `truncate` + tooltip / `line-clamp-N` + `min-h` | — |
| no image | `<svg>` placeholder + alt 텍스트 | — |
| broken image | `onError` 핸들러 → placeholder | — |
| slow network | SSE 사용 + 진행 dot pulse (`stitch-dot-pulse`) | — |
| rapid click | 버튼 `disabled` 동안 — `loading` prop | `Button.loading` |
| resize 중 | container query 권장 (`@container`) | — |

---

## 7. 접근성 (반드시)

- [ ] 모든 인터랙티브 요소에 `:focus-visible` 자동 적용 (`*:focus-visible` 전역 룰)
- [ ] NavMenu 그룹 버튼 — `aria-haspopup`, `aria-expanded`, `aria-current` ✅
- [ ] CommandPalette — `role="dialog" aria-modal aria-label`, 내부 `role="listbox"` + `aria-selected` ✅
- [ ] AvatarMenu — `role="menu" role="menuitem"`
- [ ] ESC 가 모든 모달/메뉴/팝오버를 닫음 ✅
- [ ] Tab 순서 보장 — `tabindex` 음수 사용 신중히
- [ ] screen reader: `aria-hidden="true"` 는 *의미 없는 아이콘에만*
- [ ] 색 + 텍스트 + 아이콘 3중 시그널 (예: error 는 빨강 + ⚠️ + "오류")

---

## 8. CTO 최종 판단 — YES 9개 검증

각 항목은 *실측 가능한 기준*과 *현재 상태*를 같이 적었다.

| # | 질문 | 기준 | 현재 |
|---|---|---|---|
| 1 | 색상 팔레트가 바뀌어도 UI가 깨지지 않는가? | 16 테마 전부 NavMenu/Button/Card 시각 일관 | ✅ (`text-white` 6곳 제거, `var(--accent-fg)` 로 대체) |
| 2 | primary 색이 밝아도 버튼 텍스트가 읽히는가? | mono 테마 (#e4e4e7) 에서 primary CTA 가독 | ✅ (`text-[var(--accent-fg)]` 자동 분기) |
| 3 | active 메뉴가 색 없이도 구분되는가? | NavMenu/CommandPalette 모두 4 시그널 | ✅ (배경+굵기+텍스트색+dot/막대+ARIA) |
| 4 | hover 상태가 모든 배경에서 보이는가? | `--state-hover-bg` = fg 6% — 다크/라이트 동일 효과 | ✅ |
| 5 | 모바일에서 sidebar 가 drawer 로 자연스럽게 전환? | 768px 미만 drawer + backdrop + focus trap | 🟡 (drawer 구현은 있으나 focus trap 검증 필요) |
| 6 | 긴 텍스트가 레이아웃을 부수지 않는가? | 30자 한국어 챗봇명 카드 헤더 정상 | 🟡 (`chatbots:265` line-clamp-2 + min-h 안전 패턴 — 단 카드 헤더 truncate 확정 PR 필요) |
| 7 | zoom 200% 에서도 사용 가능한가? | 320px 통과 + 키보드 흐름 정상 | 🟡 (Playwright e2e 0건 — 자동 검증 부재) |
| 8 | 메뉴가 5배 늘어나도 구조가 유지되는가? | NavMenu 그룹 룰 + `docs/ia-rules.md` 5단계 체크리스트 | ✅ (`docs/ia-rules.md` 참조) |
| 9 | 신규 개발자가 token 구조만 보고 확장할 수 있는가? | 본 문서 + 컴포넌트 contracts | ✅ (본 문서) |

🟡 항목은 후속 PR 에서 닫는다.

---

## 9. 변경 이력

- **2026-05-22** — Layer 2 (interaction state) + Layer 3 (semantic status) 토큰 추가. `text-white` 하드코딩 6곳 → `var(--accent-fg)`. `Badge.tsx` 토큰화 (emerald/amber/rose → success/warning/danger/info). NavMenu / CommandPalette active/selected 시그널을 *색만* → *bg + 굵기 + dot/막대 + ARIA* 4중 시그널로. `*:focus-visible` 전역 룰을 토큰 기반(`--focus-ring-color`) 으로 재정의.
- **2026-05-21** — `docs/ia-rules.md` (메뉴 추가 가드), `docs/audit-cto.md` (Product/IA/UX 감사).
