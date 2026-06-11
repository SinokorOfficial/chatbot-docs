# Skills 마켓플레이스

> **개발 전용(운영 미노출)** — 스킬 마켓플레이스는 ADR-0009 의 개발 티어 전용 기능입니다. 운영(`NEXT_PUBLIC_FEATURE_SET=rag` / `FEATURE_SKILLS=off`)에서는 `main.py` 가 `/skills` 라우터를 등록하지 않아 404 로 차단되고(`DEV_ONLY_PATH_PREFIXES`·`isRagOnly`로) 메뉴/화면도 숨겨집니다.

> 챗봇이 사용자 쿼리에 답할 때 자동으로 참조하는 **노하우 카드**.
> 한국 비즈니스, 개발 패턴, 디자인 가이드 등 무엇이든 마크다운으로 등록.

## 데이터 모델

### `skill_categories` 테이블
- skillsmp.com 카테고리 트리 반영 (Tools, Business, Development, ...)
- 한국 비즈니스 카테고리 별도 추가 (`korean-business`, `korean-finance`)
- 계층형: `parent_id` 로 부모 카테고리 지정

### `skills` 테이블
- `name`, `description`, `content` (마크다운)
- `category_id` (선택, FK to skill_categories)
- `tags`, `triggers` (JSONB, GIN 인덱스로 빠른 검색)
- `source` enum: `manual` / `seed` / `user` / `skillsmp` / `getdesign` / `auto_generated`
- `visibility` enum: `team` / `org` / `public`
- `author_user_id` (사용자 기여 추적)
- `upvote_count`, `success_count`, `fail_count` (커뮤니티/학습 메트릭)

## 라이프사이클

### 시드 (Seed)
- `backend/config/skill_categories.yaml` — 카테고리 트리
- `backend/config/skills/*.yaml` — 시드 스킬 (visibility=org/public)
- 부팅 시 `services/seed_loader.py` 가 자동 upsert (중복 slug 건너뜀)

### 자동 수집 (Auto-Ingest)
- 워커 핸들러 : `worker/ingest/handlers/curate_skillsmp.py`, `curate_getdesign.py`
- 외부 REST 시도 → 실패 시 큐레이트 yaml 폴백:
  - `backend/config/curate/skillsmp_known.yaml` (source="known-fallback")
  - `backend/config/curate/getdesign_known.yaml` (themes 테이블 upsert)
- 운영자가 코드 수정 없이 yaml PR 만으로 새 스킬/테마 추가 가능
- enqueue : `ingest_jobs` INSERT → Postgres `NOTIFY ingest_jobs` → 워커 즉시 처리
- 멱등성 : 같은 slug 면 skip (added/skipped 카운트 result 에 기록)

### 사용자 등록 (User Contribution)
- 누구나 `POST /skills` 로 등록 가능
- 기본 가시성: `team` (본인 팀에서만 노출)
- `org` 승격: `team_admin` 이상 필요
- `public` 승격: `super_admin` 필요

### 자동 생성 (Auto-Generated)
- `failure_logs` 가 같은 signature 로 N회 누적되면
- 별도 워커가 LLM 호출로 RecoveryTip 또는 Skill 자동 생성
- `source=auto_generated`, `auto_generated=True` 표시

## 매칭 흐름 (시스템 프롬프트 주입)

```
사용자 쿼리 → extract_keywords() → 토큰화
 ↓
 Skills 테이블 조회 (visibility-aware)
 ↓
 triggers/tags overlap 점수 + confidence 가중
 ↓
 상위 3개 선택 (4KB 제한)
 ↓
 시스템 프롬프트 ## 관련 스킬 섹션 자동 주입
```

## API

### 조회
```
GET /skills?category_id=<uuid>&source=skillsmp&q=keyword&limit=50
GET /skills/categories
GET /skills/{id}
```

### 등록 / 수정 / 삭제
```
POST /skills # 사용자 등록
PATCH /skills/{id} # 본인 작성 또는 관리자만
DELETE /skills/{id} # soft delete (is_active=False)
POST /skills/{id}/upvote # 좋아요
```

## 시드된 카테고리 (요약)

| 카테고리 | 위치 |
|---|---|
| 한국 비즈니스 | `korean-business`, `korean-finance` |
| Tools | `tools-cli`, `tools-automation` |
| Business | `business-memos`, `business-reports`, `business-comms` |
| Development | `development-backend`, `development-frontend`, `development-mobile` |
| Data & AI | `data-llm`, `data-engineering` |
| DevOps | `devops-cicd`, `devops-cloud` |
| Documentation | `documentation-tech` |
| Content & Media | `content-design`, `content-writing` |

전체 트리: 리포 내 `backend/config/skill_categories.yaml` 참조.

## 시드된 스킬 샘플

| slug | 카테고리 | 출처 |
|---|---|---|
| `ko-business-memo` | business-memos | seed (사내 표준) |
| `ko-business-report-trip` | business-reports | seed |
| `korean-stock-lookup` | korean-finance | seed |
| `dev-fastapi-async-best` | development-backend | skillsmp |
| `dev-pgvector-hnsw` | data-engineering | skillsmp |
| `design-brand-card` | content-design | getdesign |

## 향후

- [ ] skillsmp.com REST API 와 실제 동기화 (별도 워커)
- [ ] getdesign.md 사이트 크롤러 (별도 워커)
- [ ] FailureLog → 자동 RecoveryTip/Skill 생성 (별도 워커)
- [ ] 관리자 UI (스킬 승인, 카테고리 편집)
- [ ] 스킬 사용 메트릭 대시보드
