# Cards Onboarding UAT, Jira Bug Filing Automation

> ### What this repository is
>
> A method write-up of an agent-assisted UAT triage cycle, not a tool you install.
> There is no package here: the workflow runs through an AI browser-automation
> agent against a client's Jira, and both the findings and the tracker are
> confidential.
>
> `uat_bug_report.js` is the structured input that workflow consumed. It is not a
> file in this repository. Do not look for it here; its shape is documented below
> so you can build the equivalent.
>
> The numbers come from one real cycle: 29 raw findings resolving to 18 matched
> against existing tickets, 11 filed as new, and 1 comment-only. A twelfth ticket
> was created for a security issue found during testing rather than listed in the
> report, so 12 tickets were created in total.
>
> What you can take from it: the dedup-before-create ordering, the classification
> schema, and the human review gate, which is the transferable part. The Jira REST
> patterns underneath it are published as runnable code in
> [spa-automation-toolkit](https://github.com/sayandip1987/spa-automation-toolkit)
> (`examples/jira-bulk-edit.js`).

AI-assisted UAT bug triage, deduplication and batch Jira ticket creation for a
client's UK cards onboarding web application (`cards-onboarding-web`). It bridges
a structured UAT bug report (`uat_bug_report.js`) with Jira through an AI
browser-automation agent, to deduplicate findings, batch-create new tickets and
enrich existing ones in a single browser session.

> Build effort: 2 to 3 hrs per session, ongoing across releases
> Internal-team users: about 25 to 30 QA, PMs, eng leads
> Validated across 3 sessions creating 12 net-new Jira tickets across priorities P1 to P4, including a P2 security finding discovered during testing

## What the cycle produced

- 29 raw UAT findings processed into 18 matched against existing Jira tickets with no duplicates created, 12 net-new tickets with consistent shape, and 1 grammar comment on a pre-existing ticket
- Deduplication-first workflow, filtered against existing tickets carrying labels `R5` and `Onboarding`, classifying each finding as create new, enrich existing, add comment or skip. This is what stops duplicate-ticket debt accumulating across UAT cycles
- Batch ticket creation in a single browser session, using a "Create another" pattern that retains Priority, Parent epic and Labels between submissions. File all P1s together, change priority once per group, file all P4s, and so on
- One P2 security finding discovered during testing: the session return URL bypassed mobile authentication when pasted manually in a new tab after expiry. Filed independently of the original bug report
- Consistent ticket shape on every new bug. The Steps to Reproduce / Actual / Expected / Environment / Impact template is applied uniformly, so developer triage is faster and reporting and filtering are reliable
- Three reusable skills packaged in version-controlled `SKILL.md` files: `jira-creation`, `jira-sync-up`, `web-testing`, each with explicit trigger phrases, behaviours and coordinate references
- Six automation errors documented with their fixes in an Error Compendium, so future sessions do not repeat known failure modes
- Cross-session resilience. Every resumed session begins with a screenshot to verify ground-truth modal state before any action

## Workflow architecture

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

## Tech stack

| Layer | Technology | Role |
|---|---|---|
| Browser-automation agent | AI agent with Chrome connector (Cowork / Claude Code) | Drives the SPA, captures screenshots, runs JS, posts to Jira |
| Browser | Chrome with MCP extension | Logged-in session, inherits the user's auth and cookies |
| Target SPA | Atlassian Jira (Atlaskit components) | Bug board, Create modal, comments |
| UAT target | client cards onboarding web app | UAT environment for evidence capture |
| Source of truth | `uat_bug_report.js` (29 findings) | `{ id, title, severity, page, steps, actual, expected, impact }` |
| Skills | three `SKILL.md` files (YAML front-matter) | `jira-creation`, `jira-sync-up`, `web-testing` |
| Workflow contract | `CLAUDE.md` | Project memory that survives across context resets |

## How it was built, phase by phase

**Phase 0, deduplication, always first.** Run the `jira-sync-up` skill before creating anything. JQL filter: `project = PROJ AND labels = "R5" AND labels = "Onboarding" ORDER BY created DESC`. For each finding in `uat_bug_report.js`, classify as exact match (skip or enrich), partial match (create and add `Related to [PROJ-XXXX]`), no match (create new), or comment-only (add a comment to the most relevant existing ticket). Outcome on this cycle: 18 matched, 11 new, 1 comment.

**Phase 1, one-time session setup, first ticket only.** Navigate to Bug Board backlog, click `+ Create`, click ⤢ to open the expanded modal, because the compact dialog does not expose Priority, Parent or Labels. Fill Summary, then Description using the standard template. Scroll about 4 ticks down, set Priority. Set Parent (`PROJ-404`, "Cards Application Onboarding - Web"). Scroll about 2 more ticks, add Labels `R5` and `Onboarding`. Scroll to the footer and tick "Create another". That tick is the highest-risk action in the workflow: missing it forces a full re-setup of Priority, Parent and Labels on every subsequent ticket. Click Create, then verify the toast shows "1 work item created".

**Phase 2, subsequent tickets in the same priority group.** The form resets Summary and Description but retains Priority, Parent and Labels. For each subsequent ticket: scroll up about 10 ticks, Summary, Description, scroll down about 10 ticks, Create. Verify the toast counter increments by exactly 1 (`"2 work items created"`, `"3 work items created"`, and so on). If the counter does not increment the ticket was not created, so scroll up and check for red-bordered fields.

**Phase 3, changing priority between groups.** After the last ticket in a group, scroll down about 4 ticks so the Priority dropdown is visible, select the new priority, scroll back up, continue. Sequence used on this cycle: P1, P4, P3, P2. The non-linear order is intentional, grouping by failure mode: all P1 validation and input bugs together, then the lone P4 warning, then P3 display bugs, then the P2 security finding.

**Phase 4, post-creation enrichment of pre-existing tickets.** For each ticket identified in deduplication (`PROJ-1404`, `PROJ-1410`, `PROJ-1313`): open in a new tab, click into Description, replace the sparse description with the full template, attach the relevant UAT screenshots, save. For the comment-only case (`PROJ-1394`, a grammar fix): scroll to Activity and Comments, "Add a comment", type the correction, save. Do not edit the ticket summary or description for comment-only changes.

## Skills reference

| Skill | Purpose | Key behaviours |
|---|---|---|
| `jira-creation` | Create new Jira bug tickets via browser automation | Opens expanded Create Bug modal · sets Priority + Parent (PROJ-404) + Labels (R5 + Onboarding) · enables "Create another" for batch mode · confirms creation via toast counter |
| `jira-sync-up` | Read existing tickets, identify duplicates and enrichment candidates | Filters by `Onboarding` + `R5` labels · maps bug-report IDs to ticket IDs · classifies each as create new / enrich existing / add comment / skip |
| `web-testing` | UAT execution against the cards-onboarding-web app | Navigates UAT environment · captures screenshots as evidence · documents actual vs expected |

## Deduplication map (this cycle)

| Bug Report IDs | Matched Jira Ticket | Match Type | Action Taken |
|---|---|---|---|
| BUG-04, BUG-05, BUG-06, BUG-07 | PROJ-1404 | Exact | Enrich existing |
| BUG-20, BUG-21 | PROJ-1410 | Exact | Enrich existing |
| BUG-28 | PROJ-1394 | Exact | Enrich existing |
| BUG-24 | PROJ-1313 | Partial | Enrich existing (partial coverage) |
| BUG-29 | PROJ-1394 | Comment only | Add grammar comment to PROJ-1394 |
| BUG-08 → BUG-14, BUG-15, BUG-19, BUG-25, BUG-27 | — | No match | Created new tickets |
| NEW (auth bypass) | — | No match | Created new P2 ticket |

Totals: 29 raw findings resolve to 18 matched, 11 new from the report and 1 comment. The auth-bypass finding was not in the report, which brings the created total to 12.

## Description template (applied to every new ticket)

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

## Priority reference (UAT severity to Jira priority)

| UAT Severity | Jira Priority | Criteria |
|---|---|---|
| Critical | P1 | Data corruption, form submission failure, wrong data format (e.g. date locale), missing required validation, signing flow breakage |
| High | P2 | Authentication bypass, security vulnerability, session management failure, data exposure risk |
| Medium | P3 | Non-critical validation missing, UI element visible when it should be hidden, missing progress indicators |
| Low | P4 | Missing UX warnings (e.g. no session expiry notice), minor friction points |
| Cosmetic | P5 | Grammar, spelling, minor wording issues |

**Rule:** security findings are always P2 minimum, regardless of perceived severity. Auth bypasses are P2 even if exploitability appears low in UAT.

## Ticket register (this cycle, 12 created)

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

Rows 1 to 11 come from the bug report. Row 12 was found during testing. All carry labels `R5 + Onboarding` and parent epic `PROJ-404`.

## Modal navigation, coordinate reference (1469 × 837 viewport)

Atlaskit's Create Bug modal is taller than the viewport and uses styled `<div>` elements for dropdowns, with no semantic `<button>`. DOM-based discovery fails, so coordinate clicks are required.

| Element | Coordinate | Notes |
|---|---|---|
| Summary field | `[727, 493]` | When form is scrolled to top |
| Description body | `[727, 655]` | Click inside text-area body, not the toolbar, because a toolbar click is a no-op |
| Priority dropdown | `[534, 392]` | When scrolled ~4 ticks from top |
| P1 / P2 / P3 / P4 options | inside open Priority dropdown | Position varies; verify visually |
| Create button | `[1057, 733]` | Fixed in footer |
| Cancel button | `[975, 733]` | Fixed in footer |

Scroll-amount cheat-sheet (same viewport):
- `~10 ticks up` from anywhere goes to the top, with Summary visible
- `~4 ticks down` from top makes Priority and Parent visible
- `~6–7 ticks down` from top makes Labels visible
- `~10–12 ticks down` from top reaches the footer, with the Create button and "Create another" checkbox

## Error compendium, fixes from real sessions

| ID | Symptom | Root cause | Rule |
|---|---|---|---|
| ERR-001 | `Failed to execute action: Unsupported action: click` | Computer-use protocol vocabulary doesn't include `"click"` | Always pass `action: "left_click"`. Other valid mouse actions: `right_click`, `double_click`, `middle_click`, `mouse_move`, `left_click_drag` |
| ERR-002 | `Failed to execute JavaScript: 'javascript_exec' is the only supported action` | The browser JS-execution tool only accepts a single `action` value | Always pass `action: "javascript_exec"`. No other value works |
| ERR-003 | DOM query for Priority dropdown options returns empty | Atlaskit dropdowns are styled `<div>`s with no `role="button"`. Walking up 8 DOM levels from the text node finds no semantic button anywhere | For any Jira dropdown (Priority, Status, Labels, Parent) click by coordinate using `left_click`. Never attempt `querySelector('button')` or `role="button"` |
| ERR-004 | Action taken on a stale assumption about modal state after a context reset | Across context-window boundaries, the lossy summary cannot reliably encode exact form state (filled fields, "Create another" status, scroll position) | First action in any resumed session is `screenshot`. Don't type, click or scroll until ground-truth state is visually confirmed |
| ERR-005 | Modal closes after Create; Priority + Parent + Labels reset | "Create another" checkbox state is not persisted across Jira page reloads, and defaults to unchecked | In the first scroll-to-bottom of any batch session, visually verify the checkbox is ticked. Costs about 3 minutes per occurrence if missed |
| ERR-006 | Typing goes to the toolbar or no-ops on Description | Description uses a Prosemirror/Tiptap-style rich-text editor with a toolbar above and text-area body below. A click on the toolbar doesn't activate the body | Click `[727, 655]` with the form scrolled so Summary is at top. Verify cursor before typing |

## Best practices, ranked by consequence of getting them wrong

1. **Deduplication before creation, always.** Jira boards accumulate duplicate-debt fast and clean-up is expensive.
2. **Screenshot first on session resume.** Summary text from the previous context is approximate; the screenshot is ground truth.
3. **Tick "Create another" before the first Create click.** The highest-consequence omission in the workflow.
4. **Group tickets by priority, and change Priority once per group rather than per ticket.** Saves about 30 s per ticket and reduces wrong-priority risk under time pressure.
5. **Verify the toast counter after every Create.** It is the most reliable signal that a ticket was saved. If it doesn't increment, scroll up and check for red-bordered validation failures.
6. **Use the expanded modal, not the compact dialog.** The compact dialog hides Priority, Parent and Labels, so they cannot be set there.
7. **Fill Summary before Description.** Summary validation fires on Create, and an empty Summary blocks submission with a red border.
8. **For mid-session new findings, file in-batch rather than breaking out of the modal.** Maintaining batch context (Priority, Parent, Labels) is cheaper than reopening and reconfiguring.

## Patterns reused from other repositories here

| Pattern | Source | How it appears here |
|---|---|---|
| Structured input to structured output discipline (PRD to multi-query expansion) | [enterprise-rag-knowledge-base](https://github.com/sayandip1987/enterprise-rag-knowledge-base), [uk-compliance-knowledge-base](https://github.com/sayandip1987/uk-compliance-knowledge-base) | `uat_bug_report.js` is the structured input; the deduplication classifier is the structured-decision output (skip / enrich / new / comment) |
| Citation-grounded outputs, where every claim cites a source | [uk-compliance-knowledge-base](https://github.com/sayandip1987/uk-compliance-knowledge-base) | Every new Jira ticket cites bug-report IDs in the description; partial matches add `Related to [PROJ-XXXX]` |
| Validate before promote, never auto-patch | [amazon-connect-flow-tools](https://github.com/sayandip1987/amazon-connect-flow-tools) | Findings packets are validated for completeness before any Jira write; ambiguous cases get comments rather than new tickets |
| Coordinate clicks for non-semantic SPA controls; Jira REST batch ops; run inside the user's logged-in session | [spa-automation-toolkit](https://github.com/sayandip1987/spa-automation-toolkit) | Direct reuse. Every Atlaskit dropdown is a coordinate click, and every batch op piggybacks on the session's auth context |
| Skill-style runbook with version-controlled `SKILL.md` files | [spa-automation-toolkit](https://github.com/sayandip1987/spa-automation-toolkit) | Three reusable skills (`jira-creation`, `jira-sync-up`, `web-testing`) packaged the same way |

## Context engineering across a long session

A triage cycle runs longer than one context window. 29 findings, a Jira board to read, a modal to drive and screenshots as evidence will cross a reset, so the workflow is built on the assumption that the agent will forget, and that what it half-remembers is worse than nothing.

**Durable facts live in a file, not in the conversation.** `CLAUDE.md` holds the workflow contract: the JQL filter, the parent epic, the label set, the classification outcomes, the coordinate table. Those are re-read rather than recalled. Anything that must be true in session three is written down in session one, because a summary is lossy in exactly the places that matter, and it is lossy silently.

**Treat the post-reset summary as a claim, not as state.** This is what ERR-004 is really about. After a context boundary, the agent believes it knows whether "Create another" is ticked, which fields are filled and where the form is scrolled to, and that belief is reconstructed rather than observed. The rule is that the first action in any resumed session is a screenshot, and nothing is typed, clicked or scrolled until the screen has confirmed the state. Ground truth comes from the world, not from the transcript.

**Keep the working set small on purpose.** Findings are processed in priority groups rather than all 29 at once, which is a batching decision for the Jira form and a context decision as well. One group means one priority, one parent and one label set held in mind, and the only things changing per ticket are Summary and Description.

**Screenshot IDs are session-scoped, which is a context bug waiting to happen.** Capturing them as you go rather than collecting them at the end is in the list below for this reason: an ID that only exists in the current window is not a durable reference, and treating it as one loses evidence when the session ends abruptly.

## The classifier is a judge, and the gate is what makes it safe

The deduplication step asks a model to decide, for each of 29 findings, whether it matches an existing ticket. That is an LLM-as-judge call, and it is the only place in this workflow where a wrong answer is expensive: a false match silently drops a real bug, and a false miss files a duplicate.

Two things keep it usable. The question is narrow, a classification into four named outcomes (create new, enrich existing, add comment, skip) rather than an open judgement about what should happen. And nothing it decides is written without a person seeing the classification first, which is what the human gate is for. The gate is not a formality here. On this cycle it is what stood between 18 correct matches and 18 quiet omissions.

The Error Compendium works the same way an eval suite does, in prose rather than code. Each entry is a failure that happened once, its root cause, and the rule that prevents it. Adding an entry before the next session resumes is the regression test. The difference from a real harness is that it is checked by a person rather than automatically, which is a weakness I have not fixed.

**Observability here is the toast counter.** There is no logging layer; the only reliable signal that a ticket was actually saved is the counter incrementing by exactly 1. Verifying it after every Create is what turns a silent validation failure into a caught one, and it is the cheapest instrumentation in the whole workflow.

## What I'd do differently

- **Verify "Create another" was ticked before the first Create on every session, including v1.0.** A single missed tick on the very first session cost a full re-setup pass, which is the failure mode now codified in ERR-005.
- **Capture screenshot IDs as I go, not at the end.** Screenshot IDs are session-scoped, so bundling them at the end of a session loses some context if the session ends abruptly.
- **Build a small JQL helper that exports the deduplication map directly.** Right now the dedup map is hand-classified. A query returning `(label-filtered ticket → fuzzy-match score against bug-report titles)` would take the deduplication phase from minutes to seconds.
- **Write the classification cases down before running a cycle, not after.** The four outcomes were settled during the first session, by hitting ambiguous findings and deciding case by case. Writing those cases out as a specification first would have made the partial-match rule explicit before it was needed, instead of after a finding sat between two categories.
- **Treat the Error Compendium as a contract, not a log.** Any new failure mode gets added before the next session resumes. That is already the rule, and it is worth restating: skipping the write-up means the next session repeats the failure.
