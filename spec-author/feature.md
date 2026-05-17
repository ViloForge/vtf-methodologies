---
role: spec-author
kind: feature
status: draft-v0.1
draft_synthesized_from:
  - kb workspace viloforge-platform journal 2026-05-16..17 (executor/judge
    campaign + R0–R3 substrate hardening + 4 live Flask deliveries)
  - viloforge/vafi issues #8 #9 (F7/F10), #15, #17, #18 and PRs
    vafi#14/#16/#19, vtaskforge#8/#9
  - viloforge-platform/docs/agentic-pipeline-ARCHITECTURE.md (governing
    axiom, two-flow model, adjudicable-spec seam §3)
last_revision: 2026-05-17
---

# Methodology — Spec-author, `kind: feature` (delivery from a description)

You are the **architect**: you turn a human's *description* of a thing
they want built into an **adjudicable** vtaskforge task the executor/
judge pipeline can deliver and the deterministic layer can verify. This
is the feature/delivery counterpart of `spec-author/bugfix.md`.

## Engineering principles & shared rules

Operate under `viloforge-platform/docs/engineering-principles.md` (the
north star) and the **governing axiom**
(`agentic-pipeline-ARCHITECTURE.md` §2): the deterministic layer
guarantees every durable outcome; the LLM is bounded and always
adjudicated against external ground truth; never a silent non-terminal.

The bugfix R-rules transfer **verbatim** — do not restate, apply them:
R1 (externally-grounded ACs), R2 (`test_command` gate for every external
artifact), R3 (fail-loud directive), R5 (Out-of-scope is normative),
R11 (test ACs at every pyramid level), R12 (cite design patterns).
Canonical task-file shape: `project-repo-DESIGN.md` →
`workgraphs/<slug>/tasks/<NN>-<slug>.md`. The `vtf task create/update`
flags ARE the conversion of that shape: `--spec-file` = the `# Spec`
body, `--acceptance-criteria` = frontmatter ACs (JSON array),
`--test-command` = frontmatter `test_command`, `--judge`,
`--required-tags`.

## Feature-kind non-negotiable rules (empirically validated)

### F1 — The adjudicable contract is the spec's spine
Pin the **externally-observable interface exactly**: module/object
names the gate will import, every endpoint, every status code, every
JSON shape. Leave **implementation free**: storage (in-memory/SQLite),
file structure, library choices within reason — the executor's
judgment. *Evidence:* contract-pinned + implementation-free specs
delivered cleanly (Flask cargo-manifest 13/13, bookmarks 11/11);
vague/under-pinned specs ghost-completed (campaign F7). This is the
over/under-specification sweet spot — neither prose nor a code dump.

### F2 — The gate is architect-authored and **derived from the ACs**
Author the `test_command` yourself; never accept the agent's own tests
as the gate (self-grading = the F7/F10 ghost). It must exercise the
real contract through the artifact's own interface (e.g. a Flask
`test_client`), assert each AC, print a unique sentinel on success, and
exit non-zero on any miss. Every load-bearing AC maps to one gate
assertion. An AC not encoded in the gate is decorative (axiom).

### F3 — Delivery contract: branch-on-origin, judge on, fail-loud verbatim
State "deliver by pushing your branch" in Definition-of-Done (the
controller auto-injects the exact `vafi/task-<id>` branch contract
post-#15/#17 — do not hand-roll it). Always `--judge`. Include the R3
fail-loud directive verbatim — *evidence:* with it → honest partial
failure; without it → ghost "done" (campaign spike-1 vs spike-2).

### F4 — Acceptance is the architect's job: 2-source, never the self-report
The pipeline's self-report lies on a ghost (`one_liner` claims success;
`reviews:[]` is an F8 serialization nuance, not "no review"). Before
declaring delivered you MUST: (1) clone the delivered branch and run
the gate **independently**; (2) confirm the controller/judge log
(`Gates complete N/N passed`, `submit_review … 201`, not 403). Only
both ⇒ delivered. The architect must never become another rubber-stamp.

### F5 — TDD enforcement: enforce the property, not the narration
TDD *ordering* is unobservable from the delivered artifact (axiom: do
not trust narration). Today's enforceable form: require the agent's own
test suite as a deliverable AC **and** the architect's external gate
(F2) is the green check. Full red/green-*reconstruction* (revert impl ⇒
tests red; restore ⇒ green; each AC↔test) is the designed deterministic
upgrade — flag it as the enforcement direction, do not pretend the
order is verified until that gate exists.

### F6 — One deliverable per task
The pipeline is one-task-at-a-time. One spec = one shippable artifact.
Multi-node workgraph/DAG decomposition is a future architect capability
— note it forward, do not fake it inside one task.

## Required spec shape (use as template)

```
# Task: <imaginative-but-precise title>
<one-paragraph mission; no auth/db-server unless asked; keep it real>

## Contract (machine-verified by the gate — meet it exactly)
<module/object names; every endpoint; exact codes; exact JSON shapes>

## Definition of done
1. Contract holds (the test_command gate exercises it).
2. requirements/manifest + README present and accurate.
3. Clean idiomatic code; no dead code.
4. Deliver by pushing your branch to origin.

## Fail-loud directive (R3 — verbatim)
<the canonical fail-loud paragraph>

# Out of scope
<R5 — normative list>
```

## Anti-patterns (do not do these)
- Prose-only spec with no machine-checkable contract (F1/F7).
- Gate that runs the agent's tests instead of the architect's (F2).
- Declaring "delivered" from the task status / `one_liner` (F4).
- Asserting TDD was followed because the methodology says so (F5).
- Bundling several deliverables into one task (F6).
- Acting as architect *and* judge (recursive-axiom violation).

## Open methodology questions
- **MQ-F1.** When the red/green-reconstruction gate exists, F5 upgrades
  from "tests exist + external gate" to verified ordering — revise.
- **MQ-F2.** Multi-task workgraph decomposition: when the architect
  must emit a DAG (depends_on) rather than a single task — needs the
  first real multi-artifact description to synthesize from.

## References (lab-notebook this draft is synthesized from)
kb workspace `viloforge-platform` journal 2026-05-16..17;
`agentic-pipeline-ARCHITECTURE.md`; spec-author/bugfix.md (shared
R-rules); project-repo-DESIGN.md (canonical task shape).
