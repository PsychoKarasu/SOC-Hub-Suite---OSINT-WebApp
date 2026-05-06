# SOC Hub Suite — Project Context

## Overview

Multi-hub OSINT web platform for SOC analysts. Single-file HTML SPAs deployed on Cloudflare Pages, protected by Cloudflare Access (zero-trust, `@brightstarlottery.com` domain).

**Live URL:** `https://soc-hub-suite.pages.dev`
**CORS Proxy:** `https://intel-hub-proxy.onrender.com` (Render Express, Node 18)
**Repo:** `github.com/PsychoKarasu/SOC-Hub-Suite---OSINT-WebApp`
**Branch:** `claude/identity-intelligence-hub-zfs6I`

---

## Architecture

| Layer | Stack |
|---|---|
| Hosting | Cloudflare Pages |
| Auth | Cloudflare Access (email PIN, `@brightstarlottery.com`) |
| SSO cross-hub | `sessionStorage` key `socHubSession`, TTL 8h |
| Client encryption | PBKDF2 (600k iter) + AES-GCM 256 (Master Key as passphrase) |
| CORS proxy | Render Express (needed for IntelX — CF Worker IPs are blacklisted by IntelX) |
| AI analysis | Anthropic Claude (`claude-haiku-4-5`) |
| i18n | EN/IT toggle in topbar |
| Avatar | "Sakura" SVG anime state machine (idle/searching/success/error/hello/talking/angry) |

---

## Files

| File | Role |
|---|---|
| `index.html` | Portal landing — Master Key SSO login + hub cards |
| `identity-intelligence-hub.html` | Intel-Hub — identity/breach intelligence |
| `phish-hub.html` | Phish-Hub — email/phishing OSINT (crimson/amber palette, red-hair Sakura) |
| `robots.txt` | `Disallow: /` |
| `CLAUDE.local.md` | Local secrets reference — **gitignored, never commit** |

---

## Hubs

### Intel-Hub (`identity-intelligence-hub.html`)

Connectors: IntelX, HIBP, BreachDirectory (RapidAPI), LeakIX, DeHashed

**IntelX quirk:** free tier uses `free.intelx.io`, paid uses `2.intelx.io`. Toggle `intelxTier` in config. All IntelX requests must route through Render proxy (CF Worker IPs are blocked).

**BreachDirectory:** RapidAPI key required. Quota headers: `X-RateLimit-Requests-Limit` / `X-RateLimit-Requests-Remaining`.

**HIBP:** No quota API — tagged as UNLIMITED in UI. Requires HIBP API key.

**DeHashed:** Paid only. Basic auth (email:api_key base64).

**LeakIX:** Free tier with API key.

### Phish-Hub (`phish-hub.html`)

Connectors: Hunter.io, EmailRep, DNS Recon (Google DoH), URLScan.io, VirusTotal

**Hunter.io quota:** `/v2/account` → `data.requests.searches.available + used`

**VirusTotal quota:** `/api/v3/users/{key}/overall_quotas` → `data.attributes.api_requests_daily`

**URLScan quota:** `/user/quotas/` → `response.submissions` (object with `used`/`limit` per window)

**EmailRep:** No auth required for basic lookups, no quota API.

**DNS Recon:** Google DoH (`dns.google/resolve`), free, unlimited — tagged UNLIMITED in UI.

---

## Auth / Security

- Master Key validated via SHA-256 hash hardcoded in each hub
- `sessionStorage` holds `socHubSession` token (JSON: `{key, ts, hash}`) for 8h SSO
- `returnTo` param in portal URL enables redirect after login
- PBKDF2 + AES-GCM used for encrypted `localStorage` config blob (`BUNDLED_CONFIG_BLOB`)
- CSP meta tag with explicit allowlist in each HTML file
- SRI sha384 on CDN scripts: lucide@0.469.0, marked@14.1.3, dompurify@3.2.4
- DOMPurify sanitizes all Claude markdown output before DOM insertion

**See `CLAUDE.local.md` for Master Key hash and API key labels.**

---

## Git Workflow

**Current situation:** MCP tools are scoped to old owner name (`sircap-pene/...`) — GitHub redirects reads, writes may fail. Until resolved, use GitHub web UI for all file edits.

**Standard flow:**
1. Prepare exact diff (file + line range + before/after) here in chat
2. User applies via GitHub web UI → `claude/identity-intelligence-hub-zfs6I` branch
3. CF Pages auto-deploys on push (no build step, static files)

**Never push to main directly.** Feature work on `claude/identity-intelligence-hub-zfs6I`.

**Render proxy repo:** `github.com/PsychoKarasu/soc-hub-proxy` (separate repo, `server.js` Express app)

---

## Known Quirks & Gotchas

- **IntelX + Cloudflare Workers = 403/blocked.** Always proxy via Render.
- **CF Pages `_redirects` file = ERR_TOO_MANY_REDIRECTS.** Keep it deleted; CF Pages handles clean URLs natively.
- **SRI hashes must match pinned CDN versions exactly.** If updating a CDN lib, recompute sha384 and update integrity attribute.
- **`BUNDLED_CONFIG_BLOB` must replace the JS const declaration**, not be pasted inside HTML `<p>` tags.
- **CSS blocks must close all `{}`** — an unclosed rule breaks avatar-rail responsive `display: none` and all subsequent rules.
- **HTML single-file editing via web UI**: always verify the edit didn't truncate closing tags or duplicate elements.
- **Sakura avatar fullscreen bug**: caused by malformed HTML cascading into avatar container styles. Fix: validate `<div>` nesting around avatar-rail section.

---

## Deployment Checklist (after any hub edit)

1. Verify HTML is valid (no unclosed tags, no duplicate elements)
2. Check CSP header still covers any new external domains
3. Smoke test: login portal → hub → search → verify quota tags populate
4. Confirm Sakura avatar is constrained to rail (not fullscreen)
5. Test EN/IT language toggle

---

## Render Proxy (`intel-hub-proxy.onrender.com`)

Express app forwarding requests to IntelX (and generic CORS proxy for other providers if needed).
Free tier Render — cold start ~30s after inactivity. First request after idle may time out; retry.
If proxy URL needs updating: edit `PROXY_BASE` constant in each hub HTML file.
