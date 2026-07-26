# Track A — Live Trucks Plan

## Top-Level Overview

Replace the static `winn-dixie-data.js` local file with a live data feed hosted on **GitHub Pages**
(`https://vladstol223.github.io/wd.bob/`). The base warehouse, store, and delivery data becomes
a static JSON file served from that URL. The animated trucks are driven by a **deterministic,
wall-clock-synchronized simulation** that runs entirely in the browser — no server needed, no
WebSockets, no SSE. Every attendee laptop shows the same truck positions at the same real-world
moment, because all positions are computed from `Date.now()` using a fixed algorithm.

The Track A prompts in `index.html` are updated to match: Step 1 becomes "connect to the live
data URL" instead of "download a local file", and Steps 2–4 get updated prompts that teach Bob
to fetch from a URL, render animated trucks on the SVG map, and correctly orient each truck
icon toward its destination.

---

## Architecture Overview

```
GitHub Pages (static hosting)
  └── winn-dixie-data.json        ← base data: warehouses, stores, deliveries, map bg
  └── index.html                  ← Bobathon landing page (this file)
  └── winn-dixie-data.js          ← kept for legacy / local fallback

Attendee's browser
  └── dashboard.html              ← built by Bob during Track A
        ├── fetch(GitHub Pages JSON URL)   ← loads base data once on page load
        ├── Truck simulation loop          ← setInterval, deterministic by Date.now()
        └── SVG map + animated trucks      ← pure HTML/SVG/JS, no libraries
```

**Why deterministic wall-clock sync works:**
Each truck route has a fixed cycle duration (e.g. 20 minutes of sim time per one-way leg).
Position at any moment = `(Date.now() / cycleDuration) % 1`, linearly interpolated along the
route. All laptops run the same formula against the same wall clock → identical positions.

---

## Sub-Tasks

---

### Sub-Task 1 — Publish `winn-dixie-data.json` to GitHub Pages

**Intent**
Convert the existing JS variable file into a plain JSON file and set up GitHub Pages so it is
accessible at a stable public URL. This is the data source that Bob's `dashboard.html` will
`fetch()` at runtime.

**Expected Outcomes**
- A file `winn-dixie-data.json` exists in the repo root, containing the same data as
  `winn-dixie-data.js` but as valid JSON (no `const WINN_DIXIE_DATA = ...` wrapper, no trailing
  semicolon).
- GitHub Pages is enabled on the `master` branch.
- The URL `https://vladstol223.github.io/wd.bob/winn-dixie-data.json` returns the JSON with
  CORS headers (GitHub Pages sets `Access-Control-Allow-Origin: *` by default).
- The map background image is included — the `base64` field in `map_background` is kept as-is.

**Todo List**
1. Extract the value of `WINN_DIXIE_DATA` from `winn-dixie-data.js` and write it as
   `winn-dixie-data.json` (valid JSON — all keys quoted, no trailing commas).
2. Commit and push `winn-dixie-data.json` to `origin/master`.
3. In the GitHub repo settings (`https://github.com/VladStol223/wd.bob` → Settings → Pages),
   enable GitHub Pages on the `master` branch, root folder.
4. Verify the URL responds: `https://vladstol223.github.io/wd.bob/winn-dixie-data.json`

**Relevant Context**
- Source data: [`winn-dixie-data.js`](winn-dixie-data.js) — the JS file to convert.
- Repo remote: `https://github.com/VladStol223/wd.bob.git`
- GitHub Pages CORS: all GitHub Pages responses include `Access-Control-Allow-Origin: *` — no
  proxy or workaround needed.

**Status:** `[ ] pending` ← GitHub Pages hosting is already live; only the JSON file creation + push remains

---

### Sub-Task 2 — Design the truck simulation algorithm

**Intent**
Define the exact math and data contract for how truck positions are computed. This is the core
"engine" of the live animation — it must be deterministic (same result on every laptop at the
same wall-clock time), lightweight (runs in a `setInterval` at ~30fps), and encodable in a Bob
prompt without ambiguity.

**Expected Outcomes**
- A clear specification, written into this plan, of:
  - How each truck's cycle time is calculated.
  - How position along a route is computed from `Date.now()`.
  - How the truck's rotation angle is derived from the direction of travel.
  - What the truck SVG looks like (top-down, color-coded, rotates to face direction of travel).
- This spec is precise enough that a Bob prompt can implement it correctly in one shot.

**Simulation Spec**

Each delivery in the data becomes a **perpetually looping truck route**:

```
route: warehouse (x0, y0)  ←→  store (xN, yN)

cycle_ms   = 120_000          // one full round-trip = 120 real seconds
leg_ms     = 60_000           // one-way leg = 60 real seconds
phase      = (Date.now() % cycle_ms) / cycle_ms   // 0..1, same on all laptops

if phase < 0.5:               // outbound: warehouse → store
  t = phase / 0.5             // 0..1 along this leg
  x = lerp(wx, sx, t)
  y = lerp(wy, sy, t)
  angle = atan2(sy - wy, sx - wx)   // faces store
  direction = "outbound"

else:                         // return: store → warehouse
  t = (phase - 0.5) / 0.5    // 0..1 along this leg
  x = lerp(sx, wx, t)
  y = lerp(wy, sy, t)         // NOTE: lerp(sy→wy) not lerp(wy→sy)
  angle = atan2(wy - sy, wx - sx)   // faces warehouse
  direction = "return"
```

Each truck gets a **per-delivery offset** so they are not all synchronized at the same point
of their cycle:

```
offset_ms = (delivery_index / total_deliveries) * cycle_ms
phase = ((Date.now() + offset_ms) % cycle_ms) / cycle_ms
```

**Truck SVG (top-down view)**
- Shape: a rounded rectangle (cab) with a slightly narrower rectangle behind it (trailer),
  both inline in SVG, total size ~20×32px in SVG units.
- Cab is at the front (positive Y in local coords = direction of travel).
- Color:
  - Green (`#22c55e`) for `on_time` deliveries.
  - Yellow (`#eab308`) for `delayed` deliveries.
  - Grey (`#6b7280`) for `delivered` deliveries (still loop, just greyed out).
- Rotation: SVG `transform="rotate(angleDeg, cx, cy)"` applied to the truck group, where
  `angleDeg = Math.atan2(dy, dx) * 180/Math.PI + 90` (the +90 corrects for SVG's default
  upward orientation of the truck shape).
- The truck group has a `cursor: pointer` and a `data-delivery-id` attribute.

**Clicking a truck** — shows the same side panel already built in Step 3, with:
- Delivery ID, route name (WH name → Store name)
- Status badge (color-matched)
- Items being carried
- Direction label: "Outbound" or "Returning"

**Todo List**
1. Write the simulation spec into this plan file (done above).
2. Review the spec against the 15 delivery routes and confirm all `x/y` coordinates are in the
   correct 1792×1536 pixel space and will produce visually distinct routes (no two trucks
   perfectly overlapping).

**Relevant Context**
- Warehouse/store coordinates: all in 1792×1536 pixel space (see data exploration results).
- 15 delivery routes — some share routes (e.g. WH-01→ST-03 appears twice: DLV-002 and DLV-012).
  The offset_ms will spread them apart even on identical routes.

**Status:** `[x] done` ← Sim spec fully defined above; no code to write yet

---

### Sub-Task 3 — Rewrite Step 1 of Track A in `index.html`

**Intent**
Replace the current "download winn-dixie-data.js" step with a step that introduces the live
GitHub Pages data URL and instructs the attendee to tell Bob where to fetch data from. No file
download, no local file management.

**Expected Outcomes**
- Step 1 accordion in Track A reads as a "connect to live data" step.
- The download button and "confirm file is in workspace" sub-steps are removed.
- Replaced with a single copyable prompt that gives Bob the GitHub Pages JSON URL and instructs
  it to create `dashboard.html` that `fetch()`es the data at runtime.
- The "No folder open?" helper button is kept.
- The step unlocks Step 2 (accordion) when the copy button is clicked, same as the current
  confirm-button flow.

**Todo List**
1. In `index.html`, locate the Step 1 accordion inside `#card-a` (around line 1037).
2. Replace the `mode-steps` block with a new layout:
   - Single `mode-step` (step 1): copy prompt with the GitHub Pages URL.
   - Clicking copy on this step unlocks `uc-step2-a` (same as current confirm-step flow).
3. Update the accordion header description to mention "live data from GitHub Pages" instead of
   "download a file".
4. Update the `detail-intro` paragraph of Track A to reference the live data URL.

**Relevant Context**
- [`index.html`](index.html) lines ~1037–1088: current Step 1 `mode-steps` block.
- The JS unlock logic: `unlockUcStep('uc-step2-a')` is currently triggered by
  `data-confirm-step` button. After this change it should trigger from the copy button in the
  new step 1 — add a `data-unlock-after-copy="uc-step2-a"` attribute (or reuse the existing
  copy-btn logic that checks `parentUc.id`).

**Status:** `[ ] pending`

---

### Sub-Task 4 — Rewrite Step 2 Bob prompt (network map + fetch + trucks)

**Intent**
Replace the current Step 2 prompt (which loads data from a local `winn-dixie-data.js` script
tag) with a new prompt that:
1. Fetches data from the GitHub Pages JSON URL.
2. Builds the same network map (warehouses, stores, connecting lines, pan/zoom).
3. Adds the animated truck layer on top — using the deterministic simulation spec from Sub-Task 2.

This is the biggest prompt rewrite. The attendee pastes it into Bob and gets a `dashboard.html`
with live animated trucks in one step.

**Expected Outcomes**
- Step 2 prompt instructs Bob to:
  - `fetch('https://vladstol223.github.io/wd.bob/winn-dixie-data.json')` and store result in a
    module-scoped variable.
  - Render the SVG map (same as before: warehouses as gold hexagons, stores as colored circles,
    connecting lines, pan/zoom, reset view).
  - Start a `setInterval` at 33ms (~30fps) that recomputes all truck positions using the
    deterministic sim spec and updates SVG `transform` attributes in place (no DOM rebuild).
  - Each truck is a pre-created SVG `<g>` element with a `data-delivery-id`, updated each tick.
  - Truck rotation is computed using `Math.atan2(dy, dx) * 180/Math.PI + 90`.
  - Trucks are color-coded by delivery status.
- After Bob finishes, attendee opens `dashboard.html` and sees trucks moving.

**Todo List**
1. Draft the full replacement Step 2 prompt (to be written in the plan, then inserted into
   `index.html`).
2. The prompt must explicitly describe: the fetch call, the simulation loop, the SVG truck
   shape, the rotation formula, and the color mapping.
3. Update the Step 2 accordion content in `index.html` with the new prompt.
4. Update the "After Bob finishes" hint box to describe what the attendee should see (moving
   trucks, not just static nodes).

**Relevant Context**
- [`index.html`](index.html) lines ~1091–1139: current Step 2 accordion.
- The GitHub Pages URL: `https://vladstol223.github.io/wd.bob/winn-dixie-data.json`
- Simulation spec: defined in Sub-Task 2 above.
- Map spec is unchanged: 1792×1536 canvas, scale to SVG container, pan/drag/zoom, white ring
  highlight on click, pure black labels.

**Status:** `[ ] pending`

---

### Sub-Task 5 — Update Steps 3 and 4 prompts for fetch-based data

**Intent**
Steps 3 and 4 currently tell Bob to re-read `dashboard.html` and write a new version. They
reference `winn-dixie-data.js` as a script tag to leave untouched. That reference must be
updated to match the new fetch-based data loading pattern. The truck animation loop must also be
preserved when Bob rewrites the file in Step 3 and Step 4.

**Expected Outcomes**
- Step 3 prompt instructs Bob: do not remove the `fetch` call or the truck simulation loop when
  rewriting the file.
- Step 4 prompt does the same.
- Both prompts replace the `winn-dixie-data.js` script tag references with `data already loaded
  via fetch into window.WINN_DIXIE_DATA`.
- All other Step 3/4 behavior is unchanged (side panel, floor plan, delivery columns, filtering).

**Todo List**
1. In the Step 3 prompt in `index.html`, find and replace the reference to
   `winn-dixie-data.js` script tag with an instruction to preserve the fetch + truck loop.
2. In the Step 4 prompt, do the same.
3. Review both prompts to confirm no other local-file references remain.

**Relevant Context**
- [`index.html`](index.html) lines ~1142–1243: Steps 3 and 4 accordion content.
- Current Step 3 prompt says: *"Do NOT re-read or embed any content from `winn-dixie-data.js`
  — the script tag loading it is already in the file and must stay as-is."*
  → New version: *"Do NOT remove the `fetch` call that loads data from the GitHub Pages URL,
  and do NOT remove the truck simulation loop — both must be preserved exactly as-is."*

**Status:** `[ ] pending`

---

### Sub-Task 6 — Update the Track A card header copy in `index.html`

**Intent**
The Track A card summary and detail-intro currently describe the experience as "download a data
file" and reference `winn-dixie-data.js`. These need minor copy updates to match the new
live-data, live-truck experience without changing the overall tone or structure.

**Expected Outcomes**
- Card description (`card-desc`) updated to mention "live data feed" and "animated delivery
  trucks".
- `detail-intro` paragraph updated to mention that the dashboard will show trucks moving in
  real time.
- Tags updated: replace "Mock data included" with "Live data feed".
- Starting prompt quote at the bottom of the card updated to mention trucks.

**Todo List**
1. Update `.card-desc` text in `#card-a`.
2. Update the `.detail-intro` paragraph in `#card-a`.
3. Update the `card-tag` that says "Mock data included".
4. Update the `.csp-text` starting prompt quote.

**Relevant Context**
- [`index.html`](index.html) lines ~1016–1033 (card summary) and ~1033 (detail-intro).

**Status:** `[ ] pending`

---

## Implementation Order

```
Sub-Task 1  →  Sub-Task 2  →  Sub-Task 3  →  Sub-Task 4  →  Sub-Task 5  →  Sub-Task 6
  (data on        (sim          (Step 1        (Step 2        (Steps 3+4     (card copy
  GitHub)          spec)         rewrite)       rewrite)       cleanup)       polish)
```

Sub-Tasks 1 and 2 are prerequisites for 3–6, but they can be executed in parallel.
Sub-Tasks 3, 4, 5, 6 are all edits to `index.html` and should be done sequentially to avoid
merge conflicts.
