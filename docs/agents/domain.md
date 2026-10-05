# Domain documentation

This repository uses a single context: domain vocabulary belongs in root `GLOSSARY.md`, and architectural decision records belong in `docs/adr/`.

## Before exploring

Read `GLOSSARY.md` if it exists and ADRs in `docs/adr/` that concern the area you are working on. If either is absent, proceed silently. Create them lazily when terms or consequential architectural decisions are resolved through domain modeling.

## Design work

For design or implementation work, start with the [overview and shared contract](../design/shared-contract.md), then read the relevant role design:

- [User-side Plugin](../design/memory-plugin-design.md): memory use, MCP, pool binding, authentication, and submission behavior.
- [Consolidation Skill](../design/memory-consolidation-design.md): independent execution, compression, forgetting, and task outcomes.

These documents remain drafts. Keep proposals and open implementation choices in `docs/design/`. Reconcile shared-format and capacity-interface changes in both role designs.

## Use the glossary vocabulary

Use defined terms in issue titles, proposals, hypotheses, and test names. If a needed concept is absent, reconsider the wording or note the gap for domain modeling. Keep the glossary focused on vocabulary.

## Flag ADR conflicts

If a proposal contradicts an existing ADR, identify the conflict and explain why the decision should be reopened.
