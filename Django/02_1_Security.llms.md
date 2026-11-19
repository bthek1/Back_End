# Security

Security is a critical aspect of any software development process. It involves implementing measures to protect software applications from threats and vulnerabilities. This includes secure coding practices, regular security audits, and using tools to identify and mitigate risks.

Author

Benedict Thekkel

## 🔐 1. **Hashing**

| Feature           | Description                                    |
|-------------------|------------------------------------------------|
| What              | One-way transformation of data                 |
| Use Case          | Secure password storage, file integrity checks |
| Can be reversed?  | ❌ No                                          |
| Common Algorithms | SHA-256, bcrypt, PBKDF2, Argon2                |

### 🔧 Web Use

- Store only the **hashed + salted** password (e.g. Django uses `PBKDF2` by default).
- Never store plain passwords.
- Validate with `check_password()`.

------------------------------------------------------------------------

## 🔑 2. **Encryption**

| Feature          | Description                               |
|------------------|-------------------------------------------|
| What             | Reversible encoding using a key           |
| Use Case         | Secure transmission (HTTPS), data at rest |
| Can be reversed? | ✅ Yes, with key                          |
| Types            | Symmetric (AES), Asymmetric (RSA)         |

### 🔧 Web Use

- TLS/SSL encrypts HTTP traffic (HTTPS).
- Encrypt sensitive database fields (e.g. card details) using Django-encrypted-fields or Fernet.

------------------------------------------------------------------------

## 🔑 3. **Keys (Symmetric & Asymmetric)**

| Type | Use | Example |
|----|----|----|
| Symmetric | Same key for encrypt/decrypt | AES |
| Asymmetric | Public key encrypts, private key decrypts | RSA, HTTPS certificates |

### 🔧 Web Use

- HTTPS uses **TLS** with asymmetric keys to establish secure sessions.
- JWTs are signed with private keys (asymmetric) or HMAC secrets (symmetric).

------------------------------------------------------------------------

## 🔄 4. **Authentication vs Authorization**

| Term               | Means                   | Example              |
|--------------------|-------------------------|----------------------|
| **Authentication** | Verifying identity      | Login form           |
| **Authorization**  | Verifying access rights | Accessing admin page |

------------------------------------------------------------------------

## 🔐 5. **JWT (JSON Web Token)**

- Compact, self-contained token used for **stateless authentication**.
- Contains `header`, `payload`, and `signature`.

### 🔧 Web Use

- Used in APIs to authenticate users (`Authorization: Bearer <token>`).
- Must be signed and verified with a secret.

------------------------------------------------------------------------

## 🍪 6. **Cookies**

| Attribute  | Use                                    |
|------------|----------------------------------------|
| `HttpOnly` | JS cannot read (safer)                 |
| `Secure`   | Sent only over HTTPS                   |
| `SameSite` | Protects from CSRF (`Lax` or `Strict`) |

### 🔧 Web Use

- Store session IDs or refresh tokens.
- Set flags: `Secure`, `HttpOnly`, `SameSite=Lax`.

------------------------------------------------------------------------

## 🧪 7. **CSRF (Cross-Site Request Forgery)**

What \| Attack where logged-in users are tricked into submitting unintended requests \|\
Prevent with \| CSRF tokens, `SameSite` cookies, checking `Origin` headers \|

### 🔧 Web Use

- Django includes CSRF protection out of the box.
- You must use `{% csrf_token %}` in HTML forms.
- Not needed for **stateless JWT APIs**.

------------------------------------------------------------------------

## 🧼 8. **XSS (Cross-Site Scripting)**

What \| Inject malicious JS into pages \|\
Types \| Stored, Reflected, DOM \|\
Prevent with \| Escaping output, input validation, Content Security Policy (CSP) \|

### 🔧 Web Use

- Never render raw HTML from users.
- Django auto-escapes `{ variable }`.

------------------------------------------------------------------------

## 🌐 9. **CORS (Cross-Origin Resource Sharing)**

What \| Controls which origins can access your API \|\
Prevents \| Unauthorized frontend domains from calling your backend \|\
Headers \| `Access-Control-Allow-Origin`, `Access-Control-Allow-Credentials` \|

### 🔧 Web Use

- Use `django-cors-headers` to allow trusted origins (e.g., your React frontend).

------------------------------------------------------------------------

## 🔐 10. **HTTPS and SSL/TLS**

What \| Encrypts traffic between client and server \|\
Why \| Prevents MITM (Man-in-the-Middle) attacks \|\
Implement with \| TLS certificates (e.g., Let’s Encrypt) \|

### 🔧 Web Use

- Use HTTPS in production.
- Redirect all HTTP traffic to HTTPS.

------------------------------------------------------------------------

## 🚦 11. **Rate Limiting & Throttling**

What \| Limits API usage per user/IP \|\
Prevents \| Abuse, brute force, scraping \|\
Tools \| Django Ratelimit, DRF throttling \|

### 🔧 Web Use

``` python
REST_FRAMEWORK = {
  'DEFAULT_THROTTLE_CLASSES': [
    'rest_framework.throttling.UserRateThrottle',
    'rest_framework.throttling.AnonRateThrottle',
  ],
  'DEFAULT_THROTTLE_RATES': {
    'user': '1000/day',
    'anon': '100/day',
  }
}
```

------------------------------------------------------------------------

## 🧾 12. **Secure HTTP Headers**

| Header                            | Purpose                |
|-----------------------------------|------------------------|
| `X-Frame-Options: DENY`           | Prevent clickjacking   |
| `X-Content-Type-Options: nosniff` | Stop MIME-type attacks |
| `Content-Security-Policy`         | Prevent inline scripts |
| `Strict-Transport-Security`       | Enforce HTTPS          |

### 🔧 Web Use

- Use Django’s `SecurityMiddleware` and tools like `django-secure`.

------------------------------------------------------------------------

## 🧠 13. **Security Practices for Web Developers**

| Task              | Best Practice                                          |
|-------------------|--------------------------------------------------------|
| Store passwords   | Use `make_password()` and `check_password()` (Django)  |
| Store secrets     | Use `.env` files or secret managers (e.g., AWS SSM)    |
| Input validation  | Validate both client and server-side                   |
| Error handling    | Don’t expose stack traces to users                     |
| Logging           | Log auth failures, unusual access patterns             |
| Dependency checks | Use `pip-audit`, `safety` to check vulnerable packages |

------------------------------------------------------------------------

## ✅ Security Checklist Summary

| Area      | What to Do                           |
|-----------|--------------------------------------|
| Passwords | Hash + salt                          |
| Sessions  | Secure cookies                       |
| APIs      | Use tokens (JWT, OAuth)              |
| CORS      | Allow only trusted domains           |
| CSRF      | Protect forms and cookies            |
| XSS       | Escape output                        |
| HTTPS     | Use TLS and redirect HTTP            |
| Headers   | Set security headers                 |
| Secrets   | Keep out of codebase                 |
| Logs      | Monitor failures + suspicious events |

------------------------------------------------------------------------

Back to top
