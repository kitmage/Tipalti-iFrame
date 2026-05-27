# Set up payee onboarding via iFrame on an aggressively cached WordPress page (anonymous users)

This guide explains how to embed Tipalti payee onboarding in a WordPress page that is heavily cached and visited by users who are **not logged in**.

It is written for production setups where a CDN or reverse proxy serves cached HTML to anonymous traffic.

## Why this scenario is tricky

When a WordPress page is cached at the edge/CDN level, full-page HTML can be served identically to all anonymous users. That breaks any integration that needs **user-specific values** (for example, an onboarding token, session key, or payee identity) rendered directly into cached HTML.

**Rule of thumb:** keep cached HTML generic, and fetch per-user onboarding data at runtime.

---

## Architecture you should use

1. **Serve a cacheable WordPress page shell** with a placeholder container for the iFrame.
2. **Load a small JavaScript bootstrap** on page view.
3. Bootstrap calls a **non-cached backend endpoint** to get short-lived, user-specific onboarding parameters.
4. JavaScript injects/initializes the Tipalti iFrame using the runtime response.

This pattern lets the page remain aggressively cached while still onboarding anonymous users safely.

---

## Step 1: Build a cache-safe page shell in WordPress

Create a page (or template) with:

- A static mount point (for example, `<div id="tipalti-onboarding"></div>`)
- A loading state placeholder
- No embedded user-specific onboarding values in HTML

Example structure:

```html
<section>
  <h1>Complete your payout setup</h1>
  <div id="tipalti-onboarding">Loading secure onboarding…</div>
</section>
```

---

## Step 2: Add a frontend bootstrap script

Your bootstrap script should:

- Run on DOM ready
- Request onboarding parameters from your backend endpoint (not from WordPress cached HTML)
- Handle API failures gracefully
- Initialize and render the Tipalti iFrame into your mount point

Recommended behavior:

- Show retry UI on failure
- Log correlation/request IDs for support
- Avoid storing sensitive onboarding data in localStorage/sessionStorage unless absolutely required

---


### Minimal bootstrap example

```html
<script>
  document.addEventListener("DOMContentLoaded", async () => {
    const mount = document.getElementById("tipalti-onboarding");
    if (!mount) return;

    try {
      const res = await fetch("/wp-json/your-namespace/v1/onboarding-context", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        credentials: "include"
      });

      if (!res.ok) throw new Error(`Context fetch failed: ${res.status}`);
      const payload = await res.json();

      // Call your Tipalti iFrame initializer here using payload fields
      mount.textContent = "Onboarding loaded";
    } catch (err) {
      mount.innerHTML = "Unable to load onboarding right now. Please retry.";
      console.error(err);
    }
  });
</script>
```

## Step 3: Create a non-cached endpoint for anonymous users

Implement an endpoint (in WordPress, a companion service, or your app backend) that:

1. Identifies the visitor using your own onboarding flow state (for example, signed invite link, one-time code, or opaque reference).
2. Validates that state server-side.
3. Creates/retrieves the Tipalti onboarding context server-side.
4. Returns only what the frontend needs to load the iFrame.


### WordPress-specific notes

- If you expose this via the WP REST API, explicitly send `Cache-Control: no-store` from the callback.
- Also send `Vary` headers when behavior depends on request headers/cookies.
- Treat `DONOTCACHEPAGE` as plugin-local guidance only; enforce bypass at proxy/CDN too.

### Endpoint hardening checklist

- Set `Cache-Control: no-store, no-cache, must-revalidate`
- Include anti-abuse controls (rate limit, bot mitigation, replay protections)
- Keep onboarding tokens short-lived
- Never trust raw user identifiers from query strings without verification
- Return generic errors to clients; keep details in server logs

---

## Step 4: Configure WordPress and cache layers correctly

Even though the page is cached, the dynamic endpoint must not be cached.

Apply exclusions in:

- WordPress cache plugin/page cache
- Reverse proxy (Nginx/Varnish)
- CDN/edge cache (Cloudflare/Fastly/etc.)

Typical rules:

- **Do cache**: page HTML, static JS/CSS assets
- **Do not cache**: onboarding/session endpoint responses

If your CDN supports cache keys, ensure query params used for one-time onboarding links are not accidentally normalized into shared cached responses.

---

## Step 5: Handle anonymous identity safely

Because users are not logged in, use a secure onboarding handoff model:

- Send users to onboarding page with a **signed, expiring link**
- Validate signature + expiration server-side
- Bind the request to intended payee/reference
- Optionally one-time consume the link to prevent reuse

Avoid using predictable identifiers (like incremental IDs) in public URLs.

---

## Step 6: Add observability and support tooling

Track:

- Page load to iFrame-render latency
- Endpoint success/failure rates
- Expired/invalid link counts
- iFrame initialization failures by browser/device

Support team should be able to trace a failed onboarding attempt using:

- Correlation ID
- Timestamp (UTC)
- Anonymous invite/reference ID

---

## Common pitfalls

- Rendering onboarding tokens directly into cached HTML
- Letting CDN cache JSON endpoint responses
- Using long-lived onboarding links
- Missing replay protection on anonymous invite URLs
- Assuming WordPress `DONOTCACHEPAGE` affects CDN behavior (it usually does not by itself)

---

## Validation checklist before go-live

- [ ] Anonymous user opens cached page and receives fresh onboarding context
- [ ] CDN response headers confirm endpoint is never cached
- [ ] Expired link path behaves correctly and safely
- [ ] Reused link is blocked (if one-time links are required)
- [ ] iFrame renders across target browsers/devices
- [ ] Logs include correlation IDs for debugging

---

## Summary

To support Tipalti onboarding on an aggressively cached, public WordPress page, separate your implementation into:

- **Cached shell** (WordPress HTML)
- **Runtime secure data fetch** (non-cached backend endpoint)
- **Client-side iFrame initialization**

This preserves performance from caching while keeping user-specific onboarding secure and reliable for anonymous traffic.
