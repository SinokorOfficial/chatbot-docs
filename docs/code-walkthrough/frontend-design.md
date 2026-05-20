# 디자인 시스템 — Stitch 토큰

`frontend/app/globals.css` 가 단일 source of truth. Stitch taste-design 원칙 — Off-Black + Emerald accent + whisper shadow + 큰 모서리.

## 1. 핵심 토큰 (CSS 변수)

```css
:root {
  --bg: #FAFAFA;         /* Canvas White — 절대 #FFF 아님 */
  --panel: #FFFFFF;
  --fg: #18181B;         /* Off-Black — 절대 #000 아님 */
  --muted: #71717A;
  --accent: #10B981;     /* Emerald */
  --accent-soft: rgba(16,185,129,0.10);
  --stroke: rgba(0,0,0,0.06);
  --glow-a: radial-gradient(circle, rgba(16,185,129,0.18), transparent 60%);
}

[data-theme="bmw-light"] { --bg: #F5F5F7; --accent: #1C69D4; ... }
[data-theme="apple"]     { --bg: #FBFBFD; --accent: #007AFF; ... }
[data-theme="openai"]    { --bg: #FFFFFF; --accent: #10A37F; ... }
/* ... 16개 */
```

`data-theme` attribute 한 줄이면 모든 토큰이 한꺼번에 갱신 — 페이지가 즉시 새 브랜드로 보임.

## 2. 역할별 type 스케일

```css
.type-eyebrow { font-family: var(--font-mono); font-size: 10px; letter-spacing: 0.28em; text-transform: uppercase; color: var(--muted); }
.type-display { font-family: var(--font-display); font-size: clamp(2.25rem, 5vw, 3.75rem); ... }
.type-section { font-size: 1.5rem; font-weight: 600; ... }
.type-card-title { font-size: 1rem; font-weight: 600; ... }
.type-body { font-size: 0.9375rem; ... }
.type-body-sm { font-size: 0.8125rem; ... }
.type-meta { font-size: 11px; color: var(--muted); }
```

UI 의 모든 텍스트가 7개 역할 중 하나로 분류 — 임의의 font-size 쓰지 말 것. 새 페이지를 만들 때도 이 클래스만 사용.

## 3. 카드 밀도 3단계

```css
.stitch-card          { padding: 2.5rem; border-radius: 1.5rem; box-shadow: 0 20px 40px -15px rgba(0,0,0,0.05); }
.stitch-card-compact  { padding: 1.25rem; border-radius: 1rem; box-shadow: 0 8px 20px -10px rgba(0,0,0,0.04); }
.stitch-card-micro    { padding: 0.625rem; border-radius: 0.5rem; }
```

세 가지 중 하나. 새 카드가 필요하면 새 클래스 만들기보다 셋 중 적당한 것 선택.

## 4. 인터랙션 토큰

```css
.lift              { transition: transform .2s, box-shadow .2s; }
.lift:hover        { transform: translateY(-2px); box-shadow: 0 24px 48px -16px rgba(0,0,0,0.08); }
.stagger > *       { animation: stagger-fade .4s ease both; }
.stagger > *:nth-child(2) { animation-delay: 0.05s; }
.stagger > *:nth-child(3) { animation-delay: 0.10s; }
/* ... */
.stitch-grain      { background-image: url("data:image/svg+xml,..."); }  /* 미세 노이즈 */
.stitch-dot-pulse  { animation: dot-pulse 1.6s infinite; }                /* 로고 옆 점 */
.stitch-float      { animation: float 6s ease-in-out infinite; }
```

`.lift` 는 모든 카드에 자유로이 — 작은 호버 응답성으로 살아있는 느낌.

## 5. 버튼 변형

```css
.u-btn-primary { background: var(--accent); color: #fff; padding: 0.75rem 1.5rem; border-radius: 9999px; }
.u-btn-ghost   { background: transparent; border: 1px solid var(--stroke); color: var(--fg); }
.u-input       { background: var(--panel); border: 1px solid var(--stroke); border-radius: 0.75rem; ... }
```

primary 는 accent fill, ghost 는 stroke 만, input 은 panel + 미세한 stroke. 색은 항상 토큰을 거침.

## 6. 절대 금지 패턴

- **`#000` / `#fff` 직접 사용** — 절대 안 됨. `var(--fg)` / `var(--panel)` 만.
- **임의의 box-shadow** — 위의 카드 클래스만. 새 그림자가 필요하면 globals.css 에 추가.
- **neon glow** — Stitch 의 미적 대척점. 사용 금지.
- **Inter 폰트** — banned. Geist 만.
- **임의의 border-radius** — 4px / 8px / 12px / 16px / 24px (1.5rem) / 9999px (pill) 만.

## 7. 폰트

```ts
// app/layout.tsx
import { Geist, Geist_Mono } from "next/font/google";
const geist = Geist({ subsets: ["latin"], variable: "--font-display" });
const geistMono = Geist_Mono({ subsets: ["latin"], variable: "--font-mono" });
```

display = body = Geist Sans. mono 는 Geist Mono. 한국어는 시스템 폰트 fallback — 추후 Pretendard 도입 검토.

## 8. 함정·결정

- **테마는 토큰만 갈아끼움** — class 가 아니라 root attribute 로 — 컴포넌트 변경 없이 모든 화면이 따라옴.
- **defaultTokens 백업** — 새 테마가 토큰을 누락해도 [frontend-theme](frontend-theme.md) 의 mergeTokens 가 채움.
- **dark/light 어디 갔나** — `light` / `dark` 토글은 별도로 없음. 각 테마가 자기 라이트/다크 버전을 갖도록 분리 (예: bmw-light / bmw-dark).

## 관련

- [frontend-theme](frontend-theme.md) — 라디얼 피커와 토큰 주입
- 컴포넌트 카탈로그 — `frontend/shared/ui/` (Alert, EmptyState, PageHeader, BackButton, …)
