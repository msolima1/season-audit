# Season Audit Skill

A full-cycle season retrospective: reads planning commitments, audits delivery across Jira discovery items + GitLab, pulls adoption metrics from a BI dashboard, identifies slippage, and produces a polished HTML recap + a ready-to-paste narrative summary for any product pillar.

## Invocation

User types `/season-audit` or asks to do a "season audit" or "season recap" or "season retrospective".

---

## Step 0 — Intake (Always Ask First)

Before doing any work, collect the following from the user. Use `AskUserQuestion` for multi-choice fields; ask plain text questions for free-form inputs. Do NOT proceed to Step 1 until all required (*) fields are collected.

### Required Inputs

| Field | What to Ask | Notes |
|---|---|---|
| **Pillar name** * | "What product pillar is this audit for? (e.g., Maintenance, Safety, Fleet Tracking)" | Used in all titles, labels, filters |
| **Season name** * | "Which season? (e.g., Summer 2026, Fall 2026, Winter 2027)" | Maps to Jira label like `Summer2026` |
| **Season dates** * | "What are the season start and end dates? (e.g., Feb 1 – May 31, 2026)" | For Maintenance: Summer = Feb 1 – May 31, Fall = Jun 1 – Sep 30 |
| **Season planning deck** * | "Share the Google Slides URL for the season planning deck — the deck presented at the start of the season with per-stream commitment lists, L-levels, and investment ratios. Example format: `https://docs.google.com/presentation/d/<SLIDES_ID>/edit`. For Maintenance, the Summer 2026 deck is the reference example — ask the user for the link for their season." | Critical source #1. Extract commitments from here first, then cross-check against idea. |
| **Idea board** * | "Share the Jira idea board URL or project key for this pillar. Example: PROJ, or the full board link like https://<your-jira-host>/jira/polaris/projects/PROJ/ideas/view/<VIEW_ID>. The project key is what matters for JQL queries." | Critical source #2. Used to verify what actually entered season scope. |
| **Season Jira labels** * | "What Jira labels does your team use to tag ideas for this season and track slippage? Include: (a) the season label (e.g., Summer2026), (b) the next-season label for slippage (e.g., Fall2026), and (c) any other labels your team actively tracks (e.g., initiative tags, focus area labels, the annual company event items). These differ by pillar and team." | Teams use different label conventions. Don't assume — always ask. |
| **Dev teams + GitLab paths** * | "List each dev team contributing this season with their GitLab project path. Example: App Team → team-a/monorepo, Data Team → team-b/data-pipeline. Each team gets a separate GitLab audit." | Multiple teams allowed. |
| **Stream names** * | "What are the major stream or track names for this pillar this season? (e.g., Work Order Management, Insights, Platform) — used to group commitments in the recap." | Maps to the planning deck structure. |
| **BI dashboard** | "Is a BI dashboard available for this pillar? Share the dashboard ID or URL (e.g., https://superset.yourcompany.com/superset/dashboard/<ID>). This is the standard dashboard that tracks completion rate, scope churn, and sub-focus progress by pillar. If you're not sure of the ID, check with the data team." | Optional but strongly recommended. Provides scope churn, completion %, and the Removed Scope justification table. |
| **Output directory** | "Where should I save the HTML recap? (e.g., ~/projects/my-pillar-summer-2026-recap/)" | Defaults to `~/projects/<pillar-name>-<season>-recap/` |
| **Pre-read format** | "Any specific format requirements for the narrative summary? (e.g., 'Closing Summer doc asks for: headline, delivery rate, impact, descoped items')" | Optional; affects narrative structure. |

### Source Priority and Cross-Check Rules

**There are two primary sources for season commitments. Both must agree before an item is included.**

| Source | What it tells you | What it does NOT tell you |
|---|---|---|
| **Planning deck** | What was *intended* at planning time | Whether it actually entered discovery scope this season |
| **idea board** | What the team *committed to* (has the season label + correct team) | Whether the item shipped |

An item in the deck that was postponed before the season started will not appear on the idea board with the season label. **Do not include it.** When in doubt — ask the user, never assume.

Also verify **team ownership**: the idea's "Product Team[Dropdown]" field must match the pillar being audited. Items from other teams (e.g., another product team) can appear in a shared deck but must be excluded from this pillar's recap.

### Example Intake Dialogue

```
You: /season-audit

Claude: I'll run a full season audit. Let me collect the key inputs — I need both primary
sources before I can start so I can cross-check commitments properly.

1. Which product pillar? (Maintenance, Safety, Fleet Tracking, or other?)
2. Which season and dates? (e.g., Summer 2026, Feb 1 – May 31)
3. Season planning deck URL (Google Slides — the deck from the start of the season
   with commitment lists per stream and L-level tags). What's the link?
4. Idea board URL or project key (e.g., PROJ or the full board link).
5. What Jira labels does your team use? I need:
   a) The season label (e.g., Summer2026)
   b) The next-season slippage label (e.g., Fall2026)
   c) Any other labels your team tracks (initiative tags, focus areas, etc.)
6. Dev teams and their GitLab paths?
7. the data team's BI dashboard — do you have the dashboard ID or URL?
8. Output directory for the HTML recap?

[wait for all answers before proceeding to Step 1]
```

---

## Step 1 — Read the Planning Deck

**Goal:** Extract what was committed at the start of the season.

**Tool:** `workspace-cli` (Google Slides reader) or `browser-cli` if the Slides URL requires auth.

```bash
# Use workspace-cli to read slides
CLI_WS="/path/to/workspace-cli-binary"
SLIDES_ID="<extracted from URL>"
$CLI_WS slides read "$SLIDES_ID"
```

**What to extract:**
- Per-stream commitment items (L2/L3/L4/R&D tags if present)
- Investment ratio if stated (e.g., 60% new / 30% improvements / 10% tech debt)
- Number of developers per stream
- Any explicit launch targets (dates, milestones like the annual company event)
- Any KPIs or success metrics mentioned

**Store as:** `COMMITTED_ITEMS[]` — a structured list per stream.

**Fallback:** If the slides URL is inaccessible, ask the user to paste the key commitment bullets directly.

---

## Step 2 — Audit the Idea Board

**Goal:** Get current status of every idea tagged for this season, plus any with the next-season slippage label.

**Tool:** `jira-cli`

```bash
CLI_JIRA="/path/to/jira-cli-darwin-arm64"

# 2a. Get all ideas for this season
$CLI_JIRA search --jql "project = $DISCOVERY_PROJECT AND labels = \"$SEASON_LABEL\" ORDER BY updated DESC" --max 100

# 2b. Get ideas slipped to next season
$CLI_JIRA search --jql "project = $DISCOVERY_PROJECT AND labels = \"$NEXT_SEASON_LABEL\" ORDER BY updated DESC" --max 50

# 2c. Get ideas by sub-focus area (if pillar uses Sub-Focus field)
$CLI_JIRA search --jql "project = $DISCOVERY_PROJECT AND \"Product Team[Dropdown]\" = \"$PILLAR_NAME\" ORDER BY status ASC" --max 100
```

**What to extract per idea:**
- Key (e.g., PROJ-1596)
- Summary / title
- Status (Done, Measuring, Developing, Solutioning, Discovery, Net New Ideas)
- Sub-Focus area
- Assignee / team
- Launch Level (L2, L3, L4, R&D)
- Last updated date
- Linked delivery epics (look for DELIVERY-* links)

**Status mapping:**
| Idea Status | Delivery meaning |
|---|---|
| 6. Done | Shipped |
| 5. Measuring | Shipped, in observation |
| 4. Developing | In progress, not yet shipped |
| 3. Solutioning | Designed, not built |
| 2. Discovery | Early exploration |
| 1. Net New Ideas | Not started |

**Compute delivery rate:** `(Done + Measuring) / Total in season scope * 100`

---

## Step 3 — GitLab Delivery Audit (Per Team)

**Goal:** Verify delivery by checking MRs and delivery epics linked to discovery items. Find evidence of feature flag releases.

**Tool:** `gitlab-cli`

For each dev team provided by the user, run the following. Use `--project <gitlab-path>` to scope each query.

```bash
CLI_GL="/path/to/gitlab-cli-darwin-arm64"
TEAM_PROJECT="team-a/monorepo"   # substitute per team

# 3a. Search MRs by season keyword or pillar
$CLI_GL mr list --project $TEAM_PROJECT --state merged --search "$PILLAR_NAME" --limit 50

# 3b. Search for feature flag release MRs
$CLI_GL repo search "RELEASE" --ext .cs --project $TEAM_PROJECT --limit 20
$CLI_GL repo search "$PILLAR_UPPER" --project $TEAM_PROJECT --limit 20

# 3c. Look for delivery epics in DELIVERY project
$CLI_JIRA search --jql "project = DELIVERY AND text ~ \"$PILLAR_NAME\" AND \"Epic Link\" is not EMPTY" --max 20

# 3d. Check specific idea-linked stories
# For each idea key, find linked DELIVERY epics or Jira stories
$CLI_JIRA issue <IDEA_KEY>   # check "links" for delivery epic
```

**What to look for:**
- MRs with `[FF]` or `RELEASE` in title → indicates feature flag releases
- Story status "Approved for all deployment" = shipped to prod
- Delivery epic status: Active (in progress), Resolved (done), Rejected (cancelled)
- GitLab pipelines on release branches as corroborating evidence

**Cross-reference:** For each idea marked Done in Jira, find at least one MR or Jira story that confirms prod delivery.

---

## Step 4 — BI Dashboard (Optional)

**Goal:** Pull season progress metrics: completion rate, scope churn, velocity, sub-focus breakdown.

**Skip if:** User didn't provide a dashboard ID.

**Tool:** `browser-cli` (preferred, uses existing logged-in session) or `bi-dashboard-cli`

```bash
CLI_CHROME="/path/to/browser-cli-darwin-arm64"

# 4a. Preflight
$CLI_CHROME bridge preflight --log-level off

# 4b. Find or open the dashboard tab
$CLI_CHROME tab find --url-contains "superset.yourcompany.com/superset/dashboard/$DASHBOARD_ID" --limit 1

# 4c. If tab not open, open it
$CLI_CHROME tab open --url "https://superset.yourcompany.com/superset/dashboard/$DASHBOARD_ID" --active

# 4d. Screenshot each section
# Filter to the target pillar using the Product Team filter first
# Then scroll and screenshot:
#   - Season Countdown / key metrics bar
#   - Status Transitions (Throughput) chart
#   - Progress by Sub-Focus Area chart
#   - Season Ideas Collection table (all pages)
#   - Scope Changes: Added + Removed tables
```

**What to extract:**
- Season countdown / end date
- Completion Rate, Not Started, In-Progress, Delivered counts
- Scope churn: # added, # removed
- Sub-focus breakdown (completion % per area)
- Added scope items with justification
- Removed scope items with justification

**Note:** Some charts may show "Waiting on the Jira data pipeline" if the data pipeline is delayed. Screenshot what loads; note the gap.

---

## Step 5 — Slippage + Scope Volatility Analysis

**Goal:** Categorize what slipped to Fall and why.

Using data from Steps 2–4, classify each removed/slipped item:

| Category | Criteria |
|---|---|
| **Blocked** | External dependency missing (API, data, infra) |
| **Strategic** | Intentionally bundled with another initiative for Fall |
| **Deprioritized** | Capacity trade-off; lower L-level moved out |
| **Rejected** | Delivery epic formally rejected |

Pull justification text from the BI dashboard's "Removed Scope Table" (column: Justification) when available. Fall back to Jira comments or linked epic status.

---

## Step 6 — Build the HTML Season Recap

**Goal:** Generate a polished, self-contained HTML file at the user's output directory.

**Template:** See `references/html-template-guide.md` for the full page structure and CSS variables.

**Page sections (in order):**

1. **Hero** — Pillar name, season name, subtitle, season dates, team names, total ideas, scope churn
2. **Metrics Bar** — 5 KPIs: delivery rate %, discovery items shipped, slipped count, FF releases to prod, net scope churn
3. **What We Committed To** — Per-stream commitment cards with L-level tags (from Step 1)
4. **What We Delivered** — Per-sub-focus pillar cards with progress bars and per-item status dots
5. **Scope Changes** — Two-column: Added (green header, +N) and Removed (red header, -N)
6. **Slipped to Fall** — Table with columns: Idea Key, Item, Sub-Focus, Team, Reason, Type chip
7. **Key Milestones Timeline** — Two-column by stream, chronological dots
8. **Season-Close Narrative** — Dark navy box with the copyable pre-read text + clipboard button

**Example brand colors (swap for your own):**
```css
--navy: #003B5C
--blue: #0072CE
--light-blue: #00A3E0
--orange: #E87722
--green: #22c55e
--yellow: #f59e0b
--red: #ef4444
```

**After generating HTML:**
1. Start a local server: `cd <output-dir> && python3 -m http.server <port>`
2. Open in Chrome via browser-cli: `tab open --url "http://localhost:<port>" --active`
3. Screenshot the hero, delivery section, scope changes, and narrative
4. Show screenshots to user for review

---

## Step 7 — Write the Narrative Summary

**Goal:** Produce the pre-read text ready to paste into the monthly/seasonal pre-read doc.

**Structure (adapt to user's pre-read format if specified):**

```
[Season] closes with approximately [X]% delivery rate across [N] tracked [Pillar] ideas ([dates]).
The season was characterized by [strong/moderate/mixed] execution in [area] and [high/low] volatility
in scope — [N] ideas added and [N] removed mid-season.

Top impact this season: [top 2-3 shipped items and their significance]

Shortfalls to acknowledge: [what missed and why — honest, concise]

Carry-over into [Next Season]: [confirmed items + blockers to resolve]
```

**Tone:** Direct, honest, manager-grade. Acknowledge misses without spin. Name the blockers.

**Output:** Embed in the HTML recap's dark navy "Pre-Read" section with a one-click copy button.

---

## Step 8 — Final Summary to User

Present:
1. Delivery rate and headline number
2. Top 3 wins
3. Top 3 misses with root cause
4. Slippage count and breakdown by type
5. Path to HTML file and local server URL. Mention the shareable URL option if they want to distribute it.
6. Narrative text (inline, not just a link)

---

## Step 9 — Shareable URL (Optional)

The HTML file is the primary deliverable — it can be shared directly as an attachment or served locally. This step is only needed if the user wants a permanent, browser-accessible link they can send to stakeholders without sharing a file.

**How it works:** Upload the HTML to Google Drive → create an Apps Script web app that reads from that Drive file on every request → the URL always serves the latest version. A small background script (`drive-sync.py`) watches the local HTML file and syncs changes to Drive automatically.

**Only set this up if the user asks.** After presenting the HTML in Step 8, offer it as a follow-up:

> "The HTML recap is ready at `<path>`. Want me to also set up a permanent shareable link so you can send it directly to stakeholders without attaching a file?"

If yes, refer to `references/workflow-notes.md` Lesson 14 for the full setup pattern (Drive file, Apps Script `doGet()`, deployment, `drive-sync.py` watcher).

---

## Skill Rules & Constraints

- **Always ask first.** Never start the audit without completing Step 0.
- **Use the right tool per step.** Don't try to do GitLab queries from Jira or vice versa.
- **Chrome CLI requires permission.** If not already permitted in the session, ask before using browser-cli.
- **BI dashboard is optional.** Don't block if the dashboard isn't available; note the gap.
- **Token issues → stop and ask.** If any API (GitLab, Jira, Google) fails auth, stop and ask the user before any workaround.
- **Never delete or overwrite existing HTML** without confirming with the user.
- **One narrative paragraph per topic.** Don't pad. manager-grade = precise and scannable.
- **Confirm scope before heavy work.** If the audit will touch many teams or a large date range, confirm with the user before proceeding.
- **Cross-check all sources before including any item.** An item appearing in the planning deck alone is NOT sufficient to include it in the recap as a season commitment. It must also appear in the idea board (with the season label) or have a linked delivery epic in DELIVERY. Items that were in the deck but postponed before the season started (never entered discovery scope) must be excluded. When sources conflict — e.g., deck says committed but idea has no record — ask the user before including or excluding the item. Never assume; always verify across at least two sources.

---

## Reusability Notes

This skill is pillar-agnostic. Any product pillar can use it:
- Safety pillar → different idea project key, different GitLab paths, different planning deck
- Fleet Tracking → same structure, different data
- The HTML template and narrative format are identical; only the data changes

The intake step (Step 0) is what makes it reusable — it parametrizes everything.
