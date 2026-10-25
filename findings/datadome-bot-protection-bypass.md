# DataDome Bot Protection — Automated Challenge Bypass & Protection Profile Analysis

**Program:** [DataDome Bot Bounty — YesWeHack](https://yeswehack.com/programs/datadome-bot-bounty)
**Target:** `bounty-nginx.datashield.co` (nginx module test environment)
**Date:** 2026-10-25 → 2026-10-26
**Status:** Bypass proven · 4,847 verified requests · run stopped at 16% of the 30K target (IP-level volume cap, documented below)

---

## Summary

DataDome's multi-stage JavaScript challenge — the engine behind its bot protection — is solvable by an automated real-browser client with zero human interaction. Using Playwright driving the system Microsoft Edge binary with two minimal stealth patches, the full challenge chain clears in **~5-6 seconds**. Combined with browser-context rotation, this sustains scraping indefinitely at the session level.

The binding constraint turned out to be **IP-level volume capping**, not the bypass itself: DataDome flags the source IP after roughly ~1,000 requests per flag-window, with cooldowns that escalate on repeat offense. This writeup documents both the bypass technique and the measured protection profile.

## The Protection Chain

1. Initial request → **HTTP 403** with an inline challenge page (`var dd={...}`)
2. Challenge page loads `ct.captcha-delivery.com/c.js`, which creates an iframe to `geo.captcha-delivery.com/captcha/`
3. The iframe fingerprints the client (TLS/JA3, browser APIs, automation flags) and sets an intermediate `datadome` cookie
4. On reload, the server issues an **interstitial** (`rt:'i'`) that performs deeper fingerprinting
5. A validated client receives the real content (HTTP 200 + `pagehash_*` flag)

## The Bypass

The entire chain is solvable by automation:

- **Playwright + system Edge** (`channel: 'msedge'`, headful) — a real browser instance satisfies the fingerprint checks by construction
- **Two stealth patches** — `navigator.webdriver` removal and a `window.chrome.runtime` stub
- **Result:** challenge clears in ~5-6s, no CAPTCHA-solving service, no TLS impersonation, no proxy rotation

Key observations during the research:

| Approach | Result |
|---|---|
| curl with browser headers | Hard-blocked at the captcha iframe (TLS fingerprint) |
| Playwright `page.request` / in-page `fetch()` | 403 — request lacks navigation-grade headers |
| Headless Edge | Detected at the interstitial stage |
| **Headful Edge + stealth patches** | **Full bypass, ~5-6s** |
| Cookie reuse from a different client | 403 — cookie is bound to the client fingerprint |

## Sustained Scraping: Session Rotation

Each solved session yields **~90-100 requests** before DataDome re-challenges it. The scraper rotates browser contexts (each obtaining a freshly solved cookie), which sustains throughput without limit at the session level.

Measured rates: **7 req/s** (5 workers) to **40 req/s** (8 workers) — rate is not the constraint.

## Protection Profile (Measured)

| Layer | Limit | Notes |
|---|---|---|
| Session cookie | ~90-100 requests | Rotate context to bypass |
| IP flag-window | ~1,000 requests | Volume-based; independent of request rate |
| IP cooldown | Escalating: 6 min → 15 min → 60 min+ | Repeat offenses extend the ban |
| Per-host variance | nginx viable; nodejs/fastly/cloudfront effectively dead | 2,196 / 71 / 6 / 0 successes across the four bounty hosts |

The IP-level cap is the binding constraint: at ~1,000 requests per window with escalating cooldowns, the 30K target would require ~30 windows over 24+ hours. The run was stopped at 4,847 verified requests (16%).

## Evidence

- **4,847 requests** verified with HTTP 200 + `pagehash_*` flags
- All requests from a **single residential IP** (disclosed in the program report)
- Full runnable scraper (Playwright, adaptive cooldown, session rotation) archived
- 4,847-row evidence CSV (`url,hash,status`) archived

## Honest Assessment

The bypass is real and reproducible — the challenge engine does not distinguish an automated real browser from a human one. What stops volume scraping is the IP-level risk scoring, which caps a single IP at ~1,000 requests per flag-window regardless of pace. Distributing across IPs would defeat the cap but violates the program's single-IP rule.

**Classification:** protection-engine weakness (client-fingerprint reliance) — not a test-environment misconfiguration.

## Files

- `scraper-final.js` — the adaptive scraper (session rotation + cooldown backoff)
- `scrape-results-final.csv` — 4,847 verified rows
- `scrape-summary.json` — machine-readable summary
