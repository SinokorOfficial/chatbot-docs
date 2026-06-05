# Changelog

날짜는 YYYY-MM-DD, 가장 최신이 위.

## 2026-06-05 — 포맷별 페이지 분리 / OCR 폴백 매트릭스 추가 (docx 한계 명시)

“docx 첨부 시 챗 응답이 hang 걸린다” 사용자 보고를 계기로, *왜 docx 가 PDF 와 다르게 동작하는지* 한눈에 보이는 매트릭스를 [`docs/document_rag.md`](document_rag.md) §2 에 추가. 페이지 분리·citation 페이지 점프·Vision OCR 폴백 세 축으로 7개 포맷 비교. docx 한계 2가지(이미지 텍스트 무시 / citation 페이지 점프 불가) 와 해결 옵션 3가지(libreoffice 변환 / 페이지 break 마커 / 내장 이미지 OCR) 명시.

## 2026-06-04 (저녁) — 대화별 작업공간 격리 · 첨부 OCR 차단 · 단계별 지연 가시화

수백초 걸리던 챗 응답과 “모든 프로젝트가 한 패널에 섞이는” 작업공간 누적 문제를 한 번에 해소.

### Fixed
- **대화별 워크스페이스 1:1 분리** — `_ensure_conversation_workspace()` 가 새 대화에 *사용자의 가장 최근 활성* workspace 를 재사용하던 lazy bind 를 제거. `get_or_create_default_workspace` → `create_workspace` 로 바꿔서 **신규 대화 = 신규 workspace**. iphone-calculator/ 와 mario-deluxe/backend/ 가 한 사이드 패널에 섞여 보이던 누적이 사라집니다. 이미 ws 가 묶인 과거 대화는 그대로 보존. [backend/app/features/chat/router.py]
- **첨부 PDF Vision OCR 인라인 호출 차단(수백초 → 수초)** — `/chat/stream` 의 휘발성 첨부(`payload.files`) 에서 `parse_attachment_text` 가 Gemini Vision 페이지별 동기 호출을 돌려 큰 PDF 한 장이면 응답이 수백초까지 늘었습니다. `parse_attachment_text(allow_ocr=False)` 기본값으로 챗 경로에서는 **네이티브 텍스트 추출만** — OCR 이 정말 필요한 PDF 는 `/documents/upload` 인제스트 경로(백그라운드 잡)로 등록한 뒤 RAG 가 가져오게 합니다. 이미지(.png/.jpg) 첨부는 자동으로 `allow_ocr=True`. [backend/app/services/ingest.py, backend/app/features/chat/router.py]
- **단계별 지연 가시화** — `_prepare_context` 에 `search_ms` / `rerank_ms` 분리 측정 로그, 챗 라우터 첨부 처리에 `첨부 파싱 X ocr=Y Nms len=L` 로그 추가. “어디서 막혔는지 모름” 상태에서 LLM 호출 vs RAG vs 리랭커를 즉시 구분 가능. [backend/app/features/chat/router.py]
- **인제스트 OCR 동시성 4 → 8** — `_OCR_CONCURRENCY` 를 8 로 올려 대형 PDF 인제스트의 페이지 직렬 대기 감소. 챗 경로는 OCR 자체 차단이라 영향 없음, 문서 업로드 처리 속도만 ↑. [backend/app/services/ocr.py]
- **workspace race-condition lock** — SELECT FOR UPDATE 로 동시 요청 시 dangling ws 방지. 같은 대화에 대해 `/chat/stream` 이 동시에 두 번 들어와도 `_ensure_conversation_workspace` 가 Conversation 행을 잠그고 직렬화되어, 중복 workspace 가 생성된 뒤 한쪽이 버려지는 dangling 상태가 사라집니다. [backend/app/features/chat/router.py]
- **스캔본 PDF 첨부 사용자 힌트** — 네이티브 텍스트 추출 실패 시 LLM 에 안내 메시지 명시 (silent failure 제거). `parse_attachment_text` 가 빈 문자열을 돌려주는 스캔본 PDF 의 경우 LLM 프롬프트에 `[첨부 문서: X] ⚠️ 텍스트 추출 실패 — /documents 인제스트 사용` 을 끼워, 모델이 “파일 없음” 으로 오해하지 않도록 합니다. [backend/app/features/chat/router.py]
- **workspace_id 조기 SSE 발행** — 신규 대화에서 WorkspacePanel 이 빈 트리로 깜빡이지 않게 ws_id 를 agent 시작 직후 push. `gen()` 이 `_ensure_conversation_workspace` 직후 `{"workspace_id": ...}` 이벤트를 즉시 emit 해서, 프론트는 첫 agent step 을 기다리지 않고 `GET /workspaces/{id}/files` 를 띄울 수 있습니다. [backend/app/features/chat/router.py]
- **generation_ms 분리 측정** — record_llm meta 에 LLM 생성 시간만 분리 기록 (RAG/rerank 와 구분). 이미 분리된 `search_ms` / `rerank_ms` 와 짝을 맞춰 LLM streaming 구간만의 시간을 따로 집계 — 다음에 “수백초” 리포트가 들어왔을 때 어디서 시간을 썼는지 즉시 가릅니다. [backend/app/features/chat/router.py]
- **OCR concurrency 8 → 6** — rate-limit headroom 25%. Gemini Vision 레이트리밋에 대비해 동시성을 8 에서 6 으로 내려 약 25% 헤드룸을 확보하면서도 인제스트 throughput 은 거의 유지. [backend/app/services/ocr.py]

## 2026-06-04 — 클로드 코드 모드 분리 · 작업공간 영속 · 타이포 일관화 · RAG 정확도 보강

대규모 UX/백엔드 정비. 단순 채팅과 코드 작업을 모드로 가르고, 작업공간이 대화별로 살아나고, 화면 헤딩이 한 톤으로 정돈됩니다. RAG 는 무조건 검색을 멈추고 페이지 단위 인용으로 점프합니다.

### 신규 기능 (Added)
- **‘클로드 코드 모드’ 토글 + 배너** — /chat 입력창 위에 별도 모드 배너를 추가. 켜면 우측 `WorkspacePanel` 이 fullscreen 으로 펼쳐지고, Anthropic 모델만 노출 + 6단계 사고강도(none~max) + `/claude_code/account` 로 플랜·이메일 배지를 한 줄에 모은다. 키 미보유 시 등록 안내 배너로 분기. 모드 OFF 상태에서 산출물이 남아 있으면 ‘이전 클로드 코드 작업으로 돌아가기’ CTA 노출. 일반 채팅에서는 패널 자체를 렌더하지 않아 화면이 깔끔해진다. [chat/page.tsx, features/workspace/WorkspacePanel.tsx, globals.css]
- **`GET /claude_code/account` 엔드포인트** — 서버에서 `claude auth status --json` 을 5초 타임아웃으로 호출해 로그인 여부·인증 방식(claude.ai/api_key)·이메일·조직·구독 타입을 프록시. CLI 미설치/타임아웃/로그아웃은 `{available:false, reason}` 로 graceful 반환. [features/claude_code/router.py, main.py]
- **슬래시 명령 메뉴(`/`)** — 채팅 입력창에서 `/` 입력 시 `SlashCommandMenu` 팝오버. ↑↓/Enter/Tab/ESC 키보드 탐색, 핵심 6개(모드 토글·새 채팅·RAG ON/OFF·생성 중지·요약·번역)로 유지. 배너 위로 떠야 해서 chat-input-bar `overflow:visible` + backdrop-filter 제거로 stacking context 격리. 사용 시 `tried_slash` 퀘스트 진행도 기록. [features/chat/SlashCommandMenu.tsx, chat/page.tsx, globals.css]
- **게임화·라이브 통계·리더보드 위젯** — `LevelBadge`(features/gamification), `LiveHighlights`·`WeeklyLeaderboard`·`HeaderActivityTicker`(features/notices), `LiveStatsStrip`(features/auth). 로그인 화면 누적 통계 strip, /notices 라이브 하이라이트·주간 리더보드, 사이드바 LevelBadge, 헤더 활동 티커가 배치됨. `/stats/public`·`/stats/highlights`·`/stats/leaderboard`·`/stats/me/level` 4개 API 헬퍼와 타입을 shared/lib/api.ts 에 추가. [main.py 라우터 등록, features/stats/router.py, shared/lib/api.ts]
- **퀘스트 6종 + Walkthrough 분기 + ‘도움·커뮤니티’ NavMenu** — QuestId 에 five_documents/reach_intermediate/shared_skill/built_app/tried_slash/changed_theme 추가. Walkthrough 에 `branch` 스텝 모델 도입 → ‘채팅만 할게요’ vs ‘코드 프로젝트 만들기 → /mypage#api-keys’ 분기 버튼. 상단 NavMenu 에 ‘도움·커뮤니티’ 그룹과 ‘둘러보기 다시 보기’ 항목 추가. BrandThemeProvider/SkillEditor 등 행동 지점에서 `markQuestExplored` 호출. [features/onboarding/quests.ts, Walkthrough.tsx, EntryMissionBar.tsx, shared/layout/NavMenu.tsx]

### Changed (UX)
- **타이포그래피 `.type-*` 토큰 일괄 적용** — 산재해 있던 `text-lg font-medium`·`font-display ...` 류 임시 헤딩을 `.type-section`/`.type-card-title`/`.type-display`/`.type-body`/`.type-body-sm`/`.type-eyebrow` 로 통일. globals.css 의 `--stitch-font-body/display` 가 테마 토큰 `--font-display`·`--font-ui` 를 우선 참조하도록 폴백 체인 수정하고, BrandThemeProvider 가 `--font-display` 별칭을 실제 주입(이전엔 본문이 Geist 로 고정). 모든 워크스페이스 페이지·EmptyState·PageHeader·AppShell 일괄 치환. [globals.css, BrandThemeProvider.tsx, shared/ui/*, app/(workspace)/**/page.tsx 20+]
- **PageHeader `hint` 패턴 + HelpHint(?) 툴팁** — 장황한 설명을 한 줄 요약 + 옆의 `?` 호버로 옮김. PageHeader 시그니처에 `hint` prop, HelpHint 는 접근성 위해 h1 외부 형제로 배치. chatbots/tools/faq/mypage/skills 등 다수 페이지 적용. [shared/ui/PageHeader.tsx, shared/ui/HelpHint.tsx]
- **문서 picker — 폴더 트리 + 검색 + 무한 스크롤** — `DocumentPickerModal` 에 /documents 와 동일 구조의 왼쪽 폴더 트리 추가. `/documents/folders` 로 폴더 목록 + `folder` 쿼리로 백엔드 필터, 검색·페이지네이션(50건) 무한 스크롤 유지. 챗봇 상세에서는 `RagScopeChooser` 컴포넌트로 use_rag + rag_scope 통합, ‘고른 문서만’ 일 때만 picker 노출. [features/chatbots/DocumentPickerModal.tsx, RagScopeChooser.tsx]
- **문서 업로드 스테이징 + 검색 + 인라인 편집** — /documents 에 업로드 대기열(staged) 도입. 드롭/선택한 파일이 즉시 적재되지 않고 ‘확인’ 시 한 번에 commit(폴더 명시 필요), 파일명 검색은 `?q=` ILIKE 로 디바운스 전달. 설명 수정·새 폴더 입력은 window.prompt 를 걷어내고 인라인 편집(editingDescId/newFolderForId)으로 교체. [documents/page.tsx, features/documents/router.py, docs/document_rag.md 섹션0]
- **챗봇 가시성 라벨 통일 + 고급 설정 접기** — Visibility 라벨을 ‘나만/우리 팀/선택 팀(내가 고른 팀)’ 으로 통일. 생성 폼은 이름·설명·system prompt 만 노출하고 visibility/RAG 는 ‘고급 설정 ▾’ 안으로. 기본 모델은 `openai/gpt-5.4-mini` 로(저렴한 모델 우선). [chatbots/page.tsx, chatbots/[id]/page.tsx, ChatbotSelect.tsx]
- **Tools/Skills 보관함 제거 + 페이지네이션** — /tools 12개 단위 Pagination 추가(필터 변경 시 1페이지 초기화). 보관(즐겨찾기) 스코프·favorites 상태·`toggleFavorite` 핸들러와 /skills/[id] 상세의 ★ 버튼을 전부 제거 — 좋아요와 복사(fork) 두 가지에 집중. [tools/page.tsx, skills/[id]/page.tsx]
- **Team Audit 2단계 디렉터리** — /team 을 구성원 디렉토리(`/team/audit/summary`) → 선택 구성원의 대화 목록(`/team/audit/conversations?user_id=...&limit=100`) 2단계로 분리. 검색 placeholder 와 검색 대상(이름/이메일 ↔ 대화 제목/미리보기)도 단계마다 전환, ‘← 팀 구성원 전체’ 복귀 버튼. [team/page.tsx, features/team/router.py]
- **FileViewer 인라인 미리보기 강화** — PDF 는 인증 fetch → blob URL → `<iframe>` 으로 전체 페이지 인라인, .docx/.pptx/.xlsx/.hwp 등은 `GET /workspaces/{id}/files-extract/...` 로 텍스트 추출 미리보기. ‘채팅으로 돌아가기’ 버튼 + `jg-viewer-in` 슬라이드 애니메이션, 미지원은 다운로드 안내. [features/workspace/FileViewer.tsx, features/workspace/router.py]
- **RAG 토글 명칭 통일 + 챗봇 use_rag 자동 활성** — ‘회사 문서 보고 답하기’ → ‘회사 문서로 답하기’. 챗봇이 use_rag=true 면 선택 즉시 useRag 자동 ON(localStorage 전역 선호값은 보존). 챗봇 상세는 `use_rag && linked_only` 일 때만 ‘고를 문서’ 섹션 표시. [chat/page.tsx, chatbots/[id]/page.tsx, RagScopeChooser.tsx]
- **frontend dev — Turbopack 활성화** — `next dev -p 3001 --turbopack` 로 변경(Next 16 내장). HMR 속도 개선, 프로덕션 영향 없음. [frontend/package.json]

### Fixed
- **대화별 Workspace 영속화 + ESC 회귀** — `WorkspaceProvider` 에 `SET_WORKSPACE_ID` 액션 추가, 대화 전환 시 sandbox 파일을 비우고 inline 산출물만 보존. workspaceId 가 바뀌면 Provider 가 직접 `GET /workspaces/{id}/files` 를 호출해 트리 복원(패널 unmount 와 무관하게 항상 sync). 사이드바에서 과거 대화 클릭 시 `workspace_id` 주입(ADR-0006: `ConversationOut.workspace_id` 노출), claude_code 가 만든 파일 트리가 그대로 살아남. ws-fullscreen 에서도 좌측 사이드바 유지하도록 CSS 보강, 배너의 `<select>` 안에서 ESC 가 패널을 닫지 않도록 `isFormField` 가드 강화. [features/workspace/WorkspaceProvider.tsx, WorkspacePanel.tsx, chat/page.tsx, schemas.py, globals.css]
- **무조건 검색 → 조건부 RAG 플래너** — 신규 `services/rag_planner.py` (gpt-5.4-mini 1회 호출, `retrieve` + 지시어 치환된 `search_query` 동시 결정). chat_stream·chat_regenerate 양쪽에서 RAG 가 켜져 있을 때만 플래너 실행 — 잡담/요약/번역은 검색 스킵, 후속 질문은 맥락 반영 검색어로 RAG 호출. 실패 시 원 쿼리 폴백. [services/rag_planner.py, features/chat/router.py]
- **도구 화이트리스트 — 일반 채팅 최소, 챗봇은 등록 도구만** — `_compute_allowed_tool_slugs` 헬퍼 도입(stream/regenerate 공용). 챗봇 미지정이면 `claude_code`·`schedule_query` 두 핵심 도구만, 챗봇 지정이면 그 챗봇 등록·활성 도구만 화이트리스트로. `run_agent(allowed_tools=...)` 가 비어 있으면 `tools`/`tool_choice` 자체를 제거(빈 배열 거부 프로바이더 대비). 재생성 경로의 NameError 동시 수정. [features/chat/router.py, services/agent.py]
- **문서 인용 페이지 추적 — 적재·검색·응답 전 구간** — `extract_text_pages` 추가(PDF/PPTX 페이지·슬라이드 분리). 청킹 시 각 청크의 `meta.page` 저장, OCR 폴백은 `[페이지 N]` 마커로 페이지 번호 복원. `fetch_chunk_contents`·`build_numbered_context` 가 page 를 citation 에 실어 보내고, `_save_and_postprocess` 는 page·score 를 DB citation 에 보존. [services/document_parser.py, ingest.py, rag.py, features/chat/router.py]
- **RAG 출처 collapse 버그(여러 근거가 1개로 줄어들던 문제)** — `rag.py:gap_cutoff` 와 `reranker.py:gap_cutoff_scores` 의 동작을 뒤집어, 점수 차(gap)가 임계 미만일 때 `min_n=1` 로 줄이는 대신 *하드 캡까지 여러 출처를 노출*. 후보가 고르게 관련 있을수록 더 많이 보여주는 것이 의도임을 코멘트로 회귀 방지. [services/rag.py, reranker.py]
- **GPT-5.x 호환 — `max_tokens` → `max_completion_tokens` 일괄 교체** — litellm `acompletion` 호출 8곳에서 일괄 전환(faq_ai/ocr/reranker/title_gen/user_memory/workflow_engine/workflow_nl/rag_advanced/rag_hyde/ingest 요약). 커스텀 모델명 자동변환 실패로 발생하던 400 에러 차단. [services/*.py 10개 파일]
- **build intent gate + CitationPanel 페이지 점프 + 사소한 UX** — 채팅에서 ‘게임/앱/만들어줘’ 류 build intent 휴리스틱으로 감지해 보내기 전 키 보유 여부에 따라 buildGate 카드 표시. CitationPanel 타입에 `page` 필드, DocumentViewer 는 의미 토큰 기반 매칭 + 페이지 단위 에러 격리로 PDF 렌더 안정성 향상. 인용 칩은 ‘출처1’ 라벨 + rounded-md, LikeButton 이중 클릭 회귀(deps busy 제거), apiFetch/userError 의 AbortError 가독성 개선, RadialThemePicker 휠 한 틱당 한 칸 + 포커스 테마 배경색 무대 적용. [chat/page.tsx, ChatMessage.tsx, CitationPanel.tsx, DocumentViewer.tsx, LikeButton.tsx, api.ts, userError.ts, RadialThemePicker.tsx]

### DDL / Schema
- **users.level + users.level_reached_at** — `level`(int NOT NULL DEFAULT 0, overall 1~40 / 0=미초기화), `level_reached_at`(timestamptz nullable). `schema_upgrade.py` 의 idempotent `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` 로 자동 적용. stats achievement 피드가 최근 7일 레벨업을 읽어 ‘○○님이 중급 Lv.1 달성!’ 으로 노출. ERD 문서(docs/erd.md) 갱신 — 컬럼 타입/설명 포함. [db/models.py, services/schema_upgrade.py, docs/erd.md]
- **`ConversationOut.workspace_id` 노출 (ADR-0006)** — `workspace_id: UUID | None` 필드 추가. 프론트가 과거 대화를 다시 열 때 `GET /workspaces/{id}/files` 로 파일 트리를 복원. [schemas.py]

### API
- **신규 라우터 등록** — `claude_code`(앱 만들기 의도 게이트용 계정 조회), `stats`(`/stats/public` 비인증·30초 캐시 + `/stats/highlights` + `/stats/leaderboard` + `/stats/me/level`). docs/api.md 에 Stats 섹션과 Team Audit 디렉토리 드릴다운 문서화. [main.py, features/claude_code/router.py, features/stats/router.py, docs/api.md]
- **팀 감사 페이지네이션 + 사용자 필터** — `GET /team/audit/summary` 추가(구성원별 대화 건수·최근 활동 시각). 기존 `/team/audit/conversations` 는 `offset` + `user_id` 파라미터 확장. [features/team/router.py]
- **오피스/HWP 텍스트 추출 미리보기** — `GET /workspaces/{id}/files-extract/{path}` 추가. RAG 파서(`extract_text`) 재사용해 docx/xlsx/hwp 등을 20MB 까지 추출 → 200K 자 캡으로 반환. CPU 바운드 파서는 `asyncio.to_thread`. 미지원 형식은 415. [features/workspace/router.py]
- **문서 목록 검색(`q`)** — `GET /documents` 에 `q` 추가. `original_filename`·`folder`·`description` 세 컬럼 ILIKE OR. 기존 folder 정확일치·페이지네이션 유지. [features/documents/router.py]

## 2026-06-02 — 문서 기반 챗봇 선택 시 RAG 토글 자동 ON

### Fixed (UX)
- **문서 연결 챗봇을 고르면 ‘회사 문서로 답하기’ 자동 활성화** — 사용자가 문서를 연결해 둔 챗봇(`use_rag=true`)을 선택하면 RAG 토글을 자동으로 켠다. 토글이 RAG 마스터 스위치(`use_rag_effective = payload.use_rag and …`)라 꺼진 채 질문하면 문서 검색을 통째로 스킵하고 봇이 대화 기록만으로 환각 답변하던 문제를 방지. 전역 localStorage 선호값은 건드리지 않음(`setUseRag`만 호출, 수동 토글만 영구 저장). `chat/page.tsx` ChatbotSelect `onSelect`.
- **알려진 잔여 위험(미수정)**: 토글이 켜져 있어도 조건부 RAG 플래너(gpt-5.4-mini)가 *회상형* 질문("…라고 했었지?")을 "이미 대화에 있음"으로 보고 `retrieve=false`로 차단할 수 있음. 문서 연결 챗봇에서 플래너 게이팅을 우회하는 보강은 추후 옵션.

## 2026-06-02 — 출처 페이지 메타(정확 점프) 백엔드

### 신규 기능 (Added)
- **청크별 페이지 번호 저장 → 출처가 정확한 페이지로** — 적재 시 PDF/PPTX 를 **페이지(슬라이드) 단위로 청킹**하고 각 청크에 `meta.page` 기록(`document_parser.extract_text_pages`, OCR 은 `[페이지 N]` 마커로 복원). `fetch_chunk_contents`→`build_numbered_context` 가 `citation.page` 를 실어 보내고, PDF 뷰어가 그 페이지로 **바로 점프**(텍스트 매칭 불필요 → 래스터 PDF 도 정확). 기존 청크(페이지 메타 없음)는 토큰-겹침 폴백으로 계속 동작. **DB 스키마 변경 없음**(기존 JSONB `meta` 의 새 키). 대화 재진입 시에도 유지되도록 message.extra 에 page/score 보존.
- **검증**: 실제 PDF 적재 → chunk pages `[(0,1)…(7,8)]`, citation pages `[1,2,3]` 확인. 단일 페이지 포맷(docx/csv/…)은 기존과 동일한 청킹.

## 2026-06-02 — 출처 하이라이팅·페이지 이동 (토큰 겹침 매처)

### Fixed (중요)
- **출처 클릭 → 하이라이팅 + 정확한 페이지 이동 복구** — PDF 뷰어가 청크 텍스트를 *연속 부분문자열*로 찾던 방식은 슬라이드/멀티페이지에서 추출 순서·공백 차이로 매칭이 깨져 **아무 동작도 안 했다**(silent fail). 이를 **순서 무관 토큰-겹침 페이지 스코어러**로 교체: 청크 토큰과 가장 많이 겹치는 페이지를 골라 스크롤 + 해당 항목 하이라이트 + ‘출처N’ 배지. 페이지 점수는 페이지 전체 텍스트 토큰으로 계산(글자 단위 분할에도 robust). 백엔드 `chunk.meta.page` 힌트가 있으면 그 페이지를 우선(`citation.page` → PdfViewer `pageHint`). 기존 청크에도 재적재 없이 즉시 동작.
- **검증**: 실제 16페이지 인프라 PDF로 시뮬레이션 — 6개 청크 전부 정확한 페이지로 매핑(예: chunk#3→p4 overlap 162 vs 차순위 36).

## 2026-06-02 — 뷰어 끊김 수정 · 폼 간소화 · 인라인 편집 · 티커 마퀴

### Fixed
- **PDF 뷰어 ‘내려가다 중간에 끊김’** — 페이지 렌더 루프의 catch 가 루프 *바깥*에 있어 중간 한 슬라이드 렌더 실패 시 나머지 페이지가 안 그려졌다. **페이지별 try/catch + continue**로 격리(한 장 실패해도 끝까지 렌더), 첫 페이지 렌더 즉시 로딩 해제.
- 출처 클릭 → 하이라이팅 + 해당 페이지 스크롤은 기존 구현(pdfjs 청크 매칭 오버레이 + ‘출처N’ 배지). RAG 복구로 출처가 떠서 이제 동작.

### UX (Changed)
- **챗봇 만들기 폼 정보 과부하 완화** — 공개 범위·회사 문서 섹션을 **‘고급 설정’ 토글(기본 접힘)** 로. 첫 화면은 이름·모델·설명만, 토글에 현재 선택 요약(공개/문서) 표시.
- **window.prompt → 인라인 편집** — 문서 ‘설명 수정’과 ‘새 폴더’ 입력을 브라우저 prompt 대신 화면 안 인라인 입력(저장/취소)으로.
- **활동 티커 마퀴** — 우상단 티커 문구가 길면 점진적으로 좌측 스크롤(끝에서 멈춤→반복), 안 넘치면 그대로.
- **명칭 통일** — 채팅 토글 ‘회사 문서 보고 답하기’ → **‘회사 문서로 답하기’** (RagScopeChooser 와 일치).

## 2026-06-02 — 조건부·멀티턴 RAG · 다중 출처 · 오피스 뷰어

### Fixed / Changed
- **무조건 검색 → 조건부 RAG + 멀티턴** — `회사 문서 보고 답하기`가 켜져도 모든 메시지에 검색하던 것을, 싸구려 LLM *쿼리 플래너*(`rag_planner.plan_rag_query`)로 게이팅. 인사·감상·형식변환은 검색 스킵, 문서 질문만 검색. 동시에 "이거 어때?" 같은 후속 질문을 *대화 맥락 반영 독립 검색어*로 응축해 멀티턴 검색 정확도 향상. (chat stream + regenerate 양쪽 적용, 실패 시 원 쿼리 폴백) — 검증: 문서질문 retrieve=True, "안녕 고마워" retrieve=False.
- **출처가 전부 ‘1’ → 다중 출처(출처1·2·3)** — `gap_cutoff`가 점수가 비슷하면(=모두 관련) 청크를 1개로 collapse 하던 로직을 뒤집어, 뚜렷한 단절이 없으면 여러 출처를 노출. 검증: 동일 문서 다중 청크 질문에 citations=3. 출처 배지도 숫자만 → **‘출처N’** 라벨로 표기.
- **문서 뷰어 전 형식 지원** — 출처 클릭 시 DocumentViewer 는 이미 LibreOffice(soffice 설치 확인)로 DOCX/PPTX/XLSX/HWP→PDF 렌더. 작업공간 FileViewer 에도 오피스·HWP는 백엔드 추출 텍스트(`GET /workspaces/{id}/files-extract`, RAG 파서 재사용·새 의존성 없음) 미리보기 추가. PDF 는 인라인 iframe, 그 외 진짜 바이너리는 다운로드 안내.

## 2026-06-02 — 활동 티커 우상단 이동 · 문서 검색/내문서 · PDF 미리보기

### UX (Changed)
- **활동 티커 우상단 이동** — 좌측 사이드바에선 폭이 좁아 글자가 잘려, 우상단 floating pill 영역으로 이동(`HeaderActivityTicker`, 가로 280px·lg+에서 노출). 신규 등록건만 오른쪽→왼쪽 슬라이드.
- **문서 페이지: 검색 + 내 문서 강조** — 파일명·설명 검색창(백엔드 `?q=` 디바운스 300ms)으로 많은 문서에서 찾아 삭제. 내가 올린 문서는 행 배경 accent 틴트 + "나" 배지로 구분.
- **파일 뷰어 매끄럽게 + PDF 미리보기** — 작업공간 FileViewer 에 열림 애니메이션(fade/slide 240ms) 추가. PDF 는 raw 바이트를 blob 으로 받아 `<iframe>` 인라인 미리보기(이전엔 다운로드만). 오피스/HWP 등 미지원 형식은 "미리보기 미지원 — 다운로드하거나 채팅에서 물어보세요" 안내로 명확화.
- **출처 클릭 → 문서 보기** — RAG 출처 마커 [출처N] 클릭 시 DocumentViewer 모달(PDF/이미지/텍스트 렌더)로 이동. (RAG 복구로 이제 출처가 떠서 동작)

### 검증
- RAG 라이브 HTTP 경로 재확인 — 임시 팀 문서로 `use_rag` 채팅 시 citations=1 + "롤백 기준 5xx 2% 초과" 근거 답변 + 로그 `RAG ok chunks=1`. (검증 후 임시 문서 삭제)

## 2026-06-02 — RAG·자동제목 복구(근본원인) · 활동 티커

### Fixed (중요)
- **보조 LLM 호출 `max_tokens` 400 오류 일괄 수정** — 커스텀 모델명(gpt-5.4-mini/5.5 등)을 litellm 이 몰라 `max_tokens`→`max_completion_tokens` 자동변환을 못 했고, OpenAI 가 400(`Unsupported parameter: max_tokens`)으로 거부. 이 호출들이 try/except 로 조용히 삼켜져 *기능이 빈값으로 동작*하던 것을 전수 수정(`max_completion_tokens` 사용): reranker·title_gen·rag_hyde·user_memory·ingest·ocr·faq_ai·workflow_nl·workflow_engine·rag_advanced.
  - **RAG 미작동 복구** — reranker 호출이 매번 실패 → rag_context 빈값 → 챗봇이 "문서를 직접 보고 있지 않다"고 답하던 문제. 이제 내가 올린 문서(ready·16청크)에서 출처 4건 검색 확인. (기본 채팅 + ‘회사 문서 보고 답하기’ 체크 시 내 문서 즉시 활용)
  - **새 대화 자동 제목 복구** — title_gen 호출이 매번 실패 → 전부 "새 대화"였음. 이제 "피보나치 함수"처럼 대화 요약 제목 자동 생성.

### 신규 기능 (Added)
- **사이드바 ‘지금 막’ 활동 티커** — 채팅 왼쪽 사이드바에 신규 등록건(챗봇·도구·스킬)이 오른쪽→왼쪽 슬라이드로 한 줄씩 순환(`SidebarActivityTicker`). 예: "🤖 윤승현님이 챗봇 ‘test’ 등록 · 15시간 전". 성취/레벨업은 제외(등록건만).

### UX (Changed)
- 업로드 폴더 안내 문구를 사내 제품 톤으로 정정 — "싫으면 미분류" → "주제가 없으면 ‘미분류’를 선택".

## 2026-06-02 — 작업공간 전체화면 설명 · 슬래시 확장 · 퀘스트 다양화

### 신규 기능 (Added)
- **전체화면 실시간 설명** — 작업공간 전체화면에선 채팅이 docked 돼 설명 텍스트가 안 보였음. 도구 실행 화면 하단에 ‘실시간 설명’ 패널(`NarrationStreamPanel`)을 추가해 어시스턴트 narration 이 쭉쭉 흐르게. (전체화면일 때만; 작게/크게는 채팅이 보이므로 중복 방지)
- **파일 → 채팅 복귀** — FileViewer 헤더에 “💬 채팅으로” 버튼 추가(파일 닫고 작업공간을 작게 줄여 채팅 노출). 기존 ✕(파일 목록으로)·전체화면 “대화 펼치기”와 함께 복귀 동선 명확화.
- **슬래시 명령 대폭 확장** — 동작(작업공간 전체화면·다시 생성·생성 중지) + 코드 작업(코드 리뷰·설명·버그 수정·테스트 작성·리팩터링)을 `/` 메뉴에 추가. 기존 새채팅·자료토글·요약·이메일·할일·번역·앱 만들기와 합쳐 14종.
- **퀘스트 6종 추가** — 앱 만들기·스킬 공유·자료 5개·슬래시 명령·테마 변경·중급 레벨 달성. 클라이언트 행동 신호(슬래시 사용·테마 변경·스킬 생성·앱 만들기) + 서버 신호(문서 5개 소유·레벨 overall≥11). 다양한 기능을 두루 써보도록 유도.

## 2026-06-02 — 레벨업 알림(영속)

### 신규 기능 (Added)
- **레벨업 공지 알림** — `users.level`(overall 1~40) + `users.level_reached_at` 컬럼 추가(schema_upgrade idempotent DDL). `/stats/me/level` 호출 시 저장된 레벨과 비교해 *실제 레벨업*이면 `level_reached_at` 을 기록(최초 1회는 베이스라인이라 조용히 설정). `/stats/highlights.achievements` 가 최근 7일 레벨업을 "○○님이 중급 Lv.1 달성!" 으로 맨 앞에 노출. (ERD: [erd.md](erd.md) users 표 갱신 — 신규 컬럼 타입·설명 포함)

## 2026-06-01 — 의도 게이트 · 40레벨 · 검색형 선택기 · 감사 디렉토리

### 신규 기능 (Added)
- **앱 만들기 의도 게이트** — /chat 에서 “게임/앱/HTML 만들어줘” 류로 보이면(챗봇 미지정) 보내기 전에 확인 카드. Anthropic 키가 있으면 “네, 만들어 주세요”→claude_code 로 진행, 없으면 “키 등록하러 가기 →”(`/mypage#api-keys`)로 유도. “그냥 답변만”도 가능(휴리스틱 오탐 무비용). [chat/page.tsx `looksLikeBuildIntent`]
- **40레벨 시스템** — 초급/중급/고급/전문가 × 각 Lv.1~10. 활동량 XP(대화 + 제작물×15 + 받은 좋아요×8 + 문서×3)로 결정, 위 티어일수록 가파르게(고급/전문가는 기여까지 필요). `GET /stats/me/level` → `{tier,sublevel,overall(1~40),label,xp,next_label,progress}`. 사이드바 `LevelBadge` 가 “고급 Lv.3 · 다음까지 N%”.
- **공지 성취 피드** — 좋아요 3+ 받은 사용자 도구/스킬, 최고 활동가의 현재 레벨을 `/stats/highlights.achievements` 로 노출(라이브 피드에 합류). 챗봇 생성도 피드에 포함, 문구 다양화.
- **검색형 챗봇 선택기** — 네이티브 `<select>` → 커스텀 콤보박스(`ChatbotSelect`). 챗봇이 7개↑면 검색창 자동 표시(더블클릭으로도 열림).
- **감사 ‘대화 관리’ 디렉토리** — 전체 평면 로딩 → 구성원 디렉토리(`GET /team/audit/summary`: 대화한 사람·건수·최근). 구성원 클릭 시 그 사람 대화만 lazy 로딩(`/team/audit/conversations?user_id=&offset=`). 수백 명 규모 대비.
- **앱(도구) 마켓 페이지네이션** — /tools 에 12개씩 페이지 네비 추가(스킬은 기존 적용됨, 동일 구조).
- **재사용 `?` 도움말(HelpHint)** — PageHeader 에 `hint` prop 추가. 길게 늘어놓던 설명을 제목 옆 “?” 툴팁으로 접음(문서·챗봇·스킬·도구·워크플로·FAQ·마이페이지 적용).
- **상단 메뉴 ‘도움·커뮤니티’** — 최상단 네비 맨 오른쪽 그룹 추가(공지·물어보기·FAQ·메뉴얼·둘러보기 다시 보기). 이전엔 AvatarMenu/⌘K 에만 있었음.

### UX (Changed)
- **기본 모델 = 가장 저렴** — 사용자 노출 기본 모델을 gpt-5.5 → **gpt-5.4-mini**(약 33배 저렴)로. 채팅/챗봇 생성 폼 공통.

### Fixed
- **/chat 재생성 NameError** — 도구 화이트리스트(`allowed_tool_slugs`)가 stream 에만 정의돼 regenerate 에서 NameError 나던 것을 공용 helper(`_compute_allowed_tool_slugs`)로 수정.

## 2026-06-01 — 게임화 · 테마 풀반영 · 도구 정책 · 슬래시 명령

참여 유도(리더보드·레벨)와 동선 정리 일괄.

### 신규 기능 (Added)
- **주간 리더보드** — `GET /stats/leaderboard`(월 09:00 KST 기준 ‘이번 주’): 대화왕💬·열정왕🔥(비용)·자료왕📚 Top 3. /notices 상단에 메달과 함께 표시(`WeeklyLeaderboard`), 활동 없으면 숨김.
- **사용자 레벨** — `GET /stats/me/level`: 누적 대화 수로 초급🌱→중급⚡→고급🏆→마스터👑. 채팅 사이드바에 진척 바 배지(`LevelBadge`) — “다음 레벨까지 N회”로 사용 유도.
- **채팅 슬래시 명령** — 입력창에서 `/` 입력 시 *위로* 떠오르는 빠른 명령 메뉴(`SlashCommandMenu`): 새 채팅·자료 토글·요약·이메일·할 일·번역·앱 만들기. ↑↓·Enter·Esc 키 지원, 한글 IME 안전.
- **공지 라이브 피드 확장** — 도구·스킬에 더해 *챗봇 생성*도 노출. 문구를 인덱스로 돌려가며 조금씩 다르게(“만들었어요/공유했어요/추가했어요”) 최신순으로 흐르게.
- **문서 검색(q)** — `GET /documents?q=` 파일명·폴더·설명 부분일치 검색 구현(이전엔 무시됨).

### UX (Changed · Fixed)
- **/chat 기본 도구 정책** — 일반 채팅(챗봇 미지정)은 *외부 도구 없는 깔끔한 형태*. 핵심 동작(claude_code 앱 만들기·schedule_query 자연어 예약)만 항상 노출하고, web_search·이미지·Gmail·Slack 등은 **챗봇에 등록해야** 켜진다(`run_agent(allowed_tools=...)`). 챗봇 편집 ‘장착된 도구’에 도구 마켓 안내 추가.
- **테마 풀반영** — 테마 선택/스크롤 시 색뿐 아니라 **글꼴(`--font-display` alias 추가)** 과 **배경색**이 실제로 반영. 라디얼 피커 배경이 *포커스한 테마의 배경색*으로 부드럽게 전환(명도 대비 글자색 자동), 휠 스크롤은 *한 틱당 한 칸*으로 (색 2개씩 건너뛰던 버그 수정).
- **문서 고르기 = 폴더 트리** — 챗봇 문서 picker(`DocumentPickerModal`)에 좌측 폴더 트리 + 폴더별 필터(백엔드 `folder` 파라미터) 추가.
- **회사 문서 근거 선택 재설계** — “문서 검색 체크박스 + 별도 스코프 드롭다운” → 라디오 카드 한 묶음(`RagScopeChooser`): 안 봄 / 고른 문서만(선택) / 내 문서 전체(자동) / 우리 팀 문서 전체(자동). 자동 옵션은 고를 필요 없음을 명시. 생성·편집 폼 통일.
- **‘선택 팀’ 라벨 정정** — “선택 팀 — 우리 팀 + 고른 팀” → “선택 팀 — 내가 고른 팀”. 생성/편집 폼 가시성 옵션 통일.
- **업로드 취소 메시지** — `apiFetch` 가 취소(AbortError)를 서버다운/원문(“signal is aborted without reason”)으로 노출하던 문제 수정 → 호출자가 ‘취소’로 처리.
- **네비 명칭** — “내 프로젝트 (코드 마켓)” → “프로젝트 마켓” (NavMenu·⌘K).

## 2026-06-01 — UX 마감 정비 (온보딩·문서·마켓·로그인)

비전공자 신규 유저 동선을 끝까지 다듬는 일괄 정비.

### 신규 기능 (Added)
- **통계 엔드포인트** — `GET /stats/public`(비인증, 30s 캐시): 누적 대화수(=user 메시지)·활성 유저·문서 수 + 우수 기여자 Top 5. `GET /stats/highlights`(인증): 최근 사용자 등록 도구/스킬 + 오늘(KST) 가장 활발한 사용자. *읽기 전용 집계* — DB 스키마 변경 없음.
- **로그인 화면 라이브 통계** — 좌측 브랜드 패널에 누적 대화/동료/자료를 슬롯머신 카운트업 + ‘오늘의 활약’ 우수 기여자 순환(`LiveStatsStrip`). 데이터 0이거나 실패 시 *조용히 숨김*(로그인 방해 금지).
- **공지 동적 하이라이트** — /notices 상단에 “지금” 배너(`LiveHighlights`): 누가 어떤 도구/스킬을 등록했는지·오늘 가장 활발한 동료를 4초 간격 순환.
- **문서 업로드 ‘확인 후 적재’** — 끌어다 놓거나 고른 파일은 *대기열*에 담기고, 폴더를 명시적으로 고른 뒤 **확인**을 눌러야 실제 적재. 개별(✕)·전체 비우기·적재 중 취소 가능(실수 대량 업로드 방어). 폴더 미선택 시 적재 차단(‘미분류’ 명시 선택 가능).
- **/chat 코드 작업 가이드 분기** — 채팅 기본 투어 마지막에 분기 스텝: “채팅만 할게요” / “코드 프로젝트 만들기 →”(작업공간 설명 + API 키 등록 `/mypage#api-keys` 이동).

### UX (Changed · Fixed)
- **좋아요 두 번 클릭 버그 근본 수정** — 공유 `LikeButton` 의 동기화 effect deps 에 `busy` 가 있어 클릭 종료 시(busy:true→false) *갱신 안 된 부모 prop(stale)* 으로 되돌아가 좋아요가 풀리던 문제. deps 를 `[liked,count]` 로 한정 → 토글 후 prop 이 그대로면 effect 미실행, 서버 반영값 유지. (스킬·도구 공통)
- **둘러보기 전환 부드럽게** — 카드 위치 슬라이드(340ms) + 스텝 페이드/슬라이드-인 + 하이라이트 링/딤 패널 트랜지션 300ms. “확확 넘어가는” 느낌 완화.
- **문서 확장자 안내 ‘?’ 토글** — 길게 나열하던 지원 확장자 줄을 물음표 힌트로 축약(hover/focus 시 툴팁).

### 정리 (Removed)
- **‘보관(즐겨찾기)’ 개념 제거** — 도구/스킬 카드·상세의 ☆보관·보관함 탭·“내 보관함으로 복사” 전부 삭제(복사=fork 는 “내 ○○로 복사”로 정정). 백엔드 `/tools|skills/{id}/favorite` 엔드포인트·`user_*_favorites` 테이블은 *휴면*(호출 없음, 마이그레이션 불필요).

## 2026-06-01 — 기능 정비 + 보안 하드닝 + 신규 기능

5갈래 감사(기능버그·중복/UX·내용정합·기능 deepdive·데드코드)와 적대적 검증을 거친 일괄 정비.

### 보안 (Changed · Fixed)
- **교차팀 RAG 유출 차단** — `chatbot_service.resolve_rag_scope_doc_ids` 가 요청자 기준으로 동작. 챗봇 소유 팀이 아닌 사용자(shared 로 접근)는 스코프(team_all/owner_visible)와 무관하게 **챗봇에 명시적으로 연결된 문서만** 검색 → 소유팀 전체/개인 문서 유출 봉쇄.
- **SSRF 가드 신설** — `app/core/net_guard.py`. 커스텀 HTTP 도구(`tool_registry` webhook) + 워크플로 `http` 노드 양쪽에 적용. 사설/loopback/link-local/메타데이터(169.254) 차단 + DNS 리바인딩 방어(해석된 모든 IP 검사).
- **대화 복구 인가 재검증** — `/chat/stream` 기존 대화 재개 시 `get_chatbot_for_use` 로 인가 재확인. 공유 취소/가시성 변경 후 과거 대화 id 로 우회하던 경로 차단(삭제된 챗봇은 일반 채팅 폴백).

### 기능 (Fixed)
- **/chat → Claude Code** — claude_code 실패가 채팅 전체를 죽이던 버그 수정(`AgentErrorEvent`=`{agent_error:true}` 로 패널 격리). 스트리밍 경로에 빠져 있던 `apply_diff_to_db` 추가 → 워크스페이스 파일 메타가 DB 영속(새로고침/재진입 시 복원).
- **좋아요(FAQ 질문·댓글)** — 매 클릭 +1 무한 증가 → `user_faq_upvotes`·`user_faq_comment_upvotes` 조인 테이블로 **유저별 토글/중복방지** + `user_upvoted` 하트 상태. 프론트 연타 가드(in-flight) 추가.
- **HWP 압축 문서** — `document_parser` zlib 미import 로 인한 NameError 크래시 수정.

### 신규 기능 (Added)
- **챗봇 교차팀 공유(shared)** — `ChatbotVisibility.shared` + `chatbot_team_access` 테이블 + `extra_team_ids`(POST/PATCH /chatbots, shared 아니면 자동 청소) + `GET /team/list`(picker, invite_code 비노출). 가시성=나만/우리팀/선택팀(**전체 팀 선택 가능**). 생성·편집 페이지 모두 팀 multi-select + 전체팀 토글.
- **문서 폴더** — `documents.folder` 컬럼 + `GET /documents/folders` + `PATCH /documents/{id}/move` + 업로드 폴더 지정. /documents 좌측 폴더 트리 + 우측 파일목록 + 드래그성 이동.
- **대화 전체 삭제** — `DELETE /conversations/all`(내 전체 대화+메시지) + /chat 목록 "전체 삭제".
- **Interactive Walkthrough** — 최초 로그인 "가이드 받기/바로 쓰기" → 스포트라이트 스텝 투어. + **칭찬/축하 컨페티 이펙트**(`celebrate()`: 투어 단계·챗봇 생성 등 수행 시).
- **워크플로 캔버스 재구축** — 자체 SVG 폐기 → **React Flow(@xyflow/react) 2D**(미니맵·그리드스냅·드래그연결·실행상태배지) + **three/@react-three/fiber 3D**(위상 레이어 그래프, `next/dynamic ssr:false`). 도구노드에서 builtin 도구 비활성(워크플로 실행 불가 안내).

### UX (Changed)
- 추론 경과 타이머(`생각 중 · N초` / `응답 중 · N초`), 채팅 자동스크롤은 하단 근처일 때만 + "맨 아래로" 버튼.
- 파괴적 액션 `confirm()` 8곳 → 테마 일관 **ConfirmModal**(`confirmDialog`). 에러(`FailureBanner`·`ServerStatusToast`)를 하단 인라인 → **화면 정중앙 패널**.
- 워크스페이스 패널이 열려 있을 때 우상단 플로팅 pill(관리자 배지)이 헤더 버튼과 겹쳐 클릭 막히던 문제 해결(`ws-open` 시 pill 숨김).
- 공지: /chat 미니리스트에 제목+요약, /notices 제목→클릭 본문 펼치기.
- 로그인 폼 접근성(`role=alert`·`aria-invalid`·오류 시 첫 필드 focus), 모바일 미션 그리드 반응형.

### 모델 (Changed)
- 카탈로그 전면 최신화 — OpenAI GPT-5.5/5.4-mini/5.5-pro, Anthropic Opus 4.8·Sonnet 4.6·Haiku 4.5, Google Gemini 3.5 Flash·3.1 Pro·3.1 Flash-Lite. `efficiency` 다운그레이드 맵·`metrics` 가격표·fallback 체인·내부 보조모델 기본값 동기화.

### 정리 (Removed)
- 데드코드 제거 — `claude_runner` legacy blocking 구현 345줄(`_legacy_blocking_run_claude_code`·`_parse_stream_json`), 미사용 스키마 4(ConversationCreate·ApprovalDecisionIn·ChatStreamIn·ChatFileIn)·함수 2(`_assert_usable`·`all_tools`), 프론트 미사용 export(Badge·CardHeader·SectionTitle), `ThemePicker`, `Chatbot.vectorstore_partition` 컬럼.

## 2026-05-20 — 종합 감사 결과 반영 (PR1–9)

운영 가능 수준으로 끌어올리기 위한 9개 PR 머지 시퀀스.

### 보안 (Added · Changed)
- **시크릿 안전망** (PR1) — `.env.example` placeholder 값으로는 부팅이 거부됨. `JWT_SECRET` 32자 미만/placeholder, `BOOTSTRAP_ADMIN_ENABLED=true` 와 약한 비밀번호 조합 모두 차단. 운영에서 약한 시크릿이 흘러드는 경로 폐쇄.
- **tools 커스텀 도구 SSRF 차단** (PR2) — `app/core/ssrf.py` 신설. 등록 시점 + dispatch 직전 양쪽에서 사설/메타데이터/loopback IP 차단. `visibility=team/public` 은 `team_admin` 이상만 가능. `SSRF_ALLOWED_HOSTS` env 로 사내 tool-server 화이트리스트.
- **HttpOnly cookie 보조 인증** (PR7) — `/auth/login`·`/register` 응답에 `HttpOnly · SameSite=Lax` 쿠키 추가 발급. `/auth/logout` 이 쿠키 즉시 폐기. 기존 `localStorage` 흐름은 그대로 호환. 프론트 `apiFetch` 는 `credentials:'include'`.
- **CORS 외부화** — `BACKEND_ALLOWED_ORIGINS` env(콤마 구분)로 운영 도메인을 코드 변경 없이 추가.

### 운영 (Added)
- **CI 게이트** (PR3) — `.github/workflows/backend-test.yml`(uv + pytest 단위), `frontend-build.yml`(tsc `--noEmit` + lint + build).
- **헬스/레디** (PR3) — `/healthz`(liveness, 의존성 검사 없음), `/readyz`(DB ping, 실패 시 503). 기존 `/health` 호환.
- **docker-compose 정상화** (PR8) — `backend/Dockerfile` 신설(uv sync --frozen + tini PID1). `api`·`worker` 서비스가 같은 이미지로, `--profile full` 로 일괄 부팅. db 만 띄우는 기존 흐름 그대로.

### RAG (Changed)
- **citation 후검증** (PR4) — LLM 이 만든 `[출처N]` 마커 중 citation 범위 밖 N 을 응답 저장/전송 직전에 제거. citation 0개일 때 모든 마커 제거.
- **재랭킹 비용 디폴트 조정** (PR4) — `chunk_size` 1200 → 800, `chunk_overlap` 200 → 120, `rerank_candidates` 16 → 8. 채팅 1건당 재랭킹 비용 약 70% 감소 (운영 `.env` 로 언제든 덮어쓸 수 있음).
- **init-db.sql 보강** (PR4) — HNSW + GIN 인덱스 DDL 을 `IF NOT EXISTS` 로 명시. `ef_construction` 64 → 200 으로 튜닝하는 수동 재인덱싱 SQL 가이드 주석 동봉.

### 아키텍처 (Refactor)
- **SSE 이벤트 스키마 단일 진실원** (PR5) — `app/schemas_sse.py` 에 9개 wire-format 모델 + `serialize_*` 함수. chat router 의 분기/즉석 직렬화 8군데 교체. 작업 중 `StepInfo.tools` 가 `list[str]` 로 잘못 좁혀 운영에서 ValidationError 가 났을 회귀를 단위 테스트가 잡아 수정.
- **db.models 패키지 전환** (PR6) — `db/models.py`(1330줄) → `db/models/` 패키지 + `_models_full.py` 재수출. 이후 도메인 분리(`_user.py`, `_chatbot.py` …) 의 발판. 기존 `from app.db.models import User` 모두 그대로 동작.

### 문서 (PR9, 이 항목)
- `docs/api.md` 의 SSE 이벤트 표를 `schemas_sse.py` 기반으로 9개 모두 명시. 누락됐던 `pending_schedule`·`citations` 추가.
- `/healthz`·`/readyz` 와 `/auth/logout` 명시. SSRF 가드 정책 안내.
- `docs/deployment.md` — `docker compose --profile full` 절차 추가.

### 테스트 (Added)
- 단위 50+ 추가: SSRF(23), RAG citation/chunking(9), SSE wire-format(12), models package(3), auth cookie(9), compose topology(8).

### 검증
- 모든 단위 테스트 통과 (`uv run pytest tests/test_ssrf.py tests/test_rag_quality.py tests/test_sse_events.py tests/test_models_package.py tests/test_auth_cookie.py tests/test_compose_topology.py`).
- `app.main` import / 126 route 등록 정상.
- compose YAML 구조 회귀 가드 통과.

### 사용자 직접 처리 필요
- `.env` 의 OpenAI/Anthropic/Google/JWT/Bootstrap 비밀번호 **수동 회전** (히스토리에 푸시된 적은 없음을 확인했지만 권장).
- 운영 진입 시 `BOOTSTRAP_ADMIN_ENABLED=false` 로 명시.
- 사내 tool-server 있다면 `SSRF_ALLOWED_HOSTS` 등록.
- HNSW recall 개선 필요 시 `init-db.sql` 의 재인덱싱 SQL 수동 실행.

## 2026-05-20 — 인서비스 종합 가이드 + 코드 워크스루 16편 + 페이지 전수 스크린샷

### 추가 (Added)
- **`/guide` 인서비스 종합 가이드** (`frontend/app/(workspace)/guide/page.tsx`) : 23개 화면을 그룹별로 늘어놓고 **각 페이지의 목적·할 수 있는 일·내부 동작·관련 코드 경로** 를 한 화면에 정리. 캡처 이미지(`/manual-shots/<slug>.png`) 포함. 좌측 sticky 목차.
- **헤더 nav 에 "가이드" 진입점** 추가.
- **23개 페이지 전수 스크린샷** : Playwright (headless chromium) 로 인증된 세션에서 일괄 캡처 → `frontend/public/manual-shots/` (서비스용) + `docs/screenshots/` (mkdocs 사이트용) 양쪽 저장.
- **자동 재생성 스크립트** (`scripts/regen-shots.sh`) : 로그인 → storage.json → 23 페이지 순차 캡처. 새 페이지 추가 시 ROUTES 배열만 갱신.
- **코드 워크스루 16편** (`docs/code-walkthrough/`) — "한 줄씩 뜯어보기" 깊이.
    - 큰 그림: `backend-main` · `backend-models` · `backend-deps`
    - 백엔드 도메인: `backend-auth` · `backend-chat` · `backend-documents` · `backend-workflows` · `backend-faqs` · `backend-misc` (작은 모듈 묶음)
    - 프론트엔드: `frontend-shell` · `frontend-api` · `frontend-chat` · `frontend-workflow-canvas` · `frontend-theme` · `frontend-design`
- **mkdocs nav 에 "코드 워크스루" 섹션** 추가 → GitHub Pages 사이트에서도 탐색 가능.

### 검증
- 프론트 `next build` 통과 (`/guide` 11.9KB static)
- 백엔드 pytest 27/27 통과
- `mkdocs build --strict` 통과 (양쪽 리포)

## 2026-05-20 — mkdocs 기반 GitHub Pages 가이드 사이트

### 추가 (Added)
- **mkdocs.yml** : Material 테마(한국어, Pretendard, Mermaid), `docs/` 전체를 사이트 구조로 정리한 nav. 외부 노출 금지 문서는 `exclude_docs:` 처리 (`admin.md` · `operations.md` · `deployment.md` · `README.md`).
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
