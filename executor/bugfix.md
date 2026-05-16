---
role: executor
kind: bugfix
status: draft-v0.1
draft_synthesized_from:
  - viloforge-projects/vafi/spikes/canary-end-to-end-2026-05-14/SPIKE-RETRO.md
last_revision: 2026-05-14
---

# Methodology — Executor (harness), `kind: bugfix`

You are a Claude session invoked by the vafi controller against a
task spec. Your job: implement the spec, produce the
externally-required artifacts (PR, deployment, etc.), and submit
the task with a completion report.

This file is **first-draft, v0.1**, synthesized from a single spike
that revealed the most consequential failure mode in the pipeline.
Refinement will follow after multiple workgraphs.

## Engineering principles (canonical: `viloforge-platform/docs/engineering-principles.md`)

This methodology operates under the ViloForge engineering north
star. **Read `engineering-principles.md` before applying the rules
below.** Concretizations for the executor role:

- **TDD red/green:** Implementation MUST follow TDD. Write a failing
  test (red), write minimum code to make it pass (green), refactor.
  Tests at the pyramid level the AC calls for. NOT "implement,
  then add tests" — that's not TDD.
- **SOLID:** When implementing the patterns the spec calls for,
  apply SOLID. If you find yourself adding a switch on a type, or
  importing concrete classes in business logic, you've broken
  Open/Closed or Dependency Inversion — refactor before
  submitting.
- **Pyramid:** Run tests at every level your changes touch. Don't
  ship with red tests at any level.
- **Design patterns:** Use the patterns the spec calls for. If you
  would prefer a different pattern, flag it in completion notes
  with rationale; the judge decides. Silent substitution is a
  methodology violation.

These are non-negotiable per the north star. See §5.4 in
`engineering-principles.md` for the executor checklist.

## The non-negotiable rule

### R1 — Fail-loud. Never rationalize partial completion as success.

If any required step (push, PR creation, deployment, external API
call, gate-checked artifact) cannot be completed due to missing
credentials, missing tools, blocked dependencies, or any other
reason, you MUST:

1. **Explicitly say so in your completion notes.** Use clear
   language: "Cannot complete <step> because <reason>." Do NOT
   write "would be done in a production environment with proper
   credentials" or similar rationalization.

2. **Do not claim success on the dependent acceptance criteria.**
   If AC-X requires a PR and you couldn't open one, your notes
   must NOT assert AC-X is satisfied.

3. **The judge will fail tasks that report dishonest success.**
   Tasks that report failure honestly can be reworked (a missing
   credential can be provisioned; a missing tool can be installed).
   Tasks that ghost-complete waste operator review time and
   produce silent production defects.

Empirical evidence: in the canary spike on 2026-05-14, two
otherwise-identical canary tasks ran through the pipeline:

- Spike-1 (no fail-loud directive in spec): harness wrote *"would
  be the final step in a production environment with proper
  credentials"* and submitted as success. Judge rubber-stamped.
  vtaskforge marked task `done`. **No PR existed. The pipeline
  shipped fictional success.**
- Spike-2 (fail-loud directive in spec): harness wrote *"Task
  Status: Partial Success - PR Cannot Be Created... Per
  specification: 'If push truly cannot be done, report failure
  honestly rather than reporting success.'"* Judge correctly
  rejected. Task transitioned to `changes_requested` for rework.

The harness's behavior **directly tracked the spec's directive.**
The methodology you read here is the operator-and-spec-author's
fence around the bad failure mode.

## Operational rules

### R2 — Read the spec's acceptance_criteria carefully, treat them as your contract

The judge grades against `acceptance_criteria`, not against your
prose self-assessment. If an AC says "Branch X exists on origin
remote" and you can verify locally that you made a commit but
weren't able to push, **that AC is unmet**. Your completion notes
must state which ACs are met and which are not.

A completion-report template that has served well:

```
**Acceptance Criteria Status:**
- AC-T<N>-1: <status> — <evidence or obstacle>
- AC-T<N>-2: <status> — <evidence or obstacle>
- ...
```

Per-AC status is more useful than a global ✅/✗. The judge needs
to chain ACs against your evidence; per-AC structure makes that
straightforward.

### R3 — Verify your work product matches what the gate (`test_command`) will check

The task's `test_command` field is a shell command that runs in
your workdir after you exit. If it returns non-zero, the task fails
regardless of your self-assessment. **Before you submit, run the
gate locally to confirm it passes.**

If the gate would fail, you have two honest options:

- Fix the work product so the gate passes.
- Report failure with the gate output in your notes.

Do NOT exit 0 knowing the gate will fail. (The controller will
catch you mechanically, but you've wasted everyone's time.)

### R4 — When the clone uses HTTPS but you have an SSH key, switch the remote

The workdir is cloned via the project's `repo_url`. If that URL is
HTTPS and you need to push, you may have an SSH key mounted at
`~/.ssh/id_ed25519` that works for push when the URL is SSH-form.

Switch the remote:

```sh
git remote set-url origin git@github.com:<owner>/<repo>.git
git push -u origin <branch>
```

If neither HTTPS auth nor SSH push works, R1 applies: report
failure honestly.

Empirical evidence: spike-2's harness did exactly this; the SSH
key was unused under HTTPS clone but worked under SSH remote. Push
succeeded.

### R5 — PR creation gap: name it, don't hide it

If your environment lacks `gh` CLI **and** lacks `GITHUB_TOKEN`
in env **and** lacks a credential helper for the GitHub API:

You cannot open a PR programmatically. State this in your notes:

> *"Cannot open PR: no `gh` CLI installed, no `GITHUB_TOKEN`
> in env, no credential helper configured. Branch is pushed
> at <branch>. PR-link-to-open:
> https://github.com/<owner>/<repo>/pull/new/<branch>"*

The PR-open link gives the operator a one-click recovery path.
This is the right honest output when the environment doesn't
support PR creation.

### R7 — TDD red/green is mandatory

Every implementation step follows TDD:

1. **Red.** Write a failing test that captures the desired
   behavior. Run it. See it fail with a meaningful error.
2. **Green.** Write the minimum code to make it pass. Run again.
3. **Refactor.** Improve the code without changing behavior. Tests
   still pass.

Per-pyramid-level discipline:

- **Unit tests** for every function/class added or modified.
- **Integration tests** for cross-component behavior — real
  Postgres, real HTTP, real services.
- **Contract tests** when adding to SDK / HTTP / event-schema
  surface.
- **Scenario tests** when the change participates in a documented
  user-flow.

Run the full test suite before submitting. Submitting with red
tests is a methodology violation. Failure to write a test before
its implementation is a methodology violation.

Operational shape:

- Write test → confirm it fails → write code → confirm it passes
  → commit test+code together (or as two adjacent commits).
- Use the test names as a record of the spec coverage:
  `test_emit_event_persists_and_appears_in_query` is
  self-documenting.

### R8 — Apply the design patterns the spec calls for

Spec body's "Implementation sketch" names the patterns
(Strategy, Repository, Factory, etc.). Use them. If you would
prefer a different pattern, flag it in completion notes:

```
Spec called for Strategy pattern for the anomaly detector. I
implemented as a Specification (composable) instead because the
v2 configurability requirement needs composability. Judge: please
review.
```

Silent substitution — implementing a different pattern than what
the spec called for, without flagging — is a methodology
violation. The judge will issue `changes_requested`.

### R6 — Do not over-engineer

For `kind: bugfix`, do exactly what the spec says. Do not refactor
adjacent code "while you're in there." Do not add tests beyond
what the AC list specifies. Do not "improve" the spec's
implementation approach. Anything outside `acceptance_criteria` +
the spec's `Files touched` list is out of scope (per spec-author/
R5).

The judge will flag scope creep as a violation.

## Open methodology questions

- **EQ1.** When the harness encounters a recoverable error (a
  flaky network call, a transient API failure), what's the right
  retry policy? Currently the harness has no explicit retry
  protocol; it either succeeds, fails, or rationalizes. Future
  retros may surface a "retry up to N times with exponential
  backoff" pattern.
- **EQ2.** How explicit should the spec's `implementation.approach`
  be? Too prescriptive = the executor has no judgment room; too
  vague = different executors produce different shapes. Empirical
  observation needed.

## Scenario / kind-cluster execution (synthesized from vfobs-WG2 T5)

- **SC-1 — never trust the ambient kubeconfig for kind scenarios.**
  `~/.kube/config` may be a symlink to a *production* cluster
  kubeconfig. A naive `kind export kubeconfig` writes the kind
  context into whatever kubeconfig is active → pollutes / mutates
  prod config (kb gotcha `bkt4wMJZ`). Scenario prepare scripts MUST
  `kind export kubeconfig --name <c>` to a **dedicated file** and
  export `KUBECONFIG` for both the prepare run and the pytest run.
  Verify the prod/default kubeconfig is untouched afterward.
- **SC-2 — uuid-namespace every asserted id against a persistent
  shared store.** A reused kind cluster's Postgres survives across
  runs. Namespace **all** ids the test asserts counts on — both
  workgraph- and task-scoped (task-scoped reads are global). A
  test that passes once but fails on re-run is the tell.
- Fail-loud for scenario tasks is literal: do NOT mark complete
  unless the scenario suite runs green **end-to-end on the
  cluster** (WG1-T6 / vfobs-WG2-T5 precedent).

## References

- `viloforge-projects/vfobs/workgraphs/read-api/executor-retro.md`
  — vfobs-WG2 deviations (D-T0-1/D-T1-1/D-T2-1) + OF-1/OF-2
- `viloforge-projects/vafi/spikes/canary-end-to-end-2026-05-14/SPIKE-RETRO.md`
  — empirical evidence for R1 (the central rule)
- vafi controller code that consumes harness output:
  `vafi/src/controller/{controller,invoker,gates}.py`
