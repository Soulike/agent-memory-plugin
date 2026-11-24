# Agent Memory Plugin

Cross-agent memory through a shared GitHub repository. User-side agents read and record memories through Skills and MCP; independent consolidation agents compress and forget older experience records in the same pool.

> **Status: Draft — not finalized.** The Plugin has not been implemented. Cross-device reads, writes, and scheduled consolidation have not been validated.

## Design drafts

Start with the shared contract, then read the design for the role you are working on. All three documents remain open to revision.

| Document | Responsibility |
| --- | --- |
| [Overview and shared contract](docs/design/shared-contract.md) | Shared architecture, memory ownership, data and access rules, and implementation discussion boundaries |
| [User-side Plugin design](docs/design/memory-plugin-design.md) | Memory-use Skill, MCP capabilities, GitHub operations, runtime configuration, and authentication |
| [Consolidation Skill design](docs/design/memory-consolidation-design.md) | Independent execution, compression and forgetting, task outcomes, and scheduling prerequisites |

## Work tracking

Track issues and specifications in [GitHub Issues](https://github.com/Soulike/agent-memory-plugin/issues).

Engineering Skills follow [the Agent guidance](AGENTS.md), which routes to this repository's issue-tracker, triage-label, and domain documentation rules.

## Pull request reviews

Ready pull requests use [AI Review Workflow](https://github.com/Soulike/ai-review-workflow).
See [review integration and recovery](docs/agents/pull-request-review.md) for
repository configuration, verification, and failed-run recovery.

Draft pull requests start AI review when marked ready for review.
