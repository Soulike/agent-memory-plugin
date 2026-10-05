# Consolidation Skill Design

> **Status: Draft — not finalized.** This design remains open to revision.

The consolidation side runs the Plugin's memory consolidation Skill to compress and forget older experience records according to the pool policy. Users can trigger it explicitly with a preset prompt or configure the same prompt in an agent that supports scheduled tasks.

Consolidation runs independently of user-side conversations. The host starts tasks and supplies the model and access environment; the Skill defines consolidation rules and reuses the Plugin's MCP to read and write the same pool. This document follows the [shared contract](../README.md), with tools, authentication, and submission outcomes defined in the [Plugin design](memory-plugin-design.md).

## Independent execution and prerequisites

The consolidation agent does not need to connect to or wake the user-side agent, or obtain its full chat logs. In its own runtime, it needs:

- The ability to load the consolidation Skill, execute the preset prompt, and call the Plugin's MCP.
- A fixed binding to the target pool, working git / gh CLI credentials, and permission to commit.
- A host that provides the model, scheduled triggers, and access to task results.

Manual and scheduled triggers execute the same task. Scheduled-task support is only one prerequisite; the host must also run the Skill and tools. Direct compatibility with every agent product is not assumed.

GitHub Actions, an agent's built-in scheduler, or another scheduling environment can host the task when these prerequisites are met. Environment-specific configuration covers the choice of host, deployment, and model authentication. The Plugin does not embed a scheduler, another agent, or a model API adapter.

## Triggers and preset prompt

The Plugin provides a separate consolidation Skill and a reusable prompt. The memory-use Skill does not trigger it by default. A scheduler repeats the same prompt, with the target pool selected by configuration rather than memory content.

The following is a draft prompt for future implementation, not an existing executable Skill:

~~~text
Run the Plugin's memory consolidation Skill for the currently bound memory pool.

Read the current pool version and retention policy, using only current-layer material.
Select bounded batches of older experience records and expired sources by their original
time ranges. Preserve the valid current collaboration profile.
Compress older records progressively. If the capacity target is still exceeded,
forget the oldest records; older key summaries may also be lost.
Retained summaries must express necessary source information and unresolved conflicts.
Conflicting groups may be compressed or forgotten together.
Do not reset timestamps when rewriting summaries or search Git history for more content.
Use MCP to commit each batch's summaries together with their corresponding deletions.
On version conflict, read again and reconcile.
Exit when no consolidation is needed. If an outcome is unknown, confirm the original
operation before retrying a submission.
Report the scope processed, compression and forgetting, accepted versions, and any
failures or unfinished work briefly.
~~~

Retention periods, capacity values, and batch sizes come from the pool policy and tool limits. The prompt does not fix a repository address, token, or model provider.

## Consolidation rules

| Material | Treatment |
| --- | --- |
| Valid current collaboration profile | Preserve it; do not forget it based on age or infer new preferences or agreements |
| Recent detailed experience records | Retain their detail for now |
| Older experience records | Merge into summaries of background, decisions, reasons, and results |
| Still older experience records | Compress into minimal core conclusions |
| Records still exceeding capacity after compression | Forget from oldest to newest by original time; older key summaries can also be discarded |
| Expired sources | Remove bodies from the current layer, retaining necessary source information and archive markers in summaries |
| Duplicates or unresolved contradictions | Merge duplicates; compress or forget conflicting groups together without manufacturing a one-sided conclusion |

Merge within the same context and nearby time periods. Summaries inherit original time ranges; do not conceal age by merging very old content into new records. Tier names such as detailed, summary, and minimal conclusion, along with age boundaries, remain policy parameters to be decided.

Detail may be lost, but unaccepted proposals must not become decisions, and summaries must not add unsupported conclusions. MCP's structural validation cannot establish semantic correctness. Summary quality needs validation with representative material.

First remove expired sources, then compress older records. If the target is still exceeded, forget the oldest records, removing unneeded associated metadata where appropriate. Account for all current content; do not bypass the capacity target by creating another permanent summary store or log.

The current profile is excluded from age-based eviction. If the valid profile or an individual input exceeds its own budget, forgetting other records cannot resolve that limit. Explain this to the caller and shorten the relevant content. Budget values and specific validation methods remain implementation decisions.

## Task execution

1. Read online and validate the bound pool. Obtain its current version and policy, then determine the scope to process. Return without changes when consolidation is unnecessary.
2. Read bounded batches of current material eligible by age or capacity. Each batch uses an explicit version and limits input and model context size.
3. The agent produces summaries and a deletion plan according to the Skill. MCP validates format, time ranges, source relationships, processing scope, and profile protection.
4. Submit summaries and corresponding deletions through the shared batch-change capability. If the version has changed, read again and reconcile affected material rather than forcing an overwrite or blindly replaying the old plan.
5. Record a brief task result. If a multi-batch task is interrupted, distinguish accepted batches from unfinished scope. A subsequent run reassesses the current pool.

Multiple consolidation tasks and user-side writes may run concurrently. The host can reduce duplicate triggers, but scheduling locks cannot replace pool version checks. Both roles use MCP's submission protection.

Consolidation does not maintain persistent cross-device runtime state or require another long-lived task database. If a submission response is lost, confirm it using the original operation identifier. Confirmation metadata does not expose historical content to the model. Recovery follows the [shared submission rules](memory-plugin-design.md#submission-and-failure-handling).

## Completion, delays, and history boundaries

Completion means the remote has accepted changes within the reported scope. Generating summaries or creating local commits is insufficient. No work needed is a normal outcome; read failures, submission failures, and unknown outcomes are reported separately. The host stores task results and provides failure notifications and rerun mechanisms.

Scheduled tasks may not finish as expected. The consolidation Skill does not promise exact execution times. The current capacity target and user-side write admission rules must be agreed together; scheduling frequency cannot replace write validation.

Compression and forgetting modify only the current layer. Older content remains in the history actually retained by Git. Routine consolidation does not search historical content or rewrite history. Explicit user requests to retrieve old memories or clean history use their separate entry points.

## Implementation validation and open choices

Implementation validation should cover:

- Another host can trigger consolidation while the user-side agent is completely stopped.
- No unnecessary commits are created when there is no work; manual and scheduled runs follow the same rules.
- Records become progressively shorter with age and are actually forgotten when capacity is insufficient, without resetting original time ranges.
- The current profile is preserved, and summaries retain the meaning of explicit corrections and unresolved conflicts.
- Replacing original records with summaries is committed as one batch; outdated consolidation cannot overwrite concurrent new records.
- After interruption or a lost response, outcomes can be confirmed and processing can continue from the current pool without duplicate writes.
- Routine consolidation does not recall history or store another permanent copy of the content.

Consolidation implementation discussions cover time policy, merge granularity, forgetting order, prompt content, batch limits, and quality evaluation. Scheduling deployment is configured separately for the chosen host without changing the user-side Plugin's everyday interface.
