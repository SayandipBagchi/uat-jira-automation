# Cards Onboarding UAT — Jira Bug Filing Automation
> ### What this repository is
>
> A method write-up of an agent-assisted UAT triage cycle, not a tool you install.
> There is no package here: the workflow runs through an AI browser-automation
> agent against a client's Jira, and both the findings and the tracker are
> confidential.
>
> `uat_bug_report.js` is the structured input that workflow consumed. It is not a
> file in this repository — do not look for it here; its shape is documented below
> so you can build the equivalent.
>
> The numbers — 29 raw findings resolving to 18 matched, 11 new and 1 comment-only
> — come from one real cycle.
>
> What you can take from it: the dedup-before-create ordering, the classification
> schema, and the human review gate, which is the transferable part. The Jira REST
> patterns underneath it are published as runnable code in
> [spa-automation-toolkit](https://github.com/sayandip1987/spa-automation-toolkit)
> (`examples/jira-bulk-edit.js`).


AI-assisted UAT bug triage, deduplication, and batch Jira ticket creation for the client's UK cards onboarding web application (`cards-onboarding-web`). Bridges a structured UAT bug report (`uat_bug_report.js`) with Jira via an AI browser-automation agent (Claude Cowork / Claude Code) to deduplicate findings, batch-create new tickets, and enrich existing ones — all in a single browser session.

> Build effort: 2–3 hrs per session, ongoing across releases
> Internal-team users: ~25–30 — QA, PMs, eng leads
> Validated across 3 sessions creating 12 net-new Jira tickets across priorities P1–P4, plus a P2 security finding discovered during testing

## Project Outcome

- **29 raw UAT findings** processed → **18 matched** existing Jira tickets (no duplicates created), **12 net-new tickets** created with consistent shape, **1 grammar comment** added to a pre-existing ticket
- **Deduplication-first workflow** filtered against existing tickets carrying labels `R5` + `Onboarding`, classifying each finding as: create new / enrich existing / add comment / skip — eliminates duplicate-ticket debt across UAT cycles
- **Batch ticket creation in a single browser session** using a "Create another" pattern that retains Priority, Parent epic, and Labels between submissions — file all P1s together, change priority once per group, file all P4s, etc.
- **One P2 security finding discovered during testing** — session return URL bypassed mobile authentication when pasted manually in a new tab after expiry. Filed independently of the original bug report
- **Consistent ticket shape on every new bug** — Steps to Reproduce / Actual / Expected / Environment / Impact template applied uniformly so developer triage is faster and reporting/filtering are reliable
- **Three reusable skills** packaged in version-controlled `SKILL.md` files: `jira-creation`, `jira-sync-up`, `web-testing` — each with explicit trigger phrases, behaviours, and coordinate references
- **Six documented automation errors with definitive fixes** captured in an Error Compendium so future sessions don't repeat known failure modes
- **Cross-session resilience** — every resumed session begins with a screenshot to verify ground-truth modal state before any action

## Workflow Architecture

```
   uat_bug_report.js (29 findings)
            │
            ▼
   ┌────────────────────────────────────────┐
   │  Phase 0: jira-sync-up (deduplication) │  Filter Jira by labels
   │  JQL: project = PROJ AND labels = "R5" │  R5 + Onboarding,
   │       AND labels = "Onboarding"        │  classify each finding
   └────────────────────┬───────────────────┘
                        │
        ┌───────────────┼───────────────┬────────────────┐
        ▼               ▼               ▼                ▼
  EXACT MATCH    PARTIAL MATCH      NO MATCH       COMMENT ONLY
  (skip / enrich) (create + link)   (create new)   (add comment)
        │               │               │                │
        └───────────────┴───────────────┘                │
                        │                                │
                        ▼                                │
   ┌────────────────────────────────────────┐            │
   │  Phase 1: One-time setup (first ticket)│            │
   │  Open expanded modal · Set Priority,   │            │
   │  Parent (PROJ-404), Labels (R5+Onboard)│            │
   │  Tick "Create another" ◄ CRITICAL      │            │
   └────────────────────┬───────────────────┘            │
                        │                                │
                        ▼                                │
   ┌────────────────────────────────────────┐            │
   │  Phase 2: Subsequent tickets in group  │            │
   │  Form retains Priority + Parent +      │            │
   │  Labels — only Summary + Description   │            │
   │  change. Verify toast counter on Create│            │
   └────────────────────┬───────────────────┘            │
                        │                                │
                        ▼                                │
   ┌────────────────────────────────────────┐            │
   │  Phase 3: Change Priority between      │            │
   │  groups (P1 → P4 → P3 → P2 sequence)   │            │
   └────────────────────┬───────────────────┘            │
                        │                                │
                        ▼                                ▼
   ┌────────────────────────────────────────────────────────┐
   │  Phase 4: Post-creation enrichment                     │
   │  Fill richer description + screenshots on              │
   │  pre-existing tickets identified in deduplication      │
   │  (PROJ-1404, PROJ-1410, PROJ-1313, PROJ-1394 comment)  │
   └────────────────────────────────────────────────────────┘
```

## Tech Stack

| Layer | Technology | Role |
|---|---|---|
| Browser-automation agent | AI agent with Chrome connector (Cowork / Claude Code) | Drives the SPA, captures screenshots, runs JS, posts to Jira |
| Browser | Chrome with MCP extension | Logged-in session — inherits user's auth + cookies |
| Target SPA | Atlassian Jira (Atlaskit components) | Bug board, Create modal, comments |
| UAT target | client cards onboarding web app | UAT environment for evidence capture |
| Source of truth | `uat_bug_report.js` (29 findings) | `{ id, title, severity, page, steps, actual, expected, impact }` |
| Skills | three `SKILL.md` files (YAML front-matter) | `jira-creation`, `jira-sync-up`, `web-testing` |
| Workflow contract | `CLAUDE.md` | Project memory that survives across context resets |

## How It Was Built — Phase-by-Phase

**Phase 0 — Deduplication (always first).** Run the `jira-sync-up` skill before creating anything. JQL filter: `project = PROJ AND labels = "R5" AND labels = "Onboarding" ORDER BY created DESC`. For each finding in `uat_bug_report.js`, classify as exact match (skip / enrich), partial match (create + add `Related to [PROJ-XXXX]`), no match (create new), or comment-only (add comment to most relevant existing ticket). Outcome on this cycle: 18 matched, 11 new, 1 comment.

**Phase 1 — One-time session setup (first ticket only).** Navigate to Bug Board backlog → click `+ Create` → click ⤢ to open the expanded modal (the compact dialog doesn't expose Priority / Parent / Labels). Fill Summary → Description (uses the standard template). Scroll ~4 ticks down → set Priority. Set Parent (`PROJ-404` — "Cards Application Onboarding - Web"). Scroll ~2 more ticks → add Labels `R5` + `Onboarding`. Scroll to footer → **tick "Create another"** ← single highest-risk action: missing it forces full re-setup of Priority + Parent + Labels on every subsequent ticket. Click Create → verify toast shows "1 work item created".

**Phase 2 — Subsequent tickets in same priority group.** Form resets Summary + Description but retains Priority, Parent, Labels. For each subsequent ticket: scroll up ~10 ticks → Summary → Description → scroll down ~10 ticks → Create. Verify toast counter increments by exactly 1 (`"2 work items created"`, `"3 work items created"`, …). If counter doesn't increment, ticket was not created — scroll up and check for red-bordered fields.

**Phase 3 — Changing Priority between groups.** After the last ticket in a group, scroll down ~4 ticks (Priority dropdown visible), select new priority, scroll back up, continue. **Sequence used on this cycle: P1 → P4 → P3 → P2.** Non-linear order is intentional: severity grouping by failure mode — all P1 validation/input bugs together, then the lone P4 warning, then P3 display bugs, then the P2 security finding.

**Phase 4 — Post-creation enrichment of pre-existing tickets.** For each ticket identified in deduplication (`PROJ-1404`, `PROJ-1410`, `PROJ-1313`): open in new tab → click into Description → replace sparse description with the full template → attach relevant UAT screenshots → save. For the comment-only case (`PROJ-1394` grammar fix): scroll to Activity / Comments → "Add a comment" → type the minor grammar correction → save. Don't edit the ticket summary or description for comment-only changes.

## Skills Reference

| Skill | Purpose | Key behaviours |
|---|---|---|
| `jira-creation` | Create new Jira bug tickets via browser automation | Opens expanded Create Bug modal · sets Priority + Parent (PROJ-404) + Labels (R5 + Onboarding) · enables "Create another" for batch mode · confirms creation via toast counter |
| `jira-sync-up` | Read existing tickets, identify duplicates / enrichment candidates | Filters by `Onboarding` + `R5` labels · maps bug-report IDs to ticket IDs · classifies each as create new / enrich existing / add comment / skip |
| `web-testing` | UAT execution against the cards-onboarding-web app | Navigates UAT environment · captures screenshots as evidence · documents actual vs expected |

## Deduplication Map (this cycle)

| Bug Report IDs | Matched Jira Ticket | Match Type | Action Taken |
|---|---|---|---|
| BUG-04, BUG-05, BUG-06, BUG-07 | PROJ-1404 | Exact | Enrich existing |
| BUG-20, BUG-21 | PROJ-1410 | Exact | Enrich existing |
| BUG-28 | PROJ-1394 | Exact | Enrich existing |
| BUG-24 | PROJ-1313 | Partial | Enrich existing (partial coverage) |
| BUG-29 | PROJ-1394 | Comment only | Add grammar comment to PROJ-1394 |
| BUG-08 → BUG-14, BUG-15, BUG-19, BUG-25, BUG-27 | — | No match | Created new tickets |
| NEW (auth bypass) | — | No match | Created new P2 ticket |

**Total**: 29 raw findings → 18 matched · 12 new · 1 comment.

## Description Template (applied to every new ticket)

```
Steps to Reproduce:
1. [Action]
2. [Action]
3. [Observation point]

Actual Behaviour:
[Precise description of what the application does]

Expected Behaviour:
[Precise description of what the application should do, and why]

Environment:
UAT: https://[redacted].uat.example/[specific-page-path]

Impact:
[Priority level] — [User-facing or security consequence of the bug not being fixed]
```

## Priority Reference (UAT severity → Jira priority)

| UAT Severity | Jira Priority | Criteria |
|---|---|---|
| Critical | P1 | Data corruption, form submission failure, wrong data format (e.g. date locale), missing required validation, signing flow breakage |
| High | P2 | Authentication bypass, security vulnerability, session management failure, data exposure risk |
| Medium | P3 | Non-critical validation missing, UI element visible when it should be hidden, missing progress indicators |
| Low | P4 | Missing UX warnings (e.g. no session expiry notice), minor friction points |
| Cosmetic | P5 | Grammar, spelling, minor wording issues |

**Rule:** Security findings are always P2 minimum, regardless of perceived severity. Auth bypasses are P2 even if exploitability appears low in UAT.

## Ticket Register (this cycle, 12 net-new)

| # | Bug Ref | Priority | Summary |
|---|---|---|---|
| 1 | BUG-08 | P1 | Move-in date displayed as MM/DD/YYYY instead of DD/MM/YYYY |
| 2 | BUG-09 | P1 | Middle Name field accepts special characters and numbers without validation |
| 3 | BUG-10 | P1 | Middle Name field has no maximum character length enforced |
| 4 | BUG-11 | P1 | Postcode field has no maximum character length enforced |
| 5 | BUG-12 | P1 | Building Number field has no character type or length validation |
| 6 | BUG-13 | P1 | Street field has no character type or length validation |
| 7 | BUG-14 | P1 | Town field has no character type or length validation |
| 8 | BUG-27 | P1 | START SIGNING modal does not pre-populate applicant name and email |
| 9 | BUG-19 | P4 | No session expiry warning shown before automatic timeout |
| 10 | BUG-15 | P3 | Childcare Cost field is visible but disabled when number of dependants is 0 |
| 11 | BUG-25 | P3 | Step progress stepper is not displayed on the `/waiting` page |
| 12 | NEW | P2 | Session return URL bypasses mobile auth when pasted manually in a new tab |

All carry labels `R5 + Onboarding` and parent epic `PROJ-404`.

## Modal Navigation — coordinate reference (1469 × 837 viewport)

Atlaskit's Create Bug modal is taller than the viewport and uses styled `<div>` elements for dropdowns (no semantic `<button>`). DOM-based discovery fails — coordinate clicks are required.

| Element | Coordinate | Notes |
|---|---|---|
| Summary field | `[727, 493]` | When form is scrolled to top |
| Description body | `[727, 655]` | Click inside text-area body (not toolbar — toolbar click is a no-op) |
| Priority dropdown | `[534, 392]` | When scrolled ~4 ticks from top |
| P1 / P2 / P3 / P4 options | inside open Priority dropdown | Position varies; verify visually |
| Create button | `[1057, 733]` | Fixed in footer |
| Cancel button | `[975, 733]` | Fixed in footer |

Scroll-amount cheat-sheet (same viewport):
- `~10 ticks up` from anywhere → top (Summary visible)
- `~4 ticks down` from top → Priority + Parent visible
- `~6–7 ticks down` from top → Labels visible
- `~10–12 ticks down` from top → Footer (Create button + "Create another" checkbox)

## Error Compendium — definitive fixes from real sessions

| ID | Symptom | Root cause | Rule |
|---|---|---|---|
| ERR-001 | `Failed to execute action: Unsupported action: click` | Computer-use protocol vocabulary doesn't include `"click"` | Always pass `action: "left_click"`. Other valid mouse actions: `right_click`, `double_click`, `middle_click`, `mouse_move`, `left_click_drag` |
| ERR-002 | `Failed to execute JavaScript: 'javascript_exec' is the only supported action` | The browser JS-execution tool only accepts a single `action` value | Always pass `action: "javascript_exec"` — no other value works |
| ERR-003 | DOM query for Priority dropdown options returns empty | Atlaskit dropdowns are styled `<div>`s with no `role="button"`. Walking up 8 DOM levels from the text node finds no semantic button anywhere | For any Jira dropdown (Priority, Status, Labels, Parent…), click by coordinate using `left_click`. Never attempt `querySelector('button')` or `role="button"` |
| ERR-004 | Action taken on a stale assumption about modal state after a context reset | Across context-window boundaries, the lossy summary cannot reliably encode exact form state (filled fields, "Create another" status, scroll position) | First action in any resumed session is `screenshot`. Don't type / click / scroll until ground-truth state is visually confirmed |
| ERR-005 | Modal closes after Create; Priority + Parent + Labels reset | "Create another" checkbox state is not persisted across Jira page reloads — defaults to unchecked | In the first scroll-to-bottom of any batch session, visually verify the checkbox is ticked. Costs ~3 minutes per occurrence if missed |
| ERR-006 | Typing goes to the toolbar or no-ops on Description | Description uses a Prosemirror/Tiptap-style rich-text editor with toolbar above and text-area body below. Click on toolbar doesn't activate the body | Click `[727, 655]` (form scrolled so Summary is at top). Verify cursor before typing |

## Best Practices (ranked by consequence of getting them wrong)

1. **Deduplication before creation — always.** Jira boards accumulate duplicate-debt fast and clean-up is expensive.
2. **Screenshot first on session resume.** Summary text from the previous context is approximate; the screenshot is ground truth.
3. **Tick "Create another" before the first Create click.** Single highest-risk omission in the workflow.
4. **Group tickets by priority — change Priority once per group, not per ticket.** Saves ~30 s per ticket and reduces wrong-priority risk under time pressure.
5. **Verify the toast counter after every Create.** Most reliable signal that a ticket was saved. If it doesn't increment, scroll up and check for red-bordered validation failures.
6. **Use the expanded modal, not the compact dialog.** Compact dialog hides Priority / Parent / Labels — they can't be set there.
7. **Fill Summary before Description.** Summary validation fires on Create; an empty Summary blocks submission with a red border.
8. **For mid-session new findings: file in-batch, don't break out of the modal.** Maintaining batch context (Priority + Parent + Labels) is cheaper than reopening and reconfiguring.

## Patterns reused from earlier projects in this portfolio

| Pattern | Source project | How it appears here |
|---|---|---|
| Structured input → structured output discipline (PRD → multi-query expansion) | Projects 1, 2 | `uat_bug_report.js` is the structured input; the deduplication classifier is the structured-decision output (skip / enrich / new / comment) |
| Citation-grounded outputs (every claim must cite a source) | Project 2 | Every new Jira ticket cites bug-report IDs in the description; partial matches add `Related to [PROJ-XXXX]` |
| Validate before promote — never auto-patch | Project 3 | Findings packets are validated for completeness before any Jira write; ambiguous cases get comments rather than new tickets |
| Coordinate clicks for non-semantic SPA controls; Jira REST batch ops; "always run inside the user's logged-in session" | Project 4 | Direct reuse — every Atlaskit dropdown is a coordinate click, every batch op piggybacks on the session's auth context |
| Skill-style runbook + version-controlled `SKILL.md` files | Project 4 | Three reusable skills (`jira-creation`, `jira-sync-up`, `web-testing`) packaged the same way |

## What I'd Do Differently

- **Verify "Create another" was ticked before the first Create on every session, including v1.0.** A single missed tick on the very first session cost a full re-setup pass — exactly the failure mode now codified in ERR-005.
- **Capture screenshot IDs as I go, not at the end.** Screenshot IDs are session-scoped; bundling them up at the end of a session means losing some context if the session ends abruptly.
- **Build a small JQL helper that exports the deduplication map directly.** Right now the dedup map is hand-classified. A query that returns `(label-filtered ticket → fuzzy-match score against bug-report titles)` would shave the deduplication phase from minutes to seconds.
- **Treat the Error Compendium as a contract, not a log.** Any new failure mode is added before the next session resumes — that's already the rule, but worth restating: skipping the write-up means the next session repeats the failure.

---

