---
name: live-browser-js-harvesting
description: Harvest JavaScript files from a target using the live browser when curl/httpx/katana get blocked (403, WAF, bot protection), then feed the JS to jsmax for secret/endpoint extraction. Use when a target's JS is unreachable by command-line fetchers but loads fine in a real browser.
---

# Live-Browser JS Harvesting

## Problem

`jsmax` needs a list of JS URLs to analyze, and `web2-recon` harvests them
with curl-based tools (katana, httpx, LinkFinder, SecretFinder). Many
high-value targets (exchanges, fintech, large web2) return **403 / WAF blocks
/ bot challenges** to those fetchers while loading fine in a real browser.
When that happens, the JS — and the secrets, endpoints, and API routes inside
it — is invisible to the CLI pipeline.

This skill closes that gap: use the **live browser** (the user's own Chrome
via the extension) to load the page the way a real user
does, extract every JavaScript URL the page loads, download those files, and
hand them to `jsmax`. If the user's browser is unavailable, stop and tell the
user — never fall back to a cloud browser.

## When to use

- `curl`, `httpx`, or `katana` against the target returns 403 / 429 / a bot
  challenge page, but the same URL loads in a browser.
- You have a list of in-scope web hosts and need their JS, but the CLI
  harvesters come back empty or blocked.
- A specific page is suspected to load sensitive JS (auth flows, wallet
  flows, admin panels) that only renders after JavaScript executes.

## When NOT to use

- The CLI fetchers already work — use `web2-recon` + `jsmax` directly.
  The live browser is slower; only reach for it when the CLI path is blocked.
- The target is out of scope. Same authorization rules as every other
  hunting skill: only targets the user owns or that are in scope for an
  authorized bug bounty / pentest / CTF.
- The page requires logging in as the user — that needs the user's account
  and explicit approval first. Do not harvest authenticated JS without it.

## Workflow

### 1. Confirm the block

```bash
curl -s -o /dev/null -w "%{http_code}" https://target.example.com/
httpx -u https://target.example.com -sc
```

If both return 403 / 429 / a challenge page while the site loads in a
browser, the live-browser path is justified. Record the status codes.

### 2. Load the page in the live browser

Open the target URL in the live browser (the user's own Chrome via the extension
— it carries the user's real browser fingerprint and session). If the user's
browser is unavailable, stop and tell the user — never fall back to a cloud
browser. Let the page fully load, including
JavaScript execution. If the page is a single-page app, navigate through the
main flows (landing → login page → app shell) so lazy-loaded chunks are
fetched too.

### 3. Extract every JS URL the page loads

From the loaded page, collect script sources:

- All `<script src="...">` URLs in the page source.
- Dynamically loaded chunks (webpack / Vite / Next.js chunks that appear
  after interaction — check the browser's network activity / page source
  after navigating).
- Source maps (`*.js.map`) when present — they often contain original
  source with comments and variable names.

Resolve relative URLs against the page origin. Deduplicate. Keep only
first-party and CDN-hosted JS that belongs to the target (drop third-party
analytics / ads / tag-manager scripts — they are out of scope and noisy).

### 4. Download the JS files

Fetch each JS URL. Prefer the browser's own download where the CLI is
blocked; where a URL is fetchable by curl once known, use curl (faster).
Save to a working directory, e.g. `~/workspace/<target>/js/`, preserving
the original filename when possible.

```bash
mkdir -p ~/workspace/<target>/js
curl -sL -o ~/workspace/<target>/js/<name>.js "<js-url>"
```

Verify each file is actually JavaScript (not an error page or a challenge):
check the first bytes and the file size. A file that is HTML or under ~1 KB
is probably a block page — discard it and note which URLs failed.

### 5. Run jsmax on the harvested JS

Feed the downloaded files to `jsmax` (see its SKILL.md): extract secrets,
endpoints, JWTs, hashes, and internal API routes. Then triage exactly as
jsmax prescribes — a secret in JS is a lead, not a finding, until you prove
it is live and exploitable.

### 6. Record the harvest

Keep a harvest log next to the JS files (`harvest-log.md`):

- Target host, date, which browser was used.
- Which URLs were blocked for CLI fetchers (with status codes).
- How many JS files harvested, how many failed, and why.
- Which jsmax findings (if any) came out of the harvest.

This makes the next harvest on the same target fast and auditable.

## Safety and scope

- Only harvest JS from in-scope hosts. Do not use the browser to poke
  at out-of-scope domains "while you're there".
- Do not submit forms, change settings, or perform state-changing actions
  in the browser while harvesting — this is a read-only recon step.
- Do not download or execute anything the page offers (installers,
  extensions, binaries). JavaScript source files only.
- If the browser hits a login wall, stop and ask the user — do not log in
  with the user's credentials without explicit approval.

## Pairs with

- `jsmax` — consumes the harvested JS files (secret/endpoint extraction).
- `web2-recon` — the CLI-first recon pipeline; this skill is its fallback
  when the CLI path is blocked.
- `client-reverse` — if a harvested JS file reveals request signing or
  anti-bot tokens, use client-reverse to recover the signer.
