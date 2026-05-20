# 공통 의존성 — settings · DB 세션 · 인증 가드

여러 모듈이 공유하는 작은 부품들. 한 곳에 모아 두면 새로 합류한 개발자가 어디서 import 할지 헷갈리지 않음.

## 1. Settings (`core/config.py`)

```python
class Settings(BaseSettings):
    database_url: str = ...
    openai_api_key: str | None = None
    jwt_secret: str = ...
    access_token_expire_minutes: int = 60 * 24 * 7  # 7d
    upload_dir: str = "./data/uploads"
    bootstrap_admin_email: str | None = None
    bootstrap_admin_password: str | None = None
    usd_krw_fallback_rate: float = 1380.0
    rag_hyde_enabled: bool = True
    rag_hyde_alpha: float = 0.5
    rerank_enabled: bool = True
    rerank_candidates: int = 30
    rag_final_top_k: int = 6
    # ... 50+ 옵션

    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")

@lru_cache
def get_settings() -> Settings:
    return Settings()  # type: ignore
```

`lru_cache` 로 1회 평가. 환경변수 변경 시 서버 재기동 필요. dependency injection 으로 라우터/서비스에서 사용:

```python
async def endpoint(s: Annotated[Settings, Depends(get_settings)]):
    ...
```

## 2. DB 세션 (`db/database.py`)

```python
engine = create_async_engine(settings.database_url, pool_size=20, max_overflow=10, ...)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_db():
    async with SessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()
```

라우터의 `db: Annotated[AsyncSession, Depends(get_db)]` 가 매 요청마다 새 세션. **`expire_on_commit=False`** 는 commit 후에도 ORM 객체를 그대로 응답에 쓸 수 있게 — `model_dump` 가 expired column 을 다시 fetch 하지 않음.

백그라운드 태스크는 `SessionLocal()` 직접 — 요청 종료 후에도 살아있어야 하므로.

## 3. 인증 가드 (`core/deps.py`)

```python
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login", auto_error=False)

async def get_current_user(
    token: Annotated[str | None, Depends(oauth2_scheme)],
    db: Annotated[AsyncSession, Depends(get_db)],
) -> User:
    if not token:
        raise HTTPException(401, "인증이 필요합니다.")
    try:
        payload = jwt.decode(token, settings.jwt_secret, algorithms=["HS256"])
        user_id = UUID(payload["sub"])
    except (jwt.PyJWTError, ValueError, KeyError):
        raise HTTPException(401, "토큰이 유효하지 않습니다.")
    user = await db.get(User, user_id)
    if not user or not user.is_active or user.approval_status != ApprovalStatus.approved:
        raise HTTPException(403, "사용자가 비활성 또는 미승인 상태입니다.")
    return user

def require_team(user: User) -> UUID:
    if not user.team_id:
        raise HTTPException(400, "팀이 없는 사용자는 이 작업을 할 수 없습니다.")
    return user.team_id

def require_role(*allowed: UserRole):
    def _dep(user: Annotated[User, Depends(get_current_user)]) -> User:
        if user.role not in allowed:
            raise HTTPException(403, "권한이 없습니다.")
        return user
    return _dep
```

대부분의 라우트가 다음 패턴:

```python
@router.get("/admin/users")
async def list_users(
    user: Annotated[User, Depends(require_role(UserRole.team_admin, UserRole.super_admin))],
    db: Annotated[AsyncSession, Depends(get_db)],
):
    ...
```

`require_team` 은 chat/document 같이 팀 컨텍스트가 필수인 작업에서 호출 (super_admin 일 때 일부러 가드).

## 4. 보안 (`core/security.py`)

```python
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(raw: str) -> str:
    return pwd_context.hash(raw)

def verify_password(raw: str, hashed: str) -> bool:
    return pwd_context.verify(raw, hashed)

def create_access_token(subject: str, claims: dict | None = None, expires_minutes: int | None = None) -> str:
    expire = datetime.now(timezone.utc) + timedelta(minutes=expires_minutes or settings.access_token_expire_minutes)
    payload = {"sub": subject, "exp": expire, **(claims or {})}
    return jwt.encode(payload, settings.jwt_secret, algorithm="HS256")
```

bcrypt round 는 passlib 의 기본(12). JWT 는 HS256 — 운영에선 RS256 + 키 회전 권장. `claims` 에 role / team_id 가 들어가서 가드가 1차로 빠르게.

## 5. 에러 핸들러 (`core/error_handlers.py`)

```python
def register_exception_handlers(app: FastAPI):
    @app.exception_handler(HTTPException)
    async def http_handler(req, exc): return JSONResponse({"detail": exc.detail}, status_code=exc.status_code)

    @app.exception_handler(RequestValidationError)
    async def validation_handler(req, exc):
        # Pydantic v2 에러를 사람이 읽을 한 줄 메시지로
        msg = ...
        return JSONResponse({"detail": msg}, status_code=422)

    @app.exception_handler(Exception)
    async def unknown_handler(req, exc):
        log.exception(...)
        return JSONResponse({"detail": "예상치 못한 오류가 발생했습니다."}, status_code=500)
```

프론트의 `ApiUserError` 가 항상 `{detail: "..."}` 를 기대하므로, 모든 에러 응답 형식이 통일됨.

## 6. 요청 로깅 미들웨어

```python
class RequestLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        rid = uuid.uuid4().hex[:10]
        request.state.request_id = rid
        log.info("[%s] --> %s %s | client=%s", rid, request.method, request.url.path, request.client.host)
        t0 = time.perf_counter()
        try:
            response = await call_next(request)
        except Exception:
            log.exception("[%s] !! 처리 중 예외", rid)
            raise
        latency = int((time.perf_counter() - t0) * 1000)
        log.info("[%s] <-- %d %dms | %s %s", rid, response.status_code, latency, request.method, request.url.path)
        return response
```

모든 요청에 짧은 ID. 채팅 / RAG 로그가 같은 rid 로 묶여 디버깅 친화적.

## 7. 함정·결정

- **expire_on_commit=False 의 위험** — commit 후 다른 트랜잭션이 row 를 바꿔도 ORM 객체는 모름. 응답 직전엔 안전, 그 이후엔 다시 fetch 권장.
- **pool_size=20 max_overflow=10** — 동시 30 요청까지. 그 이상이면 큐잉. 트래픽이 늘면 늘려야 함.
- **JWT secret 회전** — 현재는 단일 secret. 회전 시 모든 사용자가 재로그인 강제. 운영에선 kid + multiple keys 지원 권장.
- **OAuth2PasswordBearer 의 auto_error=False** — None 이면 `get_current_user` 가 직접 401 — 메시지를 한국어로 통일하기 위함.
