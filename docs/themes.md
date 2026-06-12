# 테마 시스템 — 브랜드별 design.md

> 챗봇 UI 가 사용자/팀이 선택한 브랜드 테마에 맞춰 동적으로 색·폰트·곡률·모션을 바꿉니다.
> 출처: [getdesign.md](https://getdesign.md/design-md) (PlayStation, BMW, Apple, GitHub, Stripe 등 30+ 컬렉션).

## 데이터 모델

### `themes` 테이블
| 컬럼 | 설명 |
|---|---|
| `slug` | URL-safe 식별자 (예: `playstation`, `bmw`) |
| `name`, `brand`, `category` | 표시명/브랜드명/카테고리(`gaming` `automotive` `luxury-tech` 등) |
| `tokens` (JSONB) | 디자인 토큰. CSS 변수로 매핑됨 |
| `content_md` | design.md 마크다운 본문 (LLM 참조용) |
| `preview_image_url` | 카드 미리보기 |
| `source` | `seed` / `getdesign` / `user` |

### `users.active_theme_id` (FK → themes.id)
- 사용자별 활성 테마 (없으면 `default`)

## 디자인 토큰 표준 (확장)

`backend/config/themes.yaml` 의 각 브랜드는 다음 트리에서 **의미 있는 키만 명시**. 빠진 키는 `frontend/shared/theme/defaultTokens.ts` 의 표준 기본값으로 자동 보충 (`mergeTokens`).

```yaml
tokens:
  colors:
    bg                 # 페이지 배경
    surface            # 카드/컨테이너
    surface_strong     # 강조 카드 (세 표면 채널의 최상위)
    primary            # 핵심 액션
    primary_hover
    accent             # 보조 액센트 (예: BMW M Red)
    accent_secondary   # 추가 보조
    text / text_muted
    border / border_strong
    focus_ring         # 포커스 링 (rgba 권장)
    success / warning / error / info  # semantic
  typography:
    font_sans / font_mono / font_display
    size_xs / sm / base / lg / xl / 2xl
    tracking_tight / tracking_normal
    line_height_tight / normal / relaxed
  radius:
    sm / md / lg / full
  shadow:
    sm / md / lg       # elevation
  motion:
    duration_fast / duration / duration_slow
    easing             # 기본 cubic-bezier
    easing_bouncy      # 바운스 (선택)
    hover_scale
  spacing:
    xs / sm / md / lg / xl     # 4-32px 그리드
```

### 브랜드별 design.md 주요 특징의 토큰 매핑

| 브랜드 | design.md 핵심 | 적용 토큰 |
|---|---|---|
| BMW | 예각 강조 | `radius: {sm:0, md:2, lg:4}` |
| BMW | M Sport Red | `accent: #e22718` |
| BMW | 절제된 모션 | `duration:150ms`, ease-out-quad |
| PlayStation | 세 표면 채널 | `surface` + `surface_strong` 두 단계 |
| PlayStation | 청록 호버 스케일 | `accent:#00c8d4`, `hover_scale:1.05` |
| PlayStation | 청록 글로우 | `shadow.lg: 0 24px 60px rgba(0,200,212,0.15)` |
| Apple | 큰 여백 + 우아한 transition | `size_base:17px`, `duration:300ms` |
| Apple | 부드러운 곡선 | `radius: {sm:8, md:12, lg:18}` |
| Apple | iCloud 보라 보조 | `accent_secondary: #bf5af2` |

## 시드된 테마 (현재 15개)

| slug | 브랜드 | 카테고리 | 특징 |
|---|---|---|---|
| `default` | 장금상선 | corporate | Pretendard, 청량 블루 |
| `playstation` | PlayStation | gaming | 세 표면, 청록 호버, 다크 |
| `bmw` | BMW Group | automotive | 예각 모서리, M Blue + M Red, 0-4px radius |
| `apple` | Apple | luxury-tech | 큰 여백, SF Pro, 부드러운 곡선 |
| `github` | GitHub | developer-tools | 다크 우선, Primer, 모노스페이스 강조 |
| `stripe` | Stripe | fintech | 네이비 + 보라, 그라데이션 |
| `linear` | Linear | productivity-saas | — |
| `vercel` | Vercel | developer-tools | — |
| `openai` | OpenAI | ai-platform | — |
| `claude` | Anthropic | ai-platform | — |
| `nike` | Nike | lifestyle-sport | — |
| `spotify` | Spotify | media-consumer | — |
| `tesla` | Tesla | automotive | — |
| `airbnb` | Airbnb | lifestyle-consumer | — |
| `hyundai` | Hyundai | automotive | — |

개수·세부 토큰은 변동이 잦으므로 단일 출처인 리포 내 `backend/config/themes.yaml` 을 기준으로 한다.

## 프론트 적용 방법

### 1) 부팅 시 활성 테마 fetch
```ts
const { theme } = await fetch("/themes/active").then(r => r.json());
if (theme) applyTokens(theme.tokens);
```

### 2) CSS 변수 주입
```ts
function applyTokens(tokens: ThemeTokens) {
 const root = document.documentElement.style;
 for (const [group, vars] of Object.entries(tokens)) {
 for (const [k, v] of Object.entries(vars)) {
 root.setProperty(`--${group}-${k}`, String(v));
 }
 }
}
```

### 3) 사용자가 테마 변경
```ts
await fetch("/themes/active", {
 method: "POST",
 headers: { "Content-Type": "application/json" },
 body: JSON.stringify({ theme_id: selectedId }),
});
applyTokens(selectedTokens); // 즉시 반영
```

### 4) Tailwind 와 함께
`tailwind.config.ts` 에서 `colors.primary: 'var(--colors-primary)'` 형태로 CSS 변수 referenced.

## API

```
GET /themes # 카드 리스트 (preview 용)
GET /themes/active # 사용자 현재 테마
POST /themes/active # 활성 테마 지정 { theme_id }
GET /themes/{slug} # 상세 (tokens + content_md)
POST /themes # 사용자 직접 등록 (slug 중복 금지)
```

## design.md 표준

```markdown
# {Brand} Design

## 레이아웃
{몇 개 영역, 어떤 비율, 강조 패턴}

## 타이포그래피
{폰트 패밀리, 크기 스케일, 위계}

## 컬러
{핵심 컬러 + 액센트}

## 인터랙션
{호버, 트랜지션, 듀레이션}
```

이 마크다운은 챗봇 (디자인 어시스턴트) 가 참조해 "BMW 스타일로 카드를 디자인해줘" 같은 요청에 답합니다.

## 향후

- [ ] getdesign.md 사이트 정기 크롤러 (별도 워커, `source=getdesign`)
- [ ] 사용자 업로드 테마 (스크린샷/로고에서 자동 토큰 추출)
- [ ] 팀 단위 기본 테마 강제 (admin 설정)
- [ ] 테마 미리보기 UI (사이드바에 카드)
