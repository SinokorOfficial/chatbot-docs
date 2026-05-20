# 테마 시스템 — 16 브랜드 + 라디얼 피커

## 1. 큰 그림

```
backend/config/themes.yaml      ← 16 브랜드 시드 (BMW, Apple, OpenAI, Notion, …)
   ↓ services/seed_loader.run_all_seeds()
Theme 테이블                    ← name · slug · tokens(JSONB) · category
   ↓ GET /themes
프론트
  BrandThemeProvider            ← 마운트 시 localStorage → :root data-theme
  applyTokensInline(tokens)     ← CSS 변수 직접 주입 (호버 미리보기)
  RadialThemePicker             ← 16개를 원 위에 배치, 휠 스크롤 + 라이브 프리뷰
  BrandThemeGallery             ← 헤더의 단일 트리거 (라디얼 진입)
  defaultTokens.ts              ← 누락 토큰의 표준 기본값
```

## 2. ThemeProvider 가 하는 일

```tsx
// shared/theme/BrandThemeProvider.tsx
useEffect(() => {
  const saved = localStorage.getItem("brand-theme");
  if (saved) document.documentElement.setAttribute("data-theme", saved);
}, []);

export function applyTokensInline(rawTokens: ThemeTokens) {
  const merged = mergeTokens(defaultTokens, rawTokens);
  for (const [key, value] of Object.entries(merged)) {
    document.documentElement.style.setProperty(key, value);
  }
}
```

`data-theme` attribute 가 바뀌면 `globals.css` 의 CSS 변수 묶음이 바뀌어 모든 화면이 즉시 갱신. `applyTokensInline` 은 **persistence 없이** root 의 inline style 만 변경 — 호버 미리보기 용.

## 3. RadialThemePicker

```tsx
// shared/theme/RadialThemePicker.tsx
const RADIUS = 220;
themes.map((t, i) => {
  const angle = (i / themes.length) * Math.PI * 2 - Math.PI / 2;
  const x = Math.cos(angle) * RADIUS;
  const y = Math.sin(angle) * RADIUS;
  return <div style={{ transform: `translate(${x}px, ${y}px)`, background: t.color }} />;
})
```

원 위에 16개 점을 배치. 현재 focus 된 점은 중앙에 더 크게 표시 + 이름.

### 휠 스크롤로 회전

```tsx
const wheelAccumRef = useRef(0);
const THRESHOLD = 30;
const onWheel = (e: WheelEvent) => {
  e.preventDefault();
  wheelAccumRef.current += e.deltaY;
  while (Math.abs(wheelAccumRef.current) >= THRESHOLD) {
    const dir = wheelAccumRef.current > 0 ? 1 : -1;
    wheelAccumRef.current -= dir * THRESHOLD;
    setFocusIdx(i => dir > 0 ? (i + 1) % themes.length : (i - 1 + themes.length) % themes.length);
  }
};
```

스크롤 휠을 그대로 받지 않고 누적 → 일정 임계값을 넘어야 다음 테마로. 부드럽고 의도된 속도.

### 호버 시 즉시 미리보기

```tsx
const onPointerEnter = (theme: Theme) => {
  if (theme.tokens) applyTokensInline(theme.tokens);  // 즉시 모든 화면이 그 색으로
};
const onPointerLeave = () => {
  // 원래 selected theme 으로 복귀
  const sel = themes.find(t => t.slug === selectedSlug);
  if (sel?.tokens) applyTokensInline(sel.tokens);
};
```

선택을 안 해도 마우스가 점에 닿는 순간 전체 페이지 색이 바뀜. 라이브 프리뷰.

### Portal 마운트

```tsx
return createPortal(<div className="fixed inset-0 ...">...</div>, document.body);
```

containing block stacking context 를 피하려고 body 에 직접 마운트. 헤더의 z-index 와 충돌 안 함.

## 4. 명도 대비 가드

테마의 accent 색이 라이트 모드에선 흰 위 흰색이 될 수 있음 — 점 주변의 ring 색이 바탕과 안 보이는 사고를 막기 위해:

```tsx
function ringColorFor(token: string): string {
  // 토큰을 HSL 로 파싱, lightness > 65 면 dark ring, else light ring
  const lum = luminance(token);
  return lum > 0.55 ? "rgba(0,0,0,0.45)" : "rgba(255,255,255,0.7)";
}
```

라이트 테마 (Apple, Notion light, BMW Whisper)에선 검은 ring, 다크 테마에선 흰 ring.

## 5. defaultTokens 의 역할

`shared/theme/defaultTokens.ts` 는 표준 24개 토큰의 안전 기본값. `mergeTokens(defaultTokens, themeTokens)` 가 누락된 토큰을 자동 채워 — 새 테마가 일부만 명시해도 어색해지지 않음.

예: BMW 가 `--accent-soft` 를 명시 안 하면 defaultTokens 의 회색이 자동 적용.

## 6. 함정·결정

- **opacity 없는 accent-soft 의 위험** — 이전 버전이 `var(--accent-soft)` 를 활성 nav 배경으로 썼는데, 일부 테마가 불투명색을 줘서 글자가 사라짐. 지금은 [AppShell](frontend-shell.md) 의 linkCls 가 `color-mix(in oklch, var(--accent) 14%, transparent)` 인라인으로 강제.
- **localStorage vs DB 저장** — 현재는 localStorage 만. 로그인하면 백엔드의 User.preferred_theme 으로 옮겨야 다른 PC 에서도 동일 — 미구현.
- **휠 임계값 30** — 작은 마우스휠은 e.deltaY ≈ 100, 트랙패드는 ≈ 5. 30 이면 둘 다 자연스러움.
