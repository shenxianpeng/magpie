<!-- SPDX-License-Identifier: Apache-2.0
     https://www.apache.org/licenses/LICENSE-2.0 -->

# Bulk mode — syncing many issues in parallel

> Extracted from [`SKILL.md`](SKILL.md) so subagents that only need
> this slice can load just this file. Loaded automatically when the
> orchestrator (or a subagent) is in the matching step.

This subdoc carries the bulk-mode orchestration contract — how the orchestrator buckets trackers (CVE-affecting vs non-CVE-affecting), what subagents return, the merged-proposal review shape, hard rules, and when bulk mode is NOT appropriate.

---

## Bulk mode — syncing many issues in parallel

When the user asks for a bulk sync (*"sync all open issues"*, *"sync
#212, #214 and #218"*, *"refresh state of everything that is still
`cve allocated`"*, or a triage-sweep variant), switch into **bulk
mode**: each issue is assessed by a **separate subagent** running in
parallel, and the orchestrator merges the results into a single
combined proposal for the user to confirm once.

Running the full single-issue flow 20 times in the main agent would
blow the context window with mail threads, PR diffs, and comment
bodies the user does not need to see. Delegating per-issue gathering
to subagents keeps the main context clean and runs the reads
concurrently, which is exactly what the sync needs.

### Orchestrator responsibilities

1. **Pick the issue list.** Resolve the user's selector into a
   concrete list of issue numbers before spawning subagents. The
   selectors the skill accepts, in order of precedence:

   | User input | Resolves to |
   |---|---|
   | `sync all` | every open issue in `<tracker>` **plus recently-closed trackers still awaiting a post-close cve.org publication check**. Resolve as: `gh issue list --repo <tracker> --state open --limit 100 --json number,title,labels` ∪ `gh issue list --repo <tracker> --state closed --label "announced" --limit 50 --json number,title,labels,closedAt --jq '[.[] \| select(.closedAt > (now - 90*86400 \| todate))]'`. The closed bucket is limited to the last 90 days and to trackers carrying the `announced` label — those are the ones waiting for cve.org propagation + the final reporter notification (see [1g](gather.md#1g-recently-closed-trackers--check-cveorg-publication-state)). Everything else is a no-op on closed issues and is excluded. |
   | `sync all open` | explicit open-only variant — `gh issue list --repo <tracker> --state open --limit 100 --json number,title,labels`. No closed trackers. Use when you want the classic open-only sweep and nothing else. |
   | `sync #212`, `sync 212`, `sync #212, #214, #218`, `sync #212-#218` | the issue number(s) verbatim — no resolution needed. Works on open and closed trackers alike (the closed-issue sub-steps run when the tracker is closed with `announced`). |
   | `sync CVE-2026-40913` or `sync CVE-2026-40913, CVE-2026-40690` | regex-validate each token against `^CVE-\d{4}-\d{4,7}$` first (anything that does not match is a hard error — *never* interpolate an unvalidated free-form string into the search arg, which is in double quotes and would expand `$(...)`); then look up each validated CVE ID with `gh search issues "CVE-YYYY-NNNNN" --repo <tracker> --json number,title,body --jq '.[] | select(.body \| contains("CVE-YYYY-NNNNN")) \| .number'` (match against the body's *CVE tool link* field) and expand. |
   | `sync <free-text>` (e.g. `sync JWT`, `sync KubernetesExecutor`) | title-substring match — run `gh issue list --repo <tracker> --state open --search "<free-text> in:title" --limit 100 --json number,title` and surface the matches back to the user for confirmation before dispatching (title matches are the fuzziest selector — always confirm, never auto-dispatch). |
   | `sync <label>` (e.g. `sync announced`, `sync pr merged`) | all open issues carrying that label — `gh issue list --repo <tracker> --state open --label "<label>" --limit 100 --json number,title`. |
   | `sync announced` (as a label selector) | as above, open-only. To include the recently-closed `announced` bucket, use `sync all` (default) or `sync closed announced`. |
   | `sync closed announced` | the recently-closed `announced` bucket by itself — useful when you want to run the cve.org publication-check sweep without touching open issues (for example, as a post-release cron). |
   | `sync open` | alias for `sync all open`. |
   | `sync closed` | open *and* closed issues, **all** closed (not just recent `announced`). Explicit, narrow-scope request — most sync actions are no-ops on closed issues that are not in the `announced` bucket. |

   Selectors can be combined: `sync #212, CVE-2026-40690, JWT`
   resolves each independently and dispatches the union of the
   resulting issue numbers. After resolving, **echo the final list
   back to the user and ask for confirmation** before spawning
   subagents — this catches fuzzy-match surprises (a title-substring
   hit that was not intended, a CVE alias that matched two scope
   trackers) before they cost an API round-trip. When the open /
   closed buckets both contribute, group them in the echo so the
   user can tell at a glance *"9 open, 2 recently-closed awaiting
   cve.org"*.

   When the selector resolves to zero issues, tell the user and stop
   — do not fall back to `sync all`.

2. **Spawn one subagent per issue, in a single message.** Use the
   `general-purpose` subagent type and send all `Agent` tool calls in
   the **same assistant message** so they run concurrently. For 20
   issues, that is 20 parallel `Agent` calls in one turn.

   Each subagent prompt must be self-contained and must instruct the
   subagent to:

   - Do **only Step 1** (gather state) from this skill — no
     confirmations, no edits, no draft emails, no label changes, no
     milestone creation, no comments. The subagent is a read-only
     assessor.
   - Read the issue, its closing-PR references, the fixing PR state
     and milestone, the originating Gmail thread, and mine comments
     and mail for the signals in the table in Step 1d.
   - Return a **compact structured report** — not a freeform
     narrative. The exact shape is below.

3. **Bucket trackers by CVE-record impact.** A tracker's proposed
   changes fall into one of two buckets:

   - **CVE-affecting** — any proposal that changes a body field
     whose value lands in the regenerated CVE JSON pushed to
     Vulnogram. Concretely: *Title* (issue title; ships into
     `containers.cna.title`), *Short public summary for publish*,
     *CWE*, *Severity*, *Affected versions*, *Reporter credited
     as*, *Remediation developer*, *PR with the fix*, *Public
     advisory URL*. Also: any change to the issue title itself
     (the generator reads it verbatim into `title`). The
     [pre-push hygiene gates in Step 5b 1b](apply-and-push.md#decision-flow) all
     scan fields in this bucket; the bucket exists for the same
     reason the gates do — these are the values that ship to
     `cve.org` and stay there.
   - **Non-CVE-affecting** — label flips, milestone touches,
     assignee swaps, project-board column moves, status-rollup
     entries, reporter Gmail drafts, RM hand-off comments
     (template-bodied, no per-tracker CVE content). These
     change tracker state but do not alter the published CVE
     record.

4. **Present both buckets as merged bulk proposals; the
   CVE-affecting bucket gets a richer per-item view.** The
   two buckets are presented to the user differently:

   - **Non-CVE-affecting bucket** — fold into one combined
     proposal, same shape as the legacy bulk mode. The user
     confirms once with `all`, `NN:all`, `NN:1,3`, or per-issue
     subsets, and the orchestrator applies them sequentially.
     This bucket is bundled because the actions are reversible,
     low-blast-radius, and do not leak into public CVE surfaces.
   - **CVE-affecting bucket** — present **all proposed
     CVE-record-affecting changes from all trackers as ONE
     merged bulk proposal**, with per-tracker sections so the
     user can review every body-field rewrite, every regen+push
     target, every deferral condition at a glance. For each
     tracker section the proposal shows: CVE ID, gate-failure
     summary, every body-field update with old / new value
     side by side, the planned regen+push action, any deferral
     conditions. The user reviews the **whole bulk pack at
     once** and signals which items to apply / skip / modify
     using the same syntax as the non-CVE-affecting bucket
     (`all`, `NN:all`, `NN:1,3`, `NN:skip`, `NN:edit <item>:
     <new value>`). On confirmation the orchestrator applies
     the confirmed items across all trackers sequentially.

   **Why bulk-review (and not per-tracker walk).** Per-tracker
   walk through N CVE-affecting trackers serialises the
   confirmation cost into N round-trips and forces context
   re-loading for each one — the operator can't compare
   proposed summaries across trackers, can't notice that two
   trackers should converge on the same CWE long-form, can't
   see at a glance that three are blocked on the same missing
   field. A single merged proposal puts everything on one
   page: the operator sees the full bulk shape, edits whichever
   items they want, and the orchestrator applies the
   confirmed set in one pass. The hygiene gates in
   [Step 5b 1b](apply-and-push.md#decision-flow) still catch *mechanical* drift
   (bare CWE, missing upgrade target, etc.) on every JSON
   regen; the bulk-review surface is for the operator to make
   *judgment* calls (threat-model framing, credit-line shape,
   CWE choice) before the push fires.

   **Confirmation syntax** for the merged proposal:

   - `all` — apply every proposed change across all trackers.
   - `<N>:all` — apply every change on tracker `<N>`; skip the
     others.
   - `<N>:1,3,5` — apply only the listed items on tracker
     `<N>`.
   - `<N>:skip` — skip tracker `<N>` entirely.
   - `<N>:edit <item-number>: <new value>` — replace the
     proposed item with a free-form override before applying.
   - `cancel` / `none` — apply nothing.

   **Proposal order in the merged pack.** Trackers appear in
   **ascending tracker-number order** so the operator can
   navigate predictably across reruns. The operator can name a
   different order at confirmation (*"apply #438 first; I want
   to think about #232 last"*) and the orchestrator honours
   it.

5. **Apply sequentially, not in parallel.** Even though
   assessment ran in parallel, the apply phase must be
   sequential so `gh`-rate-limit surprises, partial failures,
   and user interrupts stay legible. Do not spawn subagents for
   the apply phase.

### Subagent report shape

Each subagent must return a single code block (or JSON) with exactly
these fields so the orchestrator can merge deterministically:

```yaml
issue: <N>
title: <one line>
scope_label: airflow | providers | chart | <missing>
current_labels: [<label>, ...]
current_milestone: <title or null>
current_assignees: [<login>, ...]
fix_pr:
  url: <<upstream> PR URL or null>
  state: open | merged | closed | null
  author: <login or null>
  author_is_security_team: true | false | null
  merged_at: <ISO8601 or null>
  milestone: <PR milestone title or null>
release_shipped: true | false | unknown
reporter:
  name: <name or null>
  email: <email or null>
  gmail_thread_id: <id or null>
  credit_confirmed_as: <string or null>
  credit_question_pending: true | false
cve_id: <CVE-YYYY-NNNNN or null>
process_step: <number from the README table>
proposed_label_add: [<label>, ...]
proposed_label_remove: [<label>, ...]
proposed_milestone: <title or null, with note "(create)" if it does not yet exist>
proposed_assignees_add: [<login>, ...]
proposed_body_field_updates: [<one-line description>, ...]
proposed_status_comment: <one-line summary or null>
proposed_reporter_email: <one-line summary or null>
blockers: [<short reason the orchestrator or user must resolve before apply>, ...]
notes: <free-form one-to-three sentences, only if something does not fit above>
```

The orchestrator uses the structured fields to produce the merged
proposal table and relies on `blockers` to flag issues that cannot
be resolved without user input (for example a missing Gmail thread
or an ambiguous credit line).

### Hard rules for bulk mode

- **No mutations in subagents.** Subagents must not call
  `gh issue edit`, `gh issue comment`, `gh api … -X PATCH/POST`,
  `gh label create`, `gh api …/milestones` (create), or any Gmail
  send / draft-create tool. They are read-only. If a subagent
  reports it did mutate something, the orchestrator must surface
  that as a bug and stop.
- **No new CVE allocations in subagents.** Printing the CVE
  allocation URL is fine; actually allocating is a human step
  anyway.
- **Gmail drafts are created by the orchestrator**, only after user
  confirmation, and only from the orchestrator's main context. This
  keeps the drafts queue linear and auditable.
- **Confidentiality still applies.** Subagents are bound by the
  same rule: no `<tracker>` content may leak into any
  public surface. This is a no-op for read-only subagents but worth
  stating.
- **Link-form self-check still applies** to the orchestrator's
  merged output — every `#NNN` must be rendered as a clickable link
  per Golden rule 2.

### When bulk mode is **not** appropriate

- The user asked for a single issue (`sync #216`). Run the normal
  flow in the main agent — spawning one subagent for one issue is
  pure overhead.
- The user wants to *drive* the sync interactively ("walk me
  through #216, I want to review each signal as we go"). Bulk mode
  collapses the per-issue detail; use single-issue mode instead.
- The proposed action requires deep multi-turn conversation with
  the user (for example "help me decide whether this is even valid").
  Single-issue mode is the right tool there.

---
