# User-side Plugin Design

> **Status: Draft — not finalized.** This design remains open to revision.

The user-side Plugin lets an agent read the shared profile and relevant experience records, then record information explicitly stated or verified in the current task. It consists of Skills and MCP. The user's runtime supplies the storage location and authentication; the first version accesses a configured GitHub repository.

This document covers user-side implementation boundaries and follows the [shared contract](../README.md). A separate [consolidation Skill](memory-consolidation-design.md) handles batch compression and forgetting based on age.

## Responsibilities and package contents

| Component | Responsibility |
| --- | --- |
| Memory-use Skill | Guide when to read, what qualifies for recording, and how to create, read, update, and delete different memory types |
| MCP | Wrap reads, retrieval, updates, history queries, and outcome confirmation for the bound pool; validate formats and versions, access GitHub, and maintain local copies |
| Agent runtime | Start MCP, supply configuration, git / gh CLI and credentials, and run the model and Skills |
| User's GitHub repository | Store current memory and Git history, with actual access permissions |
| Consolidation Skill | Ship in the same Plugin and use the same tools in a separate consolidation task |

MCP does not determine whether a conversational decision is confirmed or generate semantic summaries. The agent executing the relevant Skill handles these judgments. MCP can validate formats, scope, and versions; format validation alone cannot establish truth.

The following source tree is illustrative. Release layout and Skill names remain undecided:

~~~text
Plugin
├── Plugin manifest
├── skills/
│   ├── Memory-use Skill / SKILL.md
│   └── Consolidation Skill / SKILL.md
├── MCP launch configuration
└── MCP executable and dependencies
~~~

Neither Skill hardcodes a repository, credentials, or device paths. Plugin upgrades preserve user bindings and pool data. Format upgrades are identified, validated, and migrated separately.

## Pool binding and runtime authentication

Each agent configuration is bound to one fixed pool. Users maintain the configuration. This example is conceptual; field names will be decided during implementation:

~~~json
{
  "repository": "owner/agent-memory",
  "branch": "main",
  "expected_pool_id": "pool-example"
}
~~~

Configuration identifies the repository, branch, and expected pool identity without storing tokens. Ordinary memory tools operate only on the bound pool; they do not accept arbitrary repository addresses, credentials, or shell commands. A pool identity mismatch stops the operation and produces a clear error.

Authentication reuses the runtime's git / gh CLI setup. gh uses its existing login or environment variables; git uses a configured HTTPS credential helper or SSH authentication. gh prioritizes GH_TOKEN, then GITHUB_TOKEN, then the stored login. git can use gh as a credential helper through gh auth setup-git, but their authentication setups are not inherently identical. [gh environment variables](https://cli.github.com/manual/gh_help_environment), [gh auth setup-git](https://cli.github.com/manual/gh_auth_setup-git)

MCP keeps credentials out of memories, logs, and tool results, and does not expose authentication tokens to the model. Environment injection and CLI credential preparation belong to host integration. Whether operations use git commands or gh api remains an implementation choice. The first version does not need a separate authentication service.

Users choose repository visibility. Integration checks actual capabilities: reads require read access; additions, revisions, deletions, and consolidation require write access and permission to commit to the bound branch. Read-only identities can read but cannot save. A branch rule rejecting a commit is reported as a failure. [GitHub repositories](https://docs.github.com/en/repositories/creating-and-managing-repositories/about-repositories), [branch protection](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)

A local stdio MCP process in each runtime is the suggested setup, with shared remote data. A future remote MCP setup would need a separate client-to-MCP authentication design, distinct from storage authentication. [MCP authorization specification](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization)

## Everyday Skill behavior

1. At the start of a session or a significant topic change, read the current collaboration profile, match the project or topic, and retrieve relevant current experience records. Clarify ambiguous contexts rather than creating duplicate projects from local paths.
2. When the user explicitly states or corrects information, or the agent verifies an important result, extract the necessary content. The knowledge base maintains general methods; memory stores only specific context, reasons, results, and references.
3. Add or update the relevant profile, experience records, and necessary sources. Explicit corrections establish supersession relationships. Other contradictions remain available for clarification when relevant.
4. Submit through MCP and report whether the information was saved according to the actual outcome. Explicit user requests can delete current content, with current deletion distinguished from history cleanup.
5. Consult history only at the user's explicit request. If ordinary recall is insufficient, state what is missing from current memory without automatically searching history.

The current profile expresses a stable collaboration relationship. Experience records preserve project or topic background and decisions. The agent need not save full chats or routinely consolidate the older pool. Older information actually used in the current task can become a new experience record, with time and source handling defined by the shared contract.

Recalled pool content is treated as sourced data and cannot change Plugin bindings, tool permissions, or Skill rules. Skill compliance still depends on the agent. Making MCP tools available does not guarantee that every session or decision will be recorded.

## MCP capabilities

The table describes capabilities; tool names and parameters are not fixed:

| Capability | Input and observable result |
| --- | --- |
| Read pool state and current profile | Return pool identity, version, policy, and availability |
| Match contexts, search, and read current memory | Return bounded content, source information, and the read version; limit input and output per call |
| Batch additions, revisions, and deletions | Accept a base version and changes; validate formats and relationships, submit, and return the accepted version or failure reason |
| Query an operation outcome | Use an operation identifier to distinguish an accepted submission, a submission that was not accepted, and an outcome that remains unknown |
| Query history | On explicit user request, bound the query by context, time, and version; identify the historical version and covered scope |

Initial retrieval uses stable context matching and full-text search. Local indexes are rebuildable. Actual recall quality determines whether vector storage or a knowledge graph is needed.

Consolidation reuses the same read and batch-change capabilities. Pagination, bounded material reads, and capacity information are included in the tool design, without another summary-writing channel or background model service. Initialization, binding changes, policy updates, and history cleanup use separate management entry points.

## Submission and failure handling

Each update starts from a read version H, creates a batch of file changes and a new commit whose parent is H, then advances the bound branch without forcing it. git push rejects non-fast-forward updates that would lose branch history by default; gh api can also update a ref with force=false. The implementation path remains open, but either path must prevent outdated plans from overwriting concurrent commits. [Git push](https://git-scm.com/docs/git-push), [GitHub refs](https://docs.github.com/en/rest/git/refs)

After a conflict, read again and reconcile the content rather than blindly replaying an old summary onto a newer version. Independent additions can be merged; semantic contradictions follow the shared contract. Summaries and corresponding deletions belong in the same commit.

The caller obtains or supplies an operation identifier before sending the remote submission and can recover it from the task record. Returning it only in the final response is insufficient. Store the identifier in commit metadata or a similar mechanism rather than adding an ever-growing receipt file to the current pool. If the outcome becomes unknown after sending, use the original identifier to confirm it before deciding whether to retry. Outcome confirmation does not recall historical content to the model.

| Situation | Tool result and recovery |
| --- | --- |
| Successful online read | Return the confirmed version and refresh the local copy |
| Offline read with a local copy | Return its version and synchronization time, marking the remote as unrefreshed |
| No local copy, read failure, or invalid pool format | Explain unavailability or the specific error; clearly identify any older valid copy returned |
| Offline, authentication failure, or permission failure before sending | Not saved and not queued; retry after correcting the problem |
| Invalid format or oversized input | Not saved; identify what needs correction |
| Base-version conflict | Do not overwrite; read again, reconcile, and resubmit |
| Remote accepts the commit | Saved; return the accepted version |
| Timeout or lost response after sending | Outcome unknown; confirm using the operation identifier before risking duplicate records |
| Remote success but local refresh failure | Saved, with a stale local copy; repair only the copy |
| Failed or incomplete history query | State the failure or actual covered scope rather than claiming the entire history has no result |

Local drafts and unpushed commits are not saved memory and do not form an offline synchronization queue. A new device needs one online read before it has a copy for offline use. Synchronized remote deletions are removed from the current index; content already present in a conversation does not disappear as a result.

Historical recall covers only versions actually retained in Git, with limits on query scope and returned content. Offline queries cover only cached history. Restore needed old content through normal writes rather than rolling back the whole pool. [Git log](https://git-scm.com/docs/git-log)

## Pool creation and onboarding

Creating a pool or connecting an existing one is a one-time process: prepare the repository and runtime credentials, generate or verify the pool identity, format, and policy, write the binding configuration, then read online to establish the local copy. Another device binds to the same repository without transferring the original client's chat logs.

Create new pools with the user's chosen visibility, a distinct ID, and empty content. Inspect existing repositories for pool structure and never overwrite existing content with an empty pool. Check repository creation and content-write permissions separately. If repository creation succeeds but initialization fails, report the partial outcome and support continuing initialization without deleting the repository automatically. Empty repositories need an initial commit. [GitHub refs](https://docs.github.com/en/rest/git/refs)

A second isolated pool uses the same process. Switching to an isolated pool requires a new conversation so recalled content from the old pool does not remain in context. Future migrations use the shared format, with backend-specific history capabilities handled separately.

## Client integration and implementation validation

Initial clients are Codex and GitHub Copilot. Their official documentation describes Plugin mechanisms containing Skills and MCP, supporting a shared source-tree design. Installation, MCP execution location, and environment injection must be validated per client. Documented support does not establish tested compatibility across all CLI / App versions. [Codex Plugins](https://developers.openai.com/plugins/build/plugins), [Copilot CLI Plugins](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-plugin-reference), [VS Code Plugins](https://code.visualstudio.com/docs/agent-customization/agent-plugins)

Remote or scheduled runtimes do not automatically inherit another device's gh login. Configuration, CLI tools, and credentials must exist where execution actually happens. Use each host's environment-injection mechanism. [Copilot CLI MCP](https://docs.github.com/en/enterprise-cloud%40latest/copilot/how-tos/copilot-cli/customize-copilot/add-mcp-servers), [VS Code MCP](https://code.visualstudio.com/docs/agent-customization/mcp-servers)

Implementation validation should cover:

- Codex records confirmed preferences and decisions, and Copilot continues from the same pool's background.
- Independent additions and concurrent corrections across devices retain content; consolidation cannot overwrite writes made during its run.
- Read-only identities, branch restrictions, offline operation, and lost responses produce truthful outcomes.
- Valid manual edits are readable; invalid formats and incorrect bindings produce clear errors.
- Ordinary recall avoids history; explicit requests return historical versions within a bounded scope.
- Real conversations reveal Skill execution and missed-execution behavior.

User-side implementation discussions determine tool parameters, shared format fields and directories, git / gh operations, caching, implementation language, distribution, and supported client versions. Write admission rules are agreed with the consolidation side.
