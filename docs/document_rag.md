# 문서 파이프라인 & RAG

## 0. 업로드 UI — ‘확인 후 적재’ 대기열

/documents 업로드는 *바로 올라가지 않는다*. 끌어다 놓거나 고른 파일은 **클라이언트 대기열**(`staged`)에 담기고, 사용자가 업로드 폴더를 **명시적으로** 고른 뒤(입력하기 싫으면 ‘미분류’ 선택 필수) **확인** 버튼을 눌러야 `POST /documents/upload` 가 호출된다. 대기열에서는 개별 파일 ✕·전체 비우기·적재 중 취소(AbortController)가 가능 — 실수로 대량 투입했을 때의 안전장치. 지원 확장자는 길게 나열하지 않고 ‘?’ 힌트(hover/focus 툴팁)로 안내한다.

## 1. 업로드 → 벡터스토어 자동 생성

```
사용자 파일 (PDF/DOCX/PPTX/XLSX/CSV/HWP/HWPX/TXT/MD/HTML/JSON/RTF)
 │  (UI: 대기열에 담김 → 폴더 선택 → ‘확인’ 클릭 시 전송)
 ▼
┌─────────────────────────────────────────────────────────────┐
│ POST /documents/upload (scope=team|personal) │
│ 1) 디스크 저장: data/uploads/{team_id}/{doc_id}_{name} │
│ 2) Document 행 생성 (status=processing) │
│ 3) asyncio.create_task(process_document_job(doc_id)) │
└─────────────────────────────────────────────────────────────┘
 │
 ▼
┌─────────────────────────────────────────────────────────────┐
│ services/ingest.py :: process_document_job │
│ extract_text → split_text → embed_texts(OpenAI) → │
│ DocumentChunk N건 INSERT │
│ status = ready (or failed, error_message 기록) │
└─────────────────────────────────────────────────────────────┘
```

## 2. 지원 포맷과 파서

| 확장자 | 라이브러리 | 비고 |
| ------ | ---------- | ---- |
| `.pdf` | `pypdf.PdfReader` | 페이지별 추출 |
| `.docx` | `python-docx` | 문단 텍스트 |
| `.pptx` | `python-pptx` | 슬라이드별 `shape.text` |
| `.xlsx` / `.xls` | `openpyxl` | 행 → tab 구분 |
| `.csv` | 기본 텍스트 디코딩 | UTF-8 fallback |
| `.html` / `.htm` / `.xml` | `beautifulsoup4 + lxml` | `get_text("\n", strip=True)` |
| `.json` | `json` | pretty print |
| `.txt` / `.md` / `.markdown` / `.rtf` | 기본 텍스트 | UTF-8 fallback |
| `.hwp` (**신규**) | `olefile` 파서 또는 `pyhwp` | 아래 전략 참조 |
| `.hwpx` (**신규**) | zipfile + XML (`section0.xml` `<hp:t>` 노드) | 내장 파서로 자체 구현 가능 |

### HWP 파싱 전략

HWP/HWPX는 한국의 한컴 오피스 포맷입니다. 두 가지 접근법을 병행합니다:

1. **우선** — 순수 파이썬 zipfile + XML 파서로 `.hwpx` 처리. 별도 의존성 없음.
2. **선택** — `.hwp`(구형 OLE 포맷)은 다음 중 하나를 선택:
 - `pyhwp`, `hwp5txt` 등 서드파티 라이브러리 (Python3 호환성 확인 필요)
 - 설치되어 있지 않으면 "지원하지 않는 형식" 메시지 후 실패 상태로 저장. 파서 교체는 `document_parser.py`에서 핫스왑.

실제 구현은 `backend/app/services/document_parser.py`에 추가됩니다. 의존성 미설치 상황을 대비해 **try/except ImportError**로 감싸서 런타임 오류가 아닌 사용자-친화적 실패 메시지로 전환합니다.

## 3. 청크화 & 임베딩

```
split_text(text, size=1200, overlap=200)
 → list[str]
embed_texts(chunks, batch_size=32)
 → list[list[float]] # 1536차원
```

- OpenAI `text-embedding-3-large`, `dimensions=1536`(HNSW 차원 제한 2000 내).
- 배치화로 업로드 지연 최소화.

## 4. 삭제 연동 (ON DELETE CASCADE)

`models.py`에서 다음 관계가 CASCADE로 설정되어 있습니다:

```python
class Document(Base):
 chunks = relationship("DocumentChunk", back_populates="document",
 cascade="all, delete-orphan", passive_deletes=True)
 shares = relationship("DocumentShare", back_populates="document",
 cascade="all, delete-orphan", passive_deletes=True)
 chatbot_links = relationship("ChatbotDocument", back_populates="document",
 cascade="all, delete-orphan", passive_deletes=True)
```

그리고 FK 선언에 `ondelete="CASCADE"`를 추가합니다:

```python
document_id = mapped_column(UUID, ForeignKey("documents.id", ondelete="CASCADE"), nullable=False)
```

라우터 `DELETE /documents/{id}`는 다음을 수행합니다:

1. 권한 검증 (owner 또는 team_admin 이상)
2. `await db.delete(doc)` → SQLAlchemy가 cascade로 `document_chunks`, `document_shares`, `chatbot_documents` 삭제
3. 디스크 파일 `path.unlink()`

### 검증 포인트
- `document_chunks`에 남은 행이 없어야 함 → HNSW 인덱스에서도 자동 제거(pgvector는 삭제된 행을 백그라운드 vacuum에서 정리).
- 이벤트 소싱 등 별도 벡터스토어를 쓰게 되면 동일하게 삭제 훅이 필요. 현재는 PostgreSQL 단일 저장소이므로 추가 작업 없음.

## 5. RAG 검색 — 일반 모드

`services/rag.py :: hybrid_search_chunk_ids(user_id, team_id, query, query_embedding)`

1. 가시 문서 집합 `V` 계산
 - `team` 스코프 문서 전체 +
 - 본인 `personal` 문서 +
 - 본인에게 공유된 문서 (`document_shares`)
2. 밀집(HNSW): cosine 거리 기준 상위 24
3. 어휘(tsvector): `plainto_tsquery('simple', query)` + `ts_rank_cd` 상위 24
4. RRF 병합(k=60) → 최종 8개

## 6. RAG 검색 — 챗봇 모드 (신규)

`services/chatbot_rag.py :: search_for_chatbot(chatbot, user, db, query)`

```python
if chatbot.rag_scope == ChatbotRagScope.linked_only:
 visible_doc_ids = <chatbot_documents.document_id where chatbot_id=X>
elif chatbot.rag_scope == ChatbotRagScope.owner_visible:
 visible_doc_ids = <owner에게 보이는 모든 문서>
else: # team_all
 visible_doc_ids = <팀 전체 문서>
```

- Public 챗봇 + `linked_only` 조합이면 사용자가 누구이든 오너가 연결한 문서만 본다 — 사용자 요구사항("Public은 내가 넣은 문서만 바라보게끔") 충족.
- Private 챗봇은 오너만 쓸 수 있으므로 스코프와 독립적으로 격리됨.

## 7. 실패/재처리

- `process_document_job` 예외 발생 시 `status=failed` + 사용자용 메시지 저장.
- 재처리는 `POST /documents/{id}/reprocess` (선택) — 구현 시 기존 청크 삭제 후 재실행.
- 파싱 불가 형식은 업로드 단계에서 일찍 거부(400 Bad Request)하여 잡 실패 비율 관리.

## 8. 성능 노트

- 청크 당 임베딩 1회. 임베딩은 API 호출 비용이 크므로 동일 파일 재업로드 시 중복 방지를 위해 `documents.content_hash`(SHA256) 컬럼을 추가하는 것을 권장(후속 작업).
- HNSW `ef_search`는 기본 40; 정확도가 부족하면 챗봇 검색 경로에서만 `SET LOCAL hnsw.ef_search = 80` 옵션을 적용.
- 대용량 파일(>5MB) 업로드는 업로드 직후 200을 반환하고 백그라운드 잡으로 처리(현재 구조 그대로).
