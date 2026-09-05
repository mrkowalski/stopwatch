# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A full-screen clock + stopwatch page, deployed at https://stopwatch.mkcg.pl. The app is
`public/index.html` — markup, inline `<style>` and inline `<script>` in one ~10KB file — plus two
WOFF2 subsets in `public/fonts/`. No build step, no runtime dependencies, no tests. The only
tooling is wrangler, which publishes `public/` as a Cloudflare Workers static-asset site.

## Working on it

- **Run it:** open `public/index.html` in a browser. Prefer serving it (`npm run dev`, or
  `python3 -m http.server 8000 -d public`) when touching the Wake Lock or Fullscreen paths — the
  Screen Wake Lock API is unavailable over `file://` and the code silently swallows that
  failure, so bugs there are invisible unless served from `localhost`/https.
- **Verify by hand:** start/pause/reset, minute rollover, night mode, fullscreen, the keyboard
  shortcuts, and the 3-second idle fade. There is no automated harness.
- **The fonts live in `public/fonts/`, not in the HTML.** Two `@font-face` rules load
  `jetbrains-mono-400.woff2` and `jetbrains-mono-500.woff2` by relative URL, with matching
  `<link rel="preload">` hints in `<head>` — keep the two in sync or the browser fetches twice.
  They used to be base64 data URLs, which made the file 44KB and unreadable in full; it is ~10KB
  now, so read it normally. The subsets exist to enable the `zero` (slashed-zero) feature;
  changing the font stack means regenerating them, and the relative URLs keep `file://` working.
- **git is the source of truth.** It was not always: the `fetch from hosting` commit pulled
  `index.html` down from the deployed site rather than pushing to it, so anything before
  `wrangler and gh deploy` predates automated deploys. From that commit on, `main` is what
  production serves — see Deploying.

## Deploying

`wrangler.toml` defines a code-less Worker (no `main`) that serves `public/` as static assets on
the custom domain `stopwatch.mkcg.pl`, with `workers_dev = false` so there is no second URL.
Everything in `public/` is published; nothing outside it is.

- **CI does it.** `.github/workflows/deploy.yml` runs `wrangler deploy` on every push to `main`
  that touches a non-`.md` file, and on manual dispatch. It needs two repo secrets:
  `CLOUDFLARE_API_TOKEN` (Workers Scripts:Edit on the account, plus Zone:Read and Workers
  Routes:Edit on `mkcg.pl` for the custom domain) and `CLOUDFLARE_ACCOUNT_ID`.
- **By hand:** `npm run deploy`, after `wrangler login`. Both must run on a real machine — the
  sandboxed container has no Cloudflare credentials and no egress to `api.cloudflare.com`.
- **Version pin appears twice:** `wranglerVersion` in the workflow and the `wrangler`
  devDependency in `package.json`. Bump them together.

## Architecture

**Body classes are the state machine.** `paused`, `night`, and `idle` are toggled on `<body>` and
everything visual keys off them — CSS never reads JS state directly. Colors are five custom
properties on `:root` (`--bg`, `--ink`, `--dim`, `--line`, `--btn`); `body.night` redefines them
rather than restyling elements, so new UI should use the variables and inherit night mode for free.

**Two independent clocks share one layout.** `#clock` is the 24h wall clock (`setInterval`, 1s,
always running, always `--dim`); `#time` is the stopwatch. Both use the `.display` geometry with
`<span class="seg"><span class="num">` around each field and a `.colon` between — the colon pulses
via CSS animation only while `body` lacks `.paused`, and that animation is deliberately exempt from
`prefers-reduced-motion` (it is the running indicator).

**Stopwatch timing is monotonic, display is minute-resolution.** Elapsed time is
`accumulated` (ms banked across previous runs) plus `performance.now() - startTs` while running —
never `Date`, so clock changes and DST don't disturb it. `render()` only shows HH:MM, so a 500ms
`setTimeout` loop is enough to catch rollovers promptly; both `render()` and `updateClock()` guard
DOM writes behind a "did the visible string change" check (`lastMain`, `lastCh`/`lastCm`). Reset
must clear `lastMain` or the guard suppresses the redraw. Nothing is persisted: a reload starts at
zero.

**Wake Lock follows the running state.** Requested on start, released on pause, and re-requested on
`visibilitychange` when the tab returns visible (the browser drops the lock when hidden). All calls
are wrapped in try/catch because the API is absent in some contexts.

**Input.** Buttons plus keyboard: Space toggles, `R` resets, `N` night mode, `F` fullscreen. The
keydown handler bails out for non-Space keys while a `<button>` has focus so the browser's own
activation isn't double-fired. Any of mousemove/mousedown/touchstart/keydown calls `wake()`, which
clears `.idle` and rearms a 3s timer that hides the cursor and fades `#panel`.
