# Season Audit — Workflow Notes & Lessons Learned

This file captures real decisions, pitfalls, and patterns discovered while running the first season audit for a product pillar. Reference these when running future audits.

---

## Lesson 1: Discovery-Board View IDs != Jira Issue IDs

Jira Polaris-style board URLs like `view/5359977` are UI view identifiers, NOT Jira issue keys. You cannot look up `view/5359977` as an issue. Instead search by:
- `project = PROJ AND labels = "Summer2026"`
- `project = PROJ AND "Product Team[Dropdown]" = "<pillar>"`
- `project = PROJ AND assignee in (...) ORDER BY status`

## Lesson 2: Delivery Evidence Chain

Don't trust the discovery-item status alone. Always cross-verify:
1. Discovery item says Done -> find the linked delivery epic
2. Delivery epic -> look for Jira stories with "Approved for all deployment" status
3. Stories -> check linked MR in the code host (MR status: Merged)
4. For feature flags: search the repo for MRs containing "RELEASE" + the pillar keyword

Example chain (illustrative): discovery item (Done) -> delivery epic DELIVERY-1000 -> story PROJ-2000 "Release feature flag X" -> Status: Done

## Lesson 3: Multi-Repo Scope

For some pillars, delivery spans TWO separate codebases:
- **App team** -> the main product monorepo
- **Data team** -> a separate data-pipeline repo

Always ask the user which teams contribute. Don't assume one repo.

For other pillars, find the right repo paths by:
1. `gitlab-cli projects list --search "<pillar keyword>"`
2. Or: find a known delivery epic -> its Jira stories -> linked MR URL -> parse the project path

## Lesson 4: Dashboard "waiting on data pipeline" errors

Many chart cells show a loading error when the underlying data pipeline is stale. Don't block on them. Capture:
- Scope Churn cells (usually load fine if warehouse-backed)
- Progress by Sub-Focus bar chart (usually loads)
- Season Ideas Collection table (loads page by page -- scroll and screenshot each page)
- Status Transitions throughput chart (loads)

KPI cells (Completion Rate, Not Started, In-Progress, Delivered) often fail. Compute them manually from the Season Ideas Collection table.

## Lesson 5: Dashboard Auth

A dedicated BI-dashboard CLI's `auth` command often opens a headless browser window that requires signing in. If the user already has the dashboard open in their main browser, use a browser-automation CLI instead:
- Preflight the browser bridge -> find the tab -> screenshot

Much faster than re-authenticating through the dashboard CLI. Always check if the user is already logged in before triggering any auth flow.

## Lesson 6: Scope Churn Baseline

High scope churn (>1 change per idea) is normal in product seasons. Don't flag it as a problem unless it caused missed commitments. Context:
- Churn from the previous season's process delays = expected, no concern
- Churn from external blockers (API prerequisite, data gaps) = document clearly
- Churn from capacity trade-offs = honest deprioritization, acceptable

## Lesson 7: Slippage Classification

Use this strict taxonomy (not free-form):
- **Blocked** -- External dependency. Not a team choice. Requires infra/data/API from another team.
- **Strategic** -- Intentional bundling. PM decided to wait for a related initiative.
- **Deprioritized** -- Capacity trade-off. Team had more urgent items.
- **Rejected** -- Delivery epic formally rejected.

This classification shows up in the HTML recap table and affects the narrative tone.

## Lesson 8: Narrative Tone

Manager-grade narrative = direct, honest, scannable. Rules:
- Lead with the delivery rate number
- Name the flagship win first
- Acknowledge misses with root cause (no spin)
- Use precise names when they add clarity (feature flag name, item key) -- a PM audience can handle it
- Don't pad to fill space; 4 tight paragraphs > 8 fluffy ones

## Lesson 9: Planning Deck -- What to Look For

Planning decks are typically ~50-60 slides. Most are context-setting. Focus on:
- Slides with bullet lists of commitments per stream/team
- Slides with launch-level labels (L2/L3/L4) -- these are launch levels
- Slides with investment ratios (60/30/10)
- Slides with explicit milestones (dates, major company events)
- Ignore slides that are headers, team intros, or background context

Use your Workspace CLI's slide-reading command and filter by these patterns.

## Lesson 10: Season Dates

One example pillar used:
- Summer = Feb 1 - May 31
- Fall = Jun 1 - Sep 30 (assumed)
- Winter = Oct 1 - Jan 31 (assumed)

Other pillars may use different dates. Always confirm with the user in intake.

## Lesson 11: Team Ownership on Discovery Items -- Always Check the "Product Team" Field

The planning deck can include items from multiple teams presenting side by side. Just because an item appears in the deck does NOT mean it belongs to this pillar.

Before including any discovery item, verify its "Product Team[Dropdown]" field matches the pillar being audited. Example of the mistake: an item appeared in this pillar's planning deck context but was actually owned by a different team, confirmed by the item's team field. It was included in error and had to be removed from all 4 locations in the recap.

Rule: Deck item -> check discovery item -> check team field -> only then include.

## Lesson 12: Four Locations to Check When Removing a Misattributed Item

When an item needs to be removed from the recap, check all four places:
1. The "What We Committed To" section (planning deck cards)
2. The "What We Delivered" section (delivery status cards)
3. The "Removed Scope" list in Scope Changes section
4. The "Slipped to Next Season" table
5. The narrative paragraph (update counts: "three items were blocked" -> "two items")

Missing any one of these leaves inconsistencies that are hard to spot later.

## Lesson 13: Badge and Chip Classifications Matter -- Never Guess

From past recap work: an item was marked "Rejected" (red chip) when it was actually "Blocked" (blue chip) -- it slipped because of an external dependency on another team, not a product decision. This misleads leadership into thinking something was cancelled.

Rule: Only use Rejected when the delivery epic is formally rejected. For external blockers carrying to next season, use Blocked. For capacity trade-offs, use Deprioritized. Never default to Rejected because something didn't ship.

## Lesson 14: Shareable URL -- Script-Runtime + Drive Sync Pattern

For every recap, offer to set up a permanent shareable URL that auto-updates:

Architecture:
```
Local HTML file -> drive-sync.py (MD5 poll, 3s) -> cloud file store -> script runtime doGet() -> Stakeholder URL
```

Key facts:
- The script runtime reads from cloud storage on every request -- it always serves the latest file
- drive-sync.py is required because the script runtime can't read local files; cloud storage is the mandatory middle layer
- The shareable URL format depends on your workspace's scripting platform -- it typically includes a stable per-deployment ID
- The deployment ID is permanent -- it doesn't change when you redeploy updated versions
- Token refresh: whatever your auth CLI's "ensure" command is, updates your local workspace credentials file
- drive-sync.py reads the token from that local credentials file, keyed by account email
- Deployment often must be done manually via UI if the scripting platform's REST API requires an OAuth scope not covered by your normal credentials

Treat the file ID, deployment ID, and shareable URL as per-run secrets to note locally -- don't hardcode a specific one into this file.

## Lesson 15: Labels Differ by Team -- Always Ask

Don't assume one season label is the only one that matters. Teams also track:
- Focus-area labels (e.g., specific initiative tags)
- Annual-event labels (items targeted at a major company event)
- Cross-pillar dependency labels
- Internal prioritization tags

Asking "what labels does your team actively track this season?" surfaces these and ensures the discovery-item queries don't miss relevant scope.

---

## Tool Map

| Step | Tool | Notes |
|---|---|---|
| Read planning deck | Workspace CLI slides command | Needs Google auth |
| Query discovery-item status | Jira CLI search | Filter by project + season label |
| Verify delivery via MRs | GitLab CLI `mr list` | One command per team repo |
| Check feature-flag release stories | Jira CLI search / GitLab CLI repo search | Look for "RELEASE" in MR title |
| Pull adoption metrics | Browser-automation CLI (preferred) or BI-dashboard CLI | Ask user for permission first |
| Build HTML recap | Local file write | Save to a local output directory |
| Serve HTML | `python3 -m http.server <port>` | Pick any free local port |
| Screenshot recap | Browser-automation CLI tab/observe command | After server is running |
