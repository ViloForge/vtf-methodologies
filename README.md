# vtf-methodologies

Global methodology files for the ViloForge software factory.

## Purpose

This repo holds **role methodology files** that govern how agents (and
humans acting in those roles) execute work in the
[ViloForge software factory](https://github.com/viloforge/viloforge-platform).

A methodology file is the constitution for a role at a particular
complexity tier. It tells the architect how to architect, the
spec-author how to author specs, the verifier what to verify, the
executor how to execute, the judge how to judge.

## Layout

When populated, this repo contains:

```
architect/
  feature.md
  bugfix.md
  refactor.md
  research.md
  trivial.md
spec-author/
  feature.md
  bugfix.md
  ...
verifier/
  feature.md
  ...
executor/
  executor.md
  pi.md
judge/
  feature.md
  ...
triage/
  triage.md
retrospector/
  <kind>.md
```

Per-role × per-kind. The runtime composes the right methodology at
agent claim time using the workgraph's `kind:` field.

## Versioning

This repo is versioned via semver git tags (`v0.0`, `v0.1`, …). Each
[Project Repo](https://github.com/viloforge-projects)'s `project.yaml`
pins to a specific tag:

```yaml
methodologies_pin: vtf-methodologies@v0.1
```

Bumping a project's pin is a deliberate operator action.

## Layered composition

Methodology files in this repo are the **global** layer. Per-project
overrides live in each Project Repo's `methodologies/` directory; per-
workgraph overrides live in the workgraph's own `methodologies/`
directory. At agent claim time the runtime composes
**global → project → workgraph**, with later layers overriding
earlier ones.

See
[`vafi-runtime-DESIGN.md`](https://github.com/viloforge/viloforge-platform/blob/main/docs/vafi-runtime-DESIGN.md)
for the resolution mechanics.

## Current state — `v0.0 → drafting v0.1`

**As of 2026-05-14, four first-draft methodology files exist:**

- `spec-author/bugfix.md` — load-bearing rules for spec-authoring;
  encodes the externally-grounded AC + `test_command` gate +
  fail-loud language requirements (R1–R10).
- `executor/bugfix.md` — fail-loud central rule; HTTPS→SSH switch
  pattern; PR-creation-gap honesty pattern.
- `judge/bugfix.md` — external-verification discipline; per-criterion
  verdict structure; treating executor honesty as a positive signal.
- `verifier/bugfix.md` — 11 mandatory checks (V1–V11) that gate
  `specced → ready` transitions.

These were drafted from the **first** SDD-discipline workgraph
(`vafi-rolling-restart-fix`, 2026-05-14) plus the canary-end-to-end
spike that ran the same day. Both rolled up multiple walk-backs
(W1–W4) and an empirical pipeline-failure-mode discovery: **the
verification loop is closed-system** and rubber-stamps ghost-
completion unless the methodology forces external grounding.

**Draft synthesis is rolling, not workgraph-end-only.** Every
HIGH-maturity finding in a stage retro triggers an immediate
patch here rather than waiting for the workgraph to complete.
This means future spec-authors (autonomous or operator) read
methodology that already encodes lessons from in-flight workgraphs,
not just completed ones.

Source of truth for any rule cited here: the stage-retro file in
the workgraph dir that produced it. Cross-reference the
`draft_synthesized_from:` frontmatter on each methodology file.

See
[implementation-roadmap-PLAN.md](https://github.com/viloforge/viloforge-platform/blob/main/docs/implementation-roadmap-PLAN.md)
for the phased plan.

## References

- [`spec-driven-adoption-DESIGN.md`](https://github.com/viloforge/viloforge-platform/blob/main/docs/spec-driven-adoption-DESIGN.md) — how methodologies bind agents
- [`project-repo-DESIGN.md`](https://github.com/viloforge/viloforge-platform/blob/main/docs/project-repo-DESIGN.md) — how Project Repos consume methodologies
- [`vafi-runtime-DESIGN.md`](https://github.com/viloforge/viloforge-platform/blob/main/docs/vafi-runtime-DESIGN.md) — runtime composition mechanics
