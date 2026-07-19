---
title: "BOTSv3 in Splunk - Investigating AWS Breaches, Cryptomining, and an APT"
date: 2026-05-21
platform: TryHackMe
difficulty: Medium
category: blue-team
featured: true
featured_order: 2
summary: "The follow-up to BOTSv2. This dataset adds a big AWS component, so the investigation moves between cloud logs and endpoint logs. Notes on what the cloud side taught me."
---

## What Is Different About BOTSv3

I did BOTSv2 a week earlier. BOTSv3 is the same idea, the same fictional brewery (Frothly), the same SOC analyst role. The difference is **cloud**.

BOTSv2 was mostly on-prem: Windows logs, network streams, endpoint data. BOTSv3 adds a large **AWS** component. Now the investigation moves between two worlds. One query looks at a Windows event log. The next looks at an AWS API log. Learning to pivot between cloud and endpoint telemetry is the point of this room.

The investigation covered three themes:

- **AWS security.** IAM users, a misconfigured S3 bucket briefly made public, and a leaked access key that ended up in a public GitHub repo.
- **Cryptomining.** A browser process pinning a workstation CPU to mine Monero.
- **APT endpoint activity.** The Taedonggang group again: a macro-enabled spreadsheet, a fake service account, a network scanner, and a PowerShell C2 channel.

## CloudTrail Is The AWS Security Log

The first thing to understand for the cloud half of this room: **AWS CloudTrail** logs every API call made in an AWS account. Every time someone creates a bucket, changes a permission, or launches a server, CloudTrail records it. It is the cloud version of a Windows security event log.

The fields that matter:

- userIdentity.userName is who made the call.
- eventName is what they did (CreateBucket, PutBucketAcl, CreateAccessKey).
- requestParameters is the detail of the call.
- userAgent is the application they used.
- userIdentity.sessionContext.attributes.mfaAuthenticated says whether the session used multi-factor authentication.

Once I knew those five fields, the AWS questions felt like the same pivoting rhythm as the Windows ones. Find a value, pivot to the next query with it.

## The S3 Bucket That Went Public

The clearest lesson in the room. **S3** is AWS file storage. Each bucket has an **ACL** (access control list) that decides who can read and write it. The CloudTrail event PutBucketAcl records every change to that ACL.

There were two PutBucketAcl events on the same bucket. At the event-name level they looked identical. The difference was in the ACL body:

- One event granted the **AllUsers** group **WRITE** permission. AllUsers means *anyone on the internet*. That is what makes a bucket public.
- The other event had an empty ACL. That was someone revoking the public access.

If I had only looked at the event name, both events would have read the same. I had to open the ACL body to see which one was the breach and which one was the fix. **Read the detail, not just the event name.**

While the bucket was public, an outsider uploaded a text file to it. The filename was a blunt message telling Frothly to fix their bucket. Real attackers are not always subtle.

## Follow The URL

The single best habit this room reinforced.

One IAM access key was generating a lot of distinct AccessDenied errors. That pattern, one key failing many different API calls, is what a stolen key looks like when an attacker is probing what it can do.

AWS noticed it too. AWS proactively emails account owners when it detects a leaked credential. That email was in the dataset, and it contained a **GitHub URL**.

Following the URL led to a public Frothly repository with a file named aws_credentials.bak. The plaintext secret key was sitting in it. Someone had committed credentials to a public repo.

The decisive evidence was not in Splunk. It was in a GitHub repo that an email pointed to. If I had not clicked the link, I would have missed it. Click every URL in every email and log.

## Over-Filtering Hides Evidence

A mistake I made and want to remember.

I was trying to find the application the attacker used against AWS. I filtered the logs by the compromised access key. The query returned results, but not the attacker's tool.

The fix was to filter *less*. I broadened the search from the access key to the username. That exposed every user-agent tied to the account, including the attacker's AWS management GUI tool that the tighter filter had been hiding.

Filtering is subtraction. Every filter you add removes events. If you filter on the wrong specific value, you remove the evidence along with the noise. Start broad, narrow gradually, and if a search comes back empty or incomplete, loosen a filter before assuming the data is not there.

## Cryptomining Shows Up In Performance Data

The cryptomining thread was a different kind of pivot. Mining is invisible in most logs because it does not touch the network in an obvious way or create files. What it does is **burn CPU**.

The perfmonmk:process sourcetype records per-process CPU usage. Filtering for processes sitting at 100 percent CPU (after excluding the _Total and Idle pseudo-processes) surfaced the miner. It was a browser process, which means browser-based mining: a malicious site or extension running a miner in JavaScript.

The lesson: when an attack does not show up in the logs you expect, think about what resource it *must* consume. Mining must use CPU. That makes performance data the right place to look.

## Attribution From A User-Agent String

The APT half of the room reused the Taedonggang group from BOTSv2. One artifact was a malicious file uploaded to OneDrive. The Office 365 audit log recorded the upload, including the UserAgent of whoever did it.

The user-agent string contained ko-KP and NaenaraBrowser. ko-KP is the locale code for Korean as used in North Korea. NaenaraBrowser is the official state-developed web browser of North Korea. A web browser almost nobody outside that country runs.

That one string is strong attribution evidence. A field most people scroll past tied the activity to a specific nation-state group.

## General Lessons

- CloudTrail is the AWS security log. Learn userIdentity, eventName, requestParameters, userAgent, and the MFA field and the cloud side stops being intimidating.
- Read the detail of an event, not just its name. Two events with the same name did opposite things.
- Follow every URL in every email and log. The key evidence was in a GitHub repo, not in Splunk.
- Over-filtering hides evidence. Start broad, narrow gradually.
- When an attack is invisible in the obvious logs, think about what resource it has to consume. Mining burns CPU, so look at performance data.
- A user-agent string can carry attribution. ko-KP and NaenaraBrowser pointed straight at a North Korean group.

## What I'd Tell A Beginner

1. Do BOTSv2 before BOTSv3. The endpoint skills carry over and the cloud material is easier when the rest is familiar.
2. Learn the five core CloudTrail fields before touching the AWS questions.
3. Keep a scratch file of every pivot value, the same as any BOTS room.
4. Keep CyberChef open for the Base64 PowerShell decoding.
5. When a search returns nothing useful, loosen a filter before assuming the data is missing.

The technical version with the specific queries and screenshots is in the [companion repo on GitHub](https://github.com/barrytd/security-lab-portfolio/tree/main/labs/2026-05-21-tryhackme-splunk-3).
