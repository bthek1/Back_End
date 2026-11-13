# Authentication

Django REST Framework (DRF) provides a robust and flexible authentication system to secure your API. Here’s an overview of everything you need to know about authentication in DRF:

Author

Benedict Thekkel

## 📘 Authentication in Django REST Framework (DRF)

> 🔐 Authentication = *Who are you?*\
> 🔓 Authorization = *Are you allowed to do this?*

DRF handles **authentication** via pluggable classes set in `REST_FRAMEWORK['DEFAULT_AUTHENTICATION_CLASSES']`.

------------------------------------------------------------------------

## 🧱 Built-in & Common Authentication Classes in Django REST Framework

| Class | Description | Use Case |
|:---|:---|:---|
| `SessionAuthentication` | Uses Django’s session + CSRF cookies | Web clients using Django templates or admin |
| `BasicAuthentication` | Sends Base64-encoded username/password in header | Quick API testing; not for production |
| `TokenAuthentication` | Token in `Authorization: Token <token>` | Simple token-based API auth for mobile, SPA |
| `JWTAuthentication` (via `djangorestframework-simplejwt`) | Token with built-in expiry, stateless | Mobile apps, SPAs, modern REST APIs |
| `OAuth2Authentication` (via `django-oauth-toolkit`) | Supports Bearer tokens with scopes, introspection, refresh | SaaS-to-SaaS integrations, B2B APIs, third-party developer portals |
| `RemoteUserAuthentication` | Authenticates based on `REMOTE_USER` HTTP header | Internal apps behind enterprise SSO or reverse proxies |
| `CustomAuthentication` | You implement `authenticate(self, request)` | For API keys, mTLS, hardware-based tokens, etc. |

------------------------------------------------------------------------

## 🛠 Basic Setup

In `settings.py`:

``` python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.TokenAuthentication',
    ]
}
```

This enables **both browser and API clients** to authenticate.

------------------------------------------------------------------------

## ✅ Common Auth Workflows

| Type | User Experience | Use With |
|:---|:---|:---|
| **SessionAuth** | Log in through Django Admin or LoginView | Browsers |
| **TokenAuth** | POST username+password → get token | Postman, scripts, apps |
| **JWTAuth** | POST credentials → get access/refresh tokens | SPAs, mobile apps |
| **API Keys** | Header with a static token | 3rd-party integrations |
| **OAuth2 / SSO** | Login via Google, GitHub, etc. | SaaS, enterprise |

------------------------------------------------------------------------

## 🔑 Token Authentication (Built-in)

1.  Add `'rest_framework.authtoken'` to `INSTALLED_APPS`
2.  Run `python manage.py migrate`
3.  Add login view:

``` python
from rest_framework.authtoken.views import obtain_auth_token

urlpatterns = [
    path('api/token/', obtain_auth_token),
]
```

4.  Use it:

``` bash
POST /api/token/
{ "username": "user", "password": "pass" }

# Response:
{ "token": "abc123" }

# Then use:
Authorization: Token abc123
```

------------------------------------------------------------------------

## 🔐 JWT Authentication (Recommended for APIs)

1.  Install SimpleJWT:

``` bash
pip install djangorestframework-simplejwt
```

2.  Add to `settings.py`:

``` python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ]
}

from datetime import timedelta
SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=30),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
}
```

3.  Add routes:

``` python
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

urlpatterns = [
    path('api/token/', TokenObtainPairView.as_view()),
    path('api/token/refresh/', TokenRefreshView.as_view()),
]
```

✅ Usage:

``` bash
POST /api/token/ → get { access, refresh }
Authorization: Bearer <access>
```

------------------------------------------------------------------------

## 🔐 OAuth2.0 Authentication (Recommended for B2B & 3rd-Party APIs)

OAuth2 is an open protocol for secure delegated access — ideal for giving external apps access without exposing user credentials.

------------------------------------------------------------------------

### ✅ 1. Install dependencies

``` bash
pip install django-oauth-toolkit
```

------------------------------------------------------------------------

### 🛠 2. Add to `settings.py`

``` python
INSTALLED_APPS += [
    'oauth2_provider',
]

AUTHENTICATION_BACKENDS = (
    'django.contrib.auth.backends.ModelBackend',
    'oauth2_provider.backends.OAuth2Backend',
)

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'oauth2_provider.contrib.rest_framework.OAuth2Authentication',
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework_simplejwt.authentication.JWTAuthentication',  # optional
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}
```

------------------------------------------------------------------------

### 🔧 3. Configure OAuth2 settings (optional)

``` python
OAUTH2_PROVIDER = {
    'ACCESS_TOKEN_EXPIRE_SECONDS': 3600,
    'REFRESH_TOKEN_EXPIRE_SECONDS': 7 * 24 * 3600,
    'SCOPES': {
        'read': 'Read access to protected resources',
        'write': 'Write access to protected resources',
        'email': 'Access to your email',
        'profile': 'Access to your profile info',
    }
}
```

------------------------------------------------------------------------

### 🧩 4. Add URLs to `urls.py`

``` python
from oauth2_provider import views as oauth2_views

urlpatterns += [
    path('o/', include('oauth2_provider.urls', namespace='oauth2_provider')),
]
```

- `/o/token/`: Access token endpoint
- `/o/revoke_token/`: Token revocation
- `/o/authorize/`: Authorization code flow (if used)
- `/o/introspect/`: Validate tokens (optional)

------------------------------------------------------------------------

### 🧪 5. Create an Application

Use the Django admin panel:

**`/admin/oauth2_provider/application/add/`**

| Field | Example |
|----|----|
| Name | My API Client |
| Client type | Confidential |
| Authorization grant type | Client Credentials / Authorization Code / Password |
| User | Your internal user |

------------------------------------------------------------------------

### 📡 6. Use from Postman or `curl`

**Request token (Client Credentials grant):**

``` bash
POST /o/token/
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&
client_id=...&
client_secret=...
```

✅ Response:

``` json
{
  "access_token": "abc123",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "read write"
}
```

------------------------------------------------------------------------

### 🔐 7. Protect your views

``` python
from rest_framework import viewsets, permissions
from oauth2_provider.contrib.rest_framework import TokenHasScope

class ChatViewSet(viewsets.ModelViewSet):
    permission_classes = [permissions.IsAuthenticated, TokenHasScope]
    required_scopes = ['read', 'write']
```

------------------------------------------------------------------------

### 🔍 8. Token introspection

Validate token from external services:

``` bash
POST /o/introspect/
Authorization: Basic base64(client_id:client_secret)

token=abc123
```

✅ Response:

``` json
{
  "active": true,
  "scope": "read write",
  "client_id": "xyz"
}
```

------------------------------------------------------------------------

### 📘 Flow Options

| Grant Type              | Use Case                                  |
|-------------------------|-------------------------------------------|
| Authorization Code      | Browser-based user login (3rd-party apps) |
| Client Credentials      | Server-to-server (no user)                |
| Resource Owner Password | Trusted apps (legacy/mobile)              |
| Implicit (deprecated)   | Frontend-only apps (not recommended)      |

------------------------------------------------------------------------

### 🧪 Example: Use in `curl`

``` bash
curl -X POST http://localhost:8000/o/token/ \
  -d "grant_type=client_credentials" \
  -d "client_id=..." \
  -d "client_secret=..."
```

``` bash
curl http://localhost:8000/api/protected/ \
  -H "Authorization: Bearer <access_token>"
```

------------------------------------------------------------------------

## 🧠 Custom Authentication

Make a custom class if you want to: - Use `X-API-Key` headers - Authenticate based on certificate (mTLS) - Add throttling per user type

Example:

``` python
from rest_framework.authentication import BaseAuthentication

class CustomTokenAuth(BaseAuthentication):
    def authenticate(self, request):
        token = request.headers.get('X-Api-Key')
        if token == "expected_token":
            return (MyUserObject, None)
        return None
```

------------------------------------------------------------------------

## 🔐 Login via Session + CSRF (For Browsers)

- Requires logging in via Django LoginView or Admin
- Automatically sends `sessionid` cookie
- Must include CSRF token in all modifying requests

Good for: - Internal admin views - API browsable interface (`/api/` with DRF)

------------------------------------------------------------------------

## 🔒 Permissions (AuthZ, not AuthN)

Once the user is authenticated, **permissions** check whether they can do something:

| Class                       | Description                                 |
|:----------------------------|:--------------------------------------------|
| `AllowAny`                  | Open to everyone                            |
| `IsAuthenticated`           | Only logged-in users                        |
| `IsAdminUser`               | Only superusers                             |
| `IsAuthenticatedOrReadOnly` | Anyone can read, only authed can write      |
| Custom                      | `has_permission(self, request, view)` logic |

Used in your views like:

``` python
class MyViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]
```

------------------------------------------------------------------------

## 🔁 Session vs Token vs JWT (Comparison)

| Feature        | SessionAuth | TokenAuth          | JWT                   |
|:---------------|:------------|:-------------------|:----------------------|
| Cookie-based?  | ✅ Yes      | ❌ No              | ❌ No                 |
| Header-based?  | ❌ No       | ✅ Yes             | ✅ Yes                |
| Expiry Support | Manual      | Manual             | ✅ Built-in           |
| Stateless?     | ❌ No       | ✅ Yes             | ✅ Yes                |
| Best For       | Web apps    | Simple API clients | Production-grade APIs |

------------------------------------------------------------------------

## 🔐 Example: Login with JWT and Use API

``` bash
# Login
curl -X POST /api/token/ \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com", "password": "pass"}'

# Response:
{
  "access": "ey...",
  "refresh": "ey..."
}

# Use it
curl -H "Authorization: Bearer ey..." /api/clients/
```

------------------------------------------------------------------------

## 🔄 Refresh Tokens (JWT)

When access token expires, use refresh:

``` bash
POST /api/token/refresh/
{
  "refresh": "ey..."
}
```

Get a **new access token** without re-logging in.

------------------------------------------------------------------------

## 🔐 Advanced Use Cases

| Feature | Use |
|:---|:---|
| API key for 3rd parties | Custom auth class, or use [drf-api-key](https://github.com/florimondmanca/djangorest-framework-api-key) |
| Service accounts | Create special users, assign static tokens |
| mTLS authentication | Terminate TLS at Nginx/LB, forward identity headers |
| OAuth2 | Use `django-allauth`, `django-oauth-toolkit` |

------------------------------------------------------------------------

## 🧠 Diagram: DRF Auth Flow (JWT)

``` plaintext
[Client]
  |
  | -- POST /api/token/  -->  [DRF Login View]
  | <-- {access, refresh}
  |
  | -- GET /api/data/  (Authorization: Bearer <access>)
  | --> DRF checks JWT -> decodes -> authenticates user
```

------------------------------------------------------------------------

## ✅ Summary Cheat Sheet

| Concept | You Should Know |
|:---|:---|
| DRF supports multiple auth systems | ✅ Use `DEFAULT_AUTHENTICATION_CLASSES` |
| Token vs JWT | JWT is stateless + has expiry |
| SessionAuth | Great for browser use, requires CSRF |
| JWTAuth | Best for API use (SPAs, mobile) |
| Permissions are separate from auth | Use `permission_classes` |

------------------------------------------------------------------------

Back to top
