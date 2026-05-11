---
title: "DVWA - SQLi, API & Sessions"
date: 2026-02-17
platform: DVWA
difficulty: Medium
category: offensive
summary: "UNION-based and boolean-based SQLi at elevated DVWA security, plus an API v1/v2 exposure and predictable session ID enumeration."
---

## Overview

This DVWA session covered three issues that frequently chain together in real engagements: **UNION-based SQL injection**, **API version exposure** (an older /v1 endpoint with weaker access controls than the current /v2), and **predictable session IDs** that can be brute forced.

The lesson is that these vulnerabilities are individually well-known but constantly recur in production because they hide in less-glamorous parts of the codebase: helper modules, legacy endpoints, "we forgot to remove the v1 API" routers.

## Tools Used

- **Burp Suite** for request interception and Intruder.
- **Browser DevTools** for cookie inspection.
- **Python** for the session ID brute force.

## Methodology

### UNION-based SQLi

**Step 1 - Determine the column count.** Classic UNION SQLi needs the attacker's UNION SELECT to have the same number of columns as the original query.

```
1' ORDER BY 1-- -
1' ORDER BY 2-- -
1' ORDER BY 3-- -
```

The query that errors (or returns a "Unknown column" message at lower security levels) reveals the column count.

**Step 2 - Identify rendered columns.** Once the column count is known, UNION SELECT distinctive values to find which columns the page renders.

```
1' UNION SELECT 1,2-- -
```

The page shows *1* and *2* in the user/password fields, confirming both columns are rendered.

**Step 3 - Extract data.**

```
1' UNION SELECT user, password FROM users-- -
```

The page lists every user and their password hash from the *users* table.

**Step 4 - Schema enumeration.** Before knowing the table names, use the information_schema:

```
1' UNION SELECT table_schema, table_name FROM information_schema.tables-- -
1' UNION SELECT column_name, table_name FROM information_schema.columns WHERE table_name='users'-- -
```

These return the database structure, including every table name and every column name.

### API v1/v2 exposure

**Step 5 - Endpoint discovery.** The app has both /api/v1/ and /api/v2/ endpoints. Modern code calls v2 because the developers added access controls there. The v1 endpoints are still mounted *for backward compatibility*.

```
GET /api/v2/user/123    → 403 (correctly enforced auth)
GET /api/v1/user/123    → 200, returns the user's record
```

This is a recurring real-world finding: companies deploy a new API version, never decommission the old one, and the old one becomes the unauthenticated back door.

### Predictable session IDs

**Step 6 - Inspect the session cookie.** Open DevTools → Application → Cookies. The session cookie value is something like *PHPSESSID=00000000123*. The format reveals it's a counter, not a cryptographically random string.

**Step 7 - Brute force the cookie value.** A small Python script iterates from 1 to N, sets the cookie value, and requests an authenticated endpoint. Any response that doesn't redirect to the login page is a hit.

```python
import requests
for i in range(1, 1000):
    cookie = {"PHPSESSID": f"{i:011d}"}
    r = requests.get("http://target/dashboard", cookies=cookie)
    if "Welcome" in r.text:
        print(f"Hit at PHPSESSID={i}")
```

## Key Takeaways

- **UNION SQLi is loud but extremely productive.** When the page renders its results, UNION returns full table contents in one request.
- **information_schema is the attacker's friend.** Every MySQL/MariaDB/Postgres database exposes its own structure to any authenticated user.
- **API version drift is a real-world finding.** Whenever you see */v2/* in URLs, check if */v1/* is still there.
- **Predictable session IDs are a 1990s bug that still ships in 2024.** Any pattern that looks like a counter, a timestamp, or a username-based hash is brute-forceable.

## What a Defender Should Do

- **Parameterize every database query.** Same advice as Kioptrix and DVWA Blind: prepared statements are the structural fix.
- Decommission old API versions on a clear timeline. Add a 410 Gone response for /v1 endpoints six months after /v2 ships, and remove them entirely a year later.
- Generate session IDs from a CSPRNG (cryptographically secure pseudo-random number generator): `random.SystemRandom()` in Python, `crypto.randomBytes()` in Node, `random_bytes()` in PHP 7+. Never timestamps, never counters, never hash(username).
- Set session IDs to at least 128 bits of entropy. PHP's default is sufficient; many legacy apps use shorter formats.
- Audit endpoint-level access control on every route on every API version. The router config is the audit target.
- Log and alert on session-ID brute-force patterns: same source IP, many cookie values per second, all 302 redirecting to login.
