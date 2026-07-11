---
title: "Web Server Attacks I - Misconfigurations Across Apache, Nginx, Node, and Python"
date: 2026-07-11
platform: TryHackMe
difficulty: Easy
category: offensive
summary: "Four web servers on one host, the same misconfiguration patterns on each: version disclosure, directory listing, exposed status pages, and missing security headers."
---

Four different web servers, one machine, and the same handful of mistakes on every one. This room ran the same checks against Apache, Python's built-in HTTP server, Node.js Express, and Nginx. The lesson underneath it: default settings are built for an easy setup, not for security, so version banners, directory listings, and status pages are usually on out of the box and stay on until someone turns them off.

## Fingerprinting first

Before poking at anything, work out which server you are talking to. Two response headers do most of the work. The `Server` header names the software and often the version. `X-Powered-By` names the app framework when `Server` is generic or missing.

```bash
curl -sI http://TARGET:PORT       # headers only
curl -s http://TARGET:PORT/nope   # default 404 page, another fingerprint
```

Each server has a tell:

- Apache shows `Apache/2.4.x (Ubuntu)`.
- Python's built-in server shows `SimpleHTTP/0.6 Python/3.x`.
- Nginx shows `nginx/1.x`.
- Express sends no `Server` header at all, but its `X-Powered-By: Express` gives it away.

The 404 page also looks different on each server, so you can usually tell them apart even when the header has been stripped.

## Python HTTP server

`python3 -m http.server` serves the entire current folder with no login, no logging, and no blocklist. It will even hand over dotfiles like `.env` that Apache and Nginx would hide. The problem is not a bug in the server, it is that the server is running somewhere it should not be. Check the root listing, pull any `.env` for credentials, and download any archive you find, since backups often hold source code or database dumps.

## Apache

Default Ubuntu Apache leaves three things worth checking:

- **Directory listing** when `Options +Indexes` is on, which exposes a browsable file list at paths like `/files/`.
- **`/server-status`**, the `mod_status` page. It ships locked to localhost, but a stray `Require all granted` in a site config can quietly expose it to everyone, leaking live requests and internal paths. Always check it.
- **Backup files** in the web root. Gobuster with `-x bak,txt` finds unlinked `.bak` copies and `.htpasswd` files, which often hold credentials or config.

## Node.js Express

Express apps run code rather than serving static files, so the mistakes here are development features left switched on in production:

- A custom error handler that leaks stack traces with internal file paths and SQL queries.
- A debug route like `/api/routes` that lists every endpoint, or `/api/debug/env` that dumps `process.env` with database passwords.
- `NODE_ENV: development` on a live server, which on its own says it was never hardened.
- Config files served straight to the browser (`/static/config.js`) that leak internal hostnames and debug flags.

## Nginx

Same patterns as Apache, different setting names. `server_tokens on` (the default) shows the version in the header and the 404 footer. `autoindex on` is Nginx's directory listing. `stub_status` at a path like `/nginx_status` is the `mod_status` equivalent, leaking connection stats when it is not locked to localhost.

## The patterns that repeat

Across all four servers, the same categories keep showing up: version disclosure in headers, directory listing, an exposed status or debug page, sensitive files left reachable, and missing security headers. None of the four set `X-Frame-Options`, `X-Content-Type-Options`, `Content-Security-Policy`, or `Referrer-Policy` by default. A quick check across the ports:

```bash
for p in 80 8000 3000 8080; do echo "=== $p ==="; \
  curl -sI http://TARGET:$p/ | grep -iE "x-frame|x-content-type|content-security|referrer-policy"; done
```

Nikto automates the whole sweep. `nikto -h http://TARGET:80 -nointeractive` flags the exposed status page, the directory listing, the backup files, and the missing headers in about ten seconds.

## Key Takeaways

Looking at each server on its own hides the point: the misconfigurations repeat because the permissive defaults repeat. Fingerprint with the headers, check for directory listing and status pages, look for sensitive files, and audit the security headers. That short checklist covers most of what a misconfigured web server will hand you, whichever server it is.
