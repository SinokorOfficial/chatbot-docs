# Changelog

날짜는 YYYY-MM-DD, 가장 최신이 위.

## 2026-05-20 — mkdocs 기반 GitHub Pages 가이드 사이트

### 추가 (Added)
- **mkdocs.yml** : Material 테마(한국어, Pretendard, Mermaid), `docs/` 전체를 사이트 구조로 정리한 nav. 관리자·운영·배포 등 사내 전용 문서는 본 공개 리포에는 포함하지 않습니다.
- **GitHub Pages 자동 배포** : `.github/workflows/docs.yml` — `docs/` 또는 `mkdocs.yml` 푸시 시 `mkdocs build --strict` → Pages 배포 (총무팀과 동일 패턴).
- **docs/index.md** : 공개 랜딩 페이지. 시스템 한눈 요약, 사용자 매뉴얼 진입, 기여 규칙 안내.
- **CONTRIBUTING.md** : "기능 추가 시 항상 `docs/` 갱신" 규칙과 영역별 체크리스트 (DB → erd.md, API → api.md, UX → design-system.md 등).

### 변경 (Changed)
- **프론트 → 백엔드 기본 URL** : `frontend/next.config.ts` 의 `BACKEND` 디폴트를 `:8000` → `:8001` 로 고정 (cleanup 전용 포트). `allowedDevOrigins` 를 `10.x.x.x/16` 으로 확장 → 사내망 다른 PC 에서도 `http://10.x.x.x:3001` 로 접속 가능.
- **링크 정리** : 빌드 strict 모드 대응. `erd.md` / `themes.md` / `skills-marketplace.md` 의 리포 외 경로 링크를 경로 텍스트로 변환.

## 2026-05-20 — 공지사항, FAQ/기능 요청, 마이페이지 사용량

### 추가 (Added)
- **DB 기반 공지사항** : `notices` 테이블과 `/notices` API, `/notices` 페이지, 채팅 좌측 하단 최신 공지 미니 목록
- **FAQ/기능 요청 게시판** : `faq_posts` 테이블과 `/faq/posts` API, `/faq` 페이지. 사용자는 요청/질문 작성, 관리자는 답변과 상태 관리
- **마이페이지 사용량 요약** : `/usage/me/summary` API, `/mypage` 페이지, 채팅 좌측 하단 사용량 미니 카드. USD 비용과 KRW 환산액 동시 표시
- **사용자 매뉴얼** : `docs/user_manual.md`

### 변경 (Changed)
- 채팅 사이드바 하단을 마이페이지/공지/사용량 중심으로 정리
- `.env.example`의 API 키는 빈 값, 부트스트랩 계정은 placeholder로 정리하고 `USD_KRW_FALLBACK_RATE` 추가

### 검증
- 백엔드 compile 및 프론트 production build 대상
- 신규 공지/FAQ/사용량 API 회귀 테스트 추가

---

## 2026-05-15 — 가독성 폰트 ↑, Gmail tool-server, curate yaml 외재화 + ingest 회귀

### 추가 (Added)
- **Gmail tool-server 외재화 샘플** : `worker/tool_server/gmail/main.py`
  - `make_app("gmail.send", handle)` 표준 인터페이스 (Bearer auth, /invoke, /health)
  - `GMAIL_DRY_RUN=1` (기본) 또는 credential 없으면 dry-run 응답. 0 으로 풀고 OAuth 자격증명 연결 시 실 Gmail REST 호출
  - 401 → PermissionError ("토큰 만료/무효, /tools 재연결"), 4xx/5xx → RuntimeError
- **curate KNOWN 데이터 yaml 외재화** : 운영자가 코드 수정 없이 PR 가능
  - `backend/config/curate/skillsmp_known.yaml` (5 스킬: git-rebase / prompt-cache / api-contract / tech-writing / observability)
  - `backend/config/curate/getdesign_known.yaml` (4 테마: loom / jasper / huly / cluely + 토큰)
- `curate_skillsmp.py`, `curate_getdesign.py` 가 위 yaml 을 `lru_cache` 로 로드 (handler 모듈 import 시 1회)

### 변경 (Changed)
- 가독성: `html { font-size: 17px }` (default 16→17), `.chat-bubble-user` 16px + line-height 1.7 + padding 0.875/1.125rem
- `defaultTokens.ts` size scale 상향: sm 14 / base 16 / lg 18 / xl 22 / 2xl 28

### 검증
- 프로덕션 빌드 OK (`next build`), 17px / 16px 가 minified CSS 에 포함됨
- Ingest 회귀 end-to-end: `curate_getdesign` 잡 → 4 themes 추가 (16→20, `enriched=4` 메타 보강 성공) / `curate_skillsmp` → 5 skills 추가
- 모든 백엔드 핸들러 정상 import (parse_document / embed_chunks / curate_skillsmp / curate_getdesign / gen_recovery_tip)

---

## 2026-05-15 — 일괄 마무리: 12 브랜드 풍부화, RAG Phase 2, curate 워커, ESC 닫기

### 추가 (Added)
- **나머지 12개 브랜드 토큰 풍부화** (themes.yaml): Linear/Notion/Vercel/OpenAI/Claude/GitHub/Stripe/Nike/Spotify/Tesla/Airbnb/Hyundai 모두 shadow/semantic/focus_ring/accent_secondary/typography scale/motion easing 모두 명시. design.md 의 핵심을 토큰으로 코드화
- **RAG Phase 2 경량 구현** : `app/services/rag_advanced.py`
  - `check_sufficiency`: LLM(mini) 이 1차 검색 결과가 답하기 충분한지 JSON 응답
  - `maybe_refine_with_sufficiency`: 부족하면 sub_query 1개 생성 후 1회만 재검색 + 병합 (depth 2 한정으로 비용 통제)
  - 실 사용은 chat 흐름에서 선택적 호출 (settings flag)
- **Worker curate_getdesign 본구현** : `worker/ingest/handlers/curate_getdesign.py`
  - KNOWN_THEMES 사전 (Loom/Jasper/Huly/Cluely 등 큐레이트 토큰)
  - payload.slugs 로 선택적 동기화 또는 전체. 중복 slug skip
- **Worker curate_skillsmp 본구현** : `worker/ingest/handlers/curate_skillsmp.py`
  - KNOWN_SKILLS (Git rebase / Prompt caching / API contract / Diátaxis / Observability 3-pillar 등)
  - payload.category 필터 가능. 중복 slug skip
- **ESC 키 닫기** : SandboxPanel / SchedulePanel 에 표준 ESC keydown 핸들러 (기존 SkillDetail/SkillEditor 와 일관)

### 변경 (Changed)
- Worker handler 자동 로드 카운트: 2/5 → **4/5** (embed_chunks 만 미구현)

### 검증
- 4 handlers 자동 등록 OK
- themes 16개 응답 (모두 풍부 토큰)
- production build OK

---

## 2026-05-15 — 테마 토큰 확장 + 뒤로가기 UX + 모든 페이지 테마 picker 통일

### 추가 (Added)
- **테마 토큰 표준 확장** : `frontend/shared/theme/defaultTokens.ts` 신규.
  - 기존 colors/typography/radius/motion 외에 **shadow, semantic colors (success/warning/error/info), focus_ring, accent_secondary, border_strong, spacing scale (xs..xl), tracking, line_height** 추가
  - `mergeTokens(default, brand)` 으로 깊은 병합 — yaml 은 의미 있는 키만 명시하면 됨
- **`themes.yaml` 풍부화** : default / BMW / PlayStation / Apple 의 design.md 주요 특징을 토큰으로 반영 (BMW M Red 액센트, PS 청록 글로우, Apple iCloud 보라, 등)
- **`BackButton` 공용 컴포넌트** (`shared/ui/BackButton.tsx`) : href 있으면 Link, 없으면 router.back(). 깊은 페이지 표준 UX
- 적용: `/chatbots/[id]`, `/admin/metrics` 에 BackButton 배치
- **모든 페이지 우상단 BrandThemeGallery 통일** : 로그인 / 회원가입 / 첫화면 / workspace 모두 같은 위치
- `/themes`, `/themes/{slug}` public 라우터 (비인증 조회 가능 — 로그인 전 테마 선택)

### 변경 (Changed)
- `BrandThemeProvider.applyTokens` : DEFAULT_TOKENS 와 깊은 병합 후 주입. 옛 변수 alias 확장(--shadow-*, --focus-ring, --success/warning/error/info, --border-strong, --motion-*)
- 비인증 사용자가 선택한 테마 → localStorage 저장, 로그인 후 backend 동기화

### 검증
- 회귀 29/29 통과
- /api/themes (no auth) 200, BMW tokens primary #1c69d4

---

## 2026-05-15 — UI 깨짐 정리 + 시드 대폭 확장 + Agent 견고화

### 수정 (Fixed)
- **code_sandbox docker 없을 때 agent 죽는 버그** : `FileNotFoundError` 가 전체 에이전트 루프를 fail 시켜 사용자에게 "답변을 만들지 못했습니다" 가 표시되던 문제. 해결:
  - `code_sandbox.run_code` : `shutil.which("docker")` 로 가용성 lazy 체크, 없으면 친절한 stderr 응답
  - `agent._exec_tool` 호출부 : 도구 예외 흡수 + `error_taxonomy.diagnose` 로 카테고리 복구 힌트 자동 주입

### 추가 (Added) — 시드 확장
- **themes**: 6 → 16 (+10): Linear, Notion, Vercel, OpenAI, Claude, Nike, Spotify, Tesla, Airbnb, Hyundai
- **skills**: 6 → 15 (+9): ko-business-email-formal / meeting-notes, dev-react-server-component / typescript-strict / sql-explain, devops-docker-multistage, docs-readme-skeleton, design-dark-mode-tokens / typography-korean
- 카테고리별 한 종 이상 시드 확보 → 등록 UI 기본값 풍부

### 변경 (Changed) — UI 일관성
- 전 페이지 H1 통일 (`text-2xl font-semibold tracking-tight`)
- max-width 정책: 폼/CRUD = 6xl, 데이터 그리드 = 7xl
- shared/ui : PageHeader / Alert / EmptyState 표준 컴포넌트 추가, 일부 페이지(skills/documents/schedules/studio) 적용
- 채팅 UI: 메신저 패턴(좌/우 끝까지), 메시지간 1.25rem 간격, max-width 80rem, ChatGPT 스타일 빈 상태 (예시 prompt 카드)
- Agent: 모호 질문에 clarifying question 의무화 (단순 인사·동의는 그대로 응답)

### 검증
- 회귀 29/29 통과
- 실제 채팅 호출: "잘 처리해줘" → LLM 이 명확화 질문 응답 (의도대로)
- 테마 전환: BMW id POST → primary #1c69d4 + radius 2px 토큰 적용 확인

### 솔직히 안 한 것 — 다음 PR 후보
- **RAG Phase 2** (sufficiency check + sub-query 재귀 검색) — rag-improvements.md 에 설계 완료. 구현 시 latency·비용 증가라 별도 검토 필요
- **skillsmp.com / getdesign.md 자동 동기화 워커** — `curate_skillsmp.py`, `curate_getdesign.py` 스텁만 (runner 가 자동 로드 시 모듈 누락 warning). 실제 외부 데이터 수집은 별도 PR

---

## 2026-05-15 — 회귀 검증 + UX 원칙 + Skills UI + Worker 본구현 + 이모지 정리

### 추가 (Added)
- `docs/ux-principles.md` : 중복 기능 허용 vs 금지 기준, 라벨링·시각·견고성 원칙, 의사결정 체크리스트
- `app/(workspace)/skills/page.tsx` + `features/skills/` : 스킬 마켓 페이지 + 카드 + 상세 + 등록 폼 (마크다운 textarea)
- AppShell 네비에 `/skills` 진입점 추가
- `worker/ingest/handlers/gen_recovery_tip.py` : 스켈레톤 → **본구현** (FailureLog 3건 누적 시 LLM JSON 모드로 condition/action 추출, RecoveryTip 자동 생성)
- `globals.css` `.input-text` 공용 인풋 스타일

### 변경 (Changed)
- 모든 docs 의 이모지 일괄 제거 (chatbot_sharing/rbac/erd 등 20개 파일) — UX 원칙 5장 적용
- 회귀 테스트 입력 보정 (`/tmp/regression_test.sh`) — error_taxonomy 실제 호출 형태, /chat 등 클라이언트 컴포넌트의 SSR shell 검증으로 변경

### 검증
- 회귀 테스트 29/29 통과 (인증/Skills CRUD/Themes/Tools/Models/모듈 임포트/페이지 shell)
- tsc 0 error
- worker handler 자동 로드 2/5 (parse_document stub + gen_recovery_tip 본구현)

---

## 2026-05-15 — UI 깨짐 fix (Tailwind content + /api 프리픽스)

### 수정 (Fixed)
- **"Unexpected token '<'" 에러** : Next.js 페이지(`/documents`, `/tools`, `/schedules`, `/admin` ...) 와 backend 엔드포인트가 **같은 path** 라 fetch 가 HTML 페이지를 받던 문제 해결.
 - 모든 API 호출이 **`/api/*` 프리픽스**를 거치도록 변경. next.config rewrites 가 `/api/:path*` → `${BACKEND}/:path*` 로 프록시.
 - `.env.local` : `NEXT_PUBLIC_API_URL=/api`
 - `shared/lib/api.ts` : 미설정 시 fallback `/api`
- **테마/캘린더/대부분의 grid 깨짐** : Tailwind `content` 에 `features/`, `shared/` 빠져있어 새 위치 컴포넌트의 className 이 빌드 안 되던 문제. 두 경로 추가.
- **ThemePicker "테마" 글자 wrap** : 좁은 컨테이너에서 2줄로 깨지던 문제. `white-space: nowrap; flex-shrink: 0;` 추가.
- **Next.js dev indicator** ("Route Static / Try Turbopack") 끄기 : `devIndicators: false`.

### 검증
- 통합 smoke test 40/40 통과
- `/api/auth/login`, `/api/documents`, `/api/tools`, `/api/schedules`, `/api/skills`, `/api/themes/active` 모두 200

---

## 2026-05-15 — 견고성 강화 (logging + theme apply + RAG adaptive + worker)

### 추가 (Added)
- **Frontend Theme apply** : `shared/theme/BrandThemeProvider.tsx` — `/themes/active` 의 tokens 를 CSS 변수로 자동 주입. 로그인 시 `auth:login` 이벤트로 재로드.
- **Frontend Error Boundary** : `app/error.tsx` — 페이지 충돌 시 친절한 화면 + 콘솔에 stack + digest + requestId 자세히 출력.
- **Frontend api 로깅** : `apiFetch` 에 색깔 콘솔 로그 (요청/응답/네트워크 실패 진단). `NEXT_PUBLIC_API_DEBUG=0` 으로 끔.
- **ApiUserError 확장** : `requestId`, `path` 보관 → backend 로그 추적 가능
- **RAG 적응형 임계값** : `reranker.adaptive_min_score` — 결과가 부족하면 단계적 임계값 하향 (RAGFlow `insert_citations` 패턴)
- **Worker 스켈레톤** : `worker/ingest/runner.py` + handlers/ (parse_document, gen_recovery_tip 스텁)
 - Postgres `SELECT ... FOR UPDATE SKIP LOCKED` 로 동시 다중 워커 안전
 - max_attempts 재시도, 실패 시 영구 failed
 - SIGTERM 시 현재 잡 끝까지 + 종료

### 변경 (Changed)
- `frontend/next.config.ts` : `rewrites` 로 backend 프록시 (8000 포트 외부 노출 불필요) + `allowedDevOrigins` 추가
- `frontend/shared/lib/api.ts` : `API_URL` 의 `||` 를 `!== undefined` 체크로 (빈 문자열 정상 처리)

### 검증
- 통합 스모크 테스트 `/tmp/smoke_test.sh` : 40/40 통과 (backend 28 + frontend 12)
- Backend logging : `RequestLoggingMiddleware` + `X-Request-ID` 헤더 + exception handler 견고성 확인
- Worker handler 자동 로드 : 2/5 등록 (parse_document, gen_recovery_tip 스텁만 제공)

---

## 2026-05-15 — Skills Marketplace + Themes + RAG 고도화

### 추가 (Added)

#### Skills 마켓플레이스 (DB 기반, 사용자 등록 가능)
- `Skill` 모델에 카테고리 트리/저작자/가시성/소스 필드 추가
- `SkillCategory` 테이블 (계층형 카테고리, skillsmp.com 구조 반영)
- `backend/config/skills/` 시드 디렉토리 — 카테고리/샘플 스킬 YAML
- Skill CRUD 라우터 (`/skills`) — 조회, 검색, 사용자 등록
- 출처 표시: `manual`, `seed`, `skillsmp`, `getdesign`, `auto_generated`

#### 브랜드별 테마 (design.md 패턴)
- `Theme` 테이블 (디자인 토큰 JSONB + 마크다운 본문 + 미리보기 이미지)
- `backend/config/themes/` 시드 — playstation/bmw/apple/github/stripe 등
- 사용자 `active_theme_id` 필드 (User 모델 확장)
- `/themes` 조회 라우터, `/themes/active` 활성 테마

#### RAG 고도화 (ragflow 패턴)
- 적응형 인용 임계값 (`reranker.py` — 고정 점수 → 결과에 맞춰 decay)
- 추후: 반복적 질의 분해 (sufficiency-aware multi-hop)

#### 문서
- `docs/architecture-decisions.md` — 내재화 vs 외재화 결정
- `docs/skills-marketplace.md` — Skills 시스템 사용/등록 가이드
- `docs/themes.md` — 테마 시스템 + design.md 표준
- `docs/rag-improvements.md` — RAG 고도화 계획
- `docs/changelog.md` — 이 파일

### 변경 (Changed)
- 없음 (기존 기능 모두 유지)

### 제거 (Removed)
- 없음

---

## 2026-05-15 — assi-loop 정확도/속도/비용 패턴 이식

### 추가 (Added)
- `services/error_taxonomy.py` (9개 에러 카테고리 + 한국어 복구 전략)
- `services/efficiency.py` (token padding + microcompact + economy model routing)
- `services/tool_verifier.py` (도구 결과 silent failure 감지)
- `services/learning.py` (Skills + FailureLog + RecoveryTip 매칭/기록)
- `features/chat/prompt_builder.py` (시스템 프롬프트 모듈화, PromptContext dataclass)
- DB 테이블: `skills`, `failure_logs`, `recovery_tips`

### 변경 (Changed)
- `agent.py`: 에러 분류기/microcompact/검증기 모두 와이어링
- `chat/router.py`: `_prepare_context` 4-way `asyncio.gather` (RAG + 메모리 + 스킬 + 복구)
- `chat/router.py`: economy routing 으로 사소한 쿼리 비용↓

---

## 2026-05-15 — 정리 작업 (Cleanup)

### 추가 (Added)
- `backend/config/tools.yaml`, `backend/config/models.yaml` (YAML 단일 진실 소스)
- `services/catalog_loader.py` (YAML 로더)
- `app/core/` (config, security, deps, error_handlers, middleware)
- `app/db/` (database, models)
- `app/features/{12개 도메인}/` (router 분리)
- `frontend/features/`, `frontend/shared/` (컴포넌트 도메인 분리)

### 변경 (Changed)
- 하드코딩 secret 제거 (JWT_SECRET, BOOTSTRAP_ADMIN_PASSWORD 필수화)
- `requirements.txt` 삭제 (uv.lock 단일화)
- 67개 Python 파일 import 일괄 갱신

### 제거 (Removed)
- `requirements.txt` (uv.lock 이 진실 소스)
- 하드코딩 `***REDACTED***`, `dev-secret-change-in-production`
