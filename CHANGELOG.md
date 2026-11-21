# Changelog

All notable changes to Hellhound Spider are documented here.

## [14.6] — 2026-11-21

### Fixed
- **`--cookie`/`-C` crashed the whole scan with "Illegal key" when given a
  full `Cookie: name=value` header line** (e.g. pasted straight out of
  devtools/Burp) instead of just the `name=value` part — the leading
  `Cookie: ` text got parsed as part of the first cookie's name, and
  aiohttp's cookie jar rejected it outright. `--cookie` now strips a
  leading `Cookie:`/`cookie:` prefix automatically, and any cookie name
  that still isn't valid is skipped with a warning instead of crashing
  the scan.
- **Every crawled endpoint now gets a status code and content length
  recorded**, not just the ones a specialized probe (admin panel check,
  sensitive-file check, ...) happened to touch. A timing gap meant the
  ordinary crawl path tried to record this before the endpoint even
  existed in the store, so it silently went nowhere for most URLs —
  fixed by registering the endpoint first.
- **The final-summary endpoint table didn't show status/content-length
  either** — it only ever printed a status hint for non-200 responses,
  so the far more common 200 case showed nothing at all, and content
  length wasn't shown there regardless of status. The live feed carrying
  this information isn't enough on its own since the summary is what
  people actually read afterward — both now show on every row there too.
- **Status code and content length were both flat gray**, easy to miss
  entirely. Status is now colored by class the way httpx colors `-sc`
  (2xx green, 3xx cyan, 401/403 red, other 4xx yellow, 5xx magenta), in
  both the live feed and the final summary; content length is no longer
  gray either.

### Changed
- **Live crawl feed now shows status code and content length on every
  line** (`[200] [1.2KB] http://...`), the way you'd normally reach for
  httpx's `-sc -cl` for — no need to re-run a separate tool afterward
  just to see what each discovered URL actually returned.
- **Parameter map now groups endpoints that share the exact same
  parameter names** (method + param-name set) instead of printing an
  identical params line once per URL — the common case on any site with
  several login/register/reset-password/2FA-style endpoints that all
  take the same fields.

## [14.5] — 2026-11-09

### Fixed
- **CAPTCHA and JWT findings were visually indistinguishable in the live
  crawl feed** — both used the same generic cyan `[*]` info marker, so the
  two finding types looked identical unless you read the text closely.
  Each now gets its own color (`[CAPTCHA]` yellow, `[JWT]` blue), matching
  `[OAuth]` and `[Login]`, which already had their own.
- **Reflected-parameter hits were silent by default** — the live-feed line
  for a reflected parameter only printed with `--verbose`, unlike every
  other finding type in this release, which always prints. Now always
  shown, tagged `[Reflected]` in magenta.

## [14.4] — 2026-11-08

### Fixed
- **Form-login automation reported a successful login as a failure** whenever
  the target reuses the same session cookie across the anonymous →
  authenticated transition instead of issuing a new one on success — the
  classic PHP/`PHPSESSID` pattern, among others. Success detection
  previously hard-required a *changed* cookie; it now also accepts a
  redirect away from the login page or authenticated-looking page content
  (logout link, "welcome back", dashboard, etc.) as corroborating signals,
  with a broader error-pattern check to keep false positives down.
- **Inline "Continue with SSO" / "Sign in with Google" controls went
  undetected** when they sit next to a classic username/password form on a
  login page that itself returns 200 (no redirect) — a very common layout.
  OAuth/SSO detection and `--login`'s pre-check now also scan the page's
  own HTML for this pattern (link/button text or href), and chase it one
  hop to identify the actual provider, instead of relying solely on an
  immediate redirect from a fixed list of guessed paths.

### Fixed
- **`--no-openapi`'s OpenAPI/Swagger discovery never actually ran.** The
  probe function existed and was fully implemented, but nothing in the
  crawl ever called it — dead code, silently shipping a no-op since it was
  introduced. Now wired into recon alongside the GraphQL probe, and
  documented GET endpoints it finds are enqueued for live verification
  instead of just being recorded.

### Added
- **Reflected-parameter detection** — flags a query-parameter value this
  crawl already sent that comes back verbatim in the response body, a lead
  for reflected XSS/template injection. Passive observation only; no
  payloads are ever injected.
- **Scan resume** (`--checkpoint FILE` / `--resume FILE`) — periodically
  (and on exit, even if interrupted or crashed) snapshots crawl progress
  and every finding to disk; resuming skips already-visited URLs and
  carries all prior findings into the new run's final report, instead of
  starting a long scan over from zero after a crash or a `Ctrl-C`.
- **Subdomain takeover detection** now also runs against every subdomain
  found via crt.sh/certificate-transparency enumeration, not just the root
  domain and `www` — the same CNAME + dangling-service-fingerprint + NXDOMAIN
  check, just pointed at where it has the most value.
- **JWT structural inspection** — decodes any JWT seen in a response body or
  headers (no signature verification, forging, or cracking) and flags
  `alg: none`, HMAC algorithms worth a brute-force/confusion check, missing
  `exp`/`iat`, and claims that look like they carry elevated privilege.
- **API spec diffing** — cross-references OpenAPI/Swagger-documented
  endpoints against what the crawl actually reached, in both directions:
  documented endpoints never reached, and reached endpoints that were never
  documented (shadow API surface).
- New final-summary sections for all of the above, following the existing
  pattern (`REFLECTED PARAMETERS`, `JWT INSPECTION`, `API SPEC DIFF`), so
  these show up where the report is actually read.

## [14.3] — 2026-11-06

### Added
- **Classic HTML form-login automation** (`-l/--login URL -n/--login-user -z/--login-pass`).
  Parses the login form, submits credentials, and carries the resulting session
  cookies through the rest of the scan. Pre-checks the page for an OAuth/SSO
  redirect or a CAPTCHA and aborts with a clear report instead of attempting
  either — both are explicitly out of scope.
- **OAuth/SSO redirect detection and provider fingerprinting**, run automatically
  during unauthenticated recon (independent of `--login`). Probes common
  login-trigger paths (`/login`, `/sso`, `/oauth`, `/account`, etc.) with
  redirects disabled, classifies anything OAuth2/OIDC-authorize-shaped, and
  identifies the specific provider (Okta, Azure AD/Entra ID, Google, Auth0,
  OneLogin, Ping Identity, AWS Cognito, Keycloak, GitHub, Apple, and a generic
  OIDC fallback). Surfaces the authorize request's parameters and flags missing
  PKCE/state as recon output — detection and fingerprinting only, Spider never
  follows or automates the actual OAuth flow.
- **CAPTCHA detection** (reCAPTCHA, hCaptcha, Cloudflare Turnstile, Arkose
  Labs/FunCaptcha, and a generic fallback). Detect-and-report only; Spider
  never attempts to solve or bypass one.
- New final-summary sections — `LOGIN AUTOMATION`, `OAUTH / SSO DETECTION`,
  `CAPTCHA DETECTION` — so these findings show up where hackers actually read
  results, not just in the live feed.

### Fixed
- The aiohttp session now uses an explicit `unsafe=True` cookie jar. Without
  it, aiohttp silently refuses to persist cookies for plain IP-address hosts —
  which broke session-cookie handling (including the new `--login`) against
  bare-IP and internal targets, a routine case in pentesting.

## [14.2] — 2026-11-02

### Added
- Published to PyPI as `hellhound-spider` — `pip install hellhound-spider`
  installs the core crawler directly, no git clone needed.
- `[spa]`, `[dns]`, and `[full]` extras for optional Playwright/Patchright
  SPA support and full DNS intel (dnspython) via pip.

### Changed
- README's Installation section now documents the pip path alongside the
  existing `install.sh` source install, including what pip can't install
  for you (the Playwright browser binary itself, and WhatWeb).

## [14.1] — 2026-11-01

### Fixed
- Live crawl output could leave stray blank rows or wrap unpredictably on
  narrower or font-sensitive terminals. Root causes, now all fixed:
  - Parent-directory backtracking findings could emit very long, untruncated
    URLs into the live feed with no line-length cap — now bounded like the
    rest of the discovery feed.
  - The animator's progress bar used Braille-pattern characters, which some
    terminal/font combinations render wider than a single column, throwing
    off every carriage-return redraw. Replaced with plain ASCII.
  - Row-clearing switched from manually counting characters and overwriting
    with spaces to the terminal's own erase-in-line control sequence, so
    rendering no longer depends on guessing any glyph's on-screen width.
- Bitcoin address detection replaced a weak heuristic with real Base58Check
  checksum validation, eliminating false positives on random minified-JS
  strings that merely looked address-shaped.
- Apache `mod_autoindex` column-sort query strings (`?C=N;O=D`) were being
  picked up as real parameters. Filtered globally, at the same gate every
  discovered URL passes through — not just in the crawler, but for
  backtracking too.

### Changed
- Removed the "JS ORPHAN PARAMS" summary section (unused noise; the
  underlying data is still in the JSON export).

## [14.0] — 2026-10-31

### Added
- **Stdin/stdout piping.** `spider target.com | httpx` just works — no flag
  required. Auto-detected whenever stdout isn't a terminal; `--pipe`/`-Q`
  forces it explicitly. Defaults to plain newline-separated URLs when
  piped (`--format` still available to pick `json`, `jsonl`, `csv`, `burp`,
  or `nuclei`). Multiple targets via stdin, JSON mode, writes one compact
  JSONL line per target.
- **Parent-directory backtracking** (`--no-backtrack`/`-k` to disable, on
  by default). After the forward crawl, walks back up every discovered
  path to probe ancestor directories that were never directly linked to —
  e.g. finding `/js/` open to listing after only ever seeing individual
  `/js/*.min.js` hits. Mirrors what a recursive directory brute-forcer
  would find, without a separate wordlist pass. Findings now appear in
  both the live feed and a dedicated "PARENT DIRECTORY BACKTRACKING"
  section in the final summary.

## [13.21] — 2026-07-04

### Added
- **CTF Mode** (`--ctf`) — auto-enables sensitive file scanning, admin
  panel checks, OpenAPI/GraphQL discovery, CORS audits, flag extraction,
  high default concurrency, and highlights high-value parameters and
  backup files in summaries and reports.
- **Active CORS auditing** — built-in checks for permissive configurations
  reflecting custom origin values.
- **Parameter extraction & validation** — query parameters are now parsed
  from crawled URLs automatically and added to parameter maps.

## [13.19]

### Changed
- Admin-panel, sensitive-file, and Wayback Machine probing are now
  **off by default** (`--probe`, `--admin-probe`, `--sensitive-probe`,
  `--wayback` to opt in), for faster baseline scans.

### Added
- JavaScript endpoint extraction now detects template-literal paths
  containing runtime interpolations (e.g. `/api/user/${id}`) and flags
  them as requiring dynamic values.