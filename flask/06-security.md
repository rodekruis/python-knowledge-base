# 06 — Security

## Security Headers

Add security headers via `@app.after_request`:

```python
@app.after_request
def add_security_headers(response):
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
    response.headers["Content-Security-Policy"] = "default-src 'self'"
    response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
    response.headers["Permissions-Policy"] = "camera=(), microphone=(), geolocation=()"
    return response
```

### Header Checklist

| Header | Purpose | Value |
|--------|---------|-------|
| `X-Content-Type-Options` | Prevent MIME sniffing | `nosniff` |
| `X-Frame-Options` | Prevent clickjacking | `DENY` |
| `Strict-Transport-Security` | Force HTTPS | `max-age=31536000; includeSubDomains` |
| `Content-Security-Policy` | Control resource sources | `default-src 'self'` (adjust per app) |
| `Referrer-Policy` | Limit referrer leakage | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | Disable unused browser APIs | `camera=(), microphone=(), geolocation=()` |
| `X-XSS-Protection` | Legacy XSS filter | `0` (deprecated; use CSP instead) |

> **Drop `X-XSS-Protection: 1; mode=block`**. It is [deprecated by all major browsers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/X-XSS-Protection) and can actually introduce vulnerabilities. Set to `0` or omit entirely. Use `Content-Security-Policy` for XSS protection.

### Content Security Policy

Tune **[CSP](README.md#glossary)** based on what your app actually loads:

```python
def _build_csp():
    directives = [
        "default-src 'self'",
        "script-src 'self'",                     # Add CDN domains if needed
        "style-src 'self' 'unsafe-inline'",      # Inline styles (WTForms may need this)
        "img-src 'self' data:",                   # data: for inline images
        "font-src 'self'",
        "frame-ancestors 'none'",                 # Like X-Frame-Options: DENY
        "base-uri 'self'",
        "form-action 'self'",                     # Forms can only POST to same origin
    ]
    return "; ".join(directives)
```

> Avoid `'unsafe-inline'` for scripts. If you need inline JS, use [nonces](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP#nonces):
> ```python
> import secrets
> nonce = secrets.token_hex(16)
> response.headers["Content-Security-Policy"] = f"script-src 'nonce-{nonce}'"
> ```

## CSRF Protection

### Setup

```python
from flask_wtf.csrf import CSRFProtect
csrf = CSRFProtect()

# In create_app():
csrf.init_app(app)
```

### In forms (server-rendered)

```html
<form method="POST">
    {{ form.hidden_tag() }}   {# WTForms: includes CSRF + hidden fields #}
    {# or manually: #}
    <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
</form>
```

### In AJAX / htmx

Include the token as a meta tag and send it in headers:

```html
<meta name="csrf-token" content="{{ csrf_token() }}">
```

```javascript
// For htmx
document.body.addEventListener('htmx:configRequest', (evt) => {
    evt.detail.headers['X-CSRFToken'] = 
        document.querySelector('meta[name="csrf-token"]').content;
});

// For fetch()
fetch('/api/endpoint', {
    method: 'POST',
    headers: {
        'X-CSRFToken': document.querySelector('meta[name="csrf-token"]').content,
        'Content-Type': 'application/json',
    },
    body: JSON.stringify(data),
});
```

## Session Security

```python
class Config:
    SECRET_KEY = os.environ["SECRET_KEY"]         # Fail fast
    SESSION_COOKIE_SECURE = True                  # Only over HTTPS
    SESSION_COOKIE_HTTPONLY = True                 # No JavaScript access
    SESSION_COOKIE_SAMESITE = "Lax"               # CSRF mitigation
    PERMANENT_SESSION_LIFETIME = 1800             # 30 min timeout
```

> **Important:** `PERMANENT_SESSION_LIFETIME` only applies when `session.permanent = True`. Set this explicitly after login:
> ```python
> session.clear()
> session["user_id"] = user.id
> session.permanent = True  # Required for PERMANENT_SESSION_LIFETIME to take effect
> ```

### Session best practices

1. **Generate a strong secret key**: `python -c "import secrets; print(secrets.token_hex(32))"`
2. **Rotate the secret key** using `SECRET_KEY_FALLBACKS` (Flask 3.1+) — add old keys to this list so active sessions remain valid during rotation, then remove old keys after an appropriate period
3. **Store minimal data** in sessions — IDs and flags, not full user objects
4. **Use server-side sessions** for sensitive applications (e.g., [Flask-Session](https://flask-session.readthedocs.io/) with Redis or a database backend). Flask's default client-side cookies are signed but not encrypted — users can Base64-decode and read the contents.

## Input Validation

### Use WTForms (see [03-routes-and-templates.md](03-routes-and-templates.md))

Never trust user input. Validate with WTForms for form data.

### For non-form inputs (query params, path params)

Validate explicitly:

```python
@main_bp.route("/program/<int:program_id>")
def program_detail(program_id):
    # Flask already validated `program_id` is an int
    if program_id < 1:
        abort(400, description="Invalid program ID")
    ...
```

### Avoid SQL injection

If using raw SQL, **always** use parameterized queries:

```python
# ❌ NEVER do this
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")

# ✅ Always parameterize
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
```

If using SQLAlchemy ORM, you're safe by default — the ORM parameterizes automatically.

## Host Header Validation (Flask 3.1+)

Flask 3.1 introduced the `TRUSTED_HOSTS` config option to validate the `Host` header natively. Set this in production to prevent host header injection attacks:

```python
class ProdConfig(Config):
    TRUSTED_HOSTS = ["myapp.azurewebsites.net", ".mydomain.org"]
```

Values can be exact matches or start with `.` to match any subdomain. Invalid hosts get a 400 response.

## Sensitive Data in URLs

❌ Bad — credentials or tokens in URL:
```python
@app.route("/login/<username>/<password>")  # Logged everywhere!
```

✅ Good — use POST body for sensitive data:
```python
@app.route("/login", methods=["POST"])
def login():
    username = request.form["username"]
    password = request.form["password"]
```

## Rate Limiting

For public-facing apps, add rate limiting to prevent brute force:

```python
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(get_remote_address, app=app, default_limits=["200 per day", "50 per hour"])

@app.route("/login", methods=["POST"])
@limiter.limit("5 per minute")
def login():
    ...
```
