# RAG 고도화 계획 (RAGFlow 패턴 기반)

> [/tmp/ragflow-main](https://github.com/infiniflow/ragflow) 의 핵심 패턴을 우리 챗봇에 단계적으로 이식.

## 현 시점 능력 (이미 보유)

- **pgvector + HNSW** dense 검색 (1536d, OpenAI text-embedding-3-large)
- **하이브리드** : Dense + Lexical(GIN tsvector) + RRF 융합
- **LLM 리랭킹** : gpt-4o-mini 로 (query, chunk) 0-10 점 채점 + gap cutoff
- **적응형 임계값** : 결과 부족 시 단계적 임계 하향 (`adaptive_min_score`)
- **Sufficiency check + sub-query 1회** : 1차 결과 부족하면 LLM 이 sub-query 생성 후 재검색 (`rag_advanced.py`, depth=2 한정)
- **HyDE** : 질문을 LLM 가설 답변으로 변환 후 임베딩 → dense recall 향상 (Gao et al. 2022, opt-in flag `rag_hyde_enabled`)
- **문서 파서** : PDF / DOCX / Excel / PPT / HWP / HWPX
- **OCR 폴백** : 텍스트 추출이 부족하면 Vision LLM 으로 페이지 캡션
- **챗봇 스코프** : `chatbot_documents` 로 검색 공간 사전 필터

## HyDE 적용 가이드

- **언제 켜나** : 짧은/대화식 질문이 많은 챗봇 (FAQ 봇, 고객지원). 기술 정확도 봇(`rag_hyde_enabled=False` 권장 — 가설 답이 오답 방향으로 검색을 끌 수 있음).
- **alpha** : 0.5 가 기본 (원본 ⊕ HyDE 균등). 0.7 은 보수적(원본 비중↑), 0.3 은 공격적(HyDE 비중↑).
- **모델** : `gpt-4o-mini` 권장 — 빠르고 한국어 자연. 비용은 LLM 1회 + 임베딩 1회 추가.
- **실패 fallback** : LLM 또는 임베딩 실패 시 자동으로 원본 쿼리만 사용 (모듈: `rag_hyde.py:embed_query_with_hyde`).

## RAGFlow 에서 가져올 것 — 우선순위 표

| # | 패턴 | RAGFlow 위치 | 가치 | 난이도 | 적용 단계 |
|---|---|---|---|---|---|
| 1 | **적응형 인용 임계값** | `rag/nlp/search.py:236-326` | HIGH | EASY | Phase 1 (이번 PR) |
| 2 | **질의 분해 + 충분성 체크** | `rag/advanced_rag/tree_structured_query_decomposition_retrieval.py:93-135` | HIGH | MEDIUM | Phase 2 |
| 3 | **계층형 청킹 (타이틀 인식)** | `rag/flow/chunker/title_chunker/hierarchy_chunker.py` | MEDIUM | MEDIUM | Phase 2 |
| 4 | **PageRank + 메타 부스트** | `rag/nlp/search.py:391` (`rank_fea`) | MEDIUM | EASY | Phase 1 (선택) |
| 5 | **레이아웃 인식 파싱 (deepdoc)** | `deepdoc/parser/{mineru,docling}_parser.py` | HIGH | HARD | Phase 3 (외재화 워커) |
| 6 | **GraphRAG** (엔티티 + 관계) | `rag/graphrag/` | MEDIUM | HARD | Phase 3 (조건부) |
| 7 | **다중 임베딩 모델 라우팅** | (없음, 인프라만) | LOW | MEDIUM | 보류 |
| 8 | **HyDE (Hypothetical Doc Embeddings)** | (외부 패턴 — Gao 2022) | HIGH | EASY | Phase 1.5 (완료, opt-in) |

## Phase 별 정리

### Phase 1 — 즉시 (내재화)
**목적**: 기존 RAG 파이프라인을 건드리지 않고 정확도 추가 향상.

- [x] 가시성-aware Skills 매칭 (이번 PR)
- [ ] 적응형 인용 임계값 — 현재 `rerank_min_score=3.0` 고정 → 결과 부족 시 자동 하향 (이번/다음 PR)
- [ ] PageRank/upvote 메타 부스트 — rerank 점수에 `+ 0.5 * log(1 + upvote_count)` 가산

### Phase 2 — 다음 PR (여전히 내재화)
**목적**: 멀티-홉 검색으로 복잡 쿼리 정확도 ↑.

- [ ] **Sufficiency check**: 1차 검색 후 LLM 으로 "이 결과로 답할 수 있나?" 판단
- [ ] **Sub-query 생성**: 부족하면 LLM 이 부족한 정보 항목 + 후속 쿼리 생성
- [ ] **반복 검색**: 최대 3회 (depth=3) 까지 재귀
- [ ] **결과 병합**: 중복 청크 dedup, RRF 재정규화

예상 효과: 복잡한 다단계 질문 (`"우리 작년 매출 추이를 보고, 올해 1월 수치와 비교해줘"`) 정확도 30%+ 향상.

### Phase 3 — 외재화 (vision/heavy compute)
**목적**: 표·그림 많은 PDF 의 구조 보존.

- [ ] **deepdoc 파서 워커** (별도 컨테이너): MinerU 또는 Docling 으로 layout 추출
- [ ] **테이블 셀 인식**: row/col span 보존된 마크다운 테이블 생성
- [ ] **그림 캡션 자동 생성**: Vision LLM
- [ ] **bbox 메타데이터**: 청크에 페이지 좌표 저장 → 검색 결과에서 페이지 미리보기

이건 GPU 가 있어야 빠르므로 별도 워커. 메인 응답 흐름 안 막음.

### 보류 (LOW priority)
- 다중 임베딩 모델 라우팅 — 단일 1536d 가 충분
- GraphRAG — KG 구축 비용 큼, 우리 도메인엔 과함

## Phase 1 작업 — 적응형 임계값 (다음 PR)

현 코드 (`backend/app/services/reranker.py:170-`)는 `min_score` 고정.

개선 안:
```python
def adaptive_min_score(
 ranked: list[tuple[UUID, float]],
 *,
 target_min: int = 3,
 floor: float = 1.0, # 절대 하한
 start: float | None = None,
) -> float:
 """rerank 결과가 target_min 미만이면 점수 하한을 단계적으로 내림."""
 if not ranked:
 return floor
 start = start if start is not None else _settings.rerank_min_score
 threshold = start
 while threshold > floor:
 passing = sum(1 for _, s in ranked if s >= threshold)
 if passing >= target_min:
 return threshold
 threshold -= 0.5 # 0.5 점씩 내림
 return floor
```

호출 측 (`gap_cutoff_scores`) 에서:
```python
floor = adaptive_min_score(ranked, target_min=min_n)
filtered = [(cid, s) for cid, s in ranked if s >= floor]
```

이걸로 "결과 0건" 사례가 거의 사라지고, 임계값을 fixed 로 두는 보수성에서 벗어남.

## 측정

- `metrics` 테이블에 `phase=rag` 메트릭 기록 중
- 추가할 지표:
 - 1차 검색 결과 수
 - 리랭킹 후 keep 수
 - 적응형 임계값 발동 횟수
 - 멀티-홉 깊이 (Phase 2 이후)

## 참고

- [RAGFlow GitHub](https://github.com/infiniflow/ragflow)
- 우리 RAG 핵심 흐름: [architecture.md](architecture.md) "런타임 흐름 ② 문서 업로드"
