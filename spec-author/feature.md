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
nodes (tasks) + `depends_on` edges. Rules (validated-by-design **and
PROVEN-IN-DELIVERY**: the Workgraph Composition substrate WC-1..3 has
shipped and a real multi-task diamond DAG executed end-to-end —
`depends_on` gating, parallel fan-out, cold-start clone, integrate
SSH push + branch-create, serialized merge slot, and F7/F10 ghost-
catch all exercised. Repo `vilosource/wc3-kvstore`, 2026-05-20 2-source
verify green. Multi-task DAGs are the supported mode — single-task is
no longer a restriction):

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

### Harvested — WC substrate program (2026-05-19, multi-slice DAG, cross-repo)

Delivering Workgraph Composition as WC-1 (vtaskforge SoR) → WC-2 (vafi
controller) surfaced three SOP-grade lessons:

- **F8.a — Verify the inter-slice seam against *merged producer code*,
  at the consumer slice's design time, before writing it.** WC-1/C3
  asserted "the controller reports the outcome" but exposed no API;
  WC-2 design-grounding (reading the *merged* `tasks/views.py`) proved
  `complete()` is invalid from `integrating` and `fail()` is
  doing-scoped — a load-bearing gap caught **design-first**, not
  mid-code. For every frozen inter-node contract (F8), the consuming
  node's spec must cite the exact producer symbol/endpoint that
  satisfies it; "the producer will provide it" is not grounding.
- **F8.b — A forwarding/generated SDK is not proof a new contract
  field reaches the consumer.** WC-1's `base_ref` was silently dropped
  by an `extra="ignore"` SDK model; the consumer surface had to be
  verified to round-trip explicitly. When a DAG contract crosses a
  repo/SDK boundary, the seam test asserts the field *through the real
  client model*, not just the API response.
- **F8.c — Ground the composition ref from the real delivery
  contract.** D2 merged `gates.deliverable_branch` (`vafi/task-<id>`),
  the verified F7/F10 ref — not an assumed branch name. The
  integration-acceptance node's spec pins the *existing* deliverable
  contract; it never invents a branch convention.

### Harvested — WC-3 proving-delivery design-grounding (2026-05-19, diamond DAG)

Authoring the WC-3 proving DAG (n1 skeleton → n2 server ‖ n3 client →
n4 integration) surfaced a fourth, sharper seam lesson — caught
**design-first, before a single pipeline run was burned**, which is
exactly the value a proving delivery exists to extract:

- **F8.d — Trace the DAG *cold-start* path (empty integration branch,
  root node), not just the steady-state seam.** The steady-state seam
  (a dependent cloning the already-composed integration branch) hides
  the cold-start seam (the *first/root* node, whose integration branch
  does not exist yet). Before firing any workgraph, verify the
  dependency-less node's clone ref by composing **three** merged
  producer facts: (1) the SoR `resolve_base_ref` rule, (2) the
  controller's clone invocation + its failure mode, (3) when
  `integrate()` actually creates the integration git ref. In WC-3 this
  composition proved a real WC-1 defect: `resolve_base_ref`
  (`tasks/services.py:502-512`) returns `milestone.integration_branch`
  for the **root** node too (no `depends_on` special-case), but the
  controller's `git clone --branch <integration_branch> --single-branch`
  (`invoker.py:204-233`) hard-fails with no fallback because
  `integrate()` only creates that ref *after the first approval* — an
  unbreakable cold-start deadlock for every multi-task DAG.
  Generalization: a "proving delivery" must walk the
  empty-state/first-node path end-to-end across SoR → controller →
  integrate(); a contract correct in steady state can still be
  unsatisfiable at t=0. The dependency-less node's base **is** the
  project default — the integration branch is its *output*, created by
  its own `integrate()`, never its input.

- **F8.d.1 — Sharpening (post-fire, WC-3 n1): the cold-start trace
  must enumerate the *credential* of every git op on the multi-task
  path, not just *which ref*.** WC-3 fired after WC-1.1; n1 then
  passed gates+judge, reached `integrating`, and `integrate()` failed:
  `could not read Username for 'https://github.com'`. The multi-task
  path **re-implements** git operations the single-task path already
  solved: the executor clone uses the #15/#17 SSH credential, but
  `integrate()`'s independent clone+push never inherited it (WC-1.2,
  vafi#23 → fixed #22). Lesson: a substrate's DAG path has *parallel,
  separately-authored* git legs (executor-clone vs integrate-clone vs
  integrate-push vs branch-create); "the executor path works" does not
  imply the integrate path works. Design-grounding must walk **each
  git leg** of the cold-start path and confirm it carries the same
  auth + cold-start handling — enumerate the legs, do not assume
  shared plumbing. Two real substrate defects (WC-1.1, WC-1.2) hid on
  the never-exercised multi-task path; only proving delivery flushes
  them, and only a per-leg trace predicts them.

- **F7.a — The delivery directive must defeat the persisted-workdir
  "already complete" rationalization on rework.** WC-3 n2 (server)
  ghosted twice: run-1 never pushed; the F7 re-fire reused the
  *persisted workdir* (`invoker._ensure_repo_cloned`: existing `.git`
  ⇒ no-op, by rework design), the agent saw its own prior uncommitted
  files, declared "the implementation already existed", made **no new
  commit**, and the deliverable branch came back *identical to base*
  (the F7/F10 delivery gate caught it — the machinery worked). The
  identically-shaped sibling n3 passed first try, so the spec body was
  sound — the gap was the *delivery* directive under rework. Lesson:
  every spec's delivery section must (1) state that a reused workdir's
  pre-existing files are **not** proof of delivery, (2) mandate a
  fresh commit + push **every run** with the deliverable branch
  strictly ahead of base, and (3) forbid reporting success on
  apparent-prior-state. Pair it with R3 fail-loud. Bound re-fires (≤3)
  and *diagnose each failure distinctly* — n2's two failures had two
  different root causes; blind re-fire would have masked the second.

- **F8.e — A proving delivery's value is realized in BOTH phases;
  budget for substrate defects on the never-exercised multi-task
  path.** WC-3 confirmed the two phases catch *different* seam
  classes: design-grounding (pre-fire) caught the cold-start root-node
  base_ref seam (F8.d); execution (post-fire) caught the per-leg
  credential seam the trace under-specified (F8.d.1) and the rework
  delivery-ghost (F7.a). All three were *real* substrate defects
  (WC-1.1 root clone, WC-1.2 integrate SSH push, F7.a rework
  directive), each invisible on the single-task path that had been the
  only mode exercised. Lesson: a never-run multi-task path will carry
  ~3 latent substrate defects no trace fully predicts; only firing a
  real DAG flushes them — schedule a proving delivery as a first-class
  substrate-acceptance step, not a demo. WC-3 found exactly that and
  closed green after the three fixes shipped.

## Anti-patterns
- Prose spec, no machine-checkable contract (F1/F7-bug).
- Gate runs the agent's tests, or assertions lack AC ids (F2).
- "Delivered" from task status / `one_liner` (F4).
- Representing the presence-only test AC as a TDD/quality guarantee (F5).
- Unbounded re-fire, or relabelling an unmet AC to force closure (F7).
- Closing the cycle without the harvest journal line (skill step 8).
- Architect also acting as judge (recursive-axiom violation).
- Specifying a consumer DAG node against a producer contract that is
  *designed* but not yet merged/verified — ground the seam against
  merged producer code, across SDK boundaries (F8.a/F8.b).
- Firing a workgraph without tracing the cold-start/root-node path
  (empty integration branch at t=0) — steady-state correctness does
  not imply the first node can even clone (F8.d).

## Open methodology questions
- **MQ-F1.** Red/green-reconstruction gate → upgrades F5 from
  "external gate only" to verified ordering.
- **MQ-F2. RESOLVED + PROVEN-IN-DELIVERY** (decomp = F8). The
  Workgraph Composition substrate is delivered+merged: WC-1
  (vtaskforge#10: integration_branch, base_ref, `integrating`,
  take_merge_slot, C4 reaper) + SDK base_ref (#11) +
  integration-result endpoint (#12) + WC-2 (vafi#20: per-task
  base_ref clone, deterministic post-approve merge, fail-loud,
  idempotent) + the multi-task defects flushed by proving: WC-1.1
  (vafi#21, root cold-start clone), WC-1.2 (vafi#22, integrate SSH
  push), F7.a (rework delivery directive). Substrate DEPLOYED
  2026-05-19 (vtf-dev f5a29c6, vafi-dev 5ec61d5). **WC-3 proving
  delivery FIRED AND GREEN**: a 4-node diamond DAG (n1 skeleton → n2
  server ‖ n3 client → n4 integration) executed end-to-end against the
  live substrate, repo `vilosource/wc3-kvstore`, integration branch
  `vafi/wg-LU5GNOz7O7hkR038C-LJm`. 2-source verified independently
  2026-05-20: (1) all four externally-authored node gates re-run by
  the architect in clean venvs on the *composed* tree → GATE_OK; (2)
  the delivered in-repo pytest suite → 7 passed; the wheel builds,
  installs fresh, and `create_app`/`KVClient` round-trip + raw
  contract 400/404 checks hold. **The "single-task only" restriction
  is lifted — multi-task DAGs are proven and supported (F8).** See kb
  workspace `viloforge-platform`.
- **MQ-F3 (load-bearing).** The **spec-admission gate**: deterministic
  pre-claim check that ACs are machine-checkable, every AC has a
  labelled gate assertion (F2), fail-loud present, contract pinned.
  Until built, the architect is self-adjudicated (the KNOWN AXIOM GAP).
- **MQ-F4. RESOLVED by R4 (2026-05-20).** Harness routing is now
  deterministic by construction: the Claude and Pi pools carry disjoint
  tags (`claude` / `pi`) with no shared `executor` tag, so a task pins
  exactly one harness via `required_tags=[claude]` or `[pi]` — the
  claimer is no longer ambiguous, and bare `[executor]` matches no pool.
  Always pin the harness and record which one a delivery ran on; the
  evidence-confound that argued R4-before-heavy-harvesting is closed.
  (Single-harness coverage is still weaker than both — that's a coverage
  judgement, not a routing ambiguity.)

## References
kb workspace `viloforge-platform` journal 2026-05-16..17;
`agentic-pipeline-ARCHITECTURE.md`; `spec-author/bugfix.md` (shared
R-rules); `project-repo-DESIGN.md` (canonical task shape).
