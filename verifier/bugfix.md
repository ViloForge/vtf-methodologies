---
role: verifier
kind: bugfix
status: draft-v0.1
draft_synthesized_from:
  - viloforge-projects/vafi/workgraphs/vafi-rolling-restart-fix/verifier-retro.md
  - viloforge-projects/vafi/spikes/canary-end-to-end-2026-05-14/SPIKE-RETRO.md
last_revision: 2026-05-14
---

# Methodology — Verifier, `kind: bugfix`

You are running the verify phase between spec-authoring and
execution. Your job: read the entire workgraph (workgraph.md +
plan.md + all task specs) and confirm it can safely transition
`specced → ready` for executor pickup.

This file is **first-draft, v0.1**, synthesized from the verifier
retro of the first SDD workgraph and from the canary spike.

## Engineering principles (canonical: `viloforge-platform/docs/engineering-principles.md`)

This methodology operates under the ViloForge engineering north
star. **Read `engineering-principles.md` before running verify.**
Concretizations for the verifier role:

- **TDD enforcement:** verify every code-bearing task spec includes
  test ACs at every applicable pyramid level. Missing levels are
  rejection-grade.
- **SOLID detection:** while reading spec bodies, flag obvious SOLID
  violations in the implementation sketches (god classes,
  hardcoded concretions, switch statements where Strategy would
  fit).
- **Pattern coherence:** verify spec body claims about design
  patterns are consistent with the implementation sketch.
- **Extensibility affordances:** verify v1 contracts include
  `org_id`, `cluster_id`, and other future-feature affordances
  declared in the design.

These are non-negotiable per the north star. See §5.3 in
`engineering-principles.md` for the verifier checklist.

## Mandatory checks

Each check is a single, defined inspection. Skipping any of them
risks an executor-phase walk-back which is 10–100× more expensive
than a verify-phase catch.

### V1 — Zero `[NEEDS CLARIFICATION]` markers

Grep every task spec and the workgraph artifacts for the strings
`[NEEDS CLARIFICATION]` and `TBD`. Any hit (other than meta-
references in retros) is a defect — the spec-author left an
unfilled hole. Reject `specced → ready` until cleared.

### V2 — DAG topology is acyclic and connected

For every task in `tasks/`:

1. Its `depends_on` IDs all resolve to other tasks in the same
   workgraph dir.
2. The resulting graph has no cycles.
3. Every task is reachable from at least one root (a task with
   empty `depends_on`).

A mechanical check via topological sort suffices.

### V3 — `target_repo` values resolve

Every task's `target_repo` must point at a real, accessible repo.
For multi-repo workgraphs, verify each one independently. If a
target repo doesn't exist (or the executor lacks credentials),
flag it now — surfaces the credentials/access issue before the
executor wastes turns.

### V4 — AC verifiability (judge-gradeable)

Each `acceptance_criteria` entry must be:

- An assertion an external observer (judge or test) can evaluate.
- Specific enough that "satisfied / not satisfied" is unambiguous.
- Not phrased as a directive to the executor ("The executor should
  do X") — that's spec-body language, not an AC.

Reject ACs like "Code is high quality" (unverifiable) or "Tests
exist" (too broad — how many? testing what?).

### V5 — **Externally-grounded AC required for external-artifact tasks**

For any task whose work product is an external artifact (PR,
deployment, published package, remote branch), **at least one
acceptance criterion MUST be externally-grounded** per spec-author
methodology R1.

If a task's `target_repo` is something other than the project repo
itself, that's a strong signal it produces an external artifact;
its ACs MUST include one of:

- "A PR exists at `https://...`"
- "Branch `<name>` is present on remote `origin`"
- "Image `<registry>/<image>:<tag>` is pullable"
- "Endpoint `<url>` returns 200..."
- Equivalent externally-grounded assertion.

If absent, **reject** with a pointer to spec-author/R1.

Empirical evidence: spike-1 was missing this; ended in
ghost-completion to `done`. Spike-2 had it; ended correctly in
`changes_requested`.

### V6 — **`test_command` populated for external-artifact tasks**

For the same tasks identified in V5, the task's `test_command`
field MUST be populated with a shell command that mechanically
verifies the external artifact (per spec-author/R2).

Empty `test_command` (`{}` or absent) on an external-artifact task
is a verifier-blocking defect.

Empirical evidence: spike-1 had no gate; harness `exit 0` was
trusted blindly. Spike-2's gate forced the harness to actually
push the branch.

### V7 — Fail-loud directive present in Spec body

For external-artifact tasks, the Spec body MUST include language
that explicitly requires honest failure reporting (per
spec-author/R3). Look for phrases like "report failure honestly,"
"do not rationalize," or equivalent.

Without it, executors have been observed to rationalize partial
completion as success.

### V8 — file:line citations resolve

Spot-check 4–6 file:line citations across the workgraph. For each:

1. The file exists at the named path in the named repo.
2. The lines cited contain code with the shape the spec implies
   (a function definition, a class, a known comment, etc.).

A miss is a drift finding (per architect-retro X3). 4–6 spot
checks is enough to surface systematic drift; if multiple miss,
the workgraph is stale and needs re-spec-authoring.

### V9 — Mock-double idioms match the existing test scaffold

For each task that adds a protocol/interface method or modifies an
existing test double: read the existing test file in the target
repo and confirm the spec's mock-extension instructions match the
existing convention (AsyncMock vs custom class, init pattern,
assertion style).

Empirical evidence: spike-author missed this on the original T1
and T2 specs for vafi-rolling-restart-fix; verifier caught it
inline; specs were rewritten to use AsyncMock idioms.

### V10 — Cross-task AC consistency

For every cross-task reference (an AC that names another task by
ID or role — "T1's PR", "the integration test"), verify both
sides agree. Conflicting prescriptions are rejection-grade.

If an artifact has multiple plausible owners (e.g., the
pyproject.toml pin-bump in vafi-rolling-restart-fix's T1+T2 case),
the AC text must be neutral about ordering OR commit to a
specific order with rationale.

### V13 — Test acceptance criteria at every applicable pyramid level

For every task spec that produces code, verify
`acceptance_criteria` includes explicit test ACs at each pyramid
level the task touches:

- **Unit ACs** present if any function/class added or modified.
- **Integration ACs** present if cross-component behavior added.
- **Contract ACs** present if public-API surface added (SDK
  methods, HTTP endpoints, event schemas).
- **Scenario ACs** present if user-flow-touching work.

Sample sound AC list (from `spec-author/bugfix.md` R11):

```yaml
acceptance_criteria:
  - "Unit: tests/unit/... covers create + validate + error paths"
  - "Integration: tests/integration/... verifies round-trip"
  - "Contract: tests/contract/... validates schema"
  - "Scenario: tests/scenario/... exercises in named flow"
```

Missing levels for the scope of work = reject `specced → ready`
with pointer to `spec-author/bugfix.md` R11.

### V14 — Pattern claims coherent with implementation sketch

For every spec that names a design pattern in the Spec body, verify
the implementation sketch is consistent with the named pattern:

- Spec says "Strategy pattern" → sketch shows an interface with
  multiple implementations + a registration/dispatch site. NOT a
  switch statement.
- Spec says "Repository pattern" → sketch shows an interface
  abstracting storage + a concrete impl. NOT direct ORM calls in
  business logic.
- Spec says "Factory pattern" → sketch shows a creation site
  separate from usage. NOT inline `new` calls.

Pattern claim + inconsistent sketch = reject; ask spec-author to
either align the sketch OR drop the pattern claim.

**Pattern-theater detection (refinement from vfobs-foundation F4).**
A pattern claim with no clear "interface to conform to" is theater
— a decorative reference to a GoF name without the structural
constraint that makes it useful. Examples to reject or rewrite:

- "Adapter pattern: Alembic adapts our schema-definition to
  Postgres" — Alembic is a tool, not an inter-interface adapter.
- "Singleton pattern: the database engine is a module-level
  variable" — module-level binding is not the Singleton pattern
  unless instantiation control is in play.
- "Observer pattern: callers can listen for events" — not unless
  there's a Subject + Observer interface with a registration site.

Pattern theater confuses executors (who try to implement the
pattern faithfully and end up over-engineering) and judges (who
grade against a pattern that isn't really there). When in doubt,
drop the pattern claim — "no application-layer patterns; this task
is X" is a valid answer.

### V15 — Extensibility affordances present

For every workgraph with declared future features (per
`engineering-principles.md` §2.2):

- Multi-tenant SaaS → `org_id` field present in event/data schemas.
- Multi-cluster → `cluster_id` field present.
- Configurable rules → thresholds read from a config object, not
  literals in business logic.
- External notification routing → notification path is consumer
  of the existing event log, not a new write-side path.

Missing affordance = reject; ask architect to either add the
affordance to v1 or remove the future feature from the deferred
list.

### V11 — Methodology pinning matches kind

When `vtf-methodologies/` files exist and `project.yaml` pins
them: confirm the pinned methodology's `kind` field matches the
workgraph's `kind`. Mismatches produce architect/executor
disagreements about how the task should be executed.

(During the bootstrap period — pre-methodology-pinning — this
check is informational only.)

## Verifier output

Two outcomes:

1. **All checks pass → workgraph status transitions `specced → ready`.**
   The runtime makes all tasks claimable. Write a brief
   `verifier-retro.md` only if anything surfaced that's
   methodology-relevant.

2. **Any check fails → write findings, patch what's safe inline,
   escalate the rest to spec-author or architect.**
   - Small defects (mock idiom, citation drift) can be patched
     inline at verify time and noted.
   - Larger defects (missing AC class, DAG topology changes) go
     back to spec-author.
   - Truly structural defects (missing fundamental capability,
     mis-specced topology) go back to architect.

## Open methodology questions

- **VQ1.** How aggressive should V8 spot-checking be? Current
  spec is 4–6 citations. Future workgraphs may indicate this
  number should scale with workgraph size.
- **VQ2.** V5/V6/V7 currently apply only to external-artifact
  tasks. The "external artifact" determination is currently
  judgmental ("is this `target_repo` outside the project repo?").
  A more structural signal (e.g., a `produces_external_artifact:
  true` frontmatter field) would make the check mechanical.
- **VQ3.** What constitutes a "real defect" vs "drift to flag but
  not block"? Bias has been "any check failure blocks `ready`";
  future workgraphs may show that some classes of finding can be
  flagged without blocking.

## References

- `viloforge-projects/vafi/workgraphs/vafi-rolling-restart-fix/verifier-retro.md`
  — patterns V1–V3 (the originals before this draft synthesized them)
- `viloforge-projects/vafi/spikes/canary-end-to-end-2026-05-14/SPIKE-RETRO.md`
  — empirical basis for V5/V6/V7
- `vtf-methodologies/spec-author/bugfix.md` — companion file; the
  rules this verifier enforces are authored there
