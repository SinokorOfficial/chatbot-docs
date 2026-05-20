# faqs — 챗봇 FAQ + 기능 요청 + AI 답글

같은 파일(`features/faqs/router.py`) 이 두 가지 다른 게시판을 책임집니다.

1. **챗봇별 FAQ** (`ChatbotFaq` + `FaqComment`) — 각 챗봇 페이지에서 "이 챗봇에 대한 질문" 을 모음. AI 자동 답글 가능.
2. **공개 FAQ / 기능 요청** (`FaqPost`) — 운영팀에 요청·문의하는 전사 게시판.

두 게시판의 라우트 prefix 가 다릅니다 — `/chatbots/{id}/faqs` vs `/faq/posts`.

## 1. 챗봇 FAQ — 라우트

| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/chatbots/{cb_id}/faqs` | 챗봇별 FAQ 목록 (정렬·필터 쿼리) |
| POST | `/chatbots/{cb_id}/faqs` | 질문 생성 (open 상태) |
| GET | `/faqs/{id}` | 단건 + 댓글 묶음 |
| PATCH / DELETE | `/faqs/{id}` | 작성자/관리자만 |
| POST | `/faqs/{id}/comments` | 댓글 추가 (사람 답변) |
| POST | `/faqs/{id}/upvote` | 좋아요 토글 |
| POST | `/faqs/{id}/accept/{cid}` | 작성자가 채택 → status='answered' |
| POST | `/faqs/{id}/ai-reply` | AI 자동 답글 트리거 |

## 2. AI 답글 (`services/faq_ai.py`)

```python
async def generate_ai_answer(
    db: AsyncSession,
    faq: ChatbotFaq,
    user: User,
    use_web_search: bool = False,
    use_rag: bool = True,
) -> dict:
    chatbot = await db.get(Chatbot, faq.chatbot_id)
    msgs = [{"role": "system", "content": "..."}, {"role": "user", "content": f"질문: {faq.title}\n\n본문: {faq.body}"}]

    citations = []
    if use_rag and chatbot.use_rag:
        ctx, cits = await search_for_chatbot(db, chatbot, user, faq.title)
        if ctx:
            msgs.insert(1, {"role": "system", "content": f"[관련 문서]\n{ctx}"})
            citations = cits

    tools = []
    if use_web_search:
        tools = [WEB_SEARCH_TOOL_SCHEMA]

    t0 = time.perf_counter()
    resp = await litellm.acompletion(model="openai/gpt-4o-mini", messages=msgs, tools=tools or None, ...)
    answer = resp.choices[0].message.content
    return {
        "answer": answer,
        "ai_meta": {
            "model": "openai/gpt-4o-mini",
            "citations": citations,
            "used_rag": bool(citations),
            "used_web": use_web_search,
            "tokens": {"prompt": resp.usage.prompt_tokens, "completion": resp.usage.completion_tokens},
            "latency_ms": int((time.perf_counter() - t0) * 1000),
        },
    }
```

답글은 새 `FaqComment` 로 insert + `author_kind='ai'` 표시. `ai_meta` 는 댓글 extra JSONB 에 저장 — UI 가 "이 답변은 AI" 라벨 + 인용 카드 표시.

## 3. 공개 FAQ — 라우트

| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/faq/posts` | 공개 게시판 목록 (status 필터) |
| POST | `/faq/posts` | 요청 작성 |
| PATCH | `/faq/posts/{id}` | 관리자만 — answer/status 갱신 |
| DELETE | `/faq/posts/{id}` | 작성자/관리자 |

`FaqPostStatus` enum — open / answered / closed. 관리자가 `PATCH {status: 'answered', answer: '...'}` 로 응답.

## 4. 좋아요 dedup

`ChatbotFaq.upvotes` 같은 카운터가 아니라 **별도 upvote 테이블** 패턴은 안 씀 — `upvote_count` integer 컬럼 + 좋아요 토글 시 row-level lock 으로 안전 증감.

```python
@router.post("/faqs/{id}/upvote")
async def upvote(...):
    res = await db.execute(
        update(ChatbotFaq).where(ChatbotFaq.id == id).values(upvote_count=ChatbotFaq.upvote_count + 1)
    )
```

**문제**: 같은 사용자가 여러 번 토글 가능. 사용자별 dedup 이 필요해 `UserSkillUpvote` / `UserToolUpvote` 같은 join 테이블이 별도로 존재 ([models 워크스루](backend-models.md)). FAQ 좋아요도 동일 패턴으로 옮기는 게 다음 작업.

## 5. 함정·결정

- **두 게시판이 한 파일** — `/faq/posts` (공개) 와 `/chatbots/{id}/faqs` (챗봇별) 가 헷갈리기 쉬움. 라우트 prefix 가 prefix=""(빈) 인 router 하나에 모두 들어가 있음. UI 도 `/faq` vs `/qa` (허브) vs `/chatbots/{id}/faqs` 로 분리됨.
- **AI 답글 토큰 비용** — gpt-4o-mini + RAG ctx 평균 2-3K 토큰. 분당 다발 호출 막으려면 features/faqs/router.py 에 rate limit 미들웨어 추가 권장.
- **답글 채택은 작성자만** — `accept/{cid}` 엔드포인트 가드 `if user.id != faq.author_user_id and user.role not in (team_admin, super_admin): raise 403`.

## 관련

- 챗봇 모델 — [models 워크스루](backend-models.md)
- 챗봇 RAG 검색 — `services/chatbot_rag.py`
- AI 답글 메타 표시 — frontend `chatbots/[id]/faqs/page.tsx`
