# Issue tracker: GitHub

Issues and specifications live in [Soulike/agent-memory-plugin GitHub Issues](https://github.com/Soulike/agent-memory-plugin/issues). Use the `gh` CLI from this repository for tracker operations.

## Conventions

- When a Skill says to publish to the issue tracker, create a GitHub issue.
- When a Skill says to fetch a ticket, read the issue body, labels, and relevant comments.
- Filter issue lists by the relevant state and labels.
- Record progress and outcomes in issue comments; update labels and close issues as the workflow requires.
- Use the role-to-label mapping in [triage labels](triage-labels.md).
- GitHub issues and pull requests share a number space; resolve which kind a reference identifies before acting.

## Pull requests as a triage surface

**PRs as a request surface: no.**

## Wayfinding operations

Wayfinder uses one map issue and child issues as tickets.

- **Map:** label it `wayfinder:map`; its body holds Notes, Decisions-so-far, and Fog.
- **Child ticket:** link it as a sub-issue. If sub-issues are unavailable, use a task list in the map and a `Part of #<map>` reference in the child. Use `wayfinder:research`, `wayfinder:prototype`, `wayfinder:grilling`, or `wayfinder:task` for its type.
- **Blocking:** use native issue dependencies when available; otherwise put `Blocked by: #<number>` references in the child. A ticket is unblocked when all blockers are closed.
- **Frontier:** choose the first open child in map order with no open blockers and no assignee.
- **Claim:** assign the chosen ticket to the driving developer before working on it.
- **Resolve:** comment with the result, close the child, and add a context pointer to the map's Decisions-so-far.
