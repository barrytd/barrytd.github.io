---
title: "Decryptify"
date: 2026-04-29
platform: TryHackMe
difficulty: Medium
category: offensive
summary: "Forged invite codes by brute forcing a leaked seed constant, then padbuster against a date parameter to decrypt and re-encrypt commands for full RCE."
---

## Overview

Decryptify is a medium-difficulty TryHackMe room that focuses on **applied cryptography**: deterministic token generation, the padding oracle attack, and CBC bit-flipping. It is one of the cleanest hands-on introductions to *"the cryptography library was used correctly, but the surrounding code is the bug"*.

The chain is: an app.log leaks an invite-code-and-email pair, the deobfuscated client-side JavaScript reveals the algorithm shape, brute force recovers the one unknown constant, a forged invite code grants access, and an exposed *date* parameter that is decrypted and executed as a shell command is exploited end to end with **padbuster**.

## Tools Used

- **nmap** for the port scan.
- **nessus** for a vulnerability scan baseline.
- **gobuster** for directory discovery on the unusual port.
- **base64** for decoding the invite code.
- **php** for the brute force script.
- **Firefox** for the dashboard.
- **padbuster** (Perl-based padding oracle attack tool) for both decryption and forgery.

## Methodology

**Step 1 - Service enumeration.** Two services: SSH on 22, and a web app on the unusual port 1337 (Apache 2.4.41). Nessus surfaces 21 mostly-informational findings.

```bash
nmap -p- -sV <TARGET_IP>
```

**Step 2 - Directory brute force.** gobuster surfaces */css, /js, /javascript, /logs, /phpmyadmin*. The two leads are */logs* and */javascript*.

**Step 3 - app.log leak.** /logs/app.log is world-readable from the web root and contains a sequence of timestamps and login events. One line discloses a complete *email + invite code* pair:

```
2025-01-23 14:34:20 - Invite created, code: <base64-string> for alpha@fake.thm
```

This is the **known-plaintext lever** that will be used to reverse-engineer the secret constant in the algorithm.

**Step 4 - Recover the algorithm.** /javascript/api.js is obfuscated but the embedded string array reveals base64 encoding plus the structure of token generation. Returning to /api-docs after some probing fully exposes the *Token Generation* algorithm. Decoding the leaked invite code with base64 produces a plain integer, which means the *invite code* is the base64-encoded output of PHP's *mt_rand()* (a pseudo-random number generator) seeded by a deterministic function of the email and a single unknown constant.

```bash
echo -n "<base64-string>" | base64 -d
```

**Step 5 - Brute force the constant.** A small PHP script iterates the unknown constant from 0 upward, runs the algorithm against *alpha@fake.thm*, and stops when the produced invite code matches the leaked value. The match returns in milliseconds, which is the entire reason *mt_rand seeded with a predictable seed* is broken.

**Step 6 - Forge an invite for a fresh email.** The app.log also mentioned *"New user created: hello@fake.thm"*. Re-running the algorithm with that email and the recovered constant produces a working invite code. Submitting it on the *Login with Invite Code* tab grants authenticated access.

**Step 7 - Spot the padding oracle.** The dashboard URL accepts a *date* parameter. Sending an arbitrary plaintext value like `?date=2024-01-01` triggers two highly informative server errors:

```
Warning: openssl_decrypt(): IV passed is only 6 bytes long
Padding error: error:0606506D:digital envelope routines:EVP_DecryptFinal_ex:wrong final block length
```

This is a **padding oracle**: the server reveals whether decrypted data has valid padding (the standard filler bytes appended to make the message a multiple of the block size, called PKCS#7) through its error responses. The attacker doesn't need to know the encryption key. By tampering with the ciphertext one byte at a time and watching whether the server says *valid* or *padding error*, they can recover the original plaintext byte by byte.

**Step 8 - Decrypt with padbuster.** padbuster automates the byte-by-byte oracle queries. The block size is **8** bytes (which rules out AES and points strongly at Blowfish) and encoding mode is **0** (base64).

```bash
padbuster "http://<TARGET>:1337/dashboard.php?date=<URL_ENCODED_CT>" \
          "<URL_ENCODED_CT>" \
          8 -encoding 0 \
          -cookies "PHPSESSID=...; role=..."
```

After thousands of byte-by-byte requests, padbuster recovers the intermediate state (the in-between value the server produces during decryption, before the final XOR step that gives the real plaintext) and from there, the actual plaintext.

**Step 9 - Forge a malicious ciphertext.** The same oracle that allows decryption also allows **encryption** of attacker-chosen plaintext. This is called **CBC bit-flipping**: because CBC mode uses simple XOR math, once the intermediate values are known you can run the math backwards to build a fake ciphertext that decrypts to whatever you want, no key required.

```bash
padbuster ... 8 -encoding 0 -cookies "..." -plaintext "cat /home/ubuntu/flag.txt"
```

padbuster outputs a freshly-minted base64 ciphertext that the server's key will decrypt into the chosen command, which the dashboard then pipes into *system()*. Final flag drops out in the page footer.

## Key Takeaways

- **A padding oracle is bidirectional.** The same yes/no question pattern that lets you decrypt the server's ciphertext also lets you encrypt your own chosen text. Forgetting the encryption half is the most common reason this bug class looks "harmless" in a triage.
- **Custom token generators built on mt_rand plus a predictable seed are not random in any meaningful sense.** The moment one token leaks through a log, a support ticket, or a screenshot, the entire token space collapses.
- **Verbose error messages are not a low-severity finding when the underlying operation is cryptographic.** Either of the two errors here was enough to seed the oracle.
- **Decryption is parsing, not authentication.** Just because data was encrypted doesn't mean the decrypted value is safe to trust. Anything decrypted from user-controlled ciphertext is still attacker-influenced input.

## What a Defender Should Do

- Generate invite codes from `random_bytes()` or `openssl_random_pseudo_bytes()` (PHP's cryptographically secure random functions) and store them server-side. Never derive tokens from user-controllable inputs through deterministic functions.
- Move logs outside the web root. Strip secrets and tokens from log messages before they are written.
- Use **authenticated encryption** like AES-GCM or AES-CCM (modes that verify a built-in cryptographic signature before decrypting, so tampered ciphertext fails the check and never reveals padding info).
- If CBC must be used, prepend an HMAC and verify it in **constant time** (a comparison that always takes the same amount of time, so an attacker cannot learn from response timing how many bytes matched) before calling *openssl_decrypt*.
- Never differentiate padding errors from authentication errors in client-visible responses. Return a single generic 400 for all decryption failures and log the detail server-side only.
- Do not pass user-controlled values, encrypted or otherwise, into shell commands.
