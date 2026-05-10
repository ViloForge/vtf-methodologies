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

## Current state — `v0.0`

Empty stub. Populated during SDD adoption Phase 1 (humans run all
roles by hand on real workgraphs; methodology drafts emerge from those
runs and land here).

See
[implementation-roadmap-PLAN.md](https://github.com/viloforge/viloforge-platform/blob/main/docs/implementation-roadmap-PLAN.md)
for the phased plan.

## References

- [`spec-driven-adoption-DESIGN.md`](https://github.com/viloforge/viloforge-platform/blob/main/docs/spec-driven-adoption-DESIGN.md) — how methodologies bind agents
- [`project-repo-DESIGN.md`](https://github.com/viloforge/viloforge-platform/blob/main/docs/project-repo-DESIGN.md) — how Project Repos consume methodologies
- [`vafi-runtime-DESIGN.md`](https://github.com/viloforge/viloforge-platform/blob/main/docs/vafi-runtime-DESIGN.md) — runtime composition mechanics
