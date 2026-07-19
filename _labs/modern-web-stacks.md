---
title: "Modern Web Stacks - Fingerprint the Stack, Then Target the CVE"
date: 2026-07-11
platform: TryHackMe
difficulty: Easy
category: offensive
summary: "Four stacks, four CVEs. Reading a stack from its HTTP signals so you know the attack surface before a scanner finishes running."
---

Every web stack leaves fingerprints: in its headers, its cookie names, its error pages, and little patterns in the HTML. Once you know the stack and its version, you have a good idea of what can go wrong with it. This room ran the same three steps against four different stacks: read the signals, confirm the version, run the exploit. The point is that aiming at a known stack beats spraying a generic scanner, because the interesting bugs often live in one specific feature a scanner never thinks to test.

## The workflow

1. Fingerprint the stack from its HTTP signals, no payloads yet.
2. Confirm the version and find the CVE that matches.
3. Exploit it, and understand why the bug was there in the first place.

A version banner points you at a CVE, but it does not prove the target is actually vulnerable. A server can be patched behind the scenes while still showing an old version, or the vulnerable feature might be switched off. So the banner narrows the search, and you confirm before firing. This room made that concrete: before the Apache exploit, you check that /cgi-bin/ returns 403, which confirms the feature the exploit needs is actually turned on.

## MERN (Express) - prototype pollution

The short version: one crafted JSON object tricks the app into treating you as an admin.

Signals: X-Powered-By: Express, a connect.sid session cookie, and a plain Cannot GET /nonexistent on an unknown route.

The bug lives in a hand-written function that merges JSON into a user object without checking the keys. Sending {"__proto__": {"isAdmin": true}} does not just set a value on your object, it sets isAdmin: true on the template that every object inherits from. When the admin route checks currentUser.isAdmin and finds nothing on the object itself, it falls back to that shared template and finds the true you planted. That is an auth bypass from a single JSON payload.

## Next.js - middleware bypass (CVE-2025-29927)

The short version: one header you send yourself skips the app's login checks.

Signals: X-Powered-By: Next.js, window.__next_f in the page source, and /_next/static/chunks/ asset paths.

Next.js uses an internal header, x-middleware-subrequest, to avoid running its middleware twice on nested calls. The flaw was that it never checked whether that header came from inside the app or from the outside. Since middleware is where most Next.js apps put their login checks, sending that header yourself skips the check completely. One header, full auth bypass, rated CVSS 9.1.

## Django - SQL injection (CVE-2021-35042)

The short version: a sort parameter gets pasted straight into a database query, so you can read the database through it.

Signals: Server: WSGIServer/0.2 CPython/X.X.X, a csrftoken cookie, and the csrfmiddlewaretoken hidden field in POST forms, which is the near-certain tell for Django.

A page view drops the order parameter straight into an ORDER BY clause with no cleaning. A crafted payload forces a database error that leaks query results, and with debug mode left on, Django prints that result in the 500 error page. From there you can pull the version, the database name, and anything else you can query. This is what happens when code skips the framework's safe database layer and builds SQL by hand.

## LAMP (Apache 2.4.49) - path traversal to RCE (CVE-2021-41773)

The short version: a broken path filter lets you walk out of the web folder and run commands, with no login.

Signals: Server: Apache/2.4.49 (Unix), the version repeated in 404 footers, and /cgi-bin/ returning 403 instead of 404, which means the CGI feature is on.

Apache 2.4.49 shipped a broken filter for path traversal (the ../ trick for climbing out of a folder). The encoded form .%2e/ slips past the check, but the operating system still reads it as ../. Walk the path to a system shell, and because /cgi-bin/ is set up to run programs, Apache runs it with your request body as the commands. That is unauthenticated remote code execution. You need curl --path-as-is so curl does not "helpfully" clean up the encoded dots before sending them.

## Automation with Nikto

Nikto does the fingerprinting pass for you: nikto -h http://TARGET:PORT reads the headers and reports the server banner, the allowed methods, and misconfigurations in about ten seconds. It is a fast first look across many hosts, and for the Apache port it handed over the exact version that maps to the CVE. What it does not do is find the app-level bugs. The prototype pollution and the SQL injection both needed manual work. Nikto tells you where to look, it does not exploit.

## Key Takeaways

The banner is the lead, not the proof. Read the signals, confirm the version and that the vulnerable feature is actually on, then exploit. Fingerprinting turns a noisy guess-and-check into a targeted attack, and it is faster than waiting on a scanner, because you already know the CVE before the scan finishes.
