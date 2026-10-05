# Cross-agent Memory: Overview and Shared Contract

> **Status: Draft — not finalized.** All design documents in this repository remain open to revision.

This system has two independent roles: user-side agents use the Plugin to read and record memories; consolidation agents run the Plugin's consolidation Skill to compress and forget older experience records. The two roles run independently and share a memory pool and its data rules.

This document defines their shared contract. Implementation details are discussed separately:

| Design document | Responsibility |
| --- | --- |
| [User-side Plugin](docs/memory-plugin-design.md) | How the Skill guides memory CRUD, how MCP wraps GitHub operations, and how the runtime supplies configuration and authentication |
| [Consolidation Skill](docs/memory-consolidation-design.md) | How to select older material, compress and forget it, and run the task manually or on a schedule using a preset prompt |

These are design documents. The Plugin has not been implemented, and actual cross-device reads, writes, and scheduled consolidation have not been validated. Field names, tool names, directories, and numerical settings remain to be determined during implementation discussions.

## Shared architecture

~~~mermaid
flowchart LR
    U["User-side agent: memory-use Skill"] --> M1["Plugin MCP"]
    M1 --> R[("GitHub repository: one memory pool")]
    T["Host scheduler / explicit user trigger"] --> C["Consolidation agent: consolidation Skill"]
    C --> M2["Plugin MCP"]
    M2 --> R
~~~

The Plugin contains two Skills with different purposes and one shared set of MCP tools. Consolidation does not require the user-side agent to be running. Its own environment installs the Plugin, binds it to a pool, and supplies access credentials. The consolidation host provides scheduling and the model.

The Plugin provides rules and tools for operating on memory. The user supplies storage: in the first version, a GitHub repository selected through configuration and accessed with the runtime's git / gh CLI credentials. The Plugin's MCP implements repository operations and rebuildable local caches without providing a separate storage service.

One pool corresponds to one repository, and each agent configuration is bound to a fixed pool. Repository visibility is unrestricted; read, write, and target-branch commit permissions are checked for the requested operation. Separate pools provide isolation. Creating a new pool should quickly produce a distinct identity, empty data, and a policy. The shared pool is the sole authority for persistent memory; users disable native agent memory themselves.

## Memory content and ownership

| Content | What it stores | Owner and retention |
| --- | --- | --- |
| Pool collaboration profile | Current identity, user preferences, and explicit collaboration agreements | The pool's current profile; valid content is retained and superseded by explicit revisions |
| Project / topic experience records | Specific background, decisions, corrections, and important results | The pool's current records; compressed with age and eligible for forgetting when capacity is exceeded |
| Short-term sources | Key user quotations, verification evidence, and source identifiers | Short-term pool material; removed from the current layer on expiry, with necessary source information retained in summaries |
| General methods | How to do something, remaining valid outside a specific project | The knowledge base; memory references it and records only the adoption context, reasons, and results |
| Earlier versions | Content before revision, compression, or deletion | Git history; consulted only at the user's explicit request |
| Caches and indexes | Synchronized versions and derived retrieval data | Local to each runtime; rebuildable and without independent content authority |

Automatic recording accepts only information the user explicitly states or corrects, and important results verified by the agent. Alternatives under discussion and assistant inferences do not automatically become facts. Full chat logs remain with the original client; MCP does not collect complete sessions.

The shared identity describes collaboration roles and expectations. The runtime determines actual tools, permissions, and behavioral boundaries. Experience records describe the state at a particular confirmation point; changing code or remote state still needs verification when used. The system does not transfer running tasks, processes, or terminals.

## Shared data contract

Memory uses Markdown bodies with a small amount of metadata, allowing people to read, edit, and compare it directly. Manual edits and MCP writes follow the same format validation rules.

| Object | Required information |
| --- | --- |
| Pool | Stable pool ID, format version, retention and capacity policy |
| Context | Stable ID, project or topic, name and aliases; projects use a stable repository identity rather than a device-local path |
| Current profile | Currently valid content, sources, and explicit revision relationships |
| Experience record | Stable ID, context, original time range, compression level, status, and necessary source information |
| Correction / contradiction | Explicit supersession relationships or links between unresolved conflicting records |
| Short-term source | User quotation or verification evidence, occurrence time, and available source identifiers |
| History query result | The version and time represented, and the scope covered by the query |

Field names and file layout are determined in the shared format's implementation discussion, rather than defined independently for each role. Missing source identifiers must not be invented. When source bodies expire, retain source information and mark the bodies as absent from the current layer; do not present them as evidence that can still be read directly.

Explicit corrections can supersede earlier content. Contradictions without an explicit supersession relationship are retained and clarified when relevant. A later write is not inherently more correct. Consolidation may compress or forget a conflicting group together, but must not manufacture certainty by deleting only one side.

Record age is calculated from the original time range, which summaries inherit from their source material. Re-summarization, search hits, and routine reads do not reset age. Older content actually used in the current task may be recorded as a new experience with its sources. Access counts, last-used timestamps, and popularity scores are unnecessary.

## Shared access contract

Both roles access the bound pool through MCP. Reads and batch changes use identifiable pool versions. Submissions must check the base version to prevent an outdated plan from overwriting new records from the other role. Each batch commits summaries together with their corresponding deletions, avoiding an intermediate state where source text has been deleted but its summary has not been saved.

Updates require connectivity and count as saved only after the remote accepts the commit. Offline updates fail and are not queued. Offline reads may use a synchronized copy, identifying its version and synchronization time. Missing copies, invalid formats, and retrieval failures must not be presented as an absence of relevant memories. The [Plugin design](docs/memory-plugin-design.md#submission-and-failure-handling) defines detailed failure and recovery behavior shared by both roles.

Current memory, each recall, and Git history have separate budgets. Current capacity includes profiles, experience records, short-term sources, and metadata. Caches and indexes must not become another permanent content store. Valid current profile content is retained; experience records permit lossy compression and forgetting, including older key summaries.

Retention periods, compression levels, and capacity targets are stored in the pool policy and read by both roles. Numerical values remain open for implementation discussion. Scheduled consolidation alone cannot guarantee an immediate hard limit after every new write. The shared capacity interface must determine whether the consolidation target and write-time hard limit are the same, and how to accept new writes when consolidation falls behind.

Ordinary recall and routine consolidation process only the current layer. Git history is a cleanable archive whose content is consulted only at the user's explicit request. Version metadata used to confirm a submission is not historical-content recall. Before restoring old information to the current layer, verify it again and save the needed portion under the normal recording rules.

Deleting current content and cleaning Git history are separate operations. History cleanup requires explicit user initiation and a description of the archive scope that will be lost. It is not part of scheduled forgetting and does not guarantee physical removal of copies on other devices or old Git objects.

## Implementation discussion boundaries

The user-side discussion covers tool interfaces, git / gh operations, caching, pool creation, distribution, and client integration. The consolidation discussion covers time-based tiers, merging and forgetting rules, the consolidation prompt, batching, and summary quality. Changes to the shared format and capacity interface must be reflected in both designs.

The first version is for personal use, with reusable formats and interfaces, initially targeting Codex and GitHub Copilot. It uses the official MCP SDK with a small memory core. The running agent performs semantic judgments; MCP does not make separate model calls. [Official TypeScript SDK](https://ts.sdk.modelcontextprotocol.io/v2/)

Both roles integrate through Skills and accept some risk of missed execution. Hooks and maintained agent source forks are excluded. Other storage backends remain possible extensions outside the first version.
