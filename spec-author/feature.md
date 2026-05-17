---
role: spec-author
kind: feature
status: draft-v0.3
draft_synthesized_from:
  - kb workspace viloforge-platform journal 2026-05-16..17 (executor/judge
    campaign + R0–R3 substrate hardening + 4 live Flask deliveries)
  - viloforge/vafi issues #8 #9 (F7/F10), #15, #17, #18 and PRs
    vafi#14/#16/#19, vtaskforge#8/#9
  - viloforge-platform/docs/agentic-pipeline-ARCHITECTURE.md (governing
    axiom, two-flow model, adjudicable-spec seam §3)
  - self-review 2026-05-17 (v0.1 → v0.2 gap closure)
last_revision: 2026-05-17
---

# Methodology — Spec-author, `kind: feature` (delivery from a description)

You are the **architect**: you turn a human's *description* into an
**adjudicable** vtaskforge task the executor/judge pipeline can deliver
and the deterministic layer can verify. Feature/delivery counterpart of
`spec-author/bugfix.md`.

## Engineering principles & shared rules

Operate under `engineering-principles.md` (north star) and the
**governing axiom** (`agentic-pipeline-ARCHITECTURE.md` §2). Bugfix
R-rules transfer **verbatim** (do not restate): R1, R2, R3, R5, R11,
R12. Canonical task-file shape: `project-repo-DESIGN.md` →
`workgraphs/<slug>/tasks/<NN>-<slug>.md`. The `vtf task create/update`
flags ARE the SDD-task→vtaskforge conversion (`--spec-file` = body,
`--acceptance-criteria` = AC array, `--test-command` = gate, `--judge`,
`--required-tags`).

> **KNOWN AXIOM GAP (do not paper over).** The executor's deliverable
> is deterministically adjudicated (delivery gate + `test_command`).
> The **architect's spec is NOT** — there is no spec-admission gate
> yet, so today the architect is *self-adjudicated*, which is the same
> ad-hoc trust the axiom forbids. This methodology is the interim
> human-enforced fence; the deterministic fix is the **spec-admission
> gate** tracked in `agentic-pipeline-ARCHITECTURE.md`. State this
> honestly; do not imply the spec is verified.

## Validated envelope (confidence scope — do not over-claim)

v0.2 is synthesized from **N=4** deliveries, **all the same shape**:
single-artifact, stateless, in-memory **Python (Flask) web API**.
Validated: that shape. **Unvalidated** (mark assumptions explicitly
when you stray): multi-file/stateful services, non-Python, persistence,
concurrency, multi-task DAGs. Every evidence citation below tags the
**delivering harness**, because harness selection is currently
nondeterministic (F5/routing, unfixed) — a rule "proven" only on the
Claude executor is weaker than one proven on both. Treat single-harness
evidence as provisional.

## Feature-kind rules

### F1 — The adjudicable contract is the spec's spine
Pin the **externally-observable interface exactly** (module/object
names the gate imports, endpoints, status codes, JSON shapes); leave
**implementation free**. *Evidence:* contract-pinned + impl-free
delivered cleanly — cargo-manifest 13/13 (pi), 11/11 (claude);
bookmarks 11/11 (claude); vague specs ghosted (campaign F7, pi). The
sweet spot: neither prose nor a code dump.

### F2 — Architect-authored, AC-derived gate, with AC-id traceability
Author the `test_command` yourself; never the agent's tests
(self-grading = F7/F10 ghost). Exercise the real contract via the
artifact's own interface; sentinel on success; non-zero on any miss.
**Structural requirement:** every assertion carries its AC id, e.g.
`assert c.get('/health').status_code==200, 'AC1'`. **Coverage rule:**
every AC has ≥1 labelled assertion; an AC with none is decorative and
the spec is inadmissible. (This labelling is exactly what the future
spec-admission gate will mechanically check — write it so a machine
can verify coverage today by inspection, a gate tomorrow.)

### F3 — Delivery contract: branch-on-origin, judge on, fail-loud verbatim
Definition-of-Done says "deliver by pushing your branch" (the
controller auto-injects the exact `vafi/task-<id>` contract post-#15/
#17 — do not hand-roll). Always `--judge`. R3 fail-loud directive
verbatim. *Evidence:* with it → honest partial failure (campaign
spike-2); without → ghost "done" (spike-1).

### F4 — Acceptance is the architect's job: 2-source, never the self-report
The pipeline self-report lies on a ghost (`one_liner` claims success;
`reviews:[]` is the F8 serialization nuance, not "no review"). Before
"delivered": (1) clone the delivered branch and run the gate
**independently**; (2) confirm controller/judge log (`Gates complete
N/N passed`, `submit_review … 201` not 403). The independent gate run
(1) is what makes judge *fidelity* non-load-bearing for acceptance
(pass-2 deferred) — it is not optional. Both ⇒ delivered.

### F5 — TDD: enforce the property, never the narration; be honest what's enforced
TDD *ordering* is unobservable from the artifact (axiom: never trust
narration). **What is actually enforced today: only F2's
architect-authored external gate.** You MAY add "an automated test
suite is present and runs" as a *presence-only* AC — but per F2 that AC
can only be gate-checked for presence/exit-status, **not quality**, so
do **not** represent it as a TDD or quality guarantee. Full enforcement
(revert impl ⇒ tests red; restore ⇒ green; each AC↔test) requires the
red/green-**reconstruction** gate (MQ-F1, unbuilt). Until it exists,
the spec must not claim TDD is verified.

### F6 — One deliverable per task; multi-node = a workgraph
One spec = one shippable increment with its own gate. When the
description needs several composing artifacts, decompose into a
**workgraph DAG** per F8 — do not fake it inside one task, and do not
hand-merge between tasks (the composition substrate, not the architect,
integrates — see F8 + `agentic-pipeline-ARCHITECTURE.md` §10).

### F8 — Workgraph (multi-task DAG) decomposition discipline
For a multi-artifact deliverable the architect emits a **workgraph**:
nodes (tasks) + `depends_on` edges. Rules (validated-by-design;
**execution requires the Workgraph Composition substrate WC-1..3 —
single-task is the only mode until that ships**):

1. **Decompose by deliverable seam, not by activity.** Each node is a
   coherent shippable increment with its own AC-id'd gate (F2).
2. **File-ownership map.** Every shared/foundational file (`pyproject.
   toml`, package `__init__`, the shared contract module) has exactly
   **one owner node** (usually the root/skeleton node). Dependent nodes
   **extend via declared extension points, never rewrite owned files**
   — this is what makes the deterministic merge clean.
3. **Parallelize iff disjoint.** Two nodes may omit a `depends_on` edge
   (run in parallel) **only if their write-sets are disjoint or
   provably commutative**; otherwise add the edge to serialize.
   Compute each node's write-set from its spec and set edges
   accordingly. Coupled concerns (e.g. server↔client share a wire
   contract) ⇒ a linear edge; do not fake fan-out.
4. **Freeze inter-node contracts in the dependency.** The dependency
   node delivers + tests + documents the interface; the dependent's
   spec references that frozen contract and builds against it (F1
   generalized across nodes). The dependent only sees that code because
   the substrate merges the dependency into the integration branch
   first — never assume otherwise.
5. **Terminal integration-acceptance node.** The last node's base is
   the composed integration branch; its gate exercises the **whole
   product** (build / `pip install` / CLI / e2e) — F4 at DAG scope.
6. **Workgraph fail-loud (F7 at DAG scope).** A node's merge-conflict
   is a bounded rework on the *current* integration branch; the
   workgraph never silently stalls; on rework-exhaustion escalate the
   whole DAG to the human with per-node evidence.

### F7 — Bounded rework path (a non-terminal verdict is expected, not exceptional)
The pipeline can return `changes_requested` (judge rejected), `failed`/
`needs_attention` (gate failed / honest agent failure / R3 lease). The
architect MUST:
1. **Diagnose 2-source** (F4 method) — *which* AC/gate assertion failed,
   from the log + an independent gate run, never the self-report.
2. **Classify:** (a) spec defect (ambiguous/contradictory/inadmissible
   contract) → revise the spec, harvest the defect class into this
   file, re-fire; (b) executor miss on a sound spec → `vtf task reset
   … --status todo` to re-fire (rework); (c) environment/credential →
   fix env, re-fire.
3. **Bound it:** ≤3 re-fires for the same task (mirrors executor
   `max_rework`). On exhaustion **escalate to the human** with the
   per-AC failure evidence — do not silently keep re-firing, and never
   relabel an unmet AC as met to force closure (that is the F7/F10
   ghost at the architect level).

## Required spec shape (template)

```
# Task: <imaginative-but-precise title>
<one-paragraph mission>

## Contract (machine-verified by the gate — meet it exactly)
<names the gate imports; endpoints; exact codes; exact JSON shapes>

## Definition of done
1. Contract holds (the test_command gate exercises it).
2. requirements/manifest + README present and accurate.
3. Clean idiomatic code; no dead code.
4. Deliver by pushing your branch to origin.

## Fail-loud directive (R3 — verbatim)
<canonical fail-loud paragraph>

# Out of scope        (R5 — normative)
```
ACs: JSON array; each phrased so exactly one gate assertion (labelled
with its 1-based index, `AC1`…) verifies it.

## Worked example (canonical — bookmarks-api, 2026-05-17, claude executor)

- **Description:** "a Flask api … crud on something." Architect chose
  *bookmarks* (basic, universal).
- **Contract (excerpt):** `app.py` exposes `app`; `GET /health`→200
  `{"status":"ok"}`; `POST /bookmarks {url,title,tags?}`→201+`id`;
  missing `url`|`title`→400; `GET|PUT|DELETE /bookmarks/<id>`→
  200/200/204, 404 if absent; `requirements.txt`+`README.md`.
- **ACs (machine-checkable):** AC1 health; AC2 create+list+get-one;
  AC3 update reflected + delete 204; AC4 404 on missing id; AC5 400 on
  missing url/title; AC6 manifest+README; AC7 branch pushed.
- **Gate (architect-authored, AC-labelled):** imports `app`, drives
  `app.test_client()`, `assert …, 'AC1'` … `'AC5'`, prints `GATE_OK`,
  non-zero on any miss. (AC6/AC7 are gated by the delivery gate +
  presence checks.)
- **2-source acceptance:** independent clone + gate run → **11/11**;
  judge log `submit_review … 201 approved`. ⇒ delivered.
- **Lesson harvested:** contract-pinned/impl-free held again on a new
  domain → F1 confidence raised (still single-shape, see envelope).

## Anti-patterns
- Prose spec, no machine-checkable contract (F1/F7-bug).
- Gate runs the agent's tests, or assertions lack AC ids (F2).
- "Delivered" from task status / `one_liner` (F4).
- Representing the presence-only test AC as a TDD/quality guarantee (F5).
- Unbounded re-fire, or relabelling an unmet AC to force closure (F7).
- Closing the cycle without the harvest journal line (skill step 8).
- Architect also acting as judge (recursive-axiom violation).

## Open methodology questions
- **MQ-F1.** Red/green-reconstruction gate → upgrades F5 from
  "external gate only" to verified ordering.
- **MQ-F2. RESOLVED-BY-DESIGN** (2026-05-17, first multi-artifact
  description): decomposition discipline is F8; *execution* needs the
  Workgraph Composition substrate (ARCHITECTURE §10, WC-1..3) — built
  before multi-task DAGs run. Until then, single-task only.
- **MQ-F3 (load-bearing).** The **spec-admission gate**: deterministic
  pre-claim check that ACs are machine-checkable, every AC has a
  labelled gate assertion (F2), fail-loud present, contract pinned.
  Until built, the architect is self-adjudicated (the KNOWN AXIOM GAP).
- **MQ-F4.** Harness nondeterminism (F5/routing) pollutes the evidence
  this methodology is synthesized from — R4 (routing determinism)
  should arguably precede heavy harvesting; until then tag every
  evidence citation with its harness and treat single-harness as
  provisional.

## References
kb workspace `viloforge-platform` journal 2026-05-16..17;
`agentic-pipeline-ARCHITECTURE.md`; `spec-author/bugfix.md` (shared
R-rules); `project-repo-DESIGN.md` (canonical task shape).
