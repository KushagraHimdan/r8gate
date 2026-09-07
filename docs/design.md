# R8Gate — UI/UX Design Document

**Version:** 1.0
**Status:** Draft
**Companion documents:** PRD.md, SRS.md, architecture.md
**Owner:** [Your Name]

**Scope note:** R8Gate's UI is a single-purpose **observability dashboard** — it shows live rate-limiting activity, it does not manage config (per PRD/SRS, config is file/env-based in MVP). Keeping that scope narrow is what keeps this design simple, modern, and actually finishable. Every decision below optimizes for "an operator glances at this to understand what's happening right now," not for a full admin console.

---

## 1. Design Principles

1. **Data-first, chrome-last.** The dashboard exists to show numbers changing in real time. Nothing should compete visually with the data itself — no decorative elements, no unnecessary cards-around-cards.
2. **Legible at a glance.** An operator should understand system health within 2 seconds of looking at the screen — this drives the color and hierarchy decisions below.
3. **Calm by default, loud when it matters.** Normal operation should look quiet and unremarkable; denials/degraded states should be the only thing that visually raises its hand.
4. **Technical, not corporate.** This is a tool for engineers. Aesthetic reference points: Vercel dashboard, Linear, Upstash, Grafana — not a consumer SaaS marketing site.
5. **No dead ends.** Every state (loading, empty, error) tells the user what's happening and, where relevant, what to do next.

---

## 2. Information Architecture & Navigation

Given the MVP scope (read-only observability, no config UI), R8Gate's dashboard is intentionally **single-page**, not multi-route. This avoids the temptation to over-build navigation for a tool that doesn't need it yet.

```
┌───────────────────────────────────────────┐
│  Top Bar: Logo · Connection status · Env   │
├───────────────────────────────────────────┤
│                                             │
│  Section A — Summary strip (4 stat cards)  │
│                                             │
│  Section B — Live traffic chart            │
│  (allowed vs denied over time)             │
│                                             │
│  Section C — Per-client table              │
│  (sortable, searchable)                    │
│                                             │
└───────────────────────────────────────────┘
```

If the project grows past MVP (e.g., a future config UI), the natural next step is a left sidebar with `Overview / Clients / Config / Logs` — noted here so the layout isn't painted into a corner, but **not built now**.

---

## 3. User Journey

**Primary persona:** the operator (you, or another engineer running R8Gate), checking in on system behavior — either passively monitoring, or actively investigating "why is client X getting rate-limited."

**Journey — passive monitoring:**
1. Opens dashboard → sees summary strip → confirms deny rate looks normal → glances at chart trend → done. (~5-10 seconds)

**Journey — active investigation:**
1. Opens dashboard (or is already on it) → notices deny count spiking in summary strip or chart
2. Scans per-client table, sorted by deny count descending (default sort)
3. Identifies which client is being denied most
4. (Optional) Searches/filters table for that client to confirm pattern over time
5. Leaves the dashboard with an answer — no further UI needed, the fix happens in config/code, outside this tool

This two-journey framing is what justifies the screen layout in Section 2 — everything needed for both journeys fits on one screen without a click.

---

## 4. Screens & User Flow

### 4.1 Screen: Dashboard (the only screen)

**Section A — Summary strip (4 stat cards, single row, equal width)**
- Total Requests (current window, e.g., last 5 min)
- Allowed (count + %)
- Denied (count + %)
- Active Clients (distinct clients seen in current window)

**Section B — Live traffic chart**
- Time-series line/area chart, two series: Allowed (calm color) vs Denied (alert color)
- X-axis: rolling time window (last 5/15/60 min, simple toggle)
- Updates via polling (every 3-5s) or WebSocket push, per architecture.md

**Section C — Per-client table**
- Columns: Client Name, Tier, Requests, Allowed, Denied, Deny Rate %, Last Seen
- Default sort: Denied (descending) — surfaces the most relevant row first
- Search/filter input above the table (filters by client name)
- Row click: expands inline (no navigation) to show a small per-client mini-chart — avoids needing a second screen/route

### 4.2 Flow diagram

```
[Load dashboard]
      │
      ▼
[Fetching state] ──(success)──▶ [Populated dashboard]
      │                                │
   (error)                      (row click)
      │                                │
      ▼                                ▼
[Error state]                 [Inline expanded row]
                                        │
                                 (click again)
                                        │
                                        ▼
                              [Collapsed back]
```

There is no multi-step form flow in MVP — this document still specifies form/input patterns (Section 6) since the search/filter input and time-range toggle are the only interactive controls, and future config screens will reuse these same patterns.

---

## 5. Component Interaction

| Component | Interaction | Behavior |
|---|---|---|
| Stat cards | Static, no click action (MVP) | Number updates in place with a brief highlight flash on change (200ms background flash, not a jarring re-render) |
| Time-range toggle (5m/15m/60m) | Click to switch | Chart re-renders with new window; selected option gets an underline/active state, not a full button-style change (keep it quiet) |
| Chart | Hover | Tooltip shows exact allowed/denied counts at that timestamp |
| Table column headers | Click to sort | Ascending/descending toggle; active sort column shows a small arrow indicator |
| Search input | Type to filter | Debounced (300ms), filters table rows client-side (dataset is small enough not to need server-side search at MVP scale) |
| Table row | Click to expand | Inline expansion (accordion-style), not a modal or navigation — keeps context, avoids losing scroll position |
| Connection status (top bar) | Passive indicator | Green dot = live data connected; amber = reconnecting; red = disconnected (see Error States) |

**Interaction principle:** nothing in this dashboard should open a modal or navigate away. Everything resolves in place. This matches the "quiet, data-first" design principle and keeps the build genuinely simple.

---

## 6. Forms

MVP has exactly one input: the **client search/filter field**. Kept here as a documented pattern so future additions (e.g., a config form, if built later) stay visually consistent.

**Input field standard (applies to any future form too):**
- Label above input, not placeholder-as-label (placeholders disappear on focus and hurt accessibility/usability)
- Focus state: 2px accent-colored outline, not just a color change (contrast/accessibility — see Section 10)
- Validation (for future config forms): inline, below the field, red text + icon, appears on blur not on every keystroke
- Disabled state: reduced opacity (60%) + `not-allowed` cursor, never just grayed out with no cursor change

---

## 7. Loading, Error, and Empty States

Every one of these three states must be explicitly designed — an admin dashboard that just shows a blank white screen on any of these erodes trust immediately.

### 7.1 Loading
- **Initial load:** skeleton placeholders matching the exact shape of stat cards, chart, and table (not a generic spinner) — reduces layout shift and signals "this is what's coming"
- **Subsequent polling updates:** no loading state at all — data updates silently in place; a full-screen spinner on every 3-5s poll would be distracting and wrong

### 7.2 Error
- **Dashboard can't reach `/r8gate/metrics`:** top bar connection dot turns red, a dismissible banner appears: *"Live data unavailable — showing last known values from [timestamp]."* Stale data stays visible (grayed 80% opacity) rather than disappearing — partial information beats no information for an operator mid-investigation.
- **Total failure on first load (never connected):** the three sections show a centered message per section: *"Unable to load data. Retrying in 5s…"* with a manual "Retry now" text link.

### 7.3 Empty
- **No traffic yet (fresh install, zero requests recorded):** Section A shows zeros (not hidden), Section B shows a flat empty-state chart with a centered caption: *"No traffic yet — make a request to a protected route to see live data."* Section C table shows a single centered row: *"No clients have made requests yet."* This turns "nothing is broken, there's just no data" into an obviously different state from an error — important since both would otherwise look like a blank screen.

---

## 8. Responsive Behavior

The dashboard is primarily used on desktop (this is an engineer's tool, opened during development/monitoring), but should degrade gracefully:

| Breakpoint | Layout |
|---|---|
| **Desktop (≥1024px)** | Full layout as designed — 4-card row, full-width chart, full table |
| **Tablet (768–1023px)** | Stat cards wrap to 2×2 grid; chart and table remain full-width, stacked |
| **Mobile (<768px)** | Stat cards stack to a single column; chart remains but with fewer x-axis labels shown; table becomes a stacked card list per client instead of a horizontal table (each "row" becomes a small card with label:value pairs) — standard responsive-table pattern, avoids horizontal scrolling |

Mobile is a "should work, not the priority" tier — explicitly noting that here avoids over-investing design time in a use case (checking a rate-limiter dashboard from a phone) that isn't the primary journey.

---

## 9. Typography

| Use | Font | Notes |
|---|---|---|
| **UI text (labels, body, nav)** | **Inter** | Excellent legibility at small sizes, the de facto standard for modern technical dashboards, free/open |
| **Numeric data (stat cards, table numbers, chart axis)** | **Inter with `font-variant-numeric: tabular-nums`**, or **JetBrains Mono** for emphasis numbers | Tabular numbers prevent digits from shifting width as they update live — important since numbers change every few seconds |
| **Code/config snippets (if shown anywhere, e.g. API key display)** | **JetBrains Mono** | Standard monospace choice for technical audiences, pairs well with Inter |

**Type scale:**
| Token | Size | Weight | Use |
|---|---|---|---|
| `display` | 32px | 600 | Stat card primary numbers |
| `heading` | 20px | 600 | Section titles |
| `body` | 14px | 400 | Table content, general text |
| `label` | 12px | 500, uppercase, letter-spacing 0.04em | Stat card labels, table headers |
| `caption` | 12px | 400 | Timestamps, helper text |

---

## 10. Color & Theme

**Default theme: dark mode.** Reference point: this is a tool an engineer keeps open in a corner monitor/tab while working — dark mode is both the aesthetic convention for this category of tool (Vercel, Grafana, Upstash all default dark) and reduces eye strain for that usage pattern. Light mode is a nice-to-have toggle, not required for MVP.

### Palette

| Token | Hex | Use |
|---|---|---|
| `bg-base` | `#0B0E14` | Page background |
| `bg-surface` | `#141821` | Card/table backgrounds |
| `bg-surface-hover` | `#1B202C` | Hover state on interactive rows |
| `border` | `#232836` | Card borders, table dividers |
| `text-primary` | `#E6E9EF` | Primary text |
| `text-secondary` | `#8A93A6` | Labels, captions, secondary info |
| `accent` | `#5B8DEF` | Primary brand accent — focus rings, active states, links |
| `success / allowed` | `#3DD68C` | Allowed-request data series, healthy status dot |
| `warning / degraded` | `#F5B94D` | Degraded/reconnecting states |
| `danger / denied` | `#F0567C` | Denied-request data series, error states, error banners |

**Color usage rule (ties back to Design Principle #3):** `success`, `warning`, and `danger` colors are used **only** for their semantic meaning (allowed/denied/degraded) — never decoratively elsewhere in the UI. This is what makes the red genuinely mean something the instant an operator sees it.

**Contrast:** all text/background pairings above meet WCAG AA contrast minimums (4.5:1 for body text, 3:1 for large text) — verify with a contrast checker when implementing, especially `text-secondary` on `bg-surface`.

---

## 11. Spacing System

8px base unit, standard scale — avoids arbitrary pixel values and keeps the layout visually consistent with minimal decisions per component.

| Token | Value | Use |
|---|---|---|
| `space-1` | 4px | Icon-to-text gaps |
| `space-2` | 8px | Tight internal padding |
| `space-3` | 16px | Standard card padding, form field spacing |
| `space-4` | 24px | Section gaps (between Section A/B/C) |
| `space-5` | 32px | Page margin (desktop) |
| `space-6` | 48px | Large top-level page padding |

**Border radius:** `8px` on cards and inputs, `4px` on small elements (badges, buttons) — soft but not rounded-pill, matching the "technical, not consumer" principle.

---

## 12. Accessibility

- **Color is never the only signal.** The connection-status dot pairs color with text ("Live" / "Reconnecting" / "Disconnected"), not color alone. Allowed/Denied in the chart legend include text labels, not just colored lines.
- **Keyboard navigation:** search input, time-range toggle, and table sort headers are all reachable and operable via keyboard (Tab, Enter/Space); table row expansion is triggerable via Enter when a row is focused.
- **Focus states:** every interactive element has a visible focus ring (`2px solid accent`, per Section 6) — never `outline: none` without a replacement.
- **Live-updating regions:** the stat cards and chart use `aria-live="polite"` regions so screen readers announce changes without interrupting the user, but not on every 3-5s poll tick — throttled to avoid announcement spam (e.g., only announce if a value changed by a meaningful margin, or on a slower interval than the visual update).
- **Semantic HTML:** table uses actual `<table>` markup (not divs) so screen readers get correct row/column navigation; headings use real `<h1>`–`<h3>` hierarchy, not styled divs.
- **Minimum touch targets:** any interactive element (sort headers, toggle buttons) has at least a 32px hit area, even though desktop is the primary use case.

---

## 13. Visual Reference — What the Screen Should Look Like

```
┌────────────────────────────────────────────────────────────────┐
│  ● R8Gate            ● Live            env: local          ⚙   │  ← top bar, bg-base
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐  │
│  │ TOTAL REQ   │ │ ALLOWED     │ │ DENIED      │ │ ACTIVE      │  │
│  │ 12,480      │ │ 11,932 (96%)│ │ 548 (4%)    │ │ CLIENTS     │  │
│  │             │ │  (success)  │ │  (danger)   │ │ 23          │  │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘  │
│                                                     ↑ Section A  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Traffic (last 15 min)          [5m] [15m] [60m]          │  │
│  │                                                             │  │
│  │     ╱‾‾╲___╱‾╲__      ← allowed (success line)             │  │
│  │   ⎽⎽⎽⎽⎽⎽⎽⎽⎽⎽⎽⎽▁▁▂▃  ← denied (danger, mostly flat/low)   │  │
│  │                                                             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                     ↑ Section B  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  🔍 Search clients...                                      │  │
│  │  ┌──────────┬───────┬────────┬─────────┬──────────┬─────┐ │  │
│  │  │ Client    │ Tier  │ Total  │ Allowed │ Denied ▾ │ Last│ │  │
│  │  ├──────────┼───────┼────────┼─────────┼──────────┼─────┤ │  │
│  │  │ acme-api  │ pro   │ 4,201  │ 4,100   │ 101 (2%) │ 2s  │ │  │
│  │  │ scrapr-01 │ free  │ 890    │ 612     │ 278 (31%)│ 1s  │ │  │  ← danger-colored row accent
│  │  │ dev-key-3 │ free  │ 55     │ 55      │ 0        │ 4m  │ │  │
│  │  └──────────┴───────┴────────┴─────────┴──────────┴─────┘ │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                     ↑ Section C  │
└────────────────────────────────────────────────────────────────┘
```

**General feel:** dark background, generous spacing, restrained color (accent blue + semantic green/red only), tabular numbers that don't jitter, one screen, no clutter. If it starts to look like it needs a sidebar or a second page, that's a signal scope has crept past what MVP needs — a good gut-check to reapply while building.

---

## 14. Implementation Notes (so the build doesn't get lost)

- Recommended component library base: **Tailwind CSS** for styling (matches the spacing/color token system above directly as config values) + a lightweight chart library (**Recharts**, already available per the artifact tooling, or **Chart.js**) for Section B.
- Define the palette and type scale as **Tailwind theme config** (or CSS variables) once, at the start of frontend work — every component pulls from those tokens, nothing hardcodes a hex value or px size inline. This is what keeps "modern and simple" achievable without a design system team.
- Build order matches Section 4: static layout with mock data first → wire up polling/live data second → states (loading/error/empty) third → responsive pass last. Trying to build all of it simultaneously is the most common way this kind of dashboard scope-creeps or stalls.