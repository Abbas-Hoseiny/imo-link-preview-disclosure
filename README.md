# Zero-Click IP Leak in IMO Messenger via Client-Side Link Preview

> **Full illustrated write-up:** https://hoseiny.de/en/cvd/imo/

**Severity:** Medium — CVSS 3.1 `AV:N/AC:L/PR:N/UI:R/S:U/C:L/I:N/A:N` (Base 4.3). The impact is limited to disclosure of an IP address plus device metadata (`C:Low`), which caps the score at Medium.
**CVE:** requested — MITRE #1993721 (pending)
**Vendor:** PageBites, Inc. (IMO Messenger — Android / iOS / macOS / Windows)
**Reporter:** Abbas Hoseiny — independent · abbas@hoseiny.de
**Status:** Public. CERT/CC (case VU#138533) deprioritised the case on 2026-08-24 and advised public disclosure after roughly a year of vendor silence. All findings remain unpatched.

## TL;DR

IMO Messenger auto-fetches URLs from the recipient device when a message contains a link. The fetch leaks the recipient's real IP, device model, and online status to whoever owns the URL. No user interaction required. Affected: Android, iOS, macOS. Windows is not affected (the feature was never implemented there). 500M+ users worldwide.

## What the bug is

IMO generates link previews client-side. When a message arrives that contains a URL, The recipient's device performs an HTTP GET when the message arrives in the chat — opening the chat is enough, and on iOS it was observed firing before any UI render (confirmed by packet capture). No tap on the link is ever required. The request includes:

- Real IP (geolocation to city level)
- Device User-Agent (model, OS version)
- Accept-Language (locale)
- Online status (fetch timestamp confirms the user is reachable)

The fetch impersonates `facebookexternalhit/1.1 Facebot Twitterbot/1.0` — three social-media crawler signatures combined. On Android there is an additional `BIGO-baiguoyuan` User-Agent that contains a persistent User-ID. WAFs and analytics tools whitelist these bot signatures, which makes the leak harder to spot in server logs.

Comparable apps tested in the same session:

- WhatsApp: preview generated on the sender's device. Recipient IP never exposed.
- Telegram: preview generated on Telegram servers (149.154.161.217, NL). Recipient IP not exposed.
- Signal: no auto preview at all.

Only IMO exposes the recipient.

## How an attacker exploits it

1. Create a free IMO account.
2. Go to webhook.site, copy your unique listener URL.
3. Send the URL via IMO to the target.
4. Don't tap anything — opening the chat is enough.
5. The webhook.site dashboard shows the target's real IP, User-Agent, Accept-Language, and timestamp.

Scales to many targets simultaneously: send the URL to a group or a contact list, harvest all IPs.

PoC and detailed instructions in [poc/](poc/). No code required — webhook.site does the heavy lifting.

## What an attacker gains

- Real IP address (geolocation, ISP)
- Device model and OS version
- Persistent User-ID via BIGO-baiguoyuan supercookie (Android only)
- Language and region
- Online status confirmation
- All of the above without any victim interaction
- Mass-collection by sending to many users at once

## Affected versions

| Platform | Version | Affected | Notes |
|----------|---------|----------|-------|
| Android | 2026.01.1031 | Yes | Google Play, also leaks BIGO supercookie |
| iOS | 2026.1.3 | Yes | App Store (CVE-filed version) |
| macOS | 2.27.163 | Yes | Mac App Store |
| Windows | 1.5.3.3 | No | Feature was never implemented in the .NET build |

Tested 2026-03-01 on Android 16, iOS 18.7, macOS 15.x, and Windows 11 (build 26200). iOS leak independently confirmed via rvictl + pcap (23,927 packets, 3 simultaneous DNS queries before any UI render).

## Mitigation

**For users:** There is no setting in IMO to disable the auto-fetch. Workaround: stop using IMO for any contact you do not want to expose your IP to, or block IMO traffic at the network layer (DNS sinkhole of `imostatic.com`, `imoim.net`). If you need a similar app, Signal does not auto-fetch and WhatsApp/Telegram generate previews server-side.

**For the vendor:** Move preview generation off the recipient device. Two patterns work:

1. Server-side fetch (Telegram model) — IMO infrastructure does the GET, recipient sees only the rendered preview.
2. Sender-side fetch (WhatsApp model) — sender's device fetches, attaches preview to the outgoing message.

Either pattern eliminates the leak. The Windows build already operates this way (no fetch at all), so an implementation precedent exists inside the codebase.

## Disclosure timeline

- 2025-08-25 — First contact to PageBites via privacy@imo.im (only address available, no security@, no security.txt). No response.
- 2025-10-09 — MITRE CVE request #1914267 submitted. Rejected for insufficient version detail.
- 2026-02-16 — MITRE CVE resubmission #1993721 with full version and evidence. Pending.
- 2026-02-16 — Apple Product Security notified (iOS + macOS Store policy violations).
- 2026-03-01 — Live testing completed across all four platforms. Findings submitted to CERT/CC, case VU#138533 opened.
- 2026-03-01 — BSI CERT-Bund notified.
- 2026-04-21 — CERT/CC asked about vendor status. None.
- 2026-05-06 — CERT/CC outreach attempt to feedback@imo.im. No response.
- 2026-05-19 — CERT/CC second outreach attempt. No response.
- 2026-08-24 — CERT/CC deprioritised the case and advised public disclosure; vendor still unresponsive. All findings remain unpatched.
- 2026-10 — Public disclosure. Full illustrated write-up: https://hoseiny.de/en/cvd/imo/

Vendor remained unresponsive throughout — over a year since the first contact on 2025-08-25. All findings unpatched at publication.

**Credits:** Reported by Abbas Hoseiny (independent). Coordinated through CERT/CC (CISA Vulnerability Response and Coordination, case VU#138533). MITRE CVE assignment pending (#1993721). Parallel notifications to BSI CERT-Bund and Apple Product Security.
