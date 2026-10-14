# HTML Template Guide — Season Audit Recap

This guide defines the full structure, CSS variables, and component patterns for the Season Audit HTML recap page. Use it as the blueprint when building the output in Step 6.

## File

Single self-contained `index.html`. No external JS dependencies. Google Fonts loaded via CDN. Served via `python3 -m http.server`.

## Page Structure (in order)

```
<hero>               ← dark navy gradient, season title + meta
<metrics-bar>        ← 5 KPI cells, white bar below hero
<section: Committed> ← planning deck commitments, 2-col grid
<section: Delivered> ← 6 pillar cards, 3-col grid (white bg)
<section: Scope>     ← 2-col: added (green) / removed (red)
<section: Slipped>   ← table with chip badges
<section: Timeline>  ← 2-col by stream, dot timeline
<section: Narrative> ← navy box, copyable pre-read text
<footer>             ← sources + generated date
```

## CSS Variables

```css
:root {
  --navy: #003B5C;
  --blue: #0072CE;
  --light-blue: #00A3E0;
  --orange: #E87722;
  --green: #22c55e;
  --yellow: #f59e0b;
  --red: #ef4444;
  --slate: #64748b;
  --bg: #f8fafc;
  --card: #ffffff;
  --border: #e2e8f0;
  --text: #1e293b;
  --muted: #64748b;
}
```

## Fonts

```html
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700;800&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet">
```

- Body: `Inter`
- Monospace keys/dates/codes: `JetBrains Mono`

## Key Components

### Status Dot Colors
| Status | Color |
|---|---|
| Done | `--green` |
| Measuring | `--blue` |
| Developing | `--yellow` |
| Solutioning | `--orange` |
| Slipped/Rejected | `--red` |
| Net New | `#94a3b8` (grey) |

### Slip Type Chips
| Type | CSS class | Colors |
|---|---|---|
| Strategic | `chip-fall` | `#ede9fe` / `#6d28d9` (purple) |
| Blocked | `chip-blocked` | `#fee2e2` / `#991b1b` (red) |
| Deprioritized | `chip-deprioritized` | `#fef3c7` / `#92400e` (amber) |
| Rejected | `chip-blocked` | same as Blocked |

### L-Level Tags
| Level | CSS class | Colors |
|---|---|---|
| L2 | `tag l2` | `#dbeafe` / `#1d4ed8` (blue) |
| L3 | `tag l3` | `#ede9fe` / `#6d28d9` (purple) |
| L4 | `tag l4` | `#dcfce7` / `#166534` (green) |
| R&D | `tag rd` | `#fef3c7` / `#92400e` (amber) |

### Progress Bars
```html
<div class="progress-bar">
  <div class="progress-fill" style="width: 75%;"></div>
</div>
```
- Full green: `background: var(--green)`
- Partial/yellow: add class `partial` → `background: var(--yellow)`
- Low/orange: add class `low` → `background: var(--orange)`

### Timeline Dots
```html
<div class="tl-item">           <!-- green dot (done) -->
<div class="tl-item measuring"> <!-- blue dot (measuring/partial) -->
```

### Scope Churn Cards
- Added header: `background: #f0fdf4; color: #166534;` with `+N` in large mono font
- Removed header: `background: #fef2f2; color: #991b1b;` with `-N` in large mono font

## Parameterization Guide

When generating for a new pillar, replace these values:

| Placeholder | Replace With |
|---|---|
| `[PILLAR_NAME]` | e.g., "Safety", "Fleet Tracking" |
| `[SEASON]` | e.g., "Fall 2026" |
| `[SEASON_DATES]` | e.g., "Jun 1 – Sep 30, 2026" |
| `[TEAM_1]`, `[TEAM_2]` | Dev team names from user input |
| `[N_IDEAS]` | Total ideas tracked from Superset |
| `[SCOPE_ADDED]` | From Superset scope churn |
| `[SCOPE_REMOVED]` | From Superset scope churn |
| `[DELIVERY_RATE]` | Computed: (Done + Measuring) / total |
| `[N_SHIPPED]` | Count of Done + Measuring |
| `[N_SLIPPED]` | Count of slipped to Fall |
| `[N_FF_RELEASES]` | Feature flags released to prod |
| `[STREAM_1]`, `[STREAM_2]` | Stream names from user input |

## Delivery Pillar Cards

Each pillar card = one sub-focus area. Typically 4-6 cards for a pillar.
For each card, set:
- `progress-bar` width = (Done + Measuring) / total in that area * 100
- Label: "Excellent" (>85%), "Strong" (70-85%), "Partial" (40-69%), "Missed" (<40%)
- List each item with the correct status dot and badge

## Narrative Section

```html
<div class="narrative-box">
  <div class="narrative-label">[DATE] — Monthly Pre-Read: Closing [Season]</div>
  <h2>[Pillar] Pillar — [Season] Season Close</h2>
  <div class="narrative-text" id="narrative-content">
    <p><strong>[Season] closes with approximately [X]% delivery rate</strong>
    across [N] tracked [Pillar] ideas ([dates])...</p>
    <p><strong>Top impact this season:</strong> ...</p>
    <p><strong>Shortfalls to acknowledge:</strong> ...</p>
    <p><strong>Carry-over into [Next Season]:</strong> ...</p>
  </div>
  <button class="copy-btn" onclick="copyNarrative()">Copy narrative to clipboard</button>
</div>
```

## Footer Sources Line

```html
Sources: Jira Discovery Board ([PROJECT_KEY]), GitLab ([TEAM_PATHS]), BI Dashboard [DASHBOARD_ID], Planning Deck
```

## Local Server

```bash
cd <output-dir>
python3 -m http.server <PORT>
# Suggested ports: 8765 (season audit default), 8766, 8767 for multiple seasons
```
