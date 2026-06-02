# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project shape

Single-file vanilla-JS web app: **everything is in `index.html`** (HTML, CSS, and one IIFE script). There is no build step, no package manager, no test runner, no lint config. `satellite.js@5.0.0` is loaded from jsDelivr; no other dependencies.

A real-time satellite/ISS radar that takes the user's geolocation as the observer and plots visible satellites on a polar (azimuth/elevation) canvas, plus a "next passes" prediction list.

## Running and verifying

- **Must be served over HTTP(S).** `file://` breaks both `fetch()` to CelesTrak (CORS) and `navigator.geolocation` on most browsers. Use any static server, e.g. `python -m http.server 8000` or `npx serve .` from the repo root, then open `http://localhost:8000/`.
- Production hosting is GitHub Pages on `kiuza1004/cosmostation` `main` branch (root). To deploy: push to `main`.
- There are no automated tests. Verify changes manually in a browser:
  1. Geolocation prompt appears, allow → `#status` shows "현재 위치 위도/경도".
  2. Footer (`#source`) shows `데이터 출처: CelesTrak · 갱신 …` (not `fallback` / `cache-stale` on a healthy network).
  3. Radar redraws every 1s, `#sat-list` populates within ~10s if any satellite is above horizon, `#pass-list` populates within ~1s after location is acquired.

## Architecture (the parts that span multiple sections)

The IIFE in `index.html` is organized into labeled blocks (`// -------------------- … --------------------`). The non-obvious flow:

### Boot order matters
`boot()` → `loadTLEs()` (network, with 3-tier fallback) → `parseTLEs()` (build `satrec` objects) → `requestLocation()` → on success: `startTracking()` (1s `setInterval`) **and** `schedulePassRefresh()` (async pass prediction + 5min `setInterval`). If location fails, the radar grid still draws but no satellites or passes are computed. The `observer` global being `null` is the gate.

### TLE source pipeline (3 tiers)
`loadTLEs()` in order: (1) parallel `fetch` of CelesTrak `gp.php` endpoints — `CATNR=25544` for ISS + `GROUP=starlink` capped at `STARLINK_LIMIT` (8), (2) `localStorage` cache under `cosmostation_tle_v1` (6h TTL, but a stale cache up to 24h old is still used as last-resort over fallback), (3) hardcoded `FALLBACK_TLES`. The chosen tier is surfaced in the footer via `describeSource()` — when debugging TLE issues, check that string first.

### Coordinate math (where `NaN` bugs live)
`computePositions()`: `satellite.propagate(satrec, Date)` → ECI → `eciToEcf(eci, gmst)` → `ecfToLookAngles(observer, ecf)` → `{azimuth, elevation, rangeSat}` in **radians/km**. Every stage has a guard (`pv.position` falsy, non-finite components, try/catch around `ecfToLookAngles`). The `observer` object is in **radians** (lat/lon) and **km** (height); height defaults to 0.05 km when GPS altitude is unavailable. Azimuth is normalized via `((az * DEG) % 360 + 360) % 360` because raw output can be negative.

### Radar polar projection
Center = zenith (90° elevation), edge = horizon (0°). Mapping: `r = R * (1 - elDeg/90)`, `x = cx + r·sin(azRad)`, `y = cy − r·cos(azRad)` (N up, clockwise). Only `elevationDeg ≥ 0` satellites are drawn.

### Canvas DPR handling
`resizeCanvas()` sets `canvas.width/height = cssSize * devicePixelRatio` and `ctx.setTransform(dpr, 0, 0, dpr, 0, 0)`. `drawRadar()` then works in CSS pixels via `canvas.width / dprCache`. Don't write to `canvas.width/height` elsewhere — it resets the transform.

### Pass prediction (`findNextPass`)
Forward time-step search (`PREDICT_STEP_SEC = 30`, `PREDICT_WINDOW_SEC = 4h`). State machine: if a satellite starts above horizon, `waitingForSet` skips until it sets, then looks for the next rise. Each satellite `await`s a `setTimeout(0)` to yield to the UI — keep this if adding more sats, or the main thread will jank on mobile.

## Conventions

- Comments and user-facing strings are in Korean. Match the existing tone (terse, technical).
- Satellite "type" is `'iss' | 'starlink'`. Color/label branching keys off this string in `drawRadar()` and `renderList()`/`renderPassList()`. New types need updates in all three render sites + the legend in the HTML + the CSS dot classes.
- `setStatus(html, isError)` is the single channel for the top status box; pass `true` for the error style.
- Numeric guards use `safeNum()` / `isFinite()` — keep them; `satellite.js` can emit `NaN` or `false` positions for decayed/invalid TLEs.

## Repository notes

- GitHub remote: `https://github.com/kiuza1004/cosmostation` (main branch is the deployed branch).
- The sibling folder `현재 내 위치에서 볼 수 있는 우주정거간이나 인공위성 검색/` is an empty Korean-named scratch directory — leave it alone unless the user asks.
- No `README.md`, no `.cursor/rules/`, no `.github/copilot-instructions.md` at present.
