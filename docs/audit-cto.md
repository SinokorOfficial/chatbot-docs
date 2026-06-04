# CTO Audit — chatbot-cleanup (2026-05-21)

> "이 기능은 왜 여기 있음?" "이 메뉴는 사용자 입장에서 의미가 없음." "이건 내부 개발 구조가 그대로 노출된 것."
> 본 문서는 **PR 리뷰 톤**으로 작성한다. 좋은 말은 마지막 페이지에 한 줄로만 쓴다.

## 0. 현재 문제 진단 — 한 문장

**기능은 있다. 구조가 없다.** 13개 라우트가 평면으로 늘어선 시기를 거쳐 최근 2그룹 IA + ⌘K + MissionHub까지 들어왔지만, *백엔드 모델 경계*가 그대로 *사용자 표면 경계*가 된 흔적이 여전히 곳곳에 남아있다(공지·Q&A·FAQ 3중 구조, 도구 vs 스킬 마켓 의미 충돌, 스케줄 vs 워크플로 자동화 경로 분기). 그 결과 — *기능 발견성은 떨어지고*, *어디서 시작하든 다음 액션을 모르는 페이지가 6개 이상이며*, *온보딩은 `/chat` 한 곳에만 살아있다*.

---

## 1. Product Architecture Audit

### 1-1. PR 리뷰 톤의 비판 17개

각 항목은 "이건 왜 이러는가" 형식.

#### A. 중복·충돌

1. **`/notices` `/qa` `/faq` 가 동시에 살아있는 이유는?**
   - 같은 멘탈 모델("관리자에게 묻거나 / 동료에게 묻거나 / 봇에게 묻거나")의 *기술 구분*. 백엔드 라우터 3개(`notices`, `faq_posts`, `faqs`) + 모델 3개(`Notice`, `FaqPost`+`FaqComment`, `Faq`)가 그대로 메뉴 3개로 노출됨. 사용자는 차이를 모른다.
   - **결정**: 이미 NavMenu에서는 제거하고 AvatarMenu "도움 & 커뮤니티"로 묶음 ✅. 그러나 **URL은 여전히 분리**. 다음 단계: `/help` 단일 hub 페이지(`tabs={notices, qa, faq}`) 신설 → 기존 라우트는 deep-link로 유지.
   - 백엔드는 그대로 둔다 — 모델 머지는 비용 대비 효익 낮음.

2. **"스킬 마켓"(`/skills`)과 "도구"(`/tools`)는 둘 다 "마켓"으로 보임.**
   - `Skill` = 프롬프트/로직 패턴, `Tool` = 외부 API 연결(HTTP/OAuth/MCP). 의미가 완전히 다르지만 사용자는 "재사용 가능한 무언가의 카탈로그" 로 동일 인지.
   - **결정**: `/tools` 는 이미 Surface 제거(⌘K "고급"로) ✅. 챗봇 빌더(`/chatbots/[id]`)에서 *컨텍스트 진입*만 허용. `/skills` 는 "마켓"으로 라벨 단일화 ✅.

3. **`/schedules` 가 독립 메뉴여야 했던가?**
   - "정해진 시간에 실행"은 워크플로의 *트리거 변형*일 뿐. UI 모델이 다른 건(자연어 입력 vs 노드 캔버스) *구현 디테일*이지 사용자 분류 기준이 아니다.
   - **결정**: NavMenu에서 "자동화" 그룹 안에 같이 묶음 ✅. 다음 단계: `/workflows/new?trigger=schedule` deep-link 통합 — `/schedules` 자체는 *목록 뷰*로만 남기고 생성 흐름은 워크플로로 합류.

4. **`/admin` 과 `/admin/metrics` 의 분리 — 사용자 구분인가, 권한 구분인가?**
   - 둘 다 *팀장 이상*에게만 보임. 분리 이유는 "보안" 아니라 "구현 시점"(metrics 라우터가 나중에 들어옴). 권한이 같은데 두 라우트로 갈라진 건 *조직 구조의 노출*.
   - **결정**: `/admin`을 탭 컨테이너로 만들고 `members | quotas | metrics | teams` 4탭. 라우트는 `/admin/members` 처럼 유지(deep-link).

5. **`/team`(감사) 과 `/admin`(관리) 가 다른 이유는?**
   - RBAC 차이: `canAudit ⊃ canManageTeam`. 즉 권한 등급이 다르다. 하지만 사용자는 "내 권한이 어디까지인지"를 모름. 메뉴가 따로 있으면 "내가 못 보는 게 있다"는 *불안*만 더한다.
   - **결정**: AvatarMenu "관리" 섹션에 *권한 라벨*과 함께 같이 노출(이미 적용) ✅. 두 라우트는 유지하되 "한 권한 콘솔"의 두 탭으로 보이도록 시각 통일.

6. **`/manual`(비로그인 공개) 과 `/guide`(로그인 후) 의 분리는?**
   - 공개 매뉴얼은 SEO/마케팅 용도, 로그인 후 가이드는 기능 안내. *완전히 다른 콘텐츠*이지만 사용자에겐 둘 다 "도움말". 첫 방문 후 "아까 본 그 가이드 어디 갔지?" 가 발생함.
   - **결정**: `/guide` 안에서 `/manual` 콘텐츠 일부를 *섹션*으로 임포트, 그리고 `/manual` 페이지 헤더에 "로그인하면 더 자세한 가이드를 봅니다 →" 명시.

7. **챗봇 페이지 안의 "Q&A"(`/chatbots/[id]/faqs`) 는 `/qa` `/faq` 와 어떤 관계인가?**
   - 챗봇별 FAQ(특정 봇에 대한 사용자들의 누적 질문)는 `/qa`(제품 기능 요청)·`/faq`(전사 FAQ)와 **개념적으로 다른 객체**다. 그러나 이름이 같다는 이유로 사용자는 같은 곳이라 착각.
   - **결정**: 챗봇 페이지 내부는 "**질문 모음**"으로 라벨 변경. URL 은 deep-link 보존하되 hint에 "이 챗봇이 받는 질문" 명시.

#### B. 개발 조직 노출

8. **`/skills` 의 "출처 필터"에 `seed / manual / user / skillsmp / getdesign / auto_generated` 6종이 그대로 노출된다.**
   - 사용자에게 `skillsmp` 가 뭔지 묻지 마라. 이건 *데이터 적재 출처*다. 사용자 분류 기준이 아니다.
   - **결정**: 출처 필터는 *관리자에게만*, 사용자는 "팀 / 공개 / 즐겨찾기" 3개만.

9. **워크플로 노드 타입에 `trigger / tool / llm` 이 그대로 보인다.**
   - `llm` 노드는 사용자가 *항상 사용*하는 거고, `tool` 은 *조건부*, `trigger` 는 *진입점*. 셋을 평등하게 나열하면 인지 비용 폭증.
   - **결정**: 캔버스 좌측 팔레트에서 카테고리 라벨을 "시작 / 행동 / AI 답변"으로. 색만 다르고 라벨은 유지(`trigger / tool / llm`)인 채로 두면 *코드 명사를 그대로 노출*.

10. **`/chat` 의 사이드바 하단 빠른 링크: "챗봇 / 문서 / 스케줄 / 스킬 / 도구 / FAQ / 마이"**
    - 이것은 **NavMenu의 미니어처**. 즉 *상단 메뉴를 줄여놓고 하단에서 다시 늘려놓은* 셈. Surface 압축의 의미가 무력화됨.
    - **결정**: 사이드바 빠른 링크는 *오늘 진행 중인 미션*과 *최근 챗봇 3개* 두 블록만. 메뉴 미러링 금지.

#### C. 권한·고급 기능의 잘못된 노출

11. **`/studio` (고급 빌더, 베타) 가 ⌘K "만들기" 그룹에 있던 이력.**
    - 이미 ⌘K "고급" 그룹으로 격리 완료 ✅.

12. **`/admin/metrics` 의 모니터링 카드가 첫 방문자에게도 사이드바에서 보이던 시기**
    - 현재는 권한 체크로 가렸지만, *권한 없는 사용자에게 "관리 콘솔" 링크가 보이는 코드 분기*가 chat/page.tsx 어딘가에 남아있는지 PR 별도 검증 필요.

13. **`/qa` 의 "관리자 답변 채택" UI**
    - 일반 사용자에게 *버튼은 보이고 클릭 시 권한 거부*가 뜨면 최악의 UX. 활성화 = 행동 가능성 표현(이미 AvatarMenu 정책으로 잡힘). 다른 화면에서도 동일 룰 적용 검증 필요.

#### D. 사용자 목표와 어긋난 기능

14. **`/mypage` 에 "비밀번호 변경"이 없다.**
    - 사용자가 가장 자주 검색할 항목 중 하나인데 페이지에 항목이 빠져있다(추정). 자세히 확인 후 추가.
    - **결정**: `/mypage` 에 "계정 보안" 섹션 신설, 비밀번호 변경 + 활성 세션 + 이메일 변경.

15. **`/documents` 의 처리 상태(processing/ready/failed) 가 카드 우상단 뱃지에만 표시된다.**
    - 사용자 *목표는 "이 문서로 답할 수 있나?"* 인데, 카드 뱃지로는 그 답이 명확하지 않다. failed 인데 "ready" 와 비슷한 회색이면 더 최악.
    - **결정**: 카드 좌측 색 띠(녹/주/적) + 텍스트 라벨로 *목표 가독성* 확보. failed 는 "재시도" 버튼 노출.

#### E. 메뉴 수 축소 전략 — 구조적

16. **NavMenu 2그룹 × 4·2 항목. 다음 한 단계 더 압축할 여지.**
    - "마켓에서 가져오기" 는 *일* 그룹에 있지만 사용자 의도는 "**탐색** → 가져오기" 의 2 단계. 채팅이나 챗봇 만들기와 동급으로 두면 평등성이 *전형적이지 않다*.
    - **결정 (선택지 2개)**:
      - **(A) 유지** — 일 4개 / 자동화 2개. 단순.
      - **(B) 추가 축소** — *일* 1그룹 + *자동화* 1그룹, 마켓을 *둘러보기*로 별도 — **그러나 권장 안 함**. 이미 충분히 압축됨.

17. **사이드바·NavMenu·AvatarMenu·⌘K 4개의 *발견 채널*이 있다. 일관성 룰이 없으면 사용자는 어디서 뭘 찾을지 모름.**
    - **결정**: 발견 채널 매트릭스를 docs/ia-rules.md 표로 못 박음 ✅. 코드에 신규 항목 추가 시 그 표를 PR에 첨부 강제.

### 1-2. 정리 — 액션 5개로 요약

| # | 액션 | 비용 | 효과 | 상태 |
|---|---|---|---|---|
| 1 | `/help` hub 페이지(공지·Q&A·FAQ 탭) | 中 | 高 (도움 발견성↑) | TODO |
| 2 | `/admin` 탭 컨테이너 (members/quotas/metrics) | 中 | 中 (관리자 IA 통일) | TODO |
| 3 | `/skills` 출처 필터 단순화(3개) | 小 | 中 (개념 노출 제거) | TODO |
| 4 | `/chat` 사이드바 미러링 제거 → 미션+최근 챗봇 2블록 | 中 | 高 (Surface 압축 보존) | TODO |
| 5 | `/mypage` 계정 보안 섹션 | 中 | 高 (지원 티켓↓) | TODO |

---

## 2. UI Quality Audit

### 2-1. 깨짐 가능성이 높은 영역 (실측 기반)

| 위치 | 문제 | 트리거 | 수정 기준 |
|---|---|---|---|
| `chatbots/page.tsx:265` | `.line-clamp-2 min-h-[2.2em]` — *2줄로 잘라도* 이름이 매우 길면 카드 높이는 그대로지만 박스 자체가 좁아져 보임 | 한국어 챗봇 이름 ≥ 30자 | `truncate` + tooltip(title 속성) 또는 `line-clamp-1` 로 강제 |
| `chatbots/page.tsx:253` | 이름 옆 `visibility` 뱃지 — 이름이 길면 뱃지가 줄바꿈으로 떨어짐 | 한 줄이지만 폭 짧을 때(모바일 320px) | `flex-shrink-0` 뱃지에 명시, 이름은 `min-w-0 truncate` |
| `documents/page.tsx:260` | "아직 올린 파일이 없어요." 한 줄 텍스트 — Empty State 컴포넌트 미사용. 다른 빈 상태(`/workflows`의 EmptyState)와 *시각 일관성 X* | 페이지 첫 진입 | 통일된 `<EmptyState>` 또는 신규 `<EmptyMissionBar>` 사용 (아래 코드 변경에서 반영) |
| 모든 카드 그리드 | `sm:grid-cols-2 lg:grid-cols-3` 가 페이지마다 다르게 잡힘 (`workflows` 는 `grid-cols-3`, `chatbots`는 `lg:grid-cols-3`, `skills`는 별개) | 태블릿(768~1024px) 폭 | 표준 그리드 토큰 `grid-cards-md` / `grid-cards-lg` CSS 클래스로 통일 |
| `MissionCard` footer | `mt-auto` + `<footer>` — 부모가 `flex flex-col` 일 때만 작동. 그리드 셀이 *동일 행 안에서 다른 높이*면 카드 높이도 어긋남 | 미션 reward 텍스트 길이 차이 | 그리드 부모에 `auto-rows-fr` 추가, 카드에 `h-full min-h-[140px]` |
| `Button` size variants | `min-h-9 / 11 / 12 / icon 10×10` — 4 사이즈를 페이지마다 자유롭게 섞어 씀 | 같은 페이지에 sm + md 혼재 | 페이지당 **1개 사이즈** 룰 또는 사이즈 매트릭스(폼/페이지/모달 별) |
| `u-input` 인풋 | 텍스트 매우 길 때 placeholder 흐름과 실제 텍스트 흐름 정렬이 안 맞을 수 있음 (RTL/긴 단어) | 영어 URL 입력 시 폭 초과 | `overflow-hidden text-ellipsis` + 입력 중에도 풀텍스트 보장은 readonly 옆 미리보기로 |
| Empty State CTA | `chatbots` 페이지 빈 상태는 미션 카드 3개, `workflows` 는 단순 EmptyState, `documents` 는 텍스트 — *동일 의도, 다른 시각* | 페이지 간 이동 시 인지 부조화 | 표준 `<EmptyMissionBar>` 컴포넌트로 통일 (이번 PR) |
| Alert 컴포넌트 위치 | 에러는 페이지 상단(`Alert`), 성공은 토스트(`ServerStatusToast`), 인라인 검증은 별도 — 사용자가 *피드백을 어디서 볼지 모름* | 폼 제출 후 에러 발생 | 폼은 인라인, 시스템 레벨은 상단, 비동기 결과는 토스트 — 정책 명시 후 적용 |
| 모바일 ChatHeader | `/chat` 의 우상단 플로팅 = 아바타 + ⌘K. iPhone SE(375px) 폭에서 ⌘K 칩 "어디로?" 라벨이 잘림 — `hidden sm:inline` 처리는 됐으나 모바일 단축키 인디케이터 부재 | < 640px 폭 | 모바일에서는 ⌘K 칩 대신 *돋보기 아이콘만*, 탭하면 풀스크린 검색 |
| Toast/스피너/Skeleton 부재 | 로딩 상태가 `불러오는 중…` 텍스트 한 줄. 진행감이 없고, 0건과 *로딩 중* 의 시각 구분이 약함 | 네트워크 지연 시 | 카드 그리드는 Skeleton 카드 3개, 텍스트만 있는 곳은 spinner + "로딩 중…" pulsing |

### 2-2. 디자인 시스템 토큰 — 이미 있음, 사용 룰만 정리

CSS 변수(`globals.css` 415–600줄대)에 다음이 *이미 정의*되어 있다 — 새로 만들 필요 없음:

```
--bg / --panel / --surface / --surface-muted / --surface-strong
--fg / --muted
--stroke
--accent / --accent-rgb / --accent-fg / --accent-soft
--glow-a / --stitch-whisper-border
```

빠진 것:
- **간격 토큰** — `--space-1/2/3/4/6/8` 같은 spacing scale CSS 변수가 없음. Tailwind 의 `gap-2 / py-3` 임의 사용 중. **권고**: 새로 만들 필요는 없다(Tailwind 자체가 토큰). 단, **컴포넌트별 패딩은 Card density(default/compact/micro) 토큰에 위임**하라는 룰을 ts/PR-template에 명시.
- **카드 높이 토큰** — `--card-min-h: 140px` 같은 명시 없음. `auto-rows-fr` 만으로는 그리드 행 단위 정렬이지 개별 미니멈 보장 X.

### 2-3. 카드/버튼/메뉴 규격화 기준

| 컴포넌트 | 규칙 |
|---|---|
| Button | 한 페이지에 한 사이즈. 폼: `size="md"`, 페이지 헤더 CTA: `size="md"`, 카드 내부: `size="sm"`, 모달: `size="md"`. `size="lg"` 는 *공개 랜딩(`/`)에만*. |
| Card | 페이지 메인 그리드: `density="default"`. 사이드바 미니 카드: `density="compact"`. 인라인 뱃지·핀: `density="micro"`. 같은 그리드 안에 density 혼합 금지. |
| Empty State | 카드 그리드의 빈 상태는 `EmptyMissionBar`(이번 PR) — *3카드 액션 추천*. 검색 결과 빈 상태는 `EmptyState`(기존) — *액션 1개*. |
| 라벨 길이 | 메뉴 라벨 ≤ 14자. hint ≤ 22자. 미션 카드 title ≤ 18자, reward ≤ 60자. 초과는 ESLint custom rule(추후). |
| 아이콘 크기 | 라인 아이콘 14px(인라인), 16px(버튼 leading), 20px(헤더), 24px(섹션 헤더). svg `stroke-width` 는 1.8 또는 2 만. |

### 2-4. QA 체크리스트 (수동)

다음 8개는 *모든 페이지*가 통과해야 한다:

- [ ] 폭 320px에서 가로 스크롤이 없다
- [ ] 한국어 풀네임 30자 입력해도 카드 헤더가 깨지지 않는다
- [ ] 한국어/영어 혼합 placeholder 가 잘리지 않는다 (`overflow-hidden text-ellipsis`)
- [ ] 빈 상태가 *항상* 카드 그리드 자리에 표준 컴포넌트로 나타난다
- [ ] 로딩 상태가 텍스트 한 줄 아닌 *시각적 진행감*을 준다
- [ ] 에러는 빨강, 경고는 호박, 정보는 회색 — 색만으로 의미가 전달되지 않게 아이콘 동반
- [ ] 모든 버튼은 `min-h-11`(`size=md`) 이상 — WCAG 2.5.5 (44×44 터치 타깃)
- [ ] 키보드 ⌘K, Esc, ↑↓, Enter 가 동일 의미로 작동(이미 CommandPalette 적용)

### 2-5. 반응형 기준

| 영역 | 기본 | sm (640) | md (768) | lg (1024) | xl (1280) |
|---|---|---|---|---|---|
| 사이드바 | 숨김 | 숨김 | 숨김 | 240px 고정 | 280px |
| NavMenu | hamburger | hamburger | 인라인 | 인라인 | 인라인 |
| 카드 그리드 | 1열 | 2열 | 2열 | 3열 | 3열 |
| ⌘K | 풀스크린 | 풀스크린 | 모달 560px | 모달 560px | 모달 560px |
| Empty Mission Bar | 1열 세로 | 1열 세로 | 2열 | 3열 | 3열 |

### 2-6. UI regression test 기준 (CTO 권고)

- **시각 회귀**: Playwright + `@playwright/test` (이미 devDep 추가) — 핵심 5개 페이지(`/`, `/chat`(empty), `/chatbots`(empty), `/documents`(empty), `/workflows`(empty)) 의 스크린샷을 main 대비 diff.
- **컴포넌트 단위**: Button 4 variants × 4 sizes / Card 3 density / EmptyState · EmptyMissionBar / MissionHub 4 상태(0/1/2/3 완료) — Storybook 도입 권장. 비용 ≈ 0.5 man-week.
- **접근성**: `@axe-core/playwright` 로 critical 위반만 차단.

---

## 3. User Behavior Audit — Entry Point별 시나리오

### 3-1. 진입 매트릭스

사용자가 첫 도달할 수 있는 페이지 7개. 각각 다른 "어디까지 안내해야 하는가"가 있다.

| 진입 페이지 | 누가 | 첫 5분 목표 | 현재 안내 | 문제 |
|---|---|---|---|---|
| `/` (공개) | 회원가입 직전 | 가입 결정 | 단일 CTA "시작하기" ✅ | 가입 모드 3개의 차이를 모름 |
| `/register` | 신규 사용자 | 가입 성공 | 모드 라디오 3개 + 설명 | "새 팀 만들기" 가 *기본*이 아니라 *세 번째* 옵션. 데모/체험 사용자에게는 1순위여야 함 |
| `/chat` | 가입 직후/일반 | 첫 답변 | MissionHub ✅ (`/chat` 전용) | **다른 페이지 진입 시 미션 안내 X** |
| `/chatbots` | 마켓 링크 / 동료 추천 | 첫 챗봇 | 빈 상태에 3개 MissionCard ✅ | 미션 *진행도*(전역) 와 연결 안 됨 — 별도 카드 그리드 |
| `/documents` | 초대 직후 자료 업로드 의뢰 | 첫 자료 | "아직 올린 파일이 없어요." 텍스트만 | **온보딩 부재** |
| `/workflows` | 자동화 데모 | 첫 자동화 | EmptyState 컴포넌트 ✅ | 빈 상태가 *액션 1개* 만 — 다음 단계 없음 |
| `/admin` | 관리자 첫 로그인 | 팀 셋업 | 승인 대기 사용자 / 팀 생성 / 용량 | 첫 방문 시 *어디부터 봐야 하는지* 안내 부재 |

### 3-2. Context-aware Guided System — 설계 원칙

**1) 미션 진행도는 페이지가 아니라 *사용자 상태*다.** `frontend/features/onboarding/missionProgress.ts` 의 시그널(`hasAnyMessage / hasAnyChatbot / hasAnyDocument`)을 *모든 페이지에서* 동일하게 읽고 *현 페이지의 미완료 미션*을 강조한다.

**2) "다음 액션 1개"만 추천한다.** 다음에 할 일이 *항상 1개* 보여야 한다. 3개를 동시에 보이는 건 첫 화면(`/chat`)에만. 다른 페이지 빈 상태는 *해당 페이지 미션 1개*(highlight) + *전체 진행 도트*.

**3) 페이지마다 "여기서 시작 / 다음으로 갈 곳" 2개를 명시.** 빈 상태에서는 "여기서 시작" 1개 CTA, 미션 완료 후엔 "다음으로" 1개 CTA. 둘 다 *동사형*.

**4) 길을 잃었을 때의 복구 UX — `?` 버튼**. 모든 페이지 우상단(AvatarMenu 옆)에 작은 `?` 버튼 — 클릭 시 (a) 이 페이지의 *목적 1줄* (b) *가능한 행동 3개* (c) "가이드 전체 보기" 링크. *Linear의 helper bar 패턴*.

**5) 역할별 분기는 *최소화*.** 초보자/중급자/고급자를 *세 가지 UI*로 만들면 복잡도가 3배. 대신 **"3미션 완료 = 중급자 진입"**, 중급자에겐 미션 허브를 숨기고 *최근 활동* 위주로. 고급자(예: super_admin)는 *처음부터* 미션 허브 숨김 — `localStorage` 또는 서버 user_meta.

### 3-3. Entry Point별 UX Flow

#### A. `/chat` 첫 진입 (기존 ✅)
```
인사 → MissionHub(0/3) → quick-action 칩 → 입력
                                      ↓ 입력 후
                                  답변 + PostAnswerMissionBand("3분 챗봇 만들기 →")
                                      ↓ Y
                                  /chatbots#new (highlight: first_chatbot 미션)
```

#### B. `/chatbots` 빈 상태 (오늘 추가)
```
PageHeader "내 챗봇"
→ EmptyMissionBar (1/3 진행도 + first_chatbot 카드 highlight + 다른 2개 dim)
→ "지금 시작 → 위 폼으로 스크롤" 단일 CTA
```

#### C. `/documents` 빈 상태 (오늘 추가)
```
PageHeader "자료"
→ EmptyMissionBar (해당 미션 first_document 강조)
→ 드래그 앤 드롭 영역 (눈에 띄게)
→ 한 줄 안내: "한 파일이면 충분. 봇이 출처와 함께 답합니다."
```

#### D. `/workflows` 빈 상태 (오늘 강화)
```
PageHeader "반복 작업 자동 실행"
→ EmptyMissionBar (전체 미션 진행도 + "프롬프트 한 줄로 자동 생성" 추천)
→ 2 CTA: "프롬프트로 자동 생성" + "직접 만들기"
```

#### E. `/admin` 첫 로그인 (TODO)
```
PageHeader "관리 콘솔" + 첫 방문 banner
→ 4 step checklist: [팀 이름 확인] [초대 코드 복사] [용량 확인] [팀원 초대]
→ 각 step 클릭 시 해당 탭으로 스크롤
```

### 3-4. Role-based 분기 — 최소

| 역할 | 미션 허브 | NavMenu | AvatarMenu 관리 섹션 |
|---|---|---|---|
| 첫 방문 일반 | 표시 | 일/자동화 2그룹 | 도움만 |
| 3미션 완료 일반 | 숨김 (24h TTL) | 동일 | 도움만 |
| 감사자 | 표시 | 동일 | 도움 + **감사** |
| 팀 관리자 | 1회만 표시 | 동일 | 도움 + 팀 관리 |
| 슈퍼관리자 | 시작부터 숨김 | 동일 | 도움 + 시스템 관리 |

### 3-5. 이탈 방지·복귀 UX

- **이탈 방지**: 미션 진행 *중* 페이지를 닫으려 하면 — 안 한다. 강제 모달 금지. 대신 *진행 도트*를 사이드바에 작게(`1/3`) — 다음 방문 시 "어디까지 했지?"를 즉시 보임.
- **복귀**: 24시간 이상 미접속 후 첫 진입 시 MissionHub 가 *다시 표시*(localStorage TTL). 동시에 작은 인사말 "오랜만이에요. 어디까지 하셨는지 보여드릴까요?"
- **실패 복구**: 답변 실패 시 toast + "다시 시도" + "다른 표현으로 물어보기" 칩 3개. 챗봇 생성 실패 시 폼이 *유지*(입력 데이터 보존), 에러는 인라인.

---

## 4. Menu Redesign — 결과물

### 4-1. 추천 IA (2026-05-21 현재 + 다음 2단계)

**Phase 1 (지금)** — 이미 적용 ✅
```
NavMenu
├ 일 → 대화 시작 / 챗봇 만들기 / 자료 올리기 / 마켓에서 가져오기
└ 자동화 → 반복 작업 자동 실행 / 정해진 시간에 실행
AvatarMenu
├ 마이페이지 / 가이드
├ 도움 & 커뮤니티 → 공지 / 물어보기·기능 요청 / 자주 묻는 질문
└ (권한) 관리 → 감사 / (팀·시스템) 관리
⌘K (그룹: 일/자동화/도움/개인/고급/관리)
```

**Phase 2 (다음 1~2주)**
- `/help` 단일 hub(탭) — 공지/QA/FAQ 통합 진입 (deep-link 보존)
- `/admin` 탭 컨테이너 — members/quotas/metrics
- `/chat` 사이드바 — *메뉴 미러링 제거*, 미션+최근 챗봇 2블록

**Phase 3 (선택)**
- 마켓/스킬 통합 UX 정리 — 챗봇 빌더 안에서 *마켓 → 가져오기* 컨텍스트 진입
- `/mypage` 계정 보안 섹션

### 4-2. 라벨 안티패턴 (docs/ia-rules.md 참조)

이미 작성됨. 신규 메뉴/CTA 추가 시 PR 템플릿에 "안티패턴 단어 사용 여부 YES/NO" 강제.

---

## 5. Action-based Onboarding — 결과물

### 5-1. 미션 카탈로그 (현재 3개)

```
M1 first_answer   — 첫 답변 받기      30초  → 대화 1턴 발생
M2 first_chatbot  — 챗봇 1개 만들기   3분   → /chatbots POST 성공
M3 first_document — 자료 1개 올리기   1분   → /documents/upload 성공
```

확장 후보(검증 필요):
```
M4 first_share    — 챗봇 1개 공유      → visibility=public 변경 1회
M5 first_workflow — 자동화 1개 만들기  → /workflows POST 성공
M6 first_schedule — 예약 1개 설정      → /schedules POST 성공
```

> **CTO 비판**: 미션은 *늘리지 마라*. 3개가 적정. 6개 미션은 *체크리스트 피로*를 일으킨다. Duolingo 가 *하루 1 lesson*인 이유와 같다. M4–M6 는 "보조 미션"으로만, 메인 허브에 노출 X.

### 5-2. 진행도 저장 — 서버 vs 클라이언트

현재: 클라이언트(localStorage)에 *dismiss* 만 저장, 진행도는 서버 데이터(messages/chatbots/documents 카운트)에서 매번 재계산. **이게 맞다.**

이유:
- 서버에 별도 `onboarding_progress` 테이블을 두면 *데이터가 지워졌는데 미션은 완료*인 모순 발생
- 진행도 계산은 *카운트 쿼리 3개*면 충분 — 캐시 필요 없음
- 단, KPI 분석을 위해 *완료 시각 이벤트*는 별도 저장(아래 6장)

### 5-3. 미션 잠금 해제 — 권장 안 함

Duolingo 식 "다음 lesson 잠김" 은 학습 도메인 패턴. SaaS 도구에서는 *모든 액션이 항상 가능*해야 한다. 미션은 *추천*이지 *제약*이 아니다.

### 5-4. 실패 복구 UX

| 실패 지점 | 복구 |
|---|---|
| 답변 생성 실패 | toast + "다시 시도" + "다른 표현" 3개 chip |
| 챗봇 생성 검증 실패 | 폼 유지, 에러 inline, 자동 focus |
| 자료 업로드 실패(포맷) | 카드에 "지원 포맷: PDF/DOCX/MD/TXT" + "다시 시도" |
| 자료 처리 실패(서버) | 카드 빨강 + "재인덱싱" 버튼 |

---

## 6. CTO-level Implementation Strategy

### 6-1. React/Next.js 컴포넌트 구조

```
frontend/
├ app/
│  ├ (workspace)/   [페이지 라우트 — page.tsx 만 두고 로직은 features/로]
│  └ globals.css    [디자인 토큰 — 변경 시 모든 페이지 영향. CR 2명 이상]
├ features/         [도메인 단위 비즈니스 로직]
│  ├ chat/
│  ├ chatbots/
│  ├ documents/
│  ├ onboarding/   ← MissionHub, missionProgress, EntryMissionBar (이번 PR)
│  ├ workflows/
│  └ ...
└ shared/
   ├ layout/        [AppShell, NavMenu, AvatarMenu, CommandPalette]
   ├ lib/           [api, track(이번 PR), userError]
   ├ theme/
   └ ui/            [Button, Card, EmptyState, MissionCard, ...]
```

**CR 룰**:
- `app/` 의 page.tsx 는 *컴포넌트 조립*만. 비즈니스 로직 ≥ 30줄이면 features/로.
- `shared/ui/` 는 props 인터페이스 안정. 변경 시 BREAKING 라벨.
- `features/<domain>/` 끼리 *직접 import 금지*. shared 만 의존.

### 6-2. 온보딩 상태관리

- **클라이언트**: `useMissionProgress({ hasAnyMessage, hasAnyChatbot, hasAnyDocument })` — 입력은 *서버 신호*, 출력은 *UI 상태*.
- **서버**: 별도 state 없음. 단, *완료 이벤트 timestamp*는 `usage_events` 테이블에 `kind=onboarding_milestone` 로 1회 기록 → KPI 분석.

### 6-3. Feature Flag

현재 없음. 권장:
```ts
// shared/lib/flags.ts
export const FLAGS = {
  mission_hub_v2: true,   // 이번 PR
  help_hub: false,        // Phase 2
  admin_tabs: false,      // Phase 2
  onboarding_role_split: false,
} as const;
```
- 단순 boolean. PostHog/Unleash 같은 풀 솔루션은 *5명 미만 팀에선 과잉*. 추후 도입.

### 6-4. Role/Permission 기반 노출

이미 적용: `canAudit / canManageTeam / isSuperAdmin` props 가 AppShell → NavMenu/AvatarMenu/CommandPalette 로 흐름. 추가 룰:
- 권한 없는 항목은 *숨김*. 라우트 가드는 백엔드(`require_team` 등)가 *최종 책임*.
- 클라이언트는 *시각적 친화* 만 책임 — 보안 가정 금지.

### 6-5. 이벤트 트래킹 설계

`frontend/shared/lib/track.ts` 신규 (이번 PR). 어댑터:
- 기본: console + (옵션) PostHog/Amplitude window 객체 자동 탐지.
- 이벤트 카탈로그 *enum*화 — 자유 문자열 금지 → 분석 깨짐 방지.

핵심 이벤트:
```
nav.group.click           {group: "work"|"automate"}
nav.item.click            {href, label, source: "nav"|"avatar"|"palette"}
palette.open              {via: "shortcut"|"button"}
palette.search            {query_length}
palette.select            {id, query_length}
onboarding.mission.shown  {missionId, current, total}
onboarding.mission.click  {missionId, source: "hub"|"bar"}
onboarding.mission.complete {missionId, ms_since_signup}
onboarding.hub.dismiss
empty_state.cta.click     {page, cta}
auth.register.start       {mode}
auth.register.success     {mode, ms_on_page}
chat.first_message        {has_chatbot, model_id, ms_since_open}
```

ms_since_signup / ms_since_open 같은 시간차는 *서버 timestamp diff*로 재계산 가능하지만 클라이언트에서 *page-open* 단위 정밀도를 잡으려면 클라 발사가 정답.

### 6-6. A/B 테스트 설계

작은 팀 → *대규모 실험 인프라*는 과잉. 권장:
- *플래그 + 코호트 분리*만 자체 구현. user_id mod 2 → A/B.
- 1실험 1KPI. 메인 KPI 의 95% 신뢰구간(z-test) 만 본다 — Bayesian/멀티팩터는 도입 비용 대비 효익 낮음.
- 실험 자체는 *2주 + 코호트 ≥ 200* — 사내 도구라 트래픽이 작아 더 길게 잡는다.

### 6-7. Design System / QA 자동화

- Playwright e2e 5개 (홈/가입/첫채팅/첫챗봇/첫자료) — `@playwright/test` 이미 devDep.
- Storybook — 컴포넌트 단위 시각 검증 (선택, 0.5 man-week).
- `eslint --max-warnings 0` CI — 현재 일부 페이지에 pre-existing 에러 10개. 새 PR은 추가 에러 0.
- `tsc --noEmit` CI 통과 — 이미 통과.

### 6-8. 데이터 기반 UX 개선 루프

1. **트래킹** → 6-5의 이벤트로 *행동 깔때기* 그림.
2. **주간 리뷰** — 매주 금요일 30분: Activation 깔때기 + 가장 큰 drop-off 1곳 식별.
3. **가설 1개** — drop-off 원인 가설 1개.
4. **실험 또는 직접 수정** — 작으면 직접 수정, 크면 A/B.
5. **반복** — 다음 주 같은 페이지 측정. 개선 없으면 가설 폐기.

---

## 7. KPI & Experiment Design

### 7-1. 핵심 KPI

| KPI | 정의 | 목표 (가설) | 측정 |
|---|---|---|---|
| **Activation Rate** | 가입 후 7일 이내 미션 ≥1 완료 | 60% | `onboarding.mission.complete` 발생 / `auth.register.success` |
| **Time to First Value (TTFV)** | 가입~첫 답변까지 분 | < 5분 (중앙값) | `chat.first_message` ts − `auth.register.success` ts |
| **Onboarding Completion** | 미션 3/3 완료 비율 (가입 14일 내) | 30% | `onboarding.mission.complete` × 3 |
| **Feature Adoption** (per feature) | 가입 30일 내 1회 사용 | 챗봇 70% / 문서 50% / 자동화 25% | 각 도메인 첫 사용 이벤트 |
| **Menu Click Depth** | 항목당 클릭 / 활성 사용자 | NavMenu 평균 ≥ 1.5/주 | `nav.item.click` |
| **⌘K Adoption** | 가입 30일 내 ⌘K 사용자 비율 | 25% | `palette.open` |
| **Empty State Conversion** | 빈 상태에서 CTA 클릭→다음 액션 | 40% | `empty_state.cta.click` 직후 *해당 도메인 첫 사용 이벤트* |
| **Retention D7 / D30** | 가입 후 7/30일 재방문 | D7 50% / D30 25% | `auth.me` GET 일별 unique |
| **User Confusion Signal** | 같은 페이지에서 30초 내 NavMenu 클릭 ≥ 3회 | 5% 이하 | 클라 측 카운터 |
| **Support Ticket 감소율** | 가입 직후 4주간 *동일 사용자*의 `/qa` `/faq` 작성 수 | 분기당 20%↓ | `faq_post.created` |

### 7-2. A/B 실험 우선순위

| # | 실험명 | 메인 KPI | 가설 | 코호트 | 기간 |
|---|---|---|---|---|---|
| E1 | NavMenu 2그룹 vs 3그룹 | Menu Click Depth + Confusion Signal | 2그룹이 클릭 ↑ confusion ↓ | 가입 후 신규만 | 4주 |
| E2 | 액션형 MissionHub vs 설명형 튜토리얼 모달 | Activation Rate | 액션형이 +15%pt | 신규 100% | 6주 |
| E3 | 챗봇 중심 vs 전역 Entry-MissionBar (이 PR) | TTFV 중앙값 | 전역이 -2분 | 신규 100% | 4주 |
| E4 | 진입 모드 기본값 (`팀장 이메일` vs `새 팀 만들기`) | Register Success Rate | 데모용 진입은 새 팀이 ↑ | utm_source=demo 코호트 | 4주 |
| E5 | Empty Mission Bar 3카드 vs 1 highlight + 2 dim | Empty State Conversion | 1 highlight 가 ↑ (선택 피로↓) | 신규 100% | 4주 |

### 7-3. 측정 가능성 점검

- `auth.register.success` — 백엔드 로그에서 추출 가능, *클라 이벤트도 추가* 필요
- `chat.first_message` — 클라에서만 정밀 추적 가능
- *시간차* 분석은 user_id 기반 join — 백엔드에서 1쿼리

---

## 8. 실행 우선순위 Roadmap

### Sprint 1 (이번 주 — 이 PR)
- [x] NavMenu 2그룹 + 동사형 라벨
- [x] AvatarMenu "도움 & 커뮤니티" 분리
- [x] CommandPalette 동기화 + "고급" 그룹
- [x] MissionHub (`/chat` 빈 상태)
- [x] docs/ia-rules.md
- [x] docs/audit-cto.md (본 문서)
- [ ] **shared/lib/track.ts** — 트래킹 어댑터
- [ ] **features/onboarding/EntryMissionBar.tsx** — 컴팩트 미션 바
- [ ] `/chatbots` `/documents` `/workflows` 빈 상태에 EntryMissionBar 적용
- [ ] MissionHub / EntryMissionBar 에서 track() 호출
- [ ] mkdocs.yml — audit-cto.md 등록

### Sprint 2 (다음 주)
- [ ] `/help` 단일 hub (탭: 공지/Q&A/FAQ)
- [ ] `/admin` 탭 컨테이너
- [ ] `/skills` 출처 필터 단순화
- [ ] `/chat` 사이드바 미러링 제거
- [ ] `/mypage` 계정 보안 섹션 (비밀번호 변경)
- [ ] Playwright e2e 5개 (홈/가입/첫채팅/첫챗봇/첫자료)

### Sprint 3 (그 다음)
- [ ] Empty State / Loading Skeleton 전 페이지 표준화
- [ ] 카드/버튼 사용 룰 ESLint custom rule
- [ ] PostHog 또는 Amplitude 어댑터 (track.ts 가 이미 인터페이스)
- [ ] A/B 실험 E1, E3 시작

### Sprint 4+
- [ ] Help `?` 버튼 (전역 컨텍스트 헬퍼)
- [ ] Role 분기 (super_admin 미션 허브 자동 숨김 / 첫 방문 admin 체크리스트)
- [ ] Storybook
- [ ] 시각 회귀 테스트 CI

---

## 9. 마지막 한 줄

기능을 줄이지 않으면서 사용자에게는 단순하게 보이려면, *기술 경계를 사용자 경계로 노출하지 않을 자제력*과 *지금 보여줄 1개만 골라낼 결단력*이 필요하다. 이 두 가지를 PR 리뷰 단계에 강제해야 한다 — 그래서 본 문서가 *코드와 같이 살아야* 한다.

— end —
