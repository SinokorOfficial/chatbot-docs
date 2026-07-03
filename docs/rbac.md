# RBAC · 권한 체계 · 가입 플로우

## 1. 역할 계층

```
super_admin (최종관리자)
 ↓ 팀을 생성하고 각 팀장을 임명할 수 있음
team_admin (팀장)
 ↓ 팀원을 초대/승인, 팀 리소스 감독
team_auditor (감사자, 레거시)
 ↓ (2026-07-03 정책 변경으로 대화 감사 권한 회수 — 열람 권한은 super_admin 전용)
team_member (팀원)
 - 일반 사용자: 본인 대화·개인 문서·개인 챗봇
```

- 상위 역할은 하위 역할의 작업을 수행할 수 있습니다.
- `team_auditor`는 역할 enum 은 유지되나, **대화 감사 권한이 회수되어**(2026-07-03) 현재 고유 권한이 없는 레거시 역할입니다.
- `super_admin`은 **팀 미소속 전역 계정**입니다(2026-07-03). 팀이 없어도 채팅·문서·챗봇 등
  제품 기능을 사용할 수 있으며, 이때 콘텐츠는 `team_id = NULL`(관리자 전역 버킷)로 저장되어
  어떤 팀에도 노출되지 않습니다. 일반 사용자는 팀이 없으면 종전대로 400 차단(`require_team`).

## 2. 계정 가입 시나리오

### (a) 최초 부팅 — super_admin 자동 생성
`config.BOOTSTRAP_ADMIN_ENABLED=true` 일 때 `services/bootstrap_admin.py`가 super_admin 계정을 생성합니다. 운영에서는 `false`로 끕니다.

### (b) 신규 팀 생성 — super_admin이 수행
1. super_admin이 `/admin/teams` 에서 팀명을 입력
2. POST `/admin/teams` → `teams` INSERT, 초대코드 생성
3. 초대코드를 팀장 후보에게 전달

### (c) 팀장 가입 — 초대코드로 진입
1. 가입 페이지에서 초대코드 입력
2. POST `/auth/register { invite_code, ... }`
3. 서버 결정:
 - 초대코드 발신자가 super_admin이라면 해당 계정의 `role=team_admin`, `approval_status=approved`
 - 일반 초대코드라면 `role=team_member`, `approval_status=pending`
4. `team_admin`은 즉시 로그인 가능

### (d) 팀원 가입 — 팀장이 승인
1. 팀원이 초대코드로 가입 → `pending`
2. 팀장이 `/admin/approvals` 에서 승인 목록 확인
3. POST `/admin/approvals/{user_id}/approve` → `approval_status=approved`
4. 승인 전에는 로그인 시 403 "승인 대기 중" 응답

### (e) 팀장이 팀원을 직접 등록
이메일/비밀번호를 바로 설정해 생성 — 현재 `POST /team/members` 엔드포인트 유지. 이 경우 `approval_status=approved`로 즉시 생성.

### (f) 팀장이 팀원을 비활성화 / 재활성화
1. `/admin` → "팀원 관리" 섹션에서 [ 비활성화 ] 클릭
2. `POST /admin/users/{id}/deactivate` → `is_active=false`
3. 비활성 계정은 로그인 시 **403 "비활성화된 계정입니다."**
4. 이력(대화/문서 소유권)은 그대로 보존. 재활성화는 [ 재활성화 ] 버튼으로 즉시 복구.

제약:
- 본인 계정은 비활성화 불가(서버 400).
- `super_admin` 계정은 팀장이 비활성화 불가(서버 403).
- 다른 팀 사용자는 `super_admin`만 조작 가능.

## 3. 권한 매트릭스

| 작업 | super_admin | team_admin | team_auditor | team_member |
| ---- | :---------: | :--------: | :----------: | :---------: |
| 팀 생성 | | ❌ | ❌ | ❌ |
| 팀 삭제 / 초대코드 로테이트 | | (본인 팀) | ❌ | ❌ |
| 팀원 가입 승인/반려 | | (본인 팀) | ❌ | ❌ |
| 팀원 비활성화/재활성화 | | (본인 팀) | ❌ | ❌ |
| 팀 멤버 목록 보기 | | | | |
| 팀원 생성(직접) | | | ❌ | ❌ |
| 대화 감사(전체 인원) | | ❌ | ❌ | ❌ |
| 개인 대화/문서 | | | | |
| 팀 공용 문서 업로드 | | | ❌ | |
| 팀 공용 문서 삭제(타인) | | | ❌ | ❌ |
| 챗봇 생성 | | | | |
| Public 챗봇 공개 | | | | (본인 생성만) |
| 타인의 챗봇 수정 | | (본인 팀) | ❌ | ❌ |
| 도구 카탈로그 관리 | | ❌ | ❌ | ❌ |
| 도구 자격증명 등록(본인 계정) | | | | |

## 4. 강제 지점 (enforcement)

- **의존성 주입**: `backend/app/deps.py`
 ```python
 async def require_super_admin(user = Depends(get_current_user)):
 if user.role != UserRole.super_admin:
 raise HTTPException(403, "최종관리자만 접근 가능합니다.")

 async def require_team_admin(user = Depends(get_current_user)):
 if user.role not in (UserRole.super_admin, UserRole.team_admin):
 raise HTTPException(403, "팀 관리자 이상만 접근 가능합니다.")
 ```

- **로그인 차단**: `POST /auth/login` 응답 시 `user.approval_status != approved` 이면 403 반환.

- **라우트 예시**:
 ```python
 @router.post("/admin/teams", dependencies=[Depends(require_super_admin)])
 @router.post("/admin/approvals/{user_id}/approve", dependencies=[Depends(require_team_admin)])
 ```

## 5. 프론트엔드 가드

- `AppShell.tsx`는 `/auth/me` 응답을 기반으로 역할에 따라 네비게이션 항목을 표시/숨김 처리합니다.
- `admin`, `tools` 관리 페이지는 역할 미충족 시 `/chat`으로 리다이렉트.
- 승인 대기 상태면 로그인 직후 `/register/pending` (또는 `/login`의 에러 메시지)로 안내.

## 6. JWT 클레임

```json
{
 "sub": "<user_id>",
 "role": "team_admin",
 "team_id": "<team_id>",
 "approval": "approved",
 "exp": 1800000000
}
```

- 서버는 매 요청마다 DB에서 `User`를 다시 로드하여 상태를 재검증합니다(토큰 내 값은 UI 힌트 용도).
