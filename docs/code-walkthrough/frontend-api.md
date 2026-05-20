# API 클라이언트 + 사용자 친화 에러

`frontend/shared/lib/api.ts` (449줄) — 모든 API 호출의 단일 통로. `frontend/shared/lib/userError.ts` 는 에러 객체를 사람이 읽을 메시지로 변환.

## 1. 두 가지 진실

```ts
// shared/lib/api.ts:6
export const API_URL =
  process.env.NEXT_PUBLIC_API_URL !== undefined
    ? process.env.NEXT_PUBLIC_API_URL
    : "/api";
```

기본은 `"/api"` — 같은 origin 프록시. `next.config.ts` 의 `rewrites` 가 `/api/:path*` 를 백엔드(`http://127.0.0.1:8001`)로 보냄. 그래서:

- 브라우저는 `fetch("/api/chatbots")` 만 호출.
- next dev 가 이걸 백엔드로 forward.
- 결과: CORS 발생 안 함, 8001 포트가 LAN 에 노출될 필요 없음.

## 2. 토큰 헬퍼

```ts
export function getToken(): string | null {
  if (typeof window === "undefined") return null;
  return localStorage.getItem("token");
}
export function setToken(token: string) { localStorage.setItem("token", token); }
export function clearToken() { localStorage.removeItem("token"); }
```

`typeof window === "undefined"` 체크는 SSR 보호 — Next 의 서버 사이드 렌더 중 localStorage 가 없을 때.

## 3. apiFetch — 핵심 래퍼

```ts
export async function apiFetch(path: string, init: RequestInit = {}): Promise<Response> {
  const url = `${API_URL}${path}`;
  const headers = new Headers(init.headers);
  const t = getToken();
  if (t) headers.set("Authorization", `Bearer ${t}`);
  if (init.body !== undefined && !headers.has("Content-Type")) {
    if (typeof init.body === "string" || init.body instanceof FormData === false) {
      headers.set("Content-Type", "application/json");
    }
  }
  logRequest(init.method ?? "GET", url);
  let response: Response;
  try {
    response = await fetch(url, { ...init, headers });
  } catch (e) {
    notifyServerDown(...);
    throw new ApiUserError("서버 연결에 실패했습니다.", 0, ...);
  }
  if (!response.ok) {
    throw await ApiUserError.fromResponse(response);
  }
  notifyServerUp(...);
  return response;
}
```

3가지 보장:
1. **인증 헤더 자동 부착** — 모든 호출.
2. **Content-Type 자동 설정** — body 가 string/object 이면 JSON.
3. **에러를 ApiUserError 로 변환** — 호출자가 status / message 일관성 있게 다룸.

## 4. ApiUserError (`userError.ts`)

```ts
export class ApiUserError extends Error {
  constructor(message: string, public status: number, public payload?: unknown) {
    super(message);
  }
  static async fromResponse(r: Response): Promise<ApiUserError> {
    let payload: unknown = null;
    try { payload = await r.json(); } catch {}
    const msg = (payload as any)?.detail || (payload as any)?.message || `HTTP ${r.status}`;
    return new ApiUserError(msg, r.status, payload);
  }
}
```

서버가 `{detail: "..."}` 또는 `{message: "..."}` 어느 쪽이든 호출자가 `.message` 로 접근. `status` 으로 분기.

```ts
catch (e) {
  if (e instanceof ApiUserError && e.status === 401) router.replace("/login");
  else setError(messageFromUnknownError(e));
}
```

## 5. notifyServerDown / Up — 서버 끊김 감지

```ts
export const SERVER_DOWN_EVENT = "app:server-down";
export const SERVER_UP_EVENT = "app:server-up";

function notifyServerDown(reason: string) {
  window.dispatchEvent(new CustomEvent(SERVER_DOWN_EVENT, { detail: reason }));
}
```

`shared/ui/ServerStatusToast` 가 이 이벤트를 listen — fetch 실패가 발생하면 5초 후 자동 retry 토스트 표시. 사용자가 "서버가 죽었나" 추측하지 않게.

## 6. logout 헬퍼

```ts
export async function logout() {
  try {
    await fetch(`${API_URL}/auth/logout`, {
      method: "POST",
      headers: (() => {
        const h = new Headers();
        const t = getToken();
        if (t) h.set("Authorization", `Bearer ${t}`);
        return h;
      })(),
    }).catch(() => undefined);
  } finally {
    clearToken();
    if (typeof window !== "undefined") window.location.href = "/login";
  }
}
```

서버 요청은 실패해도 `clearToken + redirect` 는 항상 실행 — 사용자가 "로그아웃 안 됐다" 느낄 일 없음.

## 7. API_DEBUG 로깅

```ts
const API_DEBUG = typeof process !== "undefined" && process.env.NEXT_PUBLIC_API_DEBUG !== "0";
function logRequest(method: string, url: string) {
  if (!API_DEBUG || typeof window === "undefined") return;
  console.log("%c API ", "background:#10b981;color:#fff", method, url);
}
```

개발 중 모든 fetch 가 콘솔에 색깔 라벨로 표시. 운영에선 `NEXT_PUBLIC_API_DEBUG=0` 으로 비활성.

## 8. 함정·결정

- **모든 fetch 가 apiFetch** — 직접 `fetch()` 호출하면 인증/에러 일관성이 깨짐. 새 페이지 작성 시 apiFetch 만 사용.
- **`/api` prefix 가 next.config 와 결합** — 한쪽 바꾸면 다른쪽도. `next.config.ts` 의 rewrites 와 항상 한 쌍.
- **빈 토큰도 truthy 처리될 수 있음** — `if (token)` 보다 `if (token && token.length > 0)` 권장. 가입 pending 시 백엔드가 빈 토큰 반환 케이스 ([auth 워크스루](backend-auth.md)).
