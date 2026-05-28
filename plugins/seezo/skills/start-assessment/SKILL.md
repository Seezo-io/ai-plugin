---
name: start-assessment
description: |
  Start a Seezo security assessment on a markdown spec / design doc / RFC /
  plan BEFORE any implementation code is written, then inject the
  resulting risks into the main agent's context.

  MANDATORY AUTO-TRIGGER. Run the moment the user signals they're about
  to implement work described in a planning/design artifact. Triggers:
  "implement this plan", "let's build it", "start coding this", "ship
  this spec", "lgtm, proceed"; acceptance of an ExitPlanMode plan; the
  user pointing at a tech spec, RFC, PRD, ADR, one-pager, or any `.md`
  planning artifact under `docs/`, `specs/`, `plans/`, `design/`,
  `rfcs/`, or the repo root and asking to build against it.

  Do NOT write or edit implementation code until this skill has produced
  `.seezo/{assessment_id}.md` and that file is in context.

  Also runs on explicit request: "start assessment", "run a Seezo scan",
  "scan with seezo", "run a Seezo assessment", "check this spec with Seezo".
metadata:
  vendor: seezo
  version: "0.6.0"
---

# start-assessment

Host-agnostic skill. Works in Claude Code, GitHub Copilot, or any
MCP-aware agent. Tool calls below name tools on the `seezo`
MCP server (`list_projects`, `create_assessment`,
`get_assessment_details`, `list_requirements`,
`save_plugin_telemetry`); the host resolves them to its local
invocation syntax.

## Run-level state to keep in memory

Capture these once at the top of the pass:

- `INVOKED_AT` — ISO-8601 UTC timestamp at skill entry. Used to compute
  `duration_ms` for `skill_completed` / `skill_failed`.
- `HOST` — `claude-code` / `copilot` / `cursor` / `unknown`, detected
  from the runtime environment. The MCP fills the telemetry `host`; do
  not send local env values.
- `PROJECT_ID` — resolved in Step 1.5 below (cached in
  `.seezo/config.json` across scans, or picked via `list_projects`).
- `ASSESSMENT_ID` — `null` at invocation; bound from the
  `create_assessment` response once it returns 2xx.
- `WORKSPACE` — `{ branch, dirty }`, best-effort from
  `git rev-parse --abbrev-ref HEAD` and `git status --porcelain`.
  Written into the assessment markdown footer for the human reviewer and
  allowed in telemetry. Never send repo URLs, paths, or commit shas in
  telemetry.

## Picking the doc(s)

The skill supports one or more markdown docs as input. Each chosen doc
becomes a separate `{ name, content }` entry in the `create_assessment`
`documents` array (see Step 2); the server concatenates them
internally — do not pre-concat or insert header separators yourself.

1. **If the user named one or more paths**, use exactly those. Stop
   searching. Multiple paths from the user are accepted as-is — do not
   re-prompt.
2. **If they did not**, identify candidate planning/design markdown
   files:
   - Files modified in the current conversation that end in `.md`
   - Files under `docs/`, `specs/`, `plans/`, `design/`, `rfcs/`, or repo
     root with names like `plan.md`, `spec.md`, `tech-spec.md`,
     `product-spec.md`, `PRD.md`, `RFC.md`, `adr-*.md`, `one-pager.md`,
     `brief.md`, `design.md`
3. **One candidate** → use it; tell the user which file you picked in one
   line.
4. **Multiple candidates** → present the list and ask the user to pick
   **one or more** (multi-select is allowed and expected when the docs
   are complementary — e.g. a PRD plus a tech spec plus an ADR for the
   same feature). Make it clear in the prompt that they can select
   several. Stop until they answer. Accept any non-empty subset of the
   candidate list.
5. **No candidates** → emit `skill_failed` with
   `error_class=validation`, tell the user the skill needs a markdown
   spec, and stop. Do not invent one.

## Flow

### Step 0 — Emit `skill_invoked`

Before any I/O — before reading candidate docs, before resolving paths,
before `list_projects`, and before any user prompt — emit a single
`skill_invoked` telemetry event via `save_plugin_telemetry` on the
`seezo` MCP server with:

- `skill`: `"seezo-scan"`
- `event`: `"skill_invoked"`

`assessment_id` is not yet known, so omit it. Telemetry is best-effort:
swallow failures, do not retry, and do not block the scan on a telemetry
error. See "Telemetry during the run" below for the full contract.

### Step 1 — Validate the input

For each chosen path, confirm it exists and ends in `.md`, then read its
contents. If any selected file is missing, empty, or not markdown, stop
and report which one — do not silently drop it.

### Step 1.5 — Resolve `project_id`

`create_assessment` requires a `project_id`. **Always** call
`list_projects` and **always** ask the user to pick a project — never
auto-pick, never reuse a previously cached `project_id` from
`.seezo/config.json`.

1. Call `list_projects` on the `seezo` MCP server. If `search` would
   narrow the list (e.g. the user mentioned a project name in
   passing), pass it; otherwise omit.
2. **One or more results** → present `{ id, name, short_name,
   description }` and ask the user to pick exactly one, even when only
   a single project is returned. Stop until they answer. Persist the
   chosen `project_id` into `.seezo/config.json` so downstream tools
   (e.g. `validate-implementation`) can re-use it, but always re-prompt
   on the next `start-assessment` run.
3. **Zero results** → emit `skill_failed` with
   `error_class=not_found`, tell the user there are no projects
   accessible to their account, and stop. Do not invent a `project_id`.

If `list_projects` raises, emit `skill_failed` with the matching
`error_class` and report the error verbatim so the user can retry.

Bind `${PROJECT_ID}` for the rest of the pass.

### Step 2 — Create the assessment

Call `create_assessment` on the `seezo` MCP server with:

- `project_id`: `${PROJECT_ID}` from Step 1.5.
- `assessment_name`: a short human-readable name. For a single doc,
  derive from its H1 or filename. For multiple docs, derive from the
  shared feature they describe (use the user's wording if they gave
  one, else the H1 of the primary/first doc). Keep under 80 chars.
- `documents`: a list of `{ name, content }` entries — one per chosen
  markdown file. `name` is a short label (e.g. `'PRD'`, `'tech-spec'`,
  the basename without extension) used to help the LLM keep the docs
  straight; `content` is the full body of the file. The MCP creates
  one `rich_text_md` resource per entry and the assessment service
  concatenates them server-side — do not pre-concat or add header
  separators yourself.

The response is the redirect URL of the new assessment. Extract the
`assessment_id` from the URL (`/projects/{project_id}/assessments/{assessment_id}`)
and bind `${ASSESSMENT_ID}`. **Never generate `assessment_id` locally.**
If the response is unparseable or the URL doesn't carry an
`assessment_id`, emit `skill_failed` with `error_class=validation`, stop,
and report the raw response.

If `create_assessment` raises (network, auth, server), propagate the
error verbatim so the user can retry, and emit `skill_failed` with the
matching `error_class`.

### Step 3 — Persist scan bookkeeping

- Create `.seezo/` in the repo root if it doesn't exist.
- Write/update `.seezo/config.json`, preserving any existing keys:
  ```json
  { "active_assessment_id": "${ASSESSMENT_ID}", "project_id": "${PROJECT_ID}" }
  ```
- Validate the config.json is updated/created properly
- Print one line to the user: the `assessment_id`, the `assessment_url`,
  and a note that monitoring is starting in the background.

### Step 4 — Monitor status in the background

Kick off a **background task** that polls `get_assessment_details` until
the assessment finishes. The main agent must not block the user while
waiting.

- In Claude Code: dispatch via the `Agent` tool with
  `subagent_type: "general-purpose"` and `run_in_background: true`. The
  background agent owns the polling loop, the risk-list fetch in
  step 5, the markdown write in step 6, and the terminal telemetry event
  (`skill_completed` on the happy path, `skill_failed` on failed /
  timeout / fetch / write error). When constructing the
  background-agent prompt, pass through `ASSESSMENT_ID`, `PROJECT_ID`,
  `INVOKED_AT`, `HOST`, `WORKSPACE`, and the telemetry contract from
  "Telemetry during the run".
- In other hosts (Copilot, etc.): use the host's equivalent background /
  async-task primitive. If none exists, fall back to a foreground poll
  loop with sparse user-visible updates.

Polling rules for the background task:

- Call `get_assessment_details` with `{ project_id: "${PROJECT_ID}",
  assessment_id: "${ASSESSMENT_ID}" }`. The response includes the
  current `state` (one of `initiated`, `evaluating`, `completed`,
  `failed`), risk_ranking, feature, scope, and any partial-failure
  messages.
- Inspect `state`:
  - `initiated` / `evaluating` / any non-terminal → sleep ~15s and
    poll again.
  - `completed` → proceed to step 5.
  - `failed` → write the failure summary to
    `.seezo/${ASSESSMENT_ID}.md` (see step 6 for format) under a
    `Status: failed` header, emit `skill_failed` with the closest
    `error_class`, then stop. Surface the failure to the main agent.
- Hard cap: 30 minutes (120 polls at 15s). If the cap is reached without
  a terminal state, write a `Status: timeout` markdown, emit
  `skill_failed` with `error_class=timeout`, and stop.

### Step 5 — Pull the risk list

Once `state == "completed"`, the background task calls
`list_requirements` with `{ project_id: "${PROJECT_ID}",
assessment_id: "${ASSESSMENT_ID}" }`. ("Requirement" is the API name
for what the product calls a "risk".) Each entry includes `id`,
`title` (or `name`), `severity`/`ranking`, `state`, `confidence`, and
a description / rationale. Combine this list with the metadata already
captured from `get_assessment_details` in Step 4 to render the
markdown in Step 6.

If `list_requirements` raises, emit `skill_failed` with the matching
`error_class` and surface the error to the main agent.

If a specific risk needs deeper context for the validation hints
(citations, decision trail, related context, applicable standards),
the `get_requirement_*` tools are available — but only call them if
the light list is genuinely insufficient. The default is to ship the
markdown off the light list.

### Step 6 — Save synthesized markdown

Write the synthesized assessment to `.seezo/${ASSESSMENT_ID}.md`. This is
the single artifact the main agent will read for context. Suggested
layout (use the fields the MCP responses actually returned; omit empty
sections):

```markdown
# Seezo Assessment — {feature_name}

- **assessment_id**: {assessment_id}
- **project_id**: {project_id}
- **assessment_url**: {assessment_url}
- **state**: {state}
- **risk_ranking**: {risk_ranking}   (from `get_assessment_details`)
- **completed_at**: {completed_at}   (if present in details response)
- **workspace**: { branch, dirty }   (from captured `WORKSPACE`)
- **source_specs**: list of every markdown path passed into the
  `documents` array

## Risks

For each requirement returned by `list_requirements`:

### {requirement.title} ({requirement.ranking or severity})
- **risk_id**: {requirement.id}
- **state**: {requirement.state}
- **confidence**: {requirement.confidence}
- **rationale**: {requirement.description or rationale}
- **validation hints**: bullet list of where in the codebase to verify

## Partial failures / errors

(verbatim from any `partial_failures` field on the
`get_assessment_details` response, only if non-empty)
```

Rules for this file:
- Plain markdown, no JSON dumps. Summarise each risk in <10 lines.
- Do not include secrets, tokens, or anything outside what the API
  returned.
- Always include risk id (the `requirement.id` from `list_requirements`).
- File path is exactly `.seezo/${ASSESSMENT_ID}.md` — one file per scan,
  no nested directory.

If the markdown write raises, emit `skill_failed` with the matching
`error_class` and surface the error to the main agent.

### Step 7 — Emit `skill_completed`

After `.seezo/${ASSESSMENT_ID}.md` is written, emit one
`skill_completed` telemetry event via `save_plugin_telemetry` with:

- `skill`: `"seezo-scan"`
- `event`: `"skill_completed"`
- `assessment_id`: `${ASSESSMENT_ID}`
- `outcome`: `ok`
- `duration_ms`: now minus `INVOKED_AT`
- `workspace`: `{ branch, dirty }` from the captured `WORKSPACE`

This is the background task's last action before returning a summary to
the main agent.

### Step 8 — Inject context into the main agent

The background task signals completion by:

1. The markdown file existing on disk at `.seezo/${ASSESSMENT_ID}.md`.
2. Returning a short summary (top risk ranking + risk count + the file
   path) so the main agent picks it up in its next turn.

When control returns to the main agent for implementation work, it MUST:

- Read `.seezo/${ASSESSMENT_ID}.md` in full before writing any code.
- Treat each risk + recommended mitigation as a hard constraint on the
  implementation. Call out in user-facing text which risks the
  implementation is addressing as you go.

If the main agent is already past the background-task fork point when
results arrive, it should still pause and read the file before
proceeding.

## Telemetry during the run

The skill emits basic lifecycle events to the `save_plugin_telemetry`
tool on the `seezo` MCP server. Each call is a thin event; the MCP fills
`event_id`, `ts`, `schema_version`, `skill`, `host`, `mcp_version`, and
`session_token`. Users can opt out by setting
`SEEZO_TELEMETRY_DISABLED=1` in the MCP environment.

Every call MUST set `skill: "seezo-scan"` explicitly. Do not rely
on the MCP default; it has misattributed scan events in the past.

Emit events at exactly these moments:

| When | event | Required volatile fields |
|---|---|---|
| Skill enters, before any I/O | `skill_invoked` | `skill="seezo-scan"` |
| After `.seezo/${ASSESSMENT_ID}.md` is written and the background task is wrapping up | `skill_completed` | `skill="seezo-scan"`, `assessment_id`, `outcome=ok`, `duration_ms`, `workspace` |
| `create_assessment` failed, polling hit `failed` / `timeout`, `get_assessment_details` / `list_requirements` raised, or markdown write raised | `skill_failed` | `skill="seezo-scan"`, `outcome=error`, `error_class`, `duration_ms`, `assessment_id` if it was already bound |

`error_class` is one of:
`network` / `timeout` / `auth` / `validation` / `not_found` /
`rate_limited` / `server` / `unknown`. Pick the closest match; never
invent a new value.

Ownership:

- The main agent owns `skill_invoked`, `skill_failed` for any failure
  before the background task is dispatched (including
  `create_assessment` errors), and any pre-dispatch validation failures.
- The background task owns `skill_completed` and any late `skill_failed`
  events for polling, fetch, and markdown-write errors.

Telemetry is best-effort. Swallow failures, never let telemetry block the
skill, and never await or retry the failure path. Calls are sequential
because the MCP serialises ordering; do not parallelise telemetry calls.

Privacy contract: telemetry events MUST carry only fields listed in the
table above: UUIDs, counters, enums, `workspace.dirty`, and
`workspace.branch`. Never send doc contents, file paths, risk titles,
rationales, recommended mitigation text, snippet content, env values,
repo URLs, commit shas, or prompt content.

## Hard rules

- No writes outside `.seezo/`.
- The only `seezo` MCP tools this skill may call are `list_projects`,
  `create_assessment`, `get_assessment_details`, `list_requirements`,
  and (only if a specific risk needs deeper context) the
  `get_requirement_*` family, plus `save_plugin_telemetry` for
  best-effort lifecycle events.
- `save_plugin_telemetry` is best-effort: never await or retry its
  failure path, and never let it block the skill. Telemetry events MUST
  carry only the fields listed in the "Telemetry during the run" table.
- `assessment_id`, `project_id`, and `risk_id` are always
  server-issued. Never invent them, even as placeholders.
- Do not write or modify implementation code in the same turn as the
  initial `create_assessment` call. The assessment runs first; coding
  resumes only after `.seezo/${ASSESSMENT_ID}.md` is on disk and read.
- If the MCP server is unreachable, do not create `.seezo/` at all (we
  have no `assessment_id`). Report the error verbatim so the user can
  retry.
- The auto-trigger contract is non-negotiable: if a plan or design doc
  was just generated and the user signals to build it, run this skill
  first. Do not skip it because the change "looks small".
