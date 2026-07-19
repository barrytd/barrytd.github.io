---
title: "Walking An Application - Manual Web Review with Browser Dev Tools"
date: 2026-07-11
platform: TryHackMe
difficulty: Easy
category: offensive
summary: "A web app review using nothing but the browser. Page source, Inspector, Debugger, Network, and Storage, and why client-side controls are not security."
---

Before you reach for a scanner or a proxy, you can learn a lot about a web app with just the browser. This room walks through the page source and the four dev tools panels that matter for recon: Inspector, Debugger, Network, and Storage. The theme running through all of it is that anything the server sends to your browser is yours to read and change.

## Page source

Right-click and View Page Source, or put view-source: in front of the URL. What to look for:

- **HTML comments** (&lt;!-- --&gt;). Developers leave notes here that never show up on the page, sometimes pointing at pages still being built.
- **Hidden links.** Anchor tags (&lt;a href=...&gt;) can point at pages that are not in the menu, like a staff-only area.
- **Directory listing.** If images or scripts load from a folder like /assets/, browse to that folder directly. If listing is turned on, you see every file in it, including ones that were never meant to be public.
- **Framework and version.** A comment or an asset path often names the framework and version. Compare it to the framework's current version. If the site is behind, the changelog for the newer version tells you what was fixed, which is a map of what is still broken on the old one.

That last point is the useful chain: an out-of-date framework plus its public changelog tells you exactly what weakness to go looking for.

## Inspector

The Inspector shows the live page (the DOM), not the original source, so it reflects changes made by CSS and JavaScript. You can edit any element in place. The classic demo is a paywall: content covered by a floating box that is only hidden with CSS. Flip its display: block to display: none, or just delete the box, and the content behind it is right there. The lesson is that a client-side paywall hides content, it does not protect it.

## Debugger

The Debugger (called Sources in Chrome) is for reading and pausing JavaScript. Minified or scrambled files can be reformatted with Pretty Print. The powerful part is breakpoints: click a line number and the browser pauses on that line the next time it runs. If a script removes something from the page, a breakpoint on that line stops it from running, so whatever was about to be hidden stays visible.

## Network

The Network tab logs every request the page makes. Submit a form with it open and you see the background request go out. Click the request to read its headers, cookies, and the full server response. Requests you never see in the page are fully visible here, which is how you learn the way an app talks to its backend.

## Storage

The Storage tab shows what the site keeps in your browser: local storage, session storage, and cookies. Cookies are the interesting ones for a pentester, because they carry session tokens. Check their security flags:

- **HttpOnly** stops JavaScript from reading the cookie, which protects the session if the site has an XSS bug. If this is false, the token is exposed to any script.
- **Secure** sends the cookie only over HTTPS.
- **SameSite** helps limit CSRF.

A session cookie without HttpOnly set is a finding worth reporting.

## Key Takeaways

Everything the server sends to the browser can be read and edited on your side, so none of it is a real security control. Paywalls, hidden elements, and JavaScript that removes content all fall to a few clicks in dev tools. The controls that matter have to live on the server. Learning to read the source and the four dev tools panels is the cheapest recon there is, and it comes before any heavier tooling.
