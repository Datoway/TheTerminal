# DESIGN.md

TheTerminal design system: how the interface and the 3D scene look, and the rules for extending them. Every value here comes from `index.html`; if you change a token there, update this file too.

## Principles

1. **The scene is the hero.** The terminal fills the screen; UI floats above it as cards. Keep cards compact so the quay, cranes and yard stay visible.
2. **Operations board, not a toy.** Clean white (or navy) cards, one blue accent, soft shadows, real units (lifts/h, TEU, m, kn, min). The visual reference is a modern logistics SaaS dashboard.
3. **Colour means something.** Blue is the brand and interactive accent. Green, amber and red are reserved for status. Never use status colours for decoration.
4. **Summary before detail.** KPIs at the top, status pills on every row, detail one click away (inspector, tracking card, tabs).
5. **Both themes are first-class.** Light is the daylight ops board; dark is the night shift. Neither is an inversion of the other.

## Colour tokens

All colours are CSS custom properties. Components reference tokens only, never literals.

| Token | Light | Dark | Use |
|---|---|---|---|
| `--bg` | `#EDF1F7` | `#09111E` | Page background behind the scene |
| `--surface` | `#FFFFFF` | `#0F1A2C` | Cards, rows, inputs |
| `--surface-2` | `#F3F6FB` | `#16233A` | Tab tracks, neutral pills, bar tracks |
| `--glass` | `rgba(255,255,255,.95)` | `rgba(13,23,40,.93)` | Floating card and top bar fill |
| `--line` | `#E3E8F0` | `#1F2E47` | Default borders, dividers |
| `--line-strong` | `#CDD6E3` | `#2D4062` | Hover borders, button outlines, step connectors |
| `--text` | `#0F172A` | `#E8EEF6` | Primary text |
| `--muted` | `#5B6B82` | `#97A7BF` | Labels, secondary text |
| `--dim` | `#94A3B8` | `#61738F` | Tertiary text, placeholders, inactive icons |
| `--primary` | `#2F5BEA` | `#5B82FF` | Brand accent, primary buttons, icon tiles, active steps |
| `--primary-strong` | `#2449C7` | `#86A3FF` | Primary hover |
| `--primary-soft` | `#E9EEFD` | `rgba(91,130,255,.16)` | Soft tiles, info pills, focus ring halo |
| `--on-primary` | `#FFFFFF` | `#FFFFFF` | Text and icons on primary |
| `--success` / `--success-soft` | `#15803D` / `#DCFCE7` | `#4ADE80` / `rgba(74,222,128,.13)` | Working, in service, clear, positive deltas |
| `--warn` / `--warn-soft` | `#B45309` / `#FEF3C7` | `#FBBF24` / `rgba(251,191,36,.14)` | Watch, near full, waiting, export blocks |
| `--danger` / `--danger-soft` | `#DC2626` / `#FEE2E2` | `#F87171` / `rgba(248,113,113,.15)` | Act now, fault, full, negative deltas |

**Theme switching.** Light values live on bare `:root`. Dark values are defined twice with identical content: under `@media (prefers-color-scheme: dark) { :root:not([data-theme="light"]) }` and under `:root[data-theme="dark"]`, both with `color-scheme: dark`. The top-bar toggle sets `data-theme` on `<html>` and stores it in `localStorage` (`tidegrid-theme`).

**Status pattern.** Status always appears as soft background + strong foreground of the same hue: `.pill.ok|warn|crit|info`, `.tile.ok|warn|crit|soft`, `.delta.up|down|flat`.

## Typography

- **Family:** Inter (Google Fonts, weights 400–800), fallback `system-ui, -apple-system, 'Segoe UI', Roboto, sans-serif`. Font features `cv11`, `ss01`.
- **Numbers:** always `font-variant-numeric: tabular-nums` where figures update or align.

| Role | Size / weight | Example |
|---|---|---|
| Wordmark | 19px / 800, −0.02em | TheTerminal |
| KPI value | 23px / 800, −0.02em | $903k, 46 lifts/h |
| Panel title | 17px / 800, −0.015em | Mumbai Container Terminal |
| Metric value | 18px / 700 | 43%, 36 min |
| Card / section heading (`h3`) | 13px / 700 | Vessel tracking, Watch list |
| Body / rows | 12.5–13.5px / 400–600 | Row text, notes |
| Labels | 11.5–12px / 500–600, `--muted` | Treasury, Yard fill |
| Eyebrow | 11px / 700, uppercase, 0.08em, `--primary` | TERMINAL · T2 |
| Pills, chips, captions | 11–11.5px / 600 | Working, +$9k/h |

## Shape, depth, spacing

- **Radius:** cards `14px` (`--r`); tiles, inputs, rows, tab tracks `10–12px`; buttons `8–9px`; pills `999px`.
- **Shadow:** `--shadow` on floating cards; `--shadow-sm` on the active tab and top bar. No other elevation levels.
- **Borders:** 1px `--line` on every card and row; `--line-strong` on hover. Dashed border marks locked items (Berth C).
- **Spacing:** card padding 12–14px; gap between cards 12–14px; row gap 6px; section gap 16px. Use flex/grid `gap`, not margins.
- **Gutter:** HUD and top bar use `max(16px, env(safe-area-inset-*))` on both sides.

## Components

| Component | Class | Notes |
|---|---|---|
| Card | `.card` | `--glass` fill, 1px border, `--r` radius, `--shadow`. Base for every floating panel |
| Icon tile | `.tile` (`.sm`, `.soft`, `.ok/.warn/.crit`) | 36px (30px small) rounded square with a single icon. Solid primary for KPIs and headers; soft variants in lists |
| KPI card | `.kpi` | Label → tile + value + delta chip → one-line subtext. Four in a row, top-left |
| Pill | `.pill` | Status label on rows and cards. Neutral by default |
| Delta chip | `.delta.up/down/flat` | Change vs. a reference: "+$9k/h", "+3 today", "▲ Rising" |
| Button | `.btn`, `.btn.primary`, `.btn.soft`, `.btn.sm` | One primary action per card at most |
| Icon button | `.iconbtn` (`.sm`) | Map controls, theme toggle, prev/next, add crane |
| Tabs / segmented | `.tabs` (`.seg`, `.mini`) | Track in `--surface-2`; active tab is a raised `--surface` chip. Used for panel tabs, Berths/Schedule/Anchorage/Log, receiving window, hold policy, block role |
| Progress bar | `.bar > i` | 6px, primary fill; `.warn`/`.crit` recolour at thresholds |
| Composition bar | `.cbar > .ci + .ce` | Import (primary) vs export (warn) share of a block |
| Metric box | `.metric` | Label, value, bar or subtext. Two-column grid in the terminal panel |
| List row | `.row`, `.drow`, `.brow` | Bordered rows: alerts and blocks (`.row`), berth/schedule/anchorage (`.drow`, 5-column grid), block plan (`.brow`) |
| Stepper | `.steps` | 6 steps: Arrived → Anchor → Berth → Discharge → Load → Sailed. Done = filled primary with check; current = primary ring with halo; upcoming = grey outline |
| Vessel card | `.vcard` | Tile + ID (`#V-0001`) + name + line dot + status pill + export counts + actions |
| Toast | `.toast` | Top-centre, max 3, auto-dismiss after ~4 s. Icon colour follows kind |
| Switch | `.switch > .sw` | Auto-berth toggle |
| Search | `.search` + `.qres` | `/` focuses it; arrow keys and Enter pick a result |

## Iconography

- Inline SVG `<symbol>`s in the document (`#i-ship`, `#i-crane`, `#i-truck`, `#i-stack`, `#i-gate`, `#i-coin`, `#i-shield`, `#i-wave`, `#i-pulse`, `#i-anchor`, `#i-berth`, `#i-bolt`, `#i-storm`, `#i-wrench`, `#i-map`, …), rendered with `ico(name)`.
- 24×24 viewBox, stroke `currentColor`, 1.8px width, round caps and joins, no fills. Displayed at 15–19px.
- One icon per meaning: ship = vessels, crane = STS/terminal, stack = yard blocks, truck = tractors, gate = road gate, anchor = anchorage.
- No emoji anywhere in the UI.

## Layout

```
┌──────────────────────── Top bar ────────────────────────┐
│ Logo · Search · Site selector        Live clock · Speed · Theme · Profile │
├──────────────────────────────────────────────┬──────────┤
│ KPI · KPI · KPI · KPI                         │ Terminal │
│ Floating cards (brief, contract)   Map ctrls │  panel   │
│                    3D scene                   │ Overview │
│                                               │ Yard plan│
├───────────────────────────────────┬──────────┴──────────┤
│ Vessel tracking (stepper + card)   │ Berths / Schedule /  │
│                                    │ Anchorage / Log      │
└────────────────────────────────────┴──────────────────────┘
```

| Breakpoint | Behaviour |
|---|---|
| > 1240px | Full layout; right column 372px; Berths card 440px |
| ≤ 1240px | Right column 336px; profile hidden; tracking card stacks steps over the vessel card |
| ≤ 1020px | Site selector hidden |
| ≤ 900px (phone/tablet) | Search hidden; KPIs become a horizontal scroll strip; panel, tracking and berths become bottom sheets opened from a bottom tab bar (Overview · Berths · Track · Build) |
| ≤ 520px | Wordmark and 2× speed hidden; berth rows drop the progress column |

## 3D scene language

The scene uses the same palette as the UI, via `THEMES.light` / `THEMES.dark` applied in `applyTheme()`.

| Element | Light (daylight) | Dark (night shift) |
|---|---|---|
| Sky top / horizon | `#B9CFEB` / `#E9EFF6` | `#030A14` / `#0A1B2C` |
| Water deep / shallow | `#4F8FD0` / `#9CCAEE` | `#03172A` / `#0E3C5C` |
| Water grid ("tidal grid") | primary blue, 22% | `#5B82FF`, 30% |
| Ground / apron | `#E3E9F0` / `#EDF1F6` | `#0F1C29` / `#142638` |
| STS cranes, sheds, yard tractors | `#2F5BEA` | `#3F68E6` / `#1B2F66` |
| RTGs | `#F2B43A` | `#D8A038` |
| Trees | `#5CC46E` | `#1D5A3C` |
| Lighting | Bright hemisphere + sun, no lamp pools | Dim moonlight, lit windows, amber lamp pools, stars, city lights |

- **Ground markings** are painted on a canvas texture: import blocks have blue outlines; export blocks have amber outlines and an "· EXPORT" stencil.
- **Containers:** mixed industry colours; export boxes mostly carry their shipping line's accent colour, so a vessel's cargo can be traced in the yard.
- **Overlays:** ship labels are rounded cards in the theme's surface colour; selection uses a primary ring plus a blue teardrop pin; faulted cranes get a red pin.
- **Weather:** storms darken sky and fog, raise waves and add rain and lightning in both themes.

## Motion

- Cards and toasts rise in over 0.3–0.35s; the live dot pulses; pins bob gently.
- The camera eases to selections; nothing jumps.
- `prefers-reduced-motion` disables the card, toast and live-dot animations.

## Voice and copy

- Plain, specific, active voice. Name things the way a terminal operator would: berth, STS crane, RTG, tractor, TEU, draft, receiving window, short-shipped, rehandle.
- Controls say what they do ("Send to Berth A", "Sail now · leave 6", "Rush repair · $30k"); the result is confirmed in a toast or the log.
- Alerts state the problem and the fix in one line ("Export blocks at 88%. Open or switch a block, or shorten the receiving window.").
- Money is always formatted by `money()`: `$900k`, `$1.20M`, `−$3k/h`.
- Avoid marketing language, exclamation marks and emoji.

## Accessibility

- Visible focus ring (`2px solid var(--primary)`) on all interactive elements.
- Status is never conveyed by colour alone: every pill has a text label and every alert has an icon and text.
- Icon-only buttons carry `aria-label` and `title`.
- Both themes keep body text at WCAG AA contrast against `--surface`.

## Adding something new

1. Reuse an existing component before inventing one; extend with a modifier class if needed.
2. Use tokens only. If a new colour is truly needed, add a token to all three theme blocks.
3. Check light, dark, and 400px width before shipping.
4. Update this file when you add a token, component or breakpoint.
