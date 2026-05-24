# Reproduction — IMO Link Preview IP Leak

No code needed. The bug is triggered by IMO itself auto-fetching the URL.

## Steps

1. Open https://webhook.site in a browser. You will get a unique URL like `https://webhook.site/abc-def-12345`. This URL logs every HTTP request it receives, with full headers and source IP.

2. Create or use a free IMO account on any of the affected platforms:
   - Android (tested: 2026.01.1031)
   - iOS (tested: Build 846, release-2026.1.1)
   - macOS (tested: 2.27.163)

   *Windows 1.5.3.3 is not affected — the link preview feature was never implemented in the .NET build.*

3. Send the webhook.site URL to a target contact via IMO. Any chat message that contains the URL works.

4. The target only needs to **open the chat** — no tap, no click on the link itself.

5. Reload the webhook.site dashboard. You will see:
   - **Source IP** of the target device
   - **User-Agent** with one of these signatures:
     - Android: `Dalvik/2.1.0 ... BIGO-baiguoyuan/...` (contains User-ID + device model)
     - iOS: `facebookexternalhit/1.1 Facebot Twitterbot/1.0`
     - macOS: `CFNetwork/3860.400.51 Darwin/25.3.0 ... facebookexternalhit/1.1 Facebot Twitterbot/1.0`
   - **Accept-Language** (target's locale)
   - **Timestamp** (confirms the target's device is online at this moment)

## Comparison test

Send the same URL via WhatsApp, Telegram, and Signal in the same session:

- **WhatsApp:** webhook.site shows the sender's IP, not the recipient's. Preview is generated sender-side.
- **Telegram:** webhook.site shows a Telegram server IP (e.g. `149.154.161.217`, NL). Preview is generated server-side.
- **Signal:** webhook.site shows no requests at all. Signal does not auto-fetch.

Only IMO exposes the recipient.

## Why webhook.site instead of a custom script

Reproducing this bug does not need a server. Webhook.site is a free public log endpoint that any researcher or vendor can use in seconds. No Python, no Docker, no Cloudflare tunnel.

If your environment blocks webhook.site, equivalent services: requestbin.com, beeceptor.com, pipedream.com.
