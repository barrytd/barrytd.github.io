---
title: "Protocols and Servers - Reading Cleartext Email and File Transfer by Hand"
date: 2026-07-04
platform: TryHackMe
difficulty: Easy
category: offensive
summary: "FTP, SMTP, POP3, and IMAP all send credentials in the clear by default. A telnet client is enough to speak every one of them and see exactly what leaks."
---

Most of the protocols that run the internet's plumbing were designed before encryption was standard. FTP, SMTP, POP3, and IMAP all still work the same way they did decades ago, and by default they send everything in plain text, including the username and password. This post is about what that means in practice and how you can read these protocols by hand with a single tool.

## The One Lesson

Every protocol here transmits data in cleartext by default. Anyone who can see the network traffic between the client and the server can read the email content, the files, and the login credentials as they go by. That is the whole security story. If the traffic is not encrypted, capturing it is enough to steal the account.

This is why a secure alternative exists for each one. The plain versions still show up on internal networks, legacy systems, and misconfigured servers, so knowing how they work is not just history.

## Telnet Is a Universal Client

Telnet is almost never used as a remote-access protocol anymore, because it is itself cleartext and SSH replaced it. But the telnet client is still useful for one thing: it opens a raw TCP connection to any port and lets you type commands directly. Because FTP, SMTP, POP3, and IMAP are all text-based, you can speak them by hand.

```
telnet TARGET_IP PORT
```

That single command lets you talk to a web server, a mail server, or a file server and watch the exact request and response. `netcat` (`nc`) does the same job and is often preferred, but the idea is identical. This is the same skill as banner grabbing: connect to a service and read what it says about itself.

## Walking Through Each Protocol

**HTTP on port 80** is a request-and-response text protocol. You connect, send a `GET` line and a `Host` header, leave a blank line to signal the request is finished, and the server sends back headers plus the page. Doing it by hand once makes it clear that a browser is just automating these same lines.

**FTP on port 21** handles file transfer. You log in with a username and password, list files, and download them. The catch worth remembering is that FTP uses a separate data connection for the actual transfer, which is why a listing sometimes hangs until you switch to passive mode. Both the login and the files move in cleartext.

**SMTP on port 25** moves mail between servers. Connect and the server greets you with a banner starting with `220`. SMTP is the protocol behind email spoofing tests and open relay checks, because a misconfigured mail server will happily relay mail for anyone, which spammers abuse.

**POP3 on port 110** downloads mail to a client. You authenticate with `USER` and `PASS`, then `STAT` reports how many messages are waiting and their total size, and `RETR` pulls one down. POP3's default behavior is download and delete: the message lands on your device and is removed from the server.

**IMAP on port 143** also reads mail, but it keeps everything on the server and syncs state across devices. Read and unread flags, folders, and deletions all live server-side. One quirk when you drive it by hand: every IMAP command needs a short tag in front of it, like `c1 LOGIN`, so the client can match replies to commands.

## Why POP3 Versus IMAP Matters to an Attacker

The difference is not just a user convenience. POP3 downloads and deletes, so a stolen POP3 session gets whatever is on the server right now. IMAP leaves the whole mailbox on the server indefinitely, which means stolen IMAP credentials give an attacker persistent access. They can keep reading new mail as it arrives, search years of history, and hunt for password reset emails to pivot into other accounts. Mailbox access is often account access for everything whose reset flow lands in that inbox.

## Ports Worth Memorizing

Each cleartext protocol has an encrypted counterpart, usually on its own port. These come up constantly, so they are worth knowing cold.

| Protocol | Cleartext port | Secure version | Secure port |
|---|---|---|---|
| FTP | 21 | SFTP / FTPS | 22 / 990 |
| HTTP | 80 | HTTPS | 443 |
| SMTP | 25 | SMTPS / submission | 465 / 587 |
| POP3 | 110 | POP3S | 995 |
| IMAP | 143 | IMAPS | 993 |
| Telnet | 23 | SSH | 22 |

## The Blue Team View

The defensive side is straightforward and mostly about not running the plain versions where they can be sniffed. Use the encrypted variant everywhere: HTTPS, SFTP, SMTP with STARTTLS, POP3S, IMAPS, and SSH instead of telnet. Where a legacy system forces cleartext, keep it off any segment an attacker could reach and watch for `USER`/`PASS` or `LOGIN` commands crossing the wire in plain text, because seeing them at all means credentials are exposed. The single highest-value habit is treating any cleartext authentication on the network as already-compromised credentials.

## Key Takeaways

- **Cleartext means the credentials are on the wire.** For FTP, SMTP, POP3, and IMAP, capturing traffic is enough to steal the account.
- **Telnet and netcat speak any text-based protocol.** Connect to the port, type the commands, read the replies. This is the same move as banner grabbing.
- **IMAP access is more dangerous than POP3 access.** POP3 downloads and deletes, IMAP leaves the full mailbox on the server, so stolen IMAP credentials are persistent.
- **Every cleartext protocol has a secure port.** Learn the pairs: 21/22, 80/443, 25/587, 110/995, 143/993, 23/22.
- **Defenders should assume cleartext auth is already leaked.** Move to encrypted variants and isolate anything that cannot be upgraded.
