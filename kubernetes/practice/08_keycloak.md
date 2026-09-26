# [실습] Keycloak

# Keycloak 기본 실습

네. 오늘 실습한 내용 중 **LDAP ↔ Keycloak ↔ WebApp(FastAPI/Nginx)** 연결 부분만 중심으로 간명하게 정리하면 다음 구조입니다.

# 1. 전체 구성

오늘 만든 인증 구조는 다음과 같습니다.

```
                         ┌──────────────────────┐
                         │      Browser         │
                         │  WebApp Frontend     │
                         │  :31898              │
                         └──────────┬───────────┘
                                    │
                         로그인 방식 선택
                         ┌──────────┴──────────┐
                         │                     │
                     LDAP Login          Keycloak Login
                         │                     │
                         ▼                     ▼
              ┌─────────────────┐    ┌─────────────────┐
              │    FastAPI      │    │    Keycloak     │
              │    Backend      │    │    :30080       │
              │    :32277       │    │    Realm hdaic  │
              └────────┬────────┘    └────────┬────────┘
                       │                      │
                       │ LDAP 인증             │ OIDC
                       ▼                      │
              ┌─────────────────┐             │
              │    OpenLDAP     │◄────────────┘
              │ users/hdaic.com │
              └─────────────────┘

                         FastAPI
                           │
                           ▼
                    Application JWT
                           │
                    ┌──────┴──────┐
                    ▼             ▼
                 /api/me    /api/protected
```

핵심은 **LDAP과 Keycloak이 서로 다른 로그인 경로를 제공하지만, 최종적으로 WebApp에서는 동일한 Application JWT로 보호 API를 호출한다**는 것입니다.

---

# 2. 기존 LDAP 구조

기존 WebApp에는 이미 OpenLDAP이 있었습니다.

```
WebApp
   │
   │ username/password
   ▼
FastAPI
   │
   │ LDAP Bind
   ▼
OpenLDAP
```

FastAPI의 설정:

```python
LDAP_SERVER = "ldap://openldap.ldap.svc.cluster.local:389"

LDAP_BASE_DN = "ou=users,dc=hdaic,dc=com"
```

사용자가 로그인하면:

```
POST /api/login
```

요청:

```json
{
  "username": "alice",
  "password": "password"
}
```

FastAPI가 LDAP에 직접 인증합니다.

```python
user_dn = f"uid={username},{LDAP_BASE_DN}"

conn = Connection(
    server,
    user=user_dn,
    password=password,
    auto_bind=True
)
```

성공하면 FastAPI가 자체 JWT를 발급합니다.

```json
{
  "access_token": "...",
  "token_type": "bearer",
  "username": "alice",
  "provider": "ldap"
}
```

---

# 3. Keycloak 추가

여기에 Keycloak을 추가했습니다.

```
                ┌───────────────┐
                │   Web Browser  │
                └───────┬───────┘
                        │
                        ▼
                ┌───────────────┐
                │    Keycloak   │
                │   :30080      │
                │ realm: hdaic  │
                └───────┬───────┘
                        │
                        ▼
                    OpenLDAP
```

Keycloak에서는 Realm을 만들었습니다.

```
Realm
 └── hdaic
```

그리고 WebApp용 Client:

```
Client ID:
webapp
```

즉:

```
Keycloak
└── Realm: hdaic
    └── Client: webapp
```

---

# 4. Keycloak과 LDAP 연결

이번 실습에서 중요한 부분입니다.

Keycloak이 사용자 인증을 직접 관리하는 것이 아니라 **OpenLDAP을 사용자 저장소로 사용하도록 연결**했습니다.

구조:

```
WebApp
   │
   │ OIDC
   ▼
Keycloak
   │
   │ LDAP Federation
   ▼
OpenLDAP
```

따라서 사용자 입장에서는:

```
jongkim
password
```

로 Keycloak에 로그인하지만 실제 사용자 정보는 LDAP에서 가져올 수 있습니다.

즉,

```
OpenLDAP
  └── uid=alice
  └── uid=bob
  └── uid=jongkim
  └── uid=honglee
```

등의 기존 LDAP 사용자를 Keycloak에서 사용할 수 있는 구조입니다.

---

# 5. Keycloak Client 설정

Keycloak의 `webapp` Client에서 WebApp 주소를 등록했습니다.

현재 중요한 주소는:

```
WebApp Frontend
http://192.168.56.11:31898
```

FastAPI OAuth callback:

```
http://192.168.56.11:32277/oauth/callback
```

따라서 Keycloak Client의 **Valid Redirect URIs**에:

```
http://192.168.56.11:32277/oauth/callback
```

을 등록했습니다.

이 설정이 맞지 않으면 Keycloak 로그인 후 callback 단계에서 실패합니다.

---

# 6. FastAPI에 OAuth 로그인 추가

기존 LDAP 로그인:

```
POST /api/login
```

에 추가하여 Keycloak 로그인 시작 endpoint를 만들었습니다.

```
GET /oauth/login
```

사용자가 이 endpoint를 호출하면 FastAPI가 Keycloak 인증 URL을 생성합니다.

예:

```
http://192.168.56.11:30080/realms/hdaic/
protocol/openid-connect/auth
```

주요 parameter:

```
client_id=webapp
response_type=code
scope=openid profile email
redirect_uri=http://192.168.56.11:32277/oauth/callback
state=...
```

---

# 7. `/oauth/login` → Keycloak

전체 흐름은:

```
Browser
   │
   │ GET /oauth/login
   ▼
FastAPI
   │
   │ 302 Redirect
   ▼
Keycloak
   │
   │ Login
   ▼
OpenLDAP
```

FastAPI는 OAuth `state`도 session에 저장합니다.

```python
state = secrets.token_urlsafe(32)

request.session["oauth_state"] = state
```

이것은 OAuth CSRF 방지를 위한 것입니다.

---

# 8. Keycloak 로그인 → callback

사용자가 Keycloak에서 로그인에 성공하면 Keycloak이:

```
/oauth/callback?code=...&state=...
```

로 이동시킵니다.

즉:

```
Keycloak
    │
    │ authorization code
    ▼
FastAPI
/oauth/callback
```

FastAPI는 먼저 `state`를 검증합니다.

```
Keycloak state
      =
FastAPI session state
```

같아야 합니다.

이전에 발생했던:

```
mismatching_state:
CSRF Warning!
State not equal in request and response.
```

문제가 바로 이 과정에서 발생했던 것입니다.

---

# 9. Authorization Code → Token

callback에서 FastAPI는 Keycloak에 authorization code를 전달합니다.

```
FastAPI
   │
   │ code
   ▼
Keycloak Token Endpoint
```

Keycloak:

```
/realms/hdaic/protocol/openid-connect/token
```

에서 Access Token을 반환합니다.

---

# 10. Keycloak UserInfo

그 다음 FastAPI가 Keycloak UserInfo endpoint를 호출했습니다.

```
GET
/realms/hdaic/protocol/openid-connect/userinfo
```

Authorization:

```
Authorization: Bearer <Keycloak access token>
```

Keycloak은 사용자 정보를 반환합니다.

예:

```json
{
  "preferred_username": "alice",
  "email": "alice@hdaic.com"
}
```

FastAPI에서는:

```python
username = (
    user_info.get("preferred_username")
    or user_info.get("username")
    or user_info.get("email")
)
```

으로 사용자를 확인합니다.

---

# 11. Keycloak JWT를 그대로 사용하지 않고 Application JWT 발급

오늘 구성에서 중요한 설계입니다.

Keycloak Access Token을 WebApp API에 직접 사용하지 않고 FastAPI가 **자체 Application JWT**를 새로 발급했습니다.

```
Keycloak Access Token
        │
        ▼
      FastAPI
        │
        ▼
Application JWT
```

JWT에는:

```json
{
  "sub": "alice",
  "provider": "keycloak"
}
```

정보를 넣었습니다.

LDAP 로그인인 경우:

```json
{
  "sub": "alice",
  "provider": "ldap"
}
```

따라서 FastAPI 입장에서는 두 로그인 방식을 하나의 인증 체계로 통합할 수 있습니다.

---

# 12. 최종적으로 두 로그인 방식이 하나로 통합

### LDAP 로그인

```
Frontend
   │
   │ POST /api/login
   ▼
FastAPI
   │
   ▼
OpenLDAP
   │
   ▼
Application JWT
```

결과:

```json
{
  "username": "alice",
  "provider": "ldap"
}
```

### Keycloak 로그인

```
Frontend
   │
   │ /oauth/login
   ▼
Keycloak
   │
   ▼
OpenLDAP
   │
   ▼
FastAPI /oauth/callback
   │
   ▼
Application JWT
```

결과:

```json
{
  "username": "alice",
  "provider": "keycloak"
}
```

---

# 13. `/api/me`와 `/api/protected`

두 로그인 방식 모두 최종적으로 동일한 JWT를 사용합니다.

```
                Application JWT
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
          /api/me         /api/protected
```

FastAPI:

```python
@app.get("/api/me")
def me(
    user: dict = Depends(get_current_user)
):
```

그리고:

```python
@app.get("/api/protected")
def protected(
    user: dict = Depends(get_current_user)
):
```

`Authorization` header:

```
Authorization: Bearer <Application JWT>
```

를 검사합니다.

---

# 14. Nginx의 역할

Frontend는:

```
192.168.56.11:31898
```

으로 접근합니다.

Nginx가 frontend와 backend를 연결합니다.

```
Browser
   │
   │ :31898
   ▼
Nginx
   │
   ├── /              → index.html
   │
   ├── /api/          → FastAPI :8000
   │
   └── /oauth/        → FastAPI :8000
```

즉 Nginx 설정의 핵심은:

```
location /api/ {
    proxy_pass http://my-webapp-backend:8000;
}
```

그리고:

```
location /oauth/ {
    proxy_pass http://my-webapp-backend:8000;
}
```

입니다.

따라서 Browser에서는 backend의 `32277`을 직접 사용할 필요 없이:

```
/api/login
/api/me
/api/protected
/oauth/login
/oauth/callback
```

형태로 사용할 수 있습니다.

---

# 15. 최종 전체 인증 흐름

## LDAP 로그인

```
Browser
  │
  │ /api/login
  ▼
Nginx :31898
  │
  ▼
FastAPI :8000
  │
  │ LDAP Bind
  ▼
OpenLDAP
  │
  │ OK
  ▼
FastAPI
  │
  │ Application JWT
  ▼
Browser
  │
  │ Authorization: Bearer JWT
  ▼
/api/me
/api/protected
```

## Keycloak 로그인

```
Browser
  │
  │ /oauth/login
  ▼
Nginx
  │
  ▼
FastAPI
  │
  │ 302
  ▼
Keycloak :30080
  │
  │ Login
  ▼
OpenLDAP
  │
  │ 인증 성공
  ▼
Keycloak
  │
  │ code
  ▼
FastAPI /oauth/callback
  │
  │ token
  ▼
Keycloak UserInfo
  │
  │ username
  ▼
FastAPI
  │
  │ Application JWT
  ▼
Frontend
  │
  │ localStorage
  ▼
/api/me
/api/protected
```

---

# 16. 오늘 실습의 핵심 결과

결국 기존 WebApp에 다음 구조를 추가한 것입니다.

```
                 ┌─────────────┐
                 │  Frontend   │
                 │   Nginx     │
                 │   :31898    │
                 └──────┬──────┘
                        │
             ┌──────────┴──────────┐
             │                     │
        LDAP Login           Keycloak Login
             │                     │
             ▼                     ▼
          FastAPI              Keycloak
             │                     │
             │                  OpenLDAP
             │                     │
             └─────────┬───────────┘
                       ▼
                Application JWT
                       │
                ┌──────┴──────┐
                ▼             ▼
             /api/me    /api/protected
```

**핵심적으로 기억할 것은 4가지입니다.**

1. **OpenLDAP** → 실제 사용자 계정 저장
2. **Keycloak** → LDAP 사용자를 이용한 OIDC 로그인 제공
3. **FastAPI** → LDAP 로그인과 Keycloak 로그인을 모두 처리하고 Application JWT 발급
4. **Nginx** → Frontend와 FastAPI의 `/api`, `/oauth` 경로를 연결

그리고 현재 Kubernetes 서비스 기준으로는:

```
Frontend : 192.168.56.11:31898
Backend  : 192.168.56.11:32277
Keycloak : 192.168.56.11:30080
```

이라는 관계입니다.

# Fastapi 최종 코드

아래는 **오늘 실습에서 최종적으로 동작했던 구성**을 기준으로 한 `main.py` 전체 코드입니다.

- LDAP 로그인: `/api/login`
- Keycloak 로그인 시작: `/oauth/login`
- Keycloak callback: `/oauth/callback`
- Keycloak UserInfo 조회
- LDAP/Keycloak 공통 Application JWT
- `/api/me`
- `/api/protected`
- OAuth `state` 검증
- UserInfo/Token 오류 상세 출력
- Nginx `/oauth/` 프록시 환경 지원

```python
from fastapi import (
    FastAPI,
    HTTPException,
    Depends,
    Request,
)

from fastapi.middleware.cors import CORSMiddleware
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from fastapi.responses import RedirectResponse

from prometheus_fastapi_instrumentator import Instrumentator

from pydantic import BaseModel

from ldap3 import Server, Connection, ALL

from jose import jwt, JWTError

from starlette.middleware.sessions import SessionMiddleware

import os
import secrets
import urllib.parse
import requests

# ============================================================
# Configuration
# ============================================================

# ------------------------------------------------------------
# LDAP
# ------------------------------------------------------------

LDAP_SERVER = os.getenv(
    "LDAP_SERVER",
    "ldap://openldap.ldap.svc.cluster.local:389"
)

LDAP_BASE_DN = os.getenv(
    "LDAP_BASE_DN",
    "ou=users,dc=hdaic,dc=com"
)

# ------------------------------------------------------------
# Application JWT
# ------------------------------------------------------------

JWT_SECRET = os.getenv(
    "JWT_SECRET",
    "change-this-secret"
)

JWT_ALGORITHM = "HS256"

# ------------------------------------------------------------
# Session
# ------------------------------------------------------------

SESSION_SECRET = os.getenv(
    "SESSION_SECRET",
    "change-this-session-secret"
)

# ------------------------------------------------------------
# Keycloak
# ------------------------------------------------------------

KEYCLOAK_URL = os.getenv(
    "KEYCLOAK_URL",
    "http://192.168.56.11:30080"
).rstrip("/")

KEYCLOAK_REALM = os.getenv(
    "KEYCLOAK_REALM",
    "hdaic"
)

KEYCLOAK_CLIENT_ID = os.getenv(
    "KEYCLOAK_CLIENT_ID",
    "webapp"
)

# ------------------------------------------------------------
# Keycloak Client Secret
#
# Public Client:
#     ""
#
# Confidential Client:
#     환경변수로 설정
# ------------------------------------------------------------

KEYCLOAK_CLIENT_SECRET = os.getenv(
    "KEYCLOAK_CLIENT_SECRET",
    ""
)

# ------------------------------------------------------------
# OAuth Redirect URI
#
# Keycloak Client의 Valid Redirect URIs와
# 정확히 일치해야 합니다.
# ------------------------------------------------------------

OAUTH_REDIRECT_URI = os.getenv(
    "OAUTH_REDIRECT_URI",
    "http://192.168.56.11:32277/oauth/callback"
)

# ------------------------------------------------------------
# Frontend URL
#
# Keycloak 로그인 완료 후 이동할 주소
# ------------------------------------------------------------

FRONTEND_URL = os.getenv(
    "FRONTEND_URL",
    "http://192.168.56.11:31898/"
)

# ============================================================
# Keycloak Endpoints
# ============================================================

KEYCLOAK_AUTH_URL = (
    f"{KEYCLOAK_URL}"
    f"/realms/{KEYCLOAK_REALM}"
    f"/protocol/openid-connect/auth"
)

KEYCLOAK_TOKEN_URL = (
    f"{KEYCLOAK_URL}"
    f"/realms/{KEYCLOAK_REALM}"
    f"/protocol/openid-connect/token"
)

KEYCLOAK_USERINFO_URL = (
    f"{KEYCLOAK_URL}"
    f"/realms/{KEYCLOAK_REALM}"
    f"/protocol/openid-connect/userinfo"
)

# ============================================================
# FastAPI
# ============================================================

app = FastAPI(
    title="WebApp Backend",
    version="3.0.0"
)

# ============================================================
# Session Middleware
# ============================================================

app.add_middleware(
    SessionMiddleware,
    secret_key=SESSION_SECRET,
    same_site="lax",
    https_only=False,
)

# ============================================================
# CORS
# ============================================================

app.add_middleware(
    CORSMiddleware,

    allow_origins=[
        "http://192.168.56.11:31898",
        "http://192.168.56.11:32277",
    ],

    allow_credentials=True,

    allow_methods=["*"],

    allow_headers=["*"],
)

# ============================================================
# Prometheus
# ============================================================

Instrumentator().instrument(app).expose(app)

# ============================================================
# Security
# ============================================================

security = HTTPBearer()

# ============================================================
# Request Models
# ============================================================

class LoginRequest(BaseModel):

    username: str
    password: str

# ============================================================
# LDAP Authentication
# ============================================================

def authenticate_ldap(
    username: str,
    password: str
) -> bool:

    user_dn = (
        f"uid={username},"
        f"{LDAP_BASE_DN}"
    )

    server = Server(
        LDAP_SERVER,
        get_info=ALL
    )

    conn = None

    try:

        conn = Connection(
            server,
            user=user_dn,
            password=password,
            auto_bind=True
        )

        print(
            f"[LDAP] Authentication successful: "
            f"{username}"
        )

        return True

    except Exception as e:

        print(
            f"[LDAP] Authentication failed: "
            f"{type(e).__name__}: {e}"
        )

        return False

    finally:

        if conn:

            try:
                conn.unbind()
            except Exception:
                pass

# ============================================================
# Application JWT
# ============================================================

def create_access_token(
    username: str,
    provider: str = "ldap"
):

    payload = {
        "sub": username,
        "provider": provider,
    }

    return jwt.encode(
        payload,
        JWT_SECRET,
        algorithm=JWT_ALGORITHM
    )

# ============================================================
# JWT Authentication
# ============================================================

def get_current_user(
    credentials: HTTPAuthorizationCredentials =
    Depends(security)
):

    token = credentials.credentials

    try:

        payload = jwt.decode(
            token,
            JWT_SECRET,
            algorithms=[JWT_ALGORITHM]
        )

        username = payload.get("sub")

        if not username:

            raise HTTPException(
                status_code=401,
                detail="Invalid authentication token"
            )

        return {
            "username": username,
            "provider": payload.get(
                "provider",
                "unknown"
            )
        }

    except JWTError as e:

        print(
            f"[JWT] Validation failed: "
            f"{type(e).__name__}: {e}"
        )

        raise HTTPException(
            status_code=401,
            detail="Invalid authentication token"
        )

# ============================================================
# Root
# ============================================================

@app.get("/")
def root():

    return {
        "message": "Hello from FastAPI backend"
    }

# ============================================================
# Hello API
# ============================================================

@app.get("/api/hello")
def hello():

    return {
        "message": "Hello from WebApp backend"
    }

# ============================================================
# LDAP Login
# ============================================================

@app.post("/api/login")
def login(
    request: LoginRequest
):

    authenticated = authenticate_ldap(
        request.username,
        request.password
    )

    if not authenticated:

        raise HTTPException(
            status_code=401,
            detail="Invalid username or password"
        )

    access_token = create_access_token(
        request.username,
        provider="ldap"
    )

    return {
        "access_token": access_token,
        "token_type": "bearer",
        "username": request.username,
        "provider": "ldap"
    }

# ============================================================
# Keycloak OAuth Login
# ============================================================

@app.get("/oauth/login")
def oauth_login(
    request: Request
):

    # --------------------------------------------------------
    # Generate OAuth state
    # --------------------------------------------------------

    state = secrets.token_urlsafe(32)

    request.session["oauth_state"] = state

    print(
        "[OAuth] Generated state:",
        state
    )

    # --------------------------------------------------------
    # Authorization parameters
    # --------------------------------------------------------

    params = {

        "client_id":
            KEYCLOAK_CLIENT_ID,

        "response_type":
            "code",

        "scope":
            "openid profile email",

        "redirect_uri":
            OAUTH_REDIRECT_URI,

        "state":
            state,
    }

    authorization_url = (
        KEYCLOAK_AUTH_URL
        + "?"
        + urllib.parse.urlencode(params)
    )

    print(
        "[OAuth] Authorization URL:",
        authorization_url
    )

    # --------------------------------------------------------
    # Redirect browser to Keycloak
    # --------------------------------------------------------

    return RedirectResponse(
        authorization_url,
        status_code=302
    )

# ============================================================
# Keycloak OAuth Callback
# ============================================================

@app.get("/oauth/callback")
def oauth_callback(
    request: Request,

    code: str | None = None,

    state: str | None = None,

    error: str | None = None,

    error_description: str | None = None,
):

    # ========================================================
    # 1. Keycloak error
    # ========================================================

    if error:

        detail = (
            "Keycloak authentication failed: "
            f"error={error}"
        )

        if error_description:

            detail += (
                f", "
                f"description={error_description}"
            )

        print(
            "[OAuth]",
            detail
        )

        raise HTTPException(
            status_code=401,
            detail=detail
        )

    # ========================================================
    # 2. Authorization code check
    # ========================================================

    if not code:

        raise HTTPException(
            status_code=400,
            detail=(
                "Missing authorization code "
                "from Keycloak"
            )
        )

    # ========================================================
    # 3. OAuth State Validation
    # ========================================================

    saved_state = request.session.get(
        "oauth_state"
    )

    if not state:

        raise HTTPException(
            status_code=400,
            detail="Missing OAuth state"
        )

    if not saved_state:

        raise HTTPException(
            status_code=400,
            detail=(
                "OAuth session state is missing. "
                "The login session may have expired."
            )
        )

    if not secrets.compare_digest(
        saved_state,
        state
    ):

        print(
            "[OAuth] State mismatch"
        )

        print(
            "[OAuth] Saved state:",
            saved_state
        )

        print(
            "[OAuth] Received state:",
            state
        )

        raise HTTPException(
            status_code=400,
            detail="Invalid OAuth state"
        )

    # --------------------------------------------------------
    # State is one-time use
    # --------------------------------------------------------

    request.session.pop(
        "oauth_state",
        None
    )

    # ========================================================
    # 4. Authorization Code → Token
    # ========================================================

    token_data = {

        "grant_type":
            "authorization_code",

        "code":
            code,

        "redirect_uri":
            OAUTH_REDIRECT_URI,

        "client_id":
            KEYCLOAK_CLIENT_ID,
    }

    if KEYCLOAK_CLIENT_SECRET:

        token_data["client_secret"] = (
            KEYCLOAK_CLIENT_SECRET
        )

    print(
        "[OAuth] Requesting token from:",
        KEYCLOAK_TOKEN_URL
    )

    try:

        token_response = requests.post(

            KEYCLOAK_TOKEN_URL,

            data=token_data,

            timeout=10
        )

    except requests.RequestException as e:

        print(
            "[OAuth] Token request exception:",
            repr(e)
        )

        raise HTTPException(
            status_code=502,
            detail=(
                "Failed to connect to Keycloak "
                f"token endpoint: "
                f"{type(e).__name__}: {e}"
            )
        )

    # ========================================================
    # 5. Token Response Validation
    # ========================================================

    if not token_response.ok:

        print(
            "================================================"
        )

        print(
            "[OAuth] TOKEN REQUEST FAILED"
        )

        print(
            "[OAuth] URL:",
            KEYCLOAK_TOKEN_URL
        )

        print(
            "[OAuth] HTTP status:",
            token_response.status_code
        )

        print(
            "[OAuth] Response:",
            token_response.text
        )

        print(
            "================================================"
        )

        try:

            error_body = (
                token_response.json()
            )

        except ValueError:

            error_body = (
                token_response.text
            )

        raise HTTPException(
            status_code=502,
            detail={
                "message":
                    "Keycloak token request failed",

                "token_url":
                    KEYCLOAK_TOKEN_URL,

                "http_status":
                    token_response.status_code,

                "keycloak_response":
                    error_body,
            }
        )

    # ========================================================
    # 6. Parse Token JSON
    # ========================================================

    try:

        token_json = (
            token_response.json()
        )

    except ValueError as e:

        raise HTTPException(
            status_code=502,
            detail=(
                "Keycloak token endpoint returned "
                f"invalid JSON: {e}; "
                f"response={token_response.text}"
            )
        )

    access_token = token_json.get(
        "access_token"
    )

    if not access_token:

        print(
            "[OAuth] access_token missing"
        )

        print(
            "[OAuth] Token response:",
            token_json
        )

        raise HTTPException(
            status_code=502,
            detail={
                "message":
                    "Keycloak token response does not "
                    "contain access_token",

                "token_response":
                    token_json,
            }
        )

    print(
        "[OAuth] Access token received successfully"
    )

    # ========================================================
    # 7. Keycloak UserInfo
    # ========================================================

    print(
        "[OAuth] Requesting UserInfo from:",
        KEYCLOAK_USERINFO_URL
    )

    try:

        userinfo_response = requests.get(

            KEYCLOAK_USERINFO_URL,

            headers={
                "Authorization":
                    f"Bearer {access_token}",

                "Accept":
                    "application/json"
            },

            timeout=10
        )

    except requests.RequestException as e:

        print(
            "[OAuth] UserInfo request exception:",
            repr(e)
        )

        raise HTTPException(
            status_code=502,
            detail=(
                "Failed to connect to Keycloak "
                f"UserInfo endpoint: "
                f"{type(e).__name__}: {e}"
            )
        )

    # ========================================================
    # 8. UserInfo Error
    # ========================================================

    if not userinfo_response.ok:

        print(
            "================================================"
        )

        print(
            "[OAuth] USERINFO REQUEST FAILED"
        )

        print(
            "[OAuth] URL:",
            KEYCLOAK_USERINFO_URL
        )

        print(
            "[OAuth] HTTP status:",
            userinfo_response.status_code
        )

        print(
            "[OAuth] Headers:",
            dict(userinfo_response.headers)
        )

        print(
            "[OAuth] Response:",
            userinfo_response.text
        )

        print(
            "================================================"
        )

        try:

            error_body = (
                userinfo_response.json()
            )

        except ValueError:

            error_body = (
                userinfo_response.text
            )

        raise HTTPException(
            status_code=502,
            detail={
                "message":
                    "Keycloak UserInfo request failed",

                "userinfo_url":
                    KEYCLOAK_USERINFO_URL,

                "http_status":
                    userinfo_response.status_code,

                "keycloak_response":
                    error_body,
            }
        )

    # ========================================================
    # 9. Parse UserInfo JSON
    # ========================================================

    try:

        user_info = (
            userinfo_response.json()
        )

    except ValueError as e:

        print(
            "[OAuth] UserInfo invalid JSON:",
            userinfo_response.text
        )

        raise HTTPException(
            status_code=502,
            detail=(
                "Keycloak UserInfo returned "
                f"invalid JSON: {e}; "
                f"response={userinfo_response.text}"
            )
        )

    print(
        "[OAuth] UserInfo received:",
        user_info
    )

    # ========================================================
    # 10. Extract Username
    # ========================================================

    username = (

        user_info.get(
            "preferred_username"
        )

        or user_info.get(
            "username"
        )

        or user_info.get(
            "email"
        )
    )

    if not username:

        print(
            "[OAuth] Username not found"
        )

        print(
            "[OAuth] UserInfo:",
            user_info
        )

        raise HTTPException(
            status_code=502,
            detail={
                "message":
                    "Username was not found "
                    "in Keycloak UserInfo",

                "userinfo":
                    user_info,
            }
        )

    # ========================================================
    # 11. Create Application JWT
    # ========================================================

    application_token = (
        create_access_token(
            username,
            provider="keycloak"
        )
    )

    # ========================================================
    # 12. Redirect to Frontend
    #
    # Fragment (#...)은 HTTP 요청으로 서버에 전달되지
    # 않습니다.
    #
    # 따라서 Frontend JavaScript가
    # window.location.hash를 읽어서
    # access_token을 localStorage에 저장합니다.
    # ========================================================

    redirect_url = (

        FRONTEND_URL

        + "#"

        + urllib.parse.urlencode({

            "access_token":
                application_token,

            "username":
                username,

            "provider":
                "keycloak"
        })
    )

    print(
        "[OAuth] Login successful:",
        username
    )

    print(
        "[OAuth] Provider: keycloak"
    )

    print(
        "[OAuth] Redirecting to:",
        FRONTEND_URL
    )

    return RedirectResponse(
        redirect_url,
        status_code=302
    )

# ============================================================
# Current User
# ============================================================

@app.get("/api/me")
def me(
    user: dict = Depends(
        get_current_user
    )
):

    return {

        "username":
            user["username"],

        "provider":
            user["provider"],

        "message":
            f"Hello {user['username']}"
    }

# ============================================================
# Protected API
# ============================================================

@app.get("/api/protected")
def protected(
    user: dict = Depends(
        get_current_user
    )
):

    return {

        "message":
            "This is a protected API",

        "username":
            user["username"],

        "provider":
            user["provider"]
    }
```

### 현재 구성에서 반드시 맞아야 하는 값

특히 아래 3개는 현재 Kubernetes 구성과 일치해야 합니다.

```python
KEYCLOAK_URL = "http://192.168.56.11:30080"

OAUTH_REDIRECT_URI = \
    "http://192.168.56.11:32277/oauth/callback"

FRONTEND_URL = \
    "http://192.168.56.11:31898/"
```

그리고 Keycloak의 `webapp` Client에는:

```
Valid Redirect URIs

http://192.168.56.11:32277/oauth/callback
```

이 등록되어 있어야 합니다.

최종적인 로그인 흐름은:

```
LDAP 로그인
/api/login
    ↓
OpenLDAP
    ↓
Application JWT
    ↓
/api/me
/api/protected
```

또는

```
Keycloak 로그인
/oauth/login
    ↓
Keycloak :30080
    ↓
OpenLDAP
    ↓
/oauth/callback
    ↓
Keycloak UserInfo
    ↓
Application JWT
    ↓
Frontend :31898
    ↓
/api/me
/api/protected
```

입니다.
