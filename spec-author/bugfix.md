---
role: spec-author
kind: bugfix
status: draft-v0.1
draft_synthesized_from:
  - viloforge-projects/vafi/workgraphs/vafi-rolling-restart-fix/architect-retro.md
  - viloforge-projects/vafi/workgraphs/vafi-rolling-restart-fix/spec-author-retro.md
  - viloforge-projects/vafi/workgraphs/vafi-rolling-restart-fix/verifier-retro.md
  - viloforge-projects/vafi/spikes/canary-end-to-end-2026-05-14/SPIKE-RETRO.md
last_revision: 2026-05-14
---

# Methodology — Spec-author, `kind: bugfix`

You are spec-authoring a single task within a `bugfix`-kind workgraph
in the ViloForge software factory. Your output is a populated task
spec file (`tasks/<NN>-<slug>.md`): frontmatter `acceptance_criteria`
array + a Spec body that an executor can implement against verbatim
and a judge can grade per-criterion.

This file is **first-draft, v0.1**. The patterns below come from
the **first** workgraph that ran SDD discipline (`vafi-rolling-
restart-fix`, 2026-05-14) and from the canary-end-to-end spike
that ran the same day. Refinement is expected after 2–3 more
workgraphs produce corroborating retros.

## Engineering principles (canonical: `viloforge-platform/docs/engineering-principles.md`)

This methodology operates under the ViloForge engineering north
star. **Read `engineering-principles.md` before applying the rules
below.** Concretizations for the spec-author role:

- **SOLID:** When authoring a spec, cite which SOLID principles are
  load-bearing for the work. If the task introduces a strategy or
  a repository, name the pattern in the Spec body so the executor
  and judge agree on the shape.
- **TDD red/green:** Every spec's `acceptance_criteria` includes
  tests at every applicable pyramid level (unit / integration /
  contract / scenario). A spec without test ACs is a methodology
  violation that the verifier (V13) MUST reject.
- **Design patterns:** Cite the patterns the implementation will
  use in the Spec body, with rationale. Avoid pattern theater —
  the patterns must earn their inclusion.
- **Extensibility:** For any task that produces extensible code
  (e.g., a new detector class, a new event type), include an AC
  that demonstrates the extension point works (e.g., "Adding a
  new $THING follows the documented pattern; existing tests pass
  unchanged").

These are non-negotiable per the north star. See §5.2 in
`engineering-principles.md` for the spec-author checklist.

## Non-negotiable rules

The four rules in this section are HIGH-maturity, observed across
the spike + verifier retros. Skip any of them and your spec will
either be rejected at verify phase or produce a ghost-completion
at execute phase.

### R1 — Externally-grounded acceptance criteria

For any task whose work product is an **external artifact** — a PR,
a deployment, a package publication, a file in a remote repo, an
endpoint that must respond, an image that must be pullable —
**at least one acceptance criterion MUST be an externally-grounded
assertion that requires reaching outside the workdir to verify.**

Workdir-only ACs are insufficient. The judge can only reason about
what's in front of it; if every AC is checkable from local state, a
ghost-completion (local commit but no push, local artifact but no
publish) will be rubber-stamped as `done`.

Acceptable forms:

- "A PR exists at `https://github.com/<owner>/<repo>/pulls?head=<branch>`."
- "Branch `<name>` is present on remote `origin`."
- "Image `<registry>/<image>:<tag>` is pullable via `docker pull`."
- "Endpoint `<url>` returns 200 with body matching `<schema>`."
- "Commit `<sha>` appears on the upstream `main` branch."

Empirical evidence: spike-1 had only workdir-checkable ACs → judge
approved a task that produced no remote artifact. Spike-2 had one
externally-grounded AC → judge correctly rejected (changes_requested)
when the artifact wasn't produced.

### R2 — `test_command` gate for every external artifact

Populate the task's `test_command` field with a shell command that
**mechanically verifies the external artifact exists.** This runs
in the controller (`controller/gates.py`) post-harness, in the task
workdir. Exit 0 → pass; non-zero → task transitions to `fail`, not
`pending_completion_review`.

Shape:

```yaml
test_command:
  command: <single shell string>
```

The current MVP gate runner takes a single command (no array). Make
the command compose any checks you need with `&&`:

```yaml
test_command:
  command: |
    git ls-remote --heads origin canary/<branch> 2>&1 | grep -q . &&
    gh pr view <branch> --repo <owner>/<repo> >/dev/null 2>&1
```

**Why both gate AND externally-grounded AC.** Gates catch outright
absence mechanically before the judge ever sees the task. ACs catch
subtler defects (PR exists but has wrong content) at the judge
layer. They're belt + suspenders, not redundant.

Empirical evidence: spike-1 had no gate; harness exit 0 was treated
as success despite no push. Spike-2 had a branch-exists gate; gate
forced the harness to actually push.

### R3 — "Fail-loud" language in the Spec body

In the Spec body's `implementation.approach`, **explicitly require
the executor to fail honestly if it cannot complete a required
step.** Without this language, the harness has been observed to
self-rationalize gaps ("would be done in production environment
with proper credentials") and submit as success.

Sample wording (use verbatim where applicable):

> *If any required step (push, PR creation, deployment, etc.)
> cannot be completed due to missing credentials, missing tools, or
> blocked external dependencies: report the failure explicitly in
> your completion notes — do not rationalize partial completion as
> success. The judge will fail tasks that report dishonest success;
> tasks that report failure honestly can be reworked.*

Empirical evidence: spike-2's spec included a fail-loud directive;
the executor honestly named the PR-creation obstacle in its notes;
the judge correctly issued changes_requested instead of approving.
Spike-1's spec lacked this language; the executor rationalized the
gap and the judge rubber-stamped.

### R4 — file:line citations, not bare file paths

Every reference to existing code in the Spec body or References
section MUST cite the line range, not just the file path. Use
`controller.py:81-95`, not `controller.py`.

Why: bare paths force the executor to grep; line-range citations
put them in the right spot immediately, and drift surfaces faster
(a line range that no longer makes sense is a visible defect).

Spot-check: read each cited line range before submitting the spec.
A wrong line range is a verify-phase defect (V3 pattern) that you
can prevent here.

## Required spec shape (use as template)

```markdown
---
id: t_<6 alnum>
slug: <kebab-case>
title: <one-line description>
workgraph: <slug>
order: <int>
required_tags:
  - claude            # R4: pin one harness — [claude] or [pi].
                      # Never bare [executor] (matches no pool post-R4).
depends_on:
  - <other task ID>   # if any
target_repo: <owner>/<repo>
agent_model: claude-sonnet-4-20250514   # or appropriate model
judge: true
test_command:
  command: |
    <gate that verifies the EXTERNAL artifact, NOT just local state>
created_at: <ISO timestamp>
acceptance_criteria:
  - AC-T<N>-1 — <internally checkable AC>
  - AC-T<N>-2 — <internally checkable AC>
  - AC-T<N>-3 — <externally-grounded AC> (REQUIRED by R1)
  - AC-T<N>-4 — <test/contract AC>
---

# Spec

## Files touched

- `<path>:<lines>` — <what changes>

(NOT touched: <explicit list — see R5 below for OOS discipline>)

## Implementation sketch

(Concrete enough that an executor can implement against it. Cite
existing code patterns by file:line per R4. Use realistic stub
code snippets when needed.)

## Fail-loud directive (R3 — verbatim where applicable)

If any required step (push, PR creation, deployment, etc.) cannot
be completed due to missing credentials, missing tools, or blocked
external dependencies: report the failure explicitly in your
completion notes — do not rationalize partial completion as
success.

# Constraints

(Non-negotiable invariants the implementation must hold.)

# Out of scope

(Things explicitly NOT part of this task. R5 discipline.)

# References

- workgraph.md / plan.md (this directory)
- <file:line citations to existing code>
- <upstream design doc references>
```

## Other high-value patterns

### R5 — "Out of scope" is normative, not decorative (SA5)

Every spec carries an "Out of scope" section. Entries there forbid
the executor from doing the listed things even when they look like
obvious next steps. The judge SHALL flag submissions that add
out-of-scope behavior as violations, not as helpful extras.

Use the section actively:

- Explicitly defer cross-cutting concerns ("multi-replica safety",
  "pagination", "periodic reconciliation")
- Document architect-level rejections from `plan.md`
- Capture rejected-but-tempting paths ("no SIGINT-specific handling",
  "no harness-resume work")

### R6 — Cross-task AC consistency (V2)

When your spec's ACs reference another task by ID or role ("T1's
PR", "the integration test"), verify the named task's ACs **agree**
with what you're saying. Pin-bump ordering, file conventions,
shared state — any of these can produce silent conflicts if not
cross-checked.

If an artifact has multiple plausible owners (e.g., the
pyproject.toml pin-bump in our T1+T2 case), use neutral language:
"whichever T1/T2 PR lands first carries the pin-bump." Don't bake
in an execution order the DAG doesn't enforce.

### R7 — Mock-double idiom matches the existing test scaffold (V1, SA4)

If your spec introduces a new protocol/interface method or modifies
an existing test double, **open the actual test-double class** in
the target repo and check:

- The mock convention in use (AsyncMock vs custom class vs manual
  tracking)
- The expected initialization pattern
- The assertion style existing tests use

Do not invent a new mock idiom. Match the scaffold's existing
conventions; if you can't tell what the convention is, ask the
operator (or in the autonomous-agent case, surface the ambiguity
as a finding before submitting).

### R8 — Use real IDs in frontmatter even when bodies are TBD (P10)

Generate stable task IDs (`t_<6 alnum>`) at workgraph creation time.
`depends_on:` edges reference these IDs; downstream stages rely on
them. Placeholder bodies are fine during ingestion; placeholder IDs
are not.

### R9 — Verify the FULL call chain, not just the server endpoint (P3 + P12)

For any external API/SDK/library/wire-call your spec names:

1. Server-side endpoint exists at the URL + verb claimed.
2. The SDK / client library the executor will use exposes that
   endpoint via a callable method.
3. The agent's pinned version of that SDK includes the method.

Skipping step 2 or 3 produces walk-backs at executor phase.
Empirical evidence: vafi-rolling-restart-fix architect missed the
SDK gap; it surfaced at spec-author entry and forced a new T0
task into the DAG (vtaskforge-targeted).

### R10 — Verify the test-delivery mechanism, not just the test environment (P13)

For test/validation tasks, separately verify:

1. **The test runtime** — the environment/tooling the test runs IN
   (kind cluster, browser harness, simulator, database fixture).
2. **The test delivery mechanism** — what causes the test to RUN
   when it should (CI pipeline, pre-commit hook, scheduled job,
   operator runbook, deployment gate).

When (1) exists but (2) doesn't, walk back honestly: "Makefile
target today; CI wiring as a follow-up `kind: infrastructure`
workgraph."

### R11 — Test acceptance criteria at every applicable pyramid level

For every task that produces code, the `acceptance_criteria`
list MUST include explicit test ACs at each pyramid level the
task touches:

- **Unit ACs** for any function/class added or modified.
- **Integration ACs** for any cross-component behavior added.
- **Contract ACs** for any public-API surface added (SDK methods,
  HTTP endpoints, event schemas).
- **Scenario ACs** for any user-flow-touching work; or include
  one named scenario test the work participates in.

Skipping a level when work touches it is a methodology violation.

Sample AC phrasing:

```yaml
acceptance_criteria:
  - "Unit: tests/unit/test_event_factory.py covers create + validate
     for the new event type, including error paths"
  - "Integration: tests/integration/test_event_persist_query.py
     verifies the event round-trips through Postgres + the read API"
  - "Contract: tests/contract/test_event_schema.py validates the
     event matches its schema version"
  - "Scenario: tests/scenario/test_anomaly_flow.py exercises this
     event as part of the stuck-detection scenario"
```

Empirical motivation: the canary spike on 2026-05-14 showed that
without explicit test ACs, harness "success" reduces to "exited 0"
— ghost-completion. Per-pyramid-level test ACs convert "success"
into "evidence at every relevant level."

### R13 — Pydantic mutation discipline (typed-model integrity)

When a spec body's implementation sketch mutates a typed Pydantic
model via `model_copy(update={...})` or any equivalent: **only use
keys that are declared fields on the target class**. Pydantic v2's
default `extra='ignore'` causes `model_dump()` to silently drop
undeclared keys, producing a ghost-write defect.

Empirical motivation: vfobs-foundation T4 originally had an
"Enricher" CoR node that stuffed `server_received_at` into
`event.data.model_copy(update={...})`. The data sub-model didn't
declare the field; verifier pass F1 caught it before implementation.

Discipline:
- If the field belongs on the model, **declare it** (Event base class
  or the specific Data sub-model) and add a corresponding storage
  column if it's persisted.
- If the field belongs outside the typed model (server-side audit,
  request-context state), put it **somewhere typed** — a column, a
  separate context object, a header — never a Pydantic-model
  undeclared key.
- Verify in the spec body with a sketch line: "model_dump() output
  includes the field."

Spec-author check before submission: any `model_copy(update={...})`
in the sketch — confirm every key in the update dict is a declared
field on the target class.

### R14 — External-dependency stability for test scenarios

When a spec names an external Helm chart, container image, or
package by version: **probe the named artifact is still pullable**
at spec-author time AND prefer official-image manifests over
chart-bundled side services for test scenarios.

Empirical motivation: vfobs-foundation T6 originally pinned
bitnami/postgresql Helm chart 15.5.20. Bitnami removed all
`bitnami/<image>:<tag>` images from Docker Hub mid-2025; chart
15.5.20 still references `docker.io/bitnami/postgresql:16.3.0-debian-12-r23`
which now 404s. Caught at executor stage; cost a pivot to a
minimal in-cluster `postgres:15-alpine` manifest.

Discipline for any named external artifact:
- Run a probe (`docker pull <image>`, `helm template <chart>`,
  `pip index versions <pkg>`) before locking the version.
- For scenario tests that need a stock side service (Postgres,
  Redis, etc.), prefer a minimal Kubernetes manifest using the
  official image over a third-party chart. Reduces moving-target
  risk dramatically.
- If a chart MUST be used (custom operator, complex topology),
  vendor the chart (copy it into the project repo) so a future
  upstream image-deprecation doesn't break the build.

This rule is HIGH-maturity within one workgraph — sufficient to draft,
but warrants validation in WG2-3 before locking.

### R12 — Cite design patterns the implementation will use

Spec body's "Implementation sketch" section names the patterns
applied (Strategy, Repository, Factory, etc.) with a one-sentence
rationale per pattern. Helps:

- The executor: implements the pattern as named, no surprises.
- The judge: grades the impl against the claimed pattern.
- Future readers: understand the design intent without reverse-
  engineering.

Anti-pattern: silent pattern substitution by the executor.

## Anti-patterns (do not do these)

- Write ACs that are checkable only against the local workdir. (R1)
- Leave `test_command` empty for tasks that produce external
  artifacts. (R2)
- Use soft language like "open a PR" without a fail-loud directive
  for cases where push/PR isn't possible. (R3)
- Cite file paths without line ranges. (R4)
- Specify a custom mock idiom without checking the existing scaffold. (R7)
- Bake DAG-execution-order assumptions into AC text. (R6)
- Name an SDK method without verifying it exists in the async (or
  sync, whichever the agent uses) client. (R9)

## Open methodology questions

Not yet resolved — likely covered by future workgraphs' retros:

- **MQ1.** When does R3 (fail-loud directive) become unnecessary
  enough to omit? Probably never. Confirm after more workgraphs.
- **MQ2.** Should `test_command` support multiple gates someday
  (current MVP is single command)? If yes, this methodology rule
  needs an "array form" example.
- **MQ3.** R7 (mock-idiom verification) was discovered via a verify-
  phase catch on this workgraph. Does it generalize beyond
  test-double extension? Probably yes for other framework
  conventions (logging format, error-class hierarchy, etc.) but
  needs corroboration.

## References (lab-notebook entries this draft is synthesized from)

- `viloforge-projects/vafi/workgraphs/vafi-rolling-restart-fix/architect-retro.md`
  — patterns P1–P13, walk-backs W1–W4
- `viloforge-projects/vafi/workgraphs/vafi-rolling-restart-fix/spec-author-retro.md`
  — patterns SA1–SA6, cross-cutting SAX1–SAX3
- `viloforge-projects/vafi/workgraphs/vafi-rolling-restart-fix/verifier-retro.md`
  — patterns V1–V3, V4–V6 candidates emerging from spike
- `viloforge-projects/vafi/spikes/canary-end-to-end-2026-05-14/SPIKE-RETRO.md`
  — critical findings C1–C4, methodology patterns E1–E5
