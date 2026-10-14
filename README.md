# season-audit

A Claude Code skill that runs a full retrospective on a product release season: what was committed, what shipped, what slipped, and why.

## Problem

At the end of a planning season (quarter, "summer"/"fall" cycle, sprint-of-sprints, whatever your team calls it), someone has to reconcile what was promised against what actually shipped, across a planning deck, an issue tracker, and a code host -- then write an honest recap for leadership. Done by hand, this means hours of cross-referencing slides, tickets, and merge requests, and it's easy to misattribute an item to the wrong team or guess at why something slipped.

## What it does

1. Collects season inputs (pillar name, season name, date range, planning deck link, issue-tracker project/label, dev teams + repo paths) through an upfront intake step -- it never guesses these.
2. Reads planned commitments from the planning deck (Google Slides) and discovery items from the issue tracker (Jira), and requires both to agree before counting an item as an in-scope commitment.
3. Cross-references merge requests and pipeline/release evidence in the code host (GitLab) to confirm what actually shipped, rather than trusting ticket status alone.
4. Optionally pulls adoption/scope-churn metrics from a BI dashboard.
5. Classifies every slipped item into a fixed taxonomy (Blocked / Strategic / Deprioritized / Rejected) instead of free-form guessing.
6. Renders a self-contained HTML recap (hero, KPI bar, commitment/delivery cards, scope-change tables, slippage table, timeline, narrative) plus a ready-to-paste narrative summary.

## How it works (agent flow)

```
Intake (pillar, season, deck link, project key, teams)
   -> read planning deck commitments
   -> query issue tracker for season-labeled + next-season-labeled items
   -> cross-check: deck AND tracker must both show the item, else exclude
   -> verify delivery via merge requests / release evidence per team repo
   -> optional: pull BI dashboard metrics for scope churn + completion rate
   -> classify slippage (Blocked / Strategic / Deprioritized / Rejected)
   -> render HTML recap + narrative summary
   -> present to user; optionally publish a shareable link
```

## Guardrails

- **Always asks first.** The intake step is mandatory before any data pulls -- pillar, season, dates, and sources are never assumed.
- **Two-source rule.** An item only counts as an in-season commitment if it appears in both the planning deck and the issue tracker (with the season label and correct team). An item in the deck alone is not enough.
- **Team-ownership check.** Every item's "owning team" field is checked explicitly; items from other teams that merely appear in a shared deck are excluded.
- **Never guesses a slippage reason.** Slippage is only ever classified from explicit evidence (a formally rejected delivery epic, a documented external blocker, etc.) -- never defaulted to "Rejected" just because something didn't ship.
- **Token/auth failures stop the run.** If any API call fails auth, the skill stops and asks rather than working around it silently.
- **Never overwrites existing output** without confirming with the user first.
- **BI dashboard step is optional** and skipped cleanly if no dashboard is provided -- it never blocks the rest of the audit.

## Configure

- Everything season-specific (pillar name, labels, project key, team/repo paths, output directory) is collected via the intake step in `SKILL.md` -- see `references/intake-template.md` for the exact fields.
- HTML structure, CSS variables, and component patterns live in `references/html-template-guide.md` -- swap the example color palette for your own brand.
- `references/workflow-notes.md` has accumulated lessons from running real audits (evidence chains, common dashboard load failures, slippage classification pitfalls) -- read it before running one for a new pillar.

## Usage

```
/season-audit
```
