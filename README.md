## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    browser-search                       │
│                                                         │
│  ┌──────────────┐                                       │
│  │    Search    │                                       │
│  │              │                                       │
│  │  SearXNG     │   search engines → URLs               │
│  │  (Docker)    │   JSON results, fast                  │
│  │  :8080       │                                       │
│  └──────────────┘                                       │
│         │                                               │
│         │ results ready → to browse                     │
│         ↓                                               │
│  ┌─────────────────────────────────────┐                │
│  │           Browsing                  │                │
│  │                                     │                │
│  │  ┌──────────────┐                   │                │
│  │  │   Camofox    │  browser + REST   │                │
│  │  │  (Docker)    │  JS, click, eval  │                │
│  │  │  :9377       │                   │                │
│  │  └──────┬───────┘                   │                │
│  │         │                           │                │
│  │         │ if blocked                │                │
│  │         ↓                           │                │
│  │  ┌──────────────┐                   │                │
│  │  │ CloakBrowser │  stealth Chromium │                │
│  │  │   (npm)      │  anti-bot, proxy  │                │
│  │  └──────────────┘                   │                │
│  └─────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────┘
```

## How it works

### Phase 1 — Search with SearXNG

Docker container on `localhost:8080`. Metasearch engine that queries
Google, Wikipedia, Bing, DuckDuckGo and many others simultaneously.
JSON output with titles, snippets, and URLs.

**Example:**

```bash
node scripts/searxng/searxng.mjs search "largest llm benchmark 2026"
```

The agent now has a list of URLs to visit and autonomously decides
whether to browse them with Camofox or CloakBrowser based on the site.

### Phase 2 — Browse with Camofox

Docker container on `localhost:9377`. Exposes a full Firefox browser
through a REST API. The agent can create tabs, navigate, click,
scroll, execute arbitrary JavaScript, and structure data.

**Includes:** Mozilla's Readability.js for extracting clean articles,
removing nav, sidebar, and ads (~70% token savings).

**Main commands:**

```bash
# Single-URL extraction (Readability.js, auto-fallback to snapshot)
node scripts/camofox/camofox.mjs readability "https://example.com"

# JavaScript evaluation
node scripts/camofox/camofox.mjs evaluate "https://example.com" "document.title"

# Accessibility snapshot
node scripts/camofox/camofox.mjs snapshot "https://example.com"
```

### Phase 3 — Browse with CloakBrowser (when Camofox isn't enough)

npm package based on Playwright + `cloakbrowser`. Launches a Chromium
browser with advanced fingerprinting to bypass Cloudflare, Akamai,
DataDome and other anti-bot systems. Automatic challenge detection
with wait and retry.

**Available scripts:**

- `cloak-fetch.mjs` — universal fetch with challenge detection
- `cloak-script.mjs` — custom Playwright script execution

**Example:**

```bash
node scripts/cloak/cloak-fetch.mjs "https://protected-site.com"
node scripts/cloak/cloak-fetch.mjs "https://protected-site.com" --proxy socks5://... --geoip

# Markdown output (requires: pip install markitdown)
node scripts/cloak/cloak-fetch.mjs "https://example.com" --format markdown
```

> **Version note:** browser-search pins CloakBrowser `^0.5.5` and Playwright Core `1.62.1`. CloakBrowser 0.5.x adds **Linux arm64** support (Raspberry Pi) and Windows fixes. Known issue: on Windows with a **Pro** license, the browser may exit ~10s after launch ([#479](https://github.com/CloakHQ/cloakbrowser/issues/479)) — workaround: `--license-through-proxy`.
=


**Services overview:**

| Service | How | Reference |
|---|---|---|
| SearXNG | Docker, `:8080` | [docs.searxng.org](https://docs.searxng.org/admin/installation-docker.html) |
| Camofox | Docker, `:9377` | [github.com/jo-inc/camofox-browser](https://github.com/jo-inc/camofox-browser) |
| CloakBrowser | npm (requires `npm install` in skill dir + `ensureBinary`) | `scripts/cloak/cloak-fetch.mjs` + `scripts/setup.sh` / `scripts/check.sh` |


## Environment variables

| Variable             | Required for                      | Default |
|----------------------|-----------------------------------|---------|
| `CAMOFOX_API_KEY`    | evaluate, session, cleanup in Camofox | —   |
| `CAMOFOX_ADMIN_KEY`  | Camofox stop endpoint             | —       |

## What this skill does NOT do

- **Social media.** Instagram, Facebook, TikTok, LinkedIn, and Twitter/X
  require login. `browser-search` does not attempt to browse them.
- **Download files.** It is read-only (except for explicit screenshots).
- **Bypass paywalls.** Does not circumvent payment or login systems.


### Best practices

- **API keys.** Use environment variables (`$CAMOFOX_API_KEY`) or `--env-file` for Docker. Never paste keys on the command line — they appear in `ps aux` and shell history.
- **Docker binding.** Always use `127.0.0.1:` prefix for port mapping (`-p 127.0.0.1:9377:9377`). Never expose to `0.0.0.0`.
- **URL encoding.** Deterministic scripts handle encoding internally. No manual escaping needed.
- **Version pinning.** Dependencies use exact versions (no `^` caret ranges) and `package-lock.json` for reproducible builds.


## License

MIT
