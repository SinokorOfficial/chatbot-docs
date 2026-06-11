# 프론트엔드 가이드

## 1. 라우팅 개요

```
app/
├─ (public) -- 인증 불필요 (파일 구조상 그룹 없음, 그냥 루트)
│ ├─ page.tsx /
│ ├─ login/page.tsx /login
│ └─ register/page.tsx /register
└─ (workspace) -- AppShell 공통 레이아웃
 ├─ layout.tsx
 ├─ chat/page.tsx /chat
 ├─ documents/page.tsx /documents
 ├─ chatbots/page.tsx /chatbots (신규)
 ├─ chatbots/[id]/page.tsx /chatbots/{id} (신규)
 ├─ tools/page.tsx /tools (신규)
 ├─ admin/page.tsx /admin (신규)
 ├─ notices/page.tsx /notices
 ├─ faq/page.tsx /faq
 ├─ mypage/page.tsx /mypage
 ├─ studio/page.tsx /studio
 └─ team/page.tsx /team
```

## 2. AppShell

- `(workspace)` 하위 페이지 공통 헤더를 렌더.
- 헤더 요소: 브랜드 → 네비게이션(채팅·문서·챗봇·도구·FAQ·공지·마이·관리·팀) → 테마 피커 → 사용자 이름 → **로그아웃 버튼**.
- `/chat` 경로에서는 전폭 레이아웃을 위해 헤더가 감춰지지만, 좌측 사이드바 하단에 **공지 미니 목록**, **사용량 미니 카드**, **마이페이지 링크**, **로그아웃 버튼**을 배치합니다.

### 로그아웃 동작
`components/AppShell.tsx`의 헤더 우측과 `/chat`의 사이드바에 공통으로 아래 콜백을 사용합니다.

```tsx
const handleLogout = () => {
 clearToken();
 router.replace("/login");
};
```

세션 종료 후 `/login`으로 즉시 리다이렉트됩니다.

## 3. 상태 관리

- **전역 상태 없음** — 각 페이지가 `useEffect`에서 필요한 API를 호출하고 `useState`로 보관.
- **인증 토큰**: `localStorage.token`. `lib/api.ts`의 `apiFetch()`가 자동으로 `Authorization` 헤더에 주입.
- **테마**: `ThemeProvider`가 `localStorage.theme` 관리.

## 4. 챗봇 사용 UI (채팅 페이지 변경점)

- 사이드바 상단에 "챗봇 선택" 드롭다운을 추가.
- 선택된 `chatbot_id`는 새 대화 생성 시 서버에 전달.
- 선택한 챗봇의 프로필(이름, 설명, 도구 뱃지)은 입력창 위 배너에 축약 표시.

## 5. 챗봇 편집 (`/chatbots/[id]`)

3개 탭:
1. **기본** — 이름, 설명, 모델, 시스템 프롬프트, 가시성, RAG 스코프
2. **문서** — 팀 문서 + 내 개인 문서 목록에서 체크박스로 연결 선택 (`linked_only` 스코프에서 핵심)
3. **도구** — 내가 설치/자격증명 등록한 도구를 활성화 토글

"저장" 시 각각 `PATCH /chatbots/{id}` · `PUT /chatbots/{id}/documents` · `PUT /chatbots/{id}/tools`.

## 6. 도구 마켓 (`/tools`)

- 카드 그리드, 카테고리 탭 (communication / search / data / productivity / all).
- 각 카드: 아이콘, 이름, 카테고리, "연결" 버튼 또는 "연결됨" 뱃지.
- 연결이 OAuth 기반이면 서버가 반환한 consent URL로 새 창 이동.

## 7. 관리 콘솔 (`/admin`)

- 탭 1: 승인 대기 사용자 목록 → 승인/반려
- 탭 2: 팀 목록 (super_admin 전용) → 팀 신규 생성, 초대코드 복사
- 탭 3: 도구 카탈로그 (super_admin 전용) → 도구 추가/비활성화

## 8. 공지 · FAQ · 마이페이지

- `/notices`: 작게 보이던 공지 영역을 별도 페이지로 분리. 관리자는 생성/수정/비활성화 가능.
- `/faq`: 사용자가 기능 요구사항이나 질문을 작성하고, 관리자가 답변/상태를 관리.
- `/mypage`: 내 프로필과 LLM 사용량 요약. USD 비용과 KRW 환산액을 함께 표시.
- `/chat`: 좌측 하단에 최신 공지 3개와 7일 사용량 요약을 축약 표시.

## 9. 로그아웃 UX 요구사항 (체크리스트)

- [x] AppShell 헤더에 로그아웃 버튼
- [x] 채팅 페이지 사이드바에 로그아웃 버튼 (헤더 미표시 상태에서도)
- [ ] (후속) 세션 만료 자동 감지 후 토스트로 안내 → `/login` 이동

## 10. 에러 UX

- `ApiUserError`로 모든 API 오류를 통일. 401/403은 AppShell이 `/login`으로 리다이렉트, 그 외는 페이지 상단 에러 배너로 표시.
- 자격증명 만료 같은 도구 실패는 채팅 메시지 안에 에러 블록으로 표시(에이전트 출력 경로).
- **채팅 스트림 실패** — user 말풍선은 지우지 않고 그대로 유지하며 말풍선 아래 "⚠ 전송 실패" 상태를 표시(`Msg.failed`, 클라이언트 전용). `FailureBanner` 재시도가 같은 내용으로 들어오면 새 말풍선을 추가하지 않고 실패 표시만 해제한다(중복 방지). 배너를 닫아도 내가 보낸 내용이 대화에 남는다.
- **첨부 경고는 비차단** — 폴더 첨부 상한(300개) 초과 등은 native `alert()` 대신 컴포저 위 인라인 경고 바(8초 자동 소거 + 수동 닫기). 이미지 교체 확인은 `confirmDialog` 사용(native `confirm()` 금지).

## 11. 피드백 · 로딩 프리미티브 (`shared/ui/`)

모두 "모듈 싱글톤 + 전역 호스트" 패턴 — 호스트는 AppShell 에 1회 마운트, 호출은 어디서든.

| 프리미티브 | 용도 | 호출 |
| --- | --- | --- |
| `toast()` | 일상 액션 피드백("저장했어요"/"삭제했어요"/조용한 실패). 우하단 비차단 스택, 자동 소거(성공 3.5s·에러 6s) | `toast("저장했어요")`, `toast(msg, "error")` |
| `celebrate()` | 첫 챗봇 생성 같은 *달성* 순간. 컨페티 + 격려. 드물게 | `celebrate("만들었어요! 🎉")` |
| `successFollowup()` | 성공 직후 *다음 행동* 제안(액션 링크 포함) | `successFollowup({title, actions})` |
| `confirmDialog()` | 파괴적/되돌리기 어려운 액션 확인. native `confirm()` 금지 | `await confirmDialog({title, danger:true})` |
| `Alert` | 페이지 상단 인라인 오류/안내(머무르는 상태) | `<Alert variant="error">…</Alert>` |
| `Skeleton` / `SkeletonList` | 로딩 — "불러오는 중…" 텍스트 대신 콘텐츠 형태 예고 | `<SkeletonList rows={4} />` |

**중복 금지 규칙**: 같은 순간에 toast+celebrate, toast+successFollowup, toast+인라인 Alert(같은 에러) 를 동시에 쓰지 않는다. 인라인 `Alert`/`setErr` 로 이미 표시되는 에러에 toast 를 또 띄우지 않는다. `<option>` 내부 로딩 텍스트는 스켈레톤 불가 — 텍스트 유지.

## 12. 의존성

- `next@15`, `react@19`, `tailwindcss@3.4`, `react-markdown`, `rehype-highlight`, `remark-gfm`.
- 새 페이지는 `use client` 지시어 필요(모두 인터랙티브).
