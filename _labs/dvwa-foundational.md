---
title: "DVWA - Foundational Vulnerabilities"
date: 2026-02-16
platform: DVWA
difficulty: Easy
category: offensive
summary: "OWASP Top 10 coverage across DVWA's foundational modules: SQL injection, reflected and stored XSS, CSRF, command injection, IDOR, and Local File Inclusion via php://filter."
---

## Overview

This was my entry point into web application security: a single session through every foundational module in DVWA. The goal was *coverage*, not depth. Each module is a single textbook example of a class of vulnerability, and seeing them all in one app teaches the *shape* of each bug class.

The bugs covered: **SQL Injection**, **Cross-Site Scripting** (reflected and stored), **Cross-Site Request Forgery**, **OS Command Injection**, **Insecure Direct Object Reference**, and **Local File Inclusion** with PHP filter wrappers.

## Tools Used

- **A browser** and DevTools.
- **Burp Suite** for request interception and CSRF token analysis.
- **A scratch text editor** for payload templates.

## Methodology

### SQL Injection

Same shape as everything else: input concatenated into a query, classic *'OR '1'='1* style bypass at the low security level, UNION SELECT at the medium level. Covered in detail in the DVWA Blind and DVWA SQLi/API/Sessions writeups.

### Reflected XSS

The form takes a *name* parameter and echoes it back into the HTML page without escaping.

```
?name=<script>alert(1)</script>
```

The browser parses the response, sees the *script* tag, and executes the alert. In a real attack, the payload would steal the session cookie and send it to an attacker-controlled URL.

The fix is **HTML output encoding** on every user-supplied value. Modern templating engines (Twig, Jinja, ERB, Razor) do this by default. The bug here is that the DVWA code echoes raw.

### Stored XSS

The guestbook page accepts a comment, stores it in the database, and renders it to every visitor. A *script* tag in the comment executes for every viewer.

This is more dangerous than reflected XSS because it does not require the attacker to trick anyone: every visitor automatically runs the payload.

### CSRF

The password-change form accepts a new password via POST and *does not verify a CSRF token*. Any third-party page that triggers a POST to the password-change URL (via an auto-submitting form or an `<img>` with the URL as source for GET-based CSRF) can change the victim's password.

```html
<form method="POST" action="http://dvwa/vulnerabilities/csrf/" id="x">
  <input name="password_new" value="newpass">
  <input name="password_conf" value="newpass">
  <input name="Change" value="Change">
</form>
<script>document.getElementById('x').submit();</script>
```

If the victim visits a page with this code while logged into DVWA, their password is changed.

The fix is a **per-form anti-CSRF token** (a random string the server issues and validates on submit). Modern frameworks do this by default; the DVWA bug is the absence of the token at the low security level.

### Command Injection

The ping form passes user input to a shell call without sanitization.

```
127.0.0.1; cat /etc/passwd
```

Same pattern as Kioptrix Level 2.

### IDOR (Insecure Direct Object Reference)

The profile page shows `/profile?user_id=123`. Changing the value to 124 shows another user's profile. **The application authenticates the user but does not authorize each resource access**: it never asks *"is this user allowed to read user_id 124?"* before returning the data.

This is **CWE-639** and is the most common API authorization bug in real engagements.

### LFI (Local File Inclusion) via php://filter

The page includes a file based on a *page* parameter:

```
?page=about.php
```

The shape is `include($_GET['page'])`. With *allow_url_include* off, the parameter can still be local files via path traversal:

```
?page=../../../../etc/passwd
```

A more interesting payload uses **php://filter** to read PHP source files without executing them, which is the standard technique for stealing application source from an LFI.

```
?page=php://filter/convert.base64-encode/resource=config
```

The wrapper base64-encodes the file contents before returning them, which prevents PHP from interpreting them as code. The attacker then base64-decodes the response to read the source.

## Key Takeaways

- **The OWASP Top 10 is the standard reference for a reason.** Every web app has at least one of these somewhere if no one has audited it.
- **Each bug class has a *shape*.** Once you've seen reflected XSS once, you recognize it in any framework. Same for SQLi, command injection, IDOR.
- **Authorization is harder than authentication.** Authentication answers *"who is this user?"* Authorization answers *"is this user allowed to do this thing?"* IDOR is the universal authorization failure.
- **php://filter is the LFI-to-source-read pivot.** Always try it on any LFI before assuming the bug only reads flat files.

## What a Defender Should Do

- **Parameterize SQL.** Use prepared statements universally.
- **Output-encode every user-supplied value.** Use a templating engine that does this by default. Manually echoing raw input is the bug.
- **Issue and validate anti-CSRF tokens on every state-changing request.** Modern frameworks do this for free; legacy code rarely does.
- **Authorize every resource access.** Never trust a client-supplied object ID without checking that the current user owns or can access that object.
- **Strip path traversal characters from any file-reading parameter.** Better: use an allow-list of valid file IDs and map them server-side to filesystem paths.
- **Set `allow_url_include = Off` in php.ini.** It's the single line that prevents remote-file-inclusion-to-RCE.
