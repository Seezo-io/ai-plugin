---
name: validate-implementation
description: |
  Validate that the implemented code addresses every risk and abuse
  case flagged by a prior `start-assessment`. Reads
  `.seezo/{assessment_id}.md`, walks the working tree, decides a
  verdict per risk (`pass` / `fail` / `unclear`) with evidence, writes
  `.seezo/{assessment_id}.validation.md`, and posts the summary back so
  the main agent can decide whether to ship.

  MANDATORY AUTO-TRIGGER. Run the moment the implementation pass for a
  Seezo-assessed spec is complete and before the work is declared done.
  Triggers include (but are not limited to):
    - "we're done", "implementation is complete", "ready to ship",
      "wrap this up", "ready for review", "PR is ready"
    - the user asking for a final review before merge
    - the assistant about to summarise a feature as finished while
      `.seezo/config.json` has an `active_assessment_id`

  Also runs on explicit request: "validate implementation",
  "run validate-implementation", "run seezo-validate", "validate against
  the Seezo assessment", "check the code against Seezo".
metadata:
  vendor: seezo
  version: "0.6.0"
---

# validate-implementation

Host-agnostic skill. Works in Claude Code, GitHub Copilot, or any
MCP-aware agent. Tool calls below name tools on the `seezo`
MCP server (`get_assessment_details`, `list_requirements`,
`save_plugin_telemetry`); the host resolves them to its local invocation
syntax.

## Run-level state to keep in memory

At the top of the pass, capture these once and carry them through every
step:

- `ASSESSMENT_ID` — chosen via the picker below.
  `.seezo/${ASSESSMENT_ID}.md`. **Required**; if it's missing, stop and
  tell the user to re-run `start-assessment` to refresh the file. Do
  not synthesise one.
- `RUN_ID` — generate one UUID v4 at the start of the pass and reuse it
  for every verdict and every telemetry event. Two passes against the
  same assessment must have different `run_id`s. This is the only UUID
  this skill generates locally.
- `RAN_AT` — current ISO-8601 UTC timestamp at the start of the pass
  (written into the validation markdown footer).
- `HOST` — `claude-code` / `copilot` / `cursor` / `unknown`, detected
  from the runtime environment. The MCP fills the telemetry `host`; do
  not send local env values.
- `WORKSPACE` — `{ repo, branch, commit, dirty }` (see "Capturing
  workspace metadata" below). Telemetry may only include
  `{ branch, dirty }`; never send repo URLs, paths, or commit shas in
  telemetry.
- `STATS` — per-risk counters (`files_read`, `grep_calls`) you increment
  while investigating each risk. Telemetry carries the counters, never
  file paths or snippets.

## First — emit `skill_invoked`

Before any I/O — before reading `.seezo/config.json`, before listing
`.seezo/*.md`, before prompting the user, and before reading assessment
markdown — emit a single `skill_invoked` telemetry event via
`save_plugin_telemetry` on the `seezo` MCP server with:

- `skill`: `"seezo-validate"`
- `event`: `"skill_invoked"`
- `run_id`: `${RUN_ID}`
- `assessment_id`: only if the user already named one; otherwise omit

Telemetry is best-effort: swallow failures, do not retry, and do not
block validation on a telemetry error. See "Telemetry during the run" for
the full contract.

## Picking which assessment to validate against

1. **Explicit `assessment_id` from the user** → use it.
2. **Else** read `.seezo/config.json` and use `active_assessment_id` if
   present.
3. **Else** look at `.seezo/*.md` files (every file at the top level of
   `.seezo/`, excluding `*.validation.md`). Each one represents a prior
   assessment.
   - **One file** → use its basename (without `.md`) as `assessment_id`.
   - **Multiple files** → present the list with each file's H1 / feature
     name and ask the user to **pick exactly one** (single-select). Stop
     until they answer.
4. **None of the above** → emit `skill_failed` with
   `error_class=not_found`, tell the user there is no assessment to
   validate against, and offer two options:
   - Run `start-assessment` first against the relevant spec, OR
   - Provide an explicit `assessment_id` to fetch from the server.

   Present these as a clear two-option selection. Stop until they
   choose. Do not invent an `assessment_id`.

## Loading the assessment

Source of truth is `.seezo/${ASSESSMENT_ID}.md` produced by `start-assessment`.
Read it in full and extract:

- For each risk block: `title`, `severity`, `category`, `rationale`,
  `recommended_mitigation`, validation hints, **and** the `risk_id`
  line. Bind each to its block.
- Same for abuse cases (`abuse_case_id`).

If the file is missing any `risk_id`, the markdown is stale
(written by an older `start-assessment`). Tell the user to re-run
`start-assessment` to refresh the file, emit `skill_failed` with
`error_class=validation`, and stop. **Do not invent UUIDs.**

After parsing `.seezo/${ASSESSMENT_ID}.md`, emit one `assessment_loaded`
telemetry event with `run_id`, `outcome=ok`, and `duration_ms` measured
from `RAN_AT`.

If `${ASSESSMENT_ID}` was provided by the user but no local markdown
exists, re-materialise the file via the `seezo` MCP server before
proceeding: call `get_assessment_details({ project_id, assessment_id })`
for the metadata + state, and `list_requirements({ project_id,
assessment_id })` for the risk list. Resolve `project_id` from
`.seezo/config.json` (or ask the user). Write the synthesised result to
`.seezo/${ASSESSMENT_ID}.md` first — the validation always runs off the
local file.

## Source of truth: the working tree as it is

Validate against the **current working tree exactly as it sits on disk**,
including any uncommitted, unstaged, or untracked changes. Do not:

- ask the user to commit, stash, or clean their tree before running,
- run `git stash`, `git checkout`, `git reset`, or anything else that
  could mutate the tree,
- skip files because they are dirty / untracked,
- compare against `HEAD` or any other ref to decide what to read — read
  the file contents directly.

A dirty tree is the expected case: this skill runs at the end of an
implementation pass, before the user commits.

## Capturing workspace metadata

Best-effort, treat failures as `null`. These are metadata only; they
do not gate execution:

- `git rev-parse HEAD` — current commit (may be stale relative to the
  working tree)
- `git rev-parse --abbrev-ref HEAD` — branch
- `git remote get-url origin` — remote
- `git status --porcelain` — record whether the tree is dirty (count of
  changed / untracked paths). Carry this into the output as
  `workspace.dirty: true|false` so the human reviewer knows the
  validation captured uncommitted work.

The skill still proceeds for non-git workspaces.

## Per-risk validation

For each risk (and each abuse case) listed in the assessment markdown:

1. **Locate.** Decide where in the repo the mitigation would live. Start
   from the `validation hints` bullets in the assessment file. Prefer
   `rg` (ripgrep) over `find`. Read suspect files in full when they are
   under ~400 lines. Increment `STATS.files_read` for each file you
   open and `STATS.grep_calls` for each `rg` invocation; telemetry
   carries these counters per risk, never the file paths.
2. **Verdict.** One of:
   - `pass` — code clearly implements the mitigation.
   - `fail` — code does not address the risk, or addresses it
     incorrectly.
   - `unclear` — not enough signal in the repo to decide.
3. **Evidence.** Collect concrete pointers: `[{ path, lines, snippet }]`.
   Snippets MUST be ≤20 lines each. Never include secrets, tokens, or
   paths outside the repo.
4. **Checkpoint.** Append the verdict to
   `.seezo/${ASSESSMENT_ID}.validation.md` immediately after deciding it
   (see layout below). This makes the skill safe to restart after a
   crash or rate-limit — on resume, skip risks that already have a
   verdict block.
5. **Telemetry.** After the verdict is appended to disk, emit one
   `risk_validated` telemetry event carrying `risk_id`, `run_id`,
   `verdict`, `evidence_count`, the per-risk `files_read` /
   `grep_calls` counters, and `duration_ms` for this risk. Then reset
   the per-risk counters.

After every risk has a verdict block on disk, calculate
`verdict_counts` (`pass`, `fail`, `unclear`, `total`) and emit one
`skill_completed` telemetry event with `assessment_id`, `run_id`,
`verdict_counts`, `duration_ms`, and `workspace`.

The local `.seezo/${ASSESSMENT_ID}.validation.md` is the sole source of
truth for verdicts. There is no server persistence step.

## Output file: `.seezo/${ASSESSMENT_ID}.validation.md`

Single markdown file, written incrementally. Suggested layout:

```markdown
# Seezo Validation — {feature_name}

- **assessment_id**: {assessment_id}
- **project_id**: {project_id}
- **assessment_url**: {assessment_url}
- **assessment_file**: .seezo/{assessment_id}.md
- **ran_at**: {ISO-8601 timestamp}
- **workspace**: { repo, branch, commit, dirty }   (omit fields that
  are null; `dirty: true` means the validated tree had uncommitted
  changes — this is expected)

## Summary

| Verdict | Count |
| --- | --- |
| pass | N |
| fail | N |
| unclear | N |
| **total** | N |

## Verdicts

### {risk.title} — {verdict}
- **severity**: {risk.severity}
- **category**: {risk.category}
- **rationale**: short reason for the verdict
- **evidence**:
  - `path/to/file.ext:L42-L51`
    ```
    <≤20-line snippet>
    ```
  - ... (additional evidence entries)
- **gap** (only when verdict is `fail` or `unclear`): one paragraph on
  what is missing and the next step to close it.

(repeat per risk; then a separate `### Abuse case: ...` block for each
abuse case, same shape)
```

Rules:

- Writes are confined to `.seezo/`. Do not modify any other file.
- File path is exactly `.seezo/${ASSESSMENT_ID}.validation.md`. One
  validation file per assessment; overwrite on a fresh full run, append
  per-risk during a resumed run.

## Telemetry during the run

The skill emits lifecycle events to the `save_plugin_telemetry` tool on
the `seezo` MCP server. Each call is a thin event; the MCP fills
`event_id`, `ts`, `schema_version`, `skill`, `host`, `mcp_version`, and
`session_token`. Users can opt out by setting
`SEEZO_TELEMETRY_DISABLED=1` in the MCP environment.

Every call MUST set `skill: "seezo-validate"` explicitly. Do
not rely on the MCP default; explicit skill attribution keeps this
symmetric with `start-assessment`.

Emit events at exactly these moments:

| When | event | Required volatile fields |
|---|---|---|
| Skill enters, before any I/O | `skill_invoked` | `skill="seezo-validate"`, `run_id`, `assessment_id` if already known |
| After parsing `.seezo/{aid}.md` | `assessment_loaded` | `skill="seezo-validate"`, `assessment_id`, `run_id`, `outcome=ok`, `duration_ms` |
| After each per-risk verdict is appended to disk | `risk_validated` | `skill="seezo-validate"`, `assessment_id`, `risk_id`, `run_id`, `verdict`, `evidence_count`, `files_read`, `grep_calls`, `duration_ms` |
| Happy-path end | `skill_completed` | `skill="seezo-validate"`, `assessment_id`, `run_id`, `verdict_counts`, `duration_ms`, `workspace` |
| Top-level catch | `skill_failed` | `skill="seezo-validate"`, `run_id`, `outcome=error`, `error_class`, `duration_ms` |

`error_class` is one of:
`network` / `timeout` / `auth` / `validation` / `not_found` /
`rate_limited` / `server` / `unknown`. Pick the closest match; never
invent a new value.

Server persistence is not part of this skill today. If
`save_risk_validations` or an equivalent persist tool is reintroduced,
also emit these events around that call:

| When | event | Required volatile fields |
|---|---|---|
| Just before the persist call | `save_attempted` | `skill="seezo-validate"`, `assessment_id`, `run_id`, `verdict_counts` |
| Persist returned 2xx | `save_succeeded` | `skill="seezo-validate"`, `assessment_id`, `run_id`, `outcome=ok`, `duration_ms` |
| Persist threw | `save_failed` | `skill="seezo-validate"`, `assessment_id`, `run_id`, `outcome=error`, `error_class`, `duration_ms` |

On `save_failed`, append
`> Server persist failed: <error>`
as a footer to `.seezo/${ASSESSMENT_ID}.validation.md`; the local file
stays the source of truth.

Telemetry is best-effort. Swallow failures, never let telemetry block the
skill, and never await or retry the failure path. Calls are sequential
because the MCP serialises ordering; do not parallelise telemetry calls.

Privacy contract: telemetry events MUST carry only fields listed in the
tables above: UUIDs, counters, enums, `workspace.dirty`, and
`workspace.branch`. Never send rationale, gap, snippet content, file
paths, risk titles, recommended mitigation text, repo URLs, commit shas,
env values, or prompt content.

## Posting results back to the main agent

When validation finishes, return a short, structured summary to the main
agent so it can decide whether to ship:

- Counts: `pass` / `fail` / `unclear` / `total`.
- The list of `fail` risks by title, each with a one-line gap summary.
- The list of `unclear` risks by title (these still block "done" unless
  the user explicitly accepts the uncertainty).
- The path `.seezo/${ASSESSMENT_ID}.validation.md` for the full file.
- The `assessment_url` for the human reviewer.

The main agent MUST:

- Read `.seezo/${ASSESSMENT_ID}.validation.md` in full when any verdict
  is `fail` or `unclear`.
- Not declare the work complete while any `fail` exists. Surface the
  gaps to the user and propose fixes.
- Treat `unclear` verdicts as a prompt to investigate further or to ask
  the user for sign-off on the ambiguity — not as silent passes.

## Hard rules

- This skill is **read-only** over the working tree. Writes are only to
  `.seezo/`.
- The only `seezo` MCP tools this skill may call are
  `get_assessment_details` and `list_requirements` only when
  re-materialising a remote assessment whose markdown is missing locally,
  plus `save_plugin_telemetry` for best-effort lifecycle events.
- `assessment_id`, `project_id`, every `risk_id`, and every
  `abuse_case_id` are always server-issued. Never fabricate any of
  them, even as placeholders.
- `run_id` is the only UUID the skill generates locally — exactly one
  per pass, reused across all events and verdicts.
- `save_plugin_telemetry` is best-effort: never await or retry its
  failure path, and never let it block the skill. Telemetry events MUST
  carry only the fields listed in the "Telemetry during the run" tables.
- Snippets in evidence must be ≤20 lines; do not exfiltrate large file
  contents.
- If the assessment markdown cannot be loaded (file unreadable, MCP
  fetch fails), do not write a partial validation file. Emit
  `skill_failed` and report the error verbatim so the user can retry.
- The auto-trigger contract is non-negotiable: if there is an
  `active_assessment_id` and the implementation pass is wrapping up,
  this skill runs before the work is declared done. Do not skip it
  because the diff "looks small".
