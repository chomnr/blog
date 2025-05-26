+++
title = "how to block certain tlds in cloudflare"
date = 2025-05-25
+++
telemetry.zeljko.me/worker/posthog/pageview
This guide shows you how to create a Cloudflare security rule that blocks traffic from commonly abused domain extensions.

**What you'll need:**
- Cloudflare account with your domain configured
- Access to your Cloudflare dashboard

**Setting up the security rule:**

Navigate to your domain in the Cloudflare dashboard and go to **Security → Security rules**, then click **Create custom rule**.

In the rule builder, click **Edit expression** to switch to the advanced editor.

**Configuring the rule:**

Paste this expression into the editor:

```
(http.request.headers["origin"][0] contains ".xyz")
 or (http.request.headers["origin"][0] contains ".top")
 or (http.request.headers["origin"][0] contains ".club")
 or (http.request.headers["origin"][0] contains ".click")
 or (http.request.headers["origin"][0] contains ".online")
 or (http.request.headers["origin"][0] contains ".store")
 or (http.request.headers["origin"][0] contains ".info")
 or (http.request.headers["origin"][0] contains ".buzz")
 or (http.request.headers["origin"][0] contains ".work")
 or (http.request.headers["origin"][0] contains ".win")
 or (http.request.headers["origin"][0] contains ".loan")
 or (http.request.headers["origin"][0] contains ".gq")
 or (http.request.headers["origin"][0] contains ".cf")
 or (http.request.headers["origin"][0] contains ".ml")
 or (http.request.headers["origin"][0] contains ".tk")
 or (http.request.headers["referer"][0] contains ".xyz")
 or (http.request.headers["referer"][0] contains ".top")
 or (http.request.headers["referer"][0] contains ".club")
 or (http.request.headers["referer"][0] contains ".click")
 or (http.request.headers["referer"][0] contains ".online")
 or (http.request.headers["referer"][0] contains ".store")
 or (http.request.headers["referer"][0] contains ".info")
 or (http.request.headers["referer"][0] contains ".buzz")
 or (http.request.headers["referer"][0] contains ".work")
 or (http.request.headers["referer"][0] contains ".win")
 or (http.request.headers["referer"][0] contains ".loan")
 or (http.request.headers["referer"][0] contains ".gq")
 or (http.request.headers["referer"][0] contains ".cf")
 or (http.request.headers["referer"][0] contains ".ml")
 or (http.request.headers["referer"][0] contains ".tk")
```

This rule checks both the `origin` and `referer` headers for domains using these extensions, which are often associated with spam and malicious activity.

**Finalizing the rule:**

Under **Choose action**, select **Block**.

Under **Select order**, select **First**. So it runs before other rules that might allow the traffic through.

Click **Deploy** to activate the rule.

**What this blocks:**

The rule targets traffic from domains using extensions like `.xyz`, `.top`, `.tk`, and others that are frequently used for spam, phishing, and bot traffic. It checks both the origin header (where the request claims to come from) and the referer header (the page that linked to your site).