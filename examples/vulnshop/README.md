# VulnShop — `/hack-me` against a shop that doesn't list its bugs

[VulnShop](https://github.com/dev-abhrajit/VulnShop) (upstream) is a small
third-party Flask + SQLite store. The running UI brands itself **NexCart** and
the navigation is products, search, and account — there is no vulnerability menu
and no "click here to exploit" link, which is the gap DVWA leaves open.

The upstream README does name the classes (SQLi, XSS, CSRF, broken access
control). This is not a fully blind-to-the-repo find. It is a find against the
running shop: map the routes, probe, prove with a request, patch, re-verify.
We don't vendor the app.

## What `/hack-me` found

| # | Finding | Class | Status |
|---|---------|-------|--------|
| 1 | `/products?id=` UNION-dumps every plaintext password | SQL injection | Fixed → empty listing |
| 2 | `/login` accepts `' OR '1'='1` and issues an admin session | SQL injection (auth bypass) | Fixed → login page, no session |
| 3 | `/search?q=` reflects HTML unescaped (Jinja `safe` filter) | Reflected XSS | Fixed → escaped |
| 4 | Any logged-in user opens `/admin` and reads every password | Broken function auth | Fixed → 403 |
| 5 | `POST /profile` changes the email with no CSRF token | CSRF | Fixed → 400; real form still works |

Receipts, including the requests and the re-verification, are in
[`HACKME_REPORT.md`](HACKME_REPORT.md).

## Reproduce

```bash
git clone https://github.com/dev-abhrajit/VulnShop && cd VulnShop
pip install flask
python3 database.py
python3 app.py          # http://127.0.0.1:5000
# then run /hack-me (../../commands/hack-me.md) against that URL
```

Guardrails, as always: your own / authorized target, localhost only,
non-destructive proofs. The seed accounts (`admin` / `admin123`, `alice` /
`pass123`, `bob` / `bob456`) are the app's own lab data.
