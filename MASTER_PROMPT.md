# Master prompt: TheTerminal

Copy everything below the line into a capable coding model to rebuild TheTerminal from scratch. It is written to produce the current version in one pass: a single-file 3D container-terminal strategy game with export loading, light/dark themes and a Render-ready repo.

---

## Role

You are a senior game developer and product designer who knows container-terminal operations (STS cranes, RTGs, yard tractors, TEU, berth windows, vessel drafts, export receiving, short-shipping, rehandles). Build a polished, playable game, not a demo. Make sensible decisions without asking; state any assumption in a short comment at the top of the script.

## Objective

Build **TheTerminal**, a live 3D container-port operations dashboard for the fictional **Port Mumbai, Terminal 2**, that plays like a real-time strategy game. The player is the harbourmaster: berth ships, keep cranes fed with tractors, plan import and export yard blocks, and balance yard space, gate traffic, tide and weather against carrier trust and money.

## Hard constraints

- **One file:** `index.html` containing all HTML, CSS and JS. No build step, bundler, framework or ES modules.
- **3D:** three.js **r128** UMD from `https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js`, loaded before the inline script. Use only r128 APIs (`BufferGeometry`, `InstancedMesh`, `ShaderMaterial` with `extensions: { derivatives: true }`, `CanvasTexture`, `Sprite`).
- **Font:** Inter from Google Fonts (400–800) with a system fallback stack. No other external assets; draw textures on canvas.
- **Wrap the script in one IIFE.** If `THREE` is missing, show a readable error in the boot overlay.
- **Persistence:** only `localStorage` for `tidegrid-theme` and `tidegrid-hint`, wrapped in try/catch. Game state is not saved.
- **Performance:** fixed simulation step ≤ 0.034 s, sub-stepped for 2× and 4×. Pixel ratio capped at 1.75. Instanced meshes for yard and ship cargo. HUD DOM refresh every 0.25 s, not every frame.
- **Money:** US dollars, formatted by a single `money()` helper (`$900k`, `$1.20M`, `−$3k/h`).
- Must work on desktop and at 400px phone width with no horizontal page scroll.

## Code organisation

Banner-commented sections in this order: Helpers · Themes · World constants · Game state (one `S` object) · Renderer/scene · Materials · Terrain · Yard blocks + RTGs · STS cranes · Ships · Trucks · Vessel calls/exports · Crane cycle · Economy, events, environment · Camera controls · HUD · Opening scenario · Main loop.

## World and 3D scene

Units are metres; the quay runs along x, sea is +z, land is −z.

- **Terminal slab** 240 × 110 m with a canvas ground texture: apron, yellow quay coping, crane rails at z −0.8 and −7.2, dashed quay lane (z −4), spine road (z −14) with an amber centre line, aisles, back road (z −96), "BERTH A/B/C" and "GATE 2" stencils, "TRACTOR DEPOT" label. Redraw the texture on theme change and when a block's role changes.
- **Yard:** 10 block footprints (columns A–E at x −72/−36/0/36/72; rows 1 and 2 at z −26/−44). Each block is 8 bays × 6 rows × 4 tiers = **192 slots**. All yard boxes share one `InstancedMesh` (40 ft box 2.4 × 1 × 1). Locked blocks show a faint wireframe. Open blocks get an RTG (yellow gantry with a trolley) that moves to each box it handles. Stencil each open block "A1 · IMPORT" or "C2 · EXPORT"; export blocks get amber outlines.
- **Berths:** A (x −62), B (x 0), C (x 62, locked until dredged). Fenders and bollards along the quay edge.
- **STS cranes:** legs, sill beams, boom over the water, A-frame apex with stays, machinery house, trolley with operator cab, spreader on two cables that scale with hoist height, a red beacon that blinks (fast when faulted), floodlights. One to three per berth, spaced evenly. New cranes grow in.
- **Ships:** extruded hull with a pointed bow (48 × 11 m), superstructure with lit windows, funnel in the line's accent colour, mast light. Deck cargo is an `InstancedMesh` of up to 288 slots (12 bays × 8 rows × 3 tiers); lifts beyond that are "hold cargo". A canvas sprite label shows the name, "x off · y on" and draft. Ships bob at anchor and settle when moored.
- **Water:** large plane with a custom shader: vertex waves damped near the quay, fresnel tint, sun specular, a **tidal grid** (12 m lines in the primary colour, pulsing outward from the quay), quay-edge foam, exponential fog. The water level follows the tide.
- **Surroundings:** warehouses (blue walls with windows), trees (low-poly icosahedron crowns), lamp posts with light pools, a gate canopy with four lane lights, a breakwater with a flashing lighthouse, a distant ridge, city lights and stars (dark theme), rain particles during storms, lightning flashes.
- **Camera:** RTS controls. Drag to pan, right-drag or Shift-drag to rotate, wheel or pinch to zoom, WASD/arrows to pan, Q/E to rotate. Clamp pitch and distance. The camera eases to whatever is selected.
- **Picking:** raycast click on ships, cranes, tractors and blocks. Show a primary-coloured ring and a bobbing teardrop pin over the selection, and red pins over faulted cranes.

## Simulation model

Game time is in minutes. **G = 0.35 game minutes per simulated second.** Start at Day 1, 07:20, with $900k and carrier trust 58/100.

### Vessel calls
- Calls are booked ahead in a schedule: keep ETAs filled to now + 270 min. Interval between ETAs = `lerp(180, 85, trust/100) × rand(0.75, 1.3)` minutes.
- Each call has a shipping line (5 lines, each with hull and accent colours: Meridian Lines, Kestrel Ocean, Halvard Maritime, Sable Coast, Orrin Freight), a unique vessel name, imports `lifts = rand(44, 70 + trust × 0.7)`, and exports `need = lifts × rand(0.4, 0.7)`.
- Draft = `10.2 + lifts × 0.04 ± 0.4` m. Ships drawing ≥ 14.5 m can only berth when the tide is ≥ 1.0 m. Tide (m) = `1.6 + 1.35 × sin(2π (t − 300) / 745)`.
- At ETA the ship spawns offshore and sails to an anchorage slot (10 slots). Cap at 8 waiting; otherwise delay the ETA by 30 min.

### Berthing and ship phases
- `inbound → anchored → berthing → working → departing`. Auto-berth (on by default) assigns free berths in priority, then arrival order, skipping tide-held ships. Players can also send a ship to a berth manually.
- On berthing, trust changes by wait time: < 30 min +2, < 90 min +0.5, else −min(6, 3 + one per extra hour).
- **Working** has two phases: **discharge** (lift every import off), then **load** (lift exports from the yard on). The ship sails when all exports are loaded, or sails short (below). Track timestamps for arrived, anchored, cleared, berthed, discharged and sailed.

### Cranes
- Speeds (units/s): hoist 14, trolley 16, gantry 8. Autopilot upgrade = 1.12×.
- **Discharge cycle:** pick a slot in the crane's ship section (top tier first; take over a faulted neighbour's section) → travel → lower → grab → hoist → traverse to the quay lane → wait for a tractor under the boom (and import yard room) → set → release onto the tractor. Earn **$1,400 per lift** (+20% for priority calls).
- **Load cycle:** wait over the quay lane for a loaded tractor → lower → grab the box → hoist → travel to an empty ship slot (bottom tier first, own section first; use hold space if none) → lower → release (box appears in that slot, line colour kept) → hoist. Each load lift also earns $1,400.
- Wind limit 38 kn (46 kn with storm-rated spreaders). Above it all cranes stop ("wind hold").
- Faults: every 380–700 min a random working crane faults for 180 min. "Rush repair" costs $30k.

### Yard tractors and blocks
- Start with 7 tractors parked in a depot grid. Routes follow lanes: aisle → spine road → destination (quay lane or block lane under the RTG). Speed 15 (Autopilot +20%).
- **Dispatch** every 0.2 s:
  - Discharge: one empty tractor per crane, nearest first.
  - Load: assign a tractor to fetch an export box for that ship's call from the nearest block holding it, plus **one queued tractor behind each loading crane**.
  - Release tractors when a crane no longer needs them.
- **Block roles:** each open block is Import or Export (player-switchable). Discharge lands only in import blocks (least full). Gate deliveries land only in export blocks (prefer the block already holding that call's boxes).
- Per block, track `count`, `imp` + arrival timestamps, `exp[callId]` + timestamps, `expRes[callId]`, `reserved`, `outbound`. Invariant: `count = imp + Σ exp`, and `count + reserved ≤ 192`.
- Stacking is abstract LIFO (the visual top box is removed). **Rehandles:** when picking from a block, chance = `0.5 × (1 − same-category share)`. A rehandle costs $300 and adds 1.3 s of RTG time.
- RTG pick and drop animations lift, traverse and lower each box.

### Gate and exports
- **Receiving window** (player lever): 1, 2 (default) or 3 h before ETA. When it opens, shippers deliver the call's exports along a curve: 88% spread evenly up to ETA, 12% trickling in over the 90 min after ETA.
- **Gate capacity:** 40 trucks per lane per hour (2 lanes at start, up to 4), alternating between export deliveries and import pickups.
- **Import pickups:** demand accrues at 25% of imports-in-yard per hour (≈ 4 h mean dwell). Customs holds pause pickups for 80–130 min (every 800–1300 min).
- **Hold for late exports** (player lever): 30 min, 60 min (default) or until loaded. The clock only runs while the ship is idle because none of its boxes are in the yard. When it expires, or the player clicks "Sail now", the ship finishes in-flight moves and sails short. Trust −1 per 6 boxes left behind; the boxes roll to that line's next scheduled call.
- Track dwell: average age in yard and dwell at exit, for imports and exports separately.

### Economy and events
- **Prices:** tractor $45k, yard block + RTG $160k, gate lane $90k, Berth C $480k (opens with one crane), extra crane $260k (max 3 per berth), Autopilot $300k, storm-rated spreaders $220k.
- **Running costs per game hour:** crane $6k, tractor $1.5k, block $2.5k, berth $4k, gate lane $1.5k.
- **Storms:** every 600–1000 min (first at 420–640), lasting 70–130 min: wind 42–54 kn, darker sky, fog, bigger waves, rain, lightning. Normal wind is 6–24 kn and eases toward a target.
- **Priority contract offers:** every 260–420 min (first at 150). Offer: 90–140 imports, exports = 35% of that. The ship arrives 60 min after acceptance and its receiving opens immediately. Turnaround window = `(lifts + exports) × 1.3 + 70` min from arrival. Bonus = 90% of lift revenue: +4 trust if met, −6 if missed. Offers expire after 60 min.
- Day rollover logs yesterday's lifts, TEU (2 per box) and revenue, and resets daily counters.

### Opening scenario (mid-shift, not empty)
- Blocks: B1 import (104 boxes), C1 import (88), D1 export, C2 export. Four cranes: two each on Berths A and B.
- Berth A: MV Calder Reach (Meridian Lines), working, 34 of 96 discharged, 48 exports booked, 41 already in D1.
- MV Tessaline berthing at B. MV Ostra Bay at anchor with a 15.2 m draft (tide-sensitive). One more ship inbound.
- Two upcoming calls with receiving already open; exports for several calls mixed across D1 and C2, so rehandles happen.
- Pre-fill the throughput sparkline and the gate history so the dashboard reads as live from the first frame.

## Interface

Visual reference: a modern logistics SaaS dashboard floating over an isometric scene. White cards, one royal-blue accent, soft shadows, rounded corners, soft status pills.

### Design tokens (CSS custom properties; never literal colours in components)

| Token | Light | Dark |
|---|---|---|
| bg | #EDF1F7 | #09111E |
| surface / surface-2 | #FFFFFF / #F3F6FB | #0F1A2C / #16233A |
| glass (card fill) | rgba(255,255,255,.95) | rgba(13,23,40,.93) |
| line / line-strong | #E3E8F0 / #CDD6E3 | #1F2E47 / #2D4062 |
| text / muted / dim | #0F172A / #5B6B82 / #94A3B8 | #E8EEF6 / #97A7BF / #61738F |
| primary / strong / soft | #2F5BEA / #2449C7 / #E9EEFD | #5B82FF / #86A3FF / rgba(91,130,255,.16) |
| success / soft | #15803D / #DCFCE7 | #4ADE80 / rgba(74,222,128,.13) |
| warn / soft | #B45309 / #FEF3C7 | #FBBF24 / rgba(251,191,36,.14) |
| danger / soft | #DC2626 / #FEE2E2 | #F87171 / rgba(248,113,113,.15) |

- Light values on `:root`. Dark values under `@media (prefers-color-scheme: dark) { :root:not([data-theme="light"]) }` **and** `:root[data-theme="dark"]`, with `color-scheme: dark`.
- A top-bar toggle sets `data-theme` and remembers it. Watch both the media query and the attribute, and re-theme the 3D scene live (sky, fog, water, ground texture, materials, labels, pins).
- **Light scene:** daylight, light-blue water, pale ground, blue cranes, sheds and tractors, green trees.
- **Dark scene:** night shift, navy water, lit windows and lamp pools, stars and city lights.
- Card radius 14px, buttons 8–9px, pills fully rounded. Status is always soft background + strong text (green ok, amber warn, red crit, blue info). Status colours are never decoration.
- Inline SVG icon sprite (24 viewBox, 1.8px stroke, round caps, no fill): ship, crane, truck, stack, gate, coin, shield, wave, pulse, alert, check, anchor, berth, search, sun, moon, plus, minus, rotate, home, chevrons, bolt, storm, wind, grid, close, wrench, map pin. No emoji.

### Layout (desktop)
1. **Top bar:** logo tile + "TheTerminal" · search ("/" to focus; finds vessels, cranes, blocks, tractors; arrow keys + Enter) · site selector "Port Mumbai · T2 / n of m berths busy · k waiting" (click resets the view) · green Live chip with "Day 1 · 07:20" · speed control (II, 1×, 2×, 4×; Space, 1, 2, 3 keys) · theme toggle · avatar "HM / Harbourmaster".
2. **KPI cards, top-left (4):**
   - Treasury, with net $/h delta and opex subtext.
   - Quay throughput (lifts/h), with change vs 1 h ago, TEU today and lifts per crane.
   - Carrier trust (/100), with change today and tier: Preferred ≥ 75, Reliable ≥ 50, Under review.
   - Tide & wind (m with rising/falling chip), with wind vs crane limit.
3. **Terminal panel, right**, with header (eyebrow "TERMINAL · T2", "Mumbai Container Terminal", status pill Operational / Wind hold / Storm watch / Customs hold / Crane fault, and a working · at anchor · inbound line) and three tabs:
   - **Overview:**
     - Metrics: berth use, yard fill (import/export composition bar), average anchor wait, gate moves in/out per hour, import dwell, export dwell.
     - Throughput sparkline (6 h, area fill, endpoint dot).
     - Watch list rows (icon tile + text + Act now / Watch / Clear pill).
     - Yard block list (role, fill %, status pill).
     - Tractor fleet bar.
   - **Yard plan:**
     - Receiving window and hold-policy segmented controls, each with a one-paragraph explanation of the trade-off.
     - Yard dwell metrics: imports/exports in yard with average age, dwell at exit, pickups queued, short-shipped total, rehandles today.
     - Block plan rows: import/export composition bar, counts and an Import/Export toggle.
   - **Build:** upgrade cards (icon tile, title, owned count, description, price, Buy/Done) and a running-costs card.
4. **Vessel tracking, bottom-left:**
   - 6-step stepper: Arrived → Anchor → Berth → Discharge → Load → Sailed. Done = filled check; current = ring with halo plus progress such as "50/96"; Sailed shows ETC.
   - Vessel card: `#V-0001`, name, line dot, lifts, draft, status pill (Discharging / Loading / Awaiting exports / Sailing short…), berth and cranes, "Exports x/y on board · n in yard · m at gate", hold countdown, contract countdown, tide warning.
   - Actions: Send to Berth A/B/C, Sail now · leave N.
   - Prev/next arrows cycle vessels. Selecting a crane, block or tractor swaps this card for an inspector (metrics, status pill, actions such as Rush repair or the block role toggle).
5. **Operations card, bottom-right**, with tabs:
   - **Berths:** per berth, ship chip, status pill, combined progress bar, "x off" or "y on", and + crane button. Locked Berth C row with an Open button.
   - **Schedule:** ETA, countdown, vessel and line, off/on counts, Receiving / Opens hh:mm / Delayed pill, "exports in yard" progress.
   - **Anchorage:** wait time, vessel, lifts, draft, status, plus an auto-berth switch.
   - **Log:** timestamped entries coloured by severity.
6. **Floating cards** under the KPIs: a dismissible harbourmaster's brief (objective + controls) and priority-contract offers (Accept / Decline). A vertical map control (zoom in, zoom out, rotate 45°, reset) sits at the right edge of the scene.
7. **Toasts** at top centre (max 3, about 4 s) for sailings, purchases and warnings.

**Watch-list alerts:** wind hold, storm, crane faults (click selects the crane), cranes waiting on tractors, yard full or above 82%, export blocks above 85% or gate-in stalled, customs hold, tide-held vessels, anchorage wait over 90 min, ships waiting on exports with sail-short countdown, 8 or more rehandles per hour, more than 30 queued pickups, overdrawn treasury.

### Responsive
- ≤ 1240px: narrower right column, profile hidden, tracking card stacks.
- ≤ 1020px: site selector hidden.
- ≤ 900px: search hidden; KPIs become a horizontal scroll strip; the panel, tracking and berths cards become bottom sheets opened from a bottom tab bar (Overview · Berths · Track · Build). Selecting a ship opens the Track sheet.
- ≤ 520px: wordmark and 2× hidden; berth rows drop the progress column.
- Respect safe-area insets and `prefers-reduced-motion`. Visible focus rings. Icon-only buttons need aria-labels.

### Copy
Plain, specific, active voice. Use operator terms: berth, STS, RTG, tractor, TEU, draft, receiving window, short-shipped, rehandle. Buttons say what they do. Alerts name the problem and the fix in one line. No marketing language, exclamation marks or emoji.

## Implementation notes
- Render HUD sections through a `put(selector, html)` helper that string-caches and skips while a pointer is down on the HUD, so clicks are not swallowed by re-renders. Handle all actions through one delegated click handler keyed on `data-act` / `data-id`, and force a re-render after a state change.
- Keep reservation counters balanced (`reserved`, `expRes`, `fetching`), and keep crane↔tractor links consistent. Never let a ship depart while a box for it is in flight.
- Toggle visibility with the `hidden` attribute; never change `style.display`.

## Repository deliverables
- `index.html`: the game.
- `README.md`: what it is, run locally (`python3 -m http.server 8080`), controls table, deploy notes.
- `render.yaml`: Render static site, no build command, publish the repo root, headers `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `X-Frame-Options: SAMEORIGIN`.
- `CLAUDE.md`: code map, game model, invariants, UI conventions.
- `DESIGN.md`: tokens, typography, components, breakpoints, 3D palette.

## Acceptance checks (run before you finish)
1. `node --check` passes on the extracted inline script; no console errors on load.
2. The first frame shows a working terminal: ships at berth, cranes moving, KPIs populated, no empty states.
3. Fast-forward 12 game hours with no input: ships discharge, load and sail; no crane or tractor stalls forever; trust stays above about 40; export blocks approach capacity, which is the intended pressure.
4. Each lever has a visible effect: an Early window raises export dwell and yard fill; "Until loaded" stops short-shipping; switching a block's role redirects arrivals and redraws its stencil.
5. Light, dark and the toggle all re-theme the UI and the 3D scene; nothing is unreadable in either theme.
6. At 400px: no horizontal scroll, bottom sheets work, the stepper fits.
