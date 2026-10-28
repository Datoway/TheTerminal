# CLAUDE.md

Guidance for Claude (and other agents) working in this repository.

## What this is

**TheTerminal** is a single-page 3D container-terminal operations game set in the fictional **Port Mumbai** (Terminal 2). You play the harbourmaster: berth ships, discharge imports, receive and load exports, plan import/export yard blocks, and manage tide, wind, gate traffic and carrier trust.

- One file does everything: `index.html` (HTML + CSS + JS, ~2,400 lines). There is no build step, bundler or package manager.
- The only dependency is **three.js r128**, loaded as a UMD global from cdnjs:
  `https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js`.
  Stay on r128 APIs (e.g. `THREE.Geometry` is gone, but `BufferGeometry`, `InstancedMesh`, `ShaderMaterial` with `extensions.derivatives` are used). Do not switch to ES modules or a bundler without being asked.
- Fonts: Inter from Google Fonts, with a system fallback stack.
- Deployed as a **Render static site** (`render.yaml`: publish the repo root, no build command).

## Run and check

```bash
python3 -m http.server 8080          # then open http://localhost:8080
```

Quick syntax check of the inline script (Node required):

```bash
python3 -c "s=open('index.html').read(); open('/tmp/t.js','w').write(s.split('<script>')[1].split('</script>')[0])" && node --check /tmp/t.js
```

There are no automated tests. After changing simulation logic, load the page, run it at 4× for a few game hours, and watch the Watch list, Berths/Schedule tabs and the Log for stalls (cranes waiting forever, ships never sailing, blocks stuck full).

## Code map (`index.html`)

The `<script>` is one IIFE. Sections are marked with `/* ==== ... ==== */` banners, in this order:

| Section | Contents |
|---|---|
| Helpers | `rand`, `clamp`, `lerp`, `approach`, `money()` (formats `$` amounts), `esc()` |
| Themes | `THEMES.light` / `THEMES.dark` scene palettes; `TH` is the active one |
| World constants | `G` (game minutes per simulated second), prices `PRICE`, running costs `OPEX`, `LINES` (shipping lines), `BERTHS`, `BLOCK_ORDER`, `BLK` (8×6×4 = 192 slots) |
| Game state | the single `S` object: credits, trust (`rep`), clock (`minutes`), ships, trucks, cranes, blocks, calls, policies (`window`, `hold`), counters |
| Renderer / scene, Materials, Terrain | three.js setup, water shader, `M` materials, ground canvas texture (`drawGround()`) |
| Yard blocks + RTGs | `S.blocks`, `slotPos`, `setYardSlot` (one `InstancedMesh` for all yard boxes), `openBlock` |
| STS cranes | crane meshes, `addCrane`, `layoutCranes` |
| Ships | `spawnShip({ call })`, `assignBerth`, `updateShips` (discharge → load → sail), `depart` |
| Trucks | yard tractors (`S.trucks`) and gate trucks (`S.ext`), `route`, `dispatch()` |
| Vessel calls / exports | `makeCall`, `scheduleCalls`, receiving (`deliveryTarget`, `spawnExportTruck`), `startYardPick`, `rollTo` (short-shipped boxes), `rehandle`, dwell tracking |
| Crane cycle | `updateCranes`: discharge phases (`travel → lower → grab → hoist → traverse → await → set → release`) and load phases (`llower → lgrab → lhoist → ltravel → llower2 → lrelease → hoist2`) |
| Economy, events, environment | `updateEconomy` (schedule, gate, storms, customs, faults, contracts, auto-berth), `updateEnvironment` (tide, sky, fog, rain) |
| Camera controls | RTS pan/rotate/zoom, picking, `select()` |
| HUD | DOM rendering: KPI cards, terminal panel (Overview / Yard plan / Build), vessel tracking, Berths/Schedule/Anchorage/Log, search, theme toggle, click actions (`onClick` switch on `data-act`) |
| Opening scenario | `startScenario()` seeds a mid-shift state |
| Main loop | fixed-step simulation (≤ 0.034 s steps) then render; HUD refreshes every 0.25 s |

## Core game model

- **Calls before ships.** Every ship comes from a scheduled *call* (`S.calls`) booked ~4.5 game hours ahead with `lifts` (imports off) and `need` (exports on). Export boxes are keyed in blocks by **call id** (`b.exp[callId]`), not ship id.
- **Ship phases:** `inbound → anchored → berthing → working (phase 'discharge' → 'load') → departing`. A ship sails when `call.loaded >= call.need`, or sails short when the hold policy expires / the player clicks "Sail now". It only departs when `call.fetching === 0`.
- **Block roles:** each open block is `role: 'import' | 'export'`. Discharge only lands in import blocks; gate deliveries only in export blocks. Per block: `imp` count + `impT` timestamps, `exp[callId]` + `expT[callId]`, `expRes[callId]` (reserved for pickup).
- **Yard stacking is abstract LIFO:** the visual top slot is removed regardless of which box is "picked"; `rehandle()` models the cost of digging in mixed blocks.
- **Gate lanes are shared** between export deliveries and import pickups (40 trucks/lane/hour, alternating).

## Invariants — keep these true when editing

- `b.count === b.imp + Σ b.exp[*]`, and `b.count + b.reserved <= BLK.cap`.
- Every `b.reserved++` (tractor or gate truck heading to a block) must be matched by a `b.reserved--` when it lands; same for `b.expRes` and `call.fetching`.
- `c.truck` / `c.nextTruck` and `t.crane` must point at each other; clear both when a truck is released (`sendToDepot`).
- Game time is in **minutes** (`S.minutes`); per-frame `dt` is simulated seconds. Convert with `G` (`dt * G`).
- Balance knobs that interact: `G`, the arrival interval in `scheduleCalls` (`lerp(180, 85, rep/100)`), export share in `makeCall`, `PRICE`/`OPEX`, `LIFT_RATE`, `REHANDLE_COST`. If you change one, fast-forward a few game hours and check queue length and trust.

## UI conventions

- **Theming:** all colours are CSS custom properties on `:root`, redefined for dark under `@media (prefers-color-scheme: dark)` (guarded by `:root:not([data-theme="light"])`) and under `:root[data-theme="dark"]`. Never add a literal colour to a component rule. The 3D scene follows via `applyTheme()` / `THEMES`.
- Palette: primary blue `#2F5BEA` (dark `#5B82FF`); green/amber/red are **status only** (`.pill.ok/.warn/.crit`, `.tile.ok/...`).
- Icons are inline SVG `<symbol>`s (`#i-ship`, `#i-crane`, …) rendered with `ico(name)`.
- HUD HTML is rebuilt through `put(selector, html)`, which skips re-rendering while a pointer is down on the HUD (`uiHold`) and caches by string. After a click that changes state, call `forceRender()` inside `onClick`.
- New buttons: add `data-act="…" data-id="…"` and handle the action in `onClick`.
- Visibility is toggled with the `hidden` attribute (`[hidden]{display:none!important}`), not `style.display`.
- Must work at phone width (~400 px): side cards become bottom sheets via the tab bar under `@media (max-width: 900px)`.
- Copy style: plain, specific, active voice; no marketing language. Money is shown with `money()` (`$900k`, `$1.20M`).

## Persistence

Only `localStorage` keys `tidegrid-theme` and `tidegrid-hint` (wrapped in try/catch). Game state is not saved; a reload starts a new shift.

## Git and deploy

- Default branch `main`; Render auto-deploys on push once the service is connected.
- Keep the repo to `index.html`, `README.md`, `render.yaml`, `CLAUDE.md`, `DESIGN.md`, `MASTER_PROMPT.md` unless a change genuinely needs more files.
- `MASTER_PROMPT.md` is the from-scratch build spec; update its numbers when you change game constants or the UI.
- Visual and UI rules (tokens, components, breakpoints, 3D palette) live in `DESIGN.md`; follow it and update it when you change the design.
