<!-- SPDX-License-Identifier: Apache-2.0
     https://www.apache.org/licenses/LICENSE-2.0 -->

# Issue draft — Gap analysis: adoption friction and other pre-TLP maturity gaps

> **Not a committed project doc.** This file is a copy-paste-ready draft for a
> GitHub issue on `apache/magpie`. Everything below the `---` is the intended
> issue body. Delete this header block before filing.

**Suggested title:** Gap analysis: adoption friction and other maturity gaps before wider rollout

**Suggested labels:** `enhancement`, `documentation` (maintainers may want to
split this into several tracking issues — see "Suggested next steps" below)

---

## Summary

Adopting Magpie today is more manual than the framework's own goals imply.
This issue collects that gap plus a set of other maturity gaps found while
reading through `README.md`, `MISSION.md`, `PRINCIPLES.md`, `docs/modes.md`,
`docs/setup/`, and `docs/prerequisites.md`. None of these are blockers for
the framework's current pre-release, invitation-only phase, but they're
worth tracking as the project moves toward a wider rollout and an eventual
ASF TLP resolution.

Each item below is phrased as **what should happen** and **why**, per the
repo's intent-first issue convention, so any of them can be split into its
own `change_proposal` issue if a maintainer wants to pick it up
independently.

## 1. Adoption is not one command — install, uninstall, and upgrade all require a manual + conversational hybrid

**What should happen:** An adopter should be able to bootstrap, upgrade, or
remove Magpie from their repo with a single deterministic command (a real
installer/CLI), rather than a copy-pasted shell block followed by a
free-form "tell your agent to adopt" conversation.

**Why:** Today, per `docs/setup/install-recipes.md` and `skills/setup/SKILL.md`:

- Bootstrap is a two-phase process: (1) manually copy-paste one of three
  shell recipes to fetch the snapshot and copy the `setup` skill in, then
  (2) tell an LLM agent to "adopt apache/magpie" so it can write lock
  files, prompt for skill families, and create symlinks.
- Uninstall (`/magpie-setup unadopt`) is likewise agent-driven rather than
  a standalone script — there is no deterministic, independently
  verifiable removal path a maintainer can run without an LLM in the loop.
- There is no single-command installer (`npx`, `pip install`, `brew`,
  a shell one-liner piped to `sh`) — the friction is real, not just
  optics, for a maintainer evaluating whether to try the framework.

This is the single most visible barrier to adoption and is worth fixing
independent of the framework's release-distribution timeline.

## 2. No official (signed) release yet — production adopters default to tracking `main`

**What should happen:** A production-grade, versioned install path should
exist before the framework is recommended for anything beyond
experimentation.

**Why:** `docs/setup/install-recipes.md` marks **Method 1 (svn-zip, the ASF
signed release)** as "**forthcoming**." The default recipe during this
phase is `git-branch` tracking `main` directly — i.e., adopters who want to
try the framework today are, by design, tracking a moving target rather
than a pinned, checksummed release.

## 3. Cross-agent support is narrower than the docs imply

**What should happen:** Either the documented list of supported agent CLIs
should match what's actually tested, or the docs should more prominently
flag which integrations are still in progress.

**Why:** `README.md` describes `.agents/skills/` as "the path shared by
Codex, Cursor, Gemini CLI, Copilot, …" but `docs/prerequisites.md` states
that only **OpenCode** and **Claude Code** are "fully supported today,"
with Codex / Gemini CLI / Cursor / Copilot support tracked under open
"adapter" issues. A maintainer picking an agent based on the README's
framing could hit an unsupported path.

## 4. Windows / non-POSIX support is undocumented

**What should happen:** Either a documented Windows-compatible install
path, or an explicit note that POSIX (macOS/Linux/WSL) is required.

**Why:** The bootstrap recipes and the symlink-based skill-relay model
(`agents.md`) assume a POSIX shell and unprivileged `ln -s`. Windows
symlinks require elevated privileges or Developer Mode; there's no mention
of this constraint or a PowerShell-equivalent recipe anywhere in
`docs/setup/`.

## 5. No ASF Board approval yet — the whole distribution/governance story is provisional

**What should happen:** Docs adopters read before installing should make
unambiguous that the project is pre-approval.

**Why:** `MISSION.md` is explicitly framed as a "**draft** project-
establishment proposal" for a future Apache TLP. There is no Board
resolution, no `magpie.apache.org`, no `dev@`/`private@` lists, no PMC yet.
Anyone adopting today is adopting a pre-approval, potentially-renamed,
pre-infrastructure project — worth surfacing more prominently than the
current README callout.

## 6. No validated real-world pilots yet

**What should happen:** At least one documented, externally-verifiable
pilot adoption (beyond the Airflow lineage the code was extracted from).

**Why:** `MISSION.md`'s "Initial Goals" section describes "3–4 friendly
pilots within 3 months" and "at least one non-ASF project from day one" in
future tense — these are goals, not completed validations. There's
currently no public evidence the snapshot-adoption flow has been exercised
end-to-end by a project other than the framework's own origin.

## 7. The contributor-sentiment eval methodology doesn't exist yet

**What should happen:** A working eval framework (or an honest interim
methodology) for the "contributor sentiment gates every mode graduation"
principle.

**Why:** `PRINCIPLES.md` §7–8 make eval a release-blocking discipline, but
`MISSION.md` names "Apache Plumb" as an explicitly **illustrative
placeholder, not a real or planned ASF project**. The mechanism that's
supposed to gate mode graduation (e.g., before Agentic Autonomous ever
turns on) has no implementation yet.

## 8. Vendor-neutrality claims are unvalidated across backends

**What should happen:** At least one adopter pilot per backend class
(frontier-model API, local/self-hosted inference, an Apache-aligned
endpoint), as `MISSION.md` itself sets as a v1 goal.

**Why:** The "vendor neutrality is non-negotiable" principle is currently
a design commitment, not something demonstrated in production —
`inference.apache.org` / `llm.apache.org` are described as "planned," not
available.

## 9. Steep learning curve, in tension with the "weekend to a merged skill" goal

**What should happen:** A genuinely minimal quick-start (5–10 minutes to
first working skill invocation) distinct from the full reference docs.

**Why:** `MISSION.md`'s education stream promises "any motivated
maintainer can take a working agentic skill from zero to merged in a
weekend." But onboarding today routes through `PRINCIPLES.md` (18
principles), `AGENTS.md` (1000+ lines), `CONTRIBUTING.md` (985 lines), and
70+ skill directories — a heavy first read for someone evaluating adoption.

## 10. No dashboard/UI — state is only visible through CLI/agent conversation

**What should happen:** A lightweight, read-only view (even a static
generated page) of adoption state: which skill families are wired in,
lock-file drift status, audit-log history.

**Why:** Every interaction — including `/magpie-setup verify`'s drift
report — is mediated through an agent conversation. Non-technical
maintainers or PMC members who want a quick health check have no
web/dashboard option today (`dashboard-generator` exists under `tools/`,
but it's not documented as covering setup/adoption state).

## 11. Prompt-injection defenses rely on documented convention, not enforced mechanism

**What should happen:** Where feasible, promote the "external content is
data, never instructions" rule (`PRINCIPLES.md` §0) from a documented
convention that each skill's authors must remember, to something checked
by tooling (e.g., `skill-and-tool-validator`) across all skills and agent
runtimes.

**Why:** The rule's enforcement today is "documented at the framework
level, enforced at the skill level" (`MISSION.md`) — i.e., by convention
in each `SKILL.md`, not by a runtime guarantee independent of the
interpreting agent's fidelity to the instructions.

## 12. Skill maturity is uneven, and it's not always obvious at invocation time

**What should happen:** Surface each skill's `stable` / `experimental`
status inline at invocation (not just in `docs/modes.md`), so a maintainer
knows they're running something unproven before they run it.

**Why:** Per `docs/modes.md`, only the security family is broadly
`stable`; pr-management, issue-management, mentoring, pairing,
contributor-growth, and repo-health are largely `experimental`. That's a
reasonable state for a pre-release framework, but the maturity signal
should travel with the skill, not live only in a separate doc.

## 13. No published cost/economics data yet

**What should happen:** Ship the "what does each mode actually cost to
run" page `MISSION.md` commits to (token counts per invocation, per mode,
per model class).

**Why:** Without this, an adopter evaluating Magpie against a
frontier-model subscription has no way to estimate ongoing cost before
committing skill families.

## 14. Backend/tracker parity is unclear across GitHub vs. Jira/Bitbucket/SourceHut

**What should happen:** A parity matrix showing which skill families are
fully supported on which tracker/VCS backend.

**Why:** `tools/` ships `jira`, `bitbucket`, `sourcehut`, and `jira-patch`
bridges, but the documentation structure (`docs/pr-management/`,
`docs/issue-management/`, etc.) reads GitHub-first throughout; it's not
clear from the docs alone how much of each skill family actually works
end-to-end on the non-GitHub backends today.

## Out of scope for this issue

- Re-litigating the snapshot + agentic-override adoption *model* itself
  (see `PRINCIPLES.md` §13 and `RFC-AI-0006`) — the concern here is the
  friction of *executing* that model, not the model's design.
- Anything requiring ASF Board action (naming, resolution, infra
  provisioning) — those are tracked by the MISSION.md process already.

## Suggested next steps

This issue is intentionally broad — a gap-analysis snapshot rather than a
single actionable change. Maintainers may want to:

1. Confirm which of the above are already tracked elsewhere (e.g., the
   "open adapter issues" referenced in `docs/prerequisites.md` likely
   already cover #3).
2. Split items #1, #9, and #10 into standalone `change_proposal` issues —
   these look like the highest-leverage adoption-friction fixes.
3. Treat #2, #5, #6, #7, #8 as tracked-but-not-actionable-yet, pending the
   framework's own release/TLP timeline.

## References

- `README.md` — "How adoption works" / "Adopting the framework"
- `MISSION.md` — Initial Goals, Vendor neutrality section
- `PRINCIPLES.md` — §0, §7, §8, §9, §13
- `docs/modes.md` — status legend and per-skill maturity table
- `docs/setup/install-recipes.md`
- `docs/prerequisites.md`
- `skills/setup/SKILL.md`
