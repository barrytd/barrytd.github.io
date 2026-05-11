---
title: "DVWA - Blind SQLi (High)"
date: 2026-02-18
platform: DVWA
difficulty: Medium
category: offensive
summary: "Boolean-based blind SQL injection at the High security level via Burp Intruder and response-length differentials, extracting the database one character at a time."
---

## Overview

DVWA (*Damn Vulnerable Web Application*) is a deliberately broken PHP app maintained for security learning. The blind SQL injection challenge at the **High** security level removes every easy SQLi technique: no error messages reach the client, the response is identical for true and false queries (in content), and the security level enforces session-bound CSRF tokens.

The only signal left is **response length**. Boolean-based blind SQLi against this configuration is the textbook example of *why timing/length oracles matter*.

## Tools Used

- **Burp Suite** with **Intruder** for the automated sweep.
- **Browser** to observe baseline responses.
- **A scratch text editor** for the SQLi payload templates.

## Methodology

**Step 1 - Understand the security level.** DVWA's *High* SQLi page accepts a *User ID* parameter. The query is:

```sql
SELECT first_name, last_name FROM users WHERE user_id = '<INPUT>';
```

On classic SQLi pages, the response shows the matched rows. On the High level, the response is a *single line*: *"User ID exists in the database."* if any rows match, identical content if none match (or the rows are hidden). There is no error message and no row-count display.

The blind-injection signal is something subtler: the **response length differs** between *exists* and *not exists*.

**Step 2 - Confirm with a manual probe.** Submit *1* and *9999999*. The first returns the "exists" response (true). The second returns either "exists" or "not exists" (false). The length differs.

**Step 3 - Craft a boolean-based payload.** The classic blind-SQLi shape is:

```sql
1' AND (SELECT 1 FROM users WHERE SUBSTRING(password,1,1)='a' AND username='admin') --
```

If the first character of admin's password is *a*, the AND is true, the row matches, response is long. If not, response is short.

**Step 4 - Automate with Burp Intruder.** Burp Suite proxies the browser request, then *Intruder* lets you replace any part of the payload with a list of values and run them as separate requests. With Intruder configured to swap the guessed character through *a-z, 0-9*, and the response-length column shown, the correct character is the one with a different length.

**Step 5 - Walk the string.** Once the first character is recovered, change the SUBSTRING offset to 2 and repeat. Then 3, 4, ... until the full string is extracted. For a 32-character MD5 hash, that's 32 × 36 = 1152 requests.

**Step 6 - Scope reduction.** Each character has 36 possibilities (lowercase + digits) for an MD5 hex hash. Binary search (`<` and `>` comparisons) reduces that to ~6 requests per character, or 192 total instead of 1152.

```sql
1' AND (SELECT 1 FROM users WHERE ASCII(SUBSTRING(password,1,1)) < 100 AND username='admin') --
```

Each request narrows the range by half. Walking from 32-127 (ASCII printable range) to a single character is log2(96) ≈ 7 requests.

## Key Takeaways

- **Blind SQLi works as long as *any* observable property differs between true and false.** Response length, response time, redirect location, HTTP status code: any of them is enough.
- **Binary search drastically reduces request count.** Walking a 32-char string with 7 requests per character is 224 total requests, which finishes in seconds. Linear scanning is 1000+ requests.
- **Burp Intruder is the standard automation tool.** Free version is rate-limited but enough to learn the technique.
- **High-security flag in DVWA mostly just adds CSRF tokens and removes error messages.** The underlying bug is unchanged.

## What a Defender Should Do

- Use parameterized queries. Prepared statements with bind parameters make SQL injection structurally impossible. There is no good reason to concatenate user input into a SQL string in 2024+.
- Apply WAF rules that detect SQLi patterns in URL parameters (single quotes, common keywords, length-based brute-force patterns). ModSecurity OWASP CRS is mature and free.
- Rate-limit query-style endpoints. 1000 requests to */vulnerabilities/sqli/* in a minute from one IP is obviously not human.
- Strip error messages from production responses. The High-level page does this; lower levels leak error detail in dev configs that often make it to prod.
- Audit web app logs for boolean-style payloads. *AND 1=1*, *OR '1'='1'*, *SUBSTRING(...)*, *SLEEP(...)* are all distinctive signatures.
