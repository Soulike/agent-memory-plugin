# Pull request reviews

Ready pull requests use [AI Review Workflow](https://github.com/Soulike/ai-review-workflow)
through [the repository caller](../../.github/workflows/review-pr.yml). The caller
follows the shared workflow's `main`; upstream changes take effect without a
version-update PR. AI review adds a comment review and the
`Review / Engine / AI review gate` check. High or medium findings block the gate.
A passing gate does not supply a human approval or authorize a merge.

## Maintain the integration

The [upstream setup guide](https://github.com/Soulike/ai-review-workflow/blob/main/docs/consumer-setup.md)
owns the workflow interface, credential prerequisites, and detailed recovery
instructions. This repository configures:

- The Actions secret `TAVILY_API_KEY`, passed explicitly to the shared workflow.
  Copilot uses the built-in `GITHUB_TOKEN`; model access and billing must be
  available to that token in this repository.
- The Actions variables `AI_REVIEW_MODEL` and `AI_REVIEW_REASONING_EFFORT`.
  Keep the effort supported by the selected model; an unset effort fails setup.
- An active Actions event policy allowing `pull_request_target` for
  `.github/workflows/review-pr.yml`.

Manage these through [Actions settings](https://github.com/Soulike/agent-memory-plugin/settings/actions)
and [secrets and variables](https://github.com/Soulike/agent-memory-plugin/settings/secrets/actions).
The [branch rules](https://github.com/Soulike/agent-memory-plugin/settings/rules)
and [repository settings](https://github.com/Soulike/agent-memory-plugin/settings)
control required checks, human approval, bypass permissions, PR admission, and
merge methods. Update those settings when changing the policy. Keep the calling
job's name aligned with the required check name, and bind the check to GitHub
Actions.

## Verify changes and recover

The caller runs on PR opening, reopening, updates, and conversion to ready.
Converting a PR to draft cancels its active review. Drafts cannot pass the gate.

`pull_request_target` loads the caller from the default branch. After deploying
a new caller there, trigger a fresh supported event on a ready PR. Verify setup,
inference, publication, and the gate, check that the bot's review identifies the
current head, and confirm there is no Actions event-policy warning. Make the
gate required only after this repository emits and passes the check.

If a review needs changes, address or discuss its findings and push the updated
head. If setup, inference, or publication fails, inspect the failed job, repair
the prerequisite, and rerun failed jobs. If an artifact expired or conflicts on
rerun, rerun all jobs. Rerunning only the gate uses the existing verdict; it
cannot produce a new review. Rerunning publication can add another visible
comment review.

A rerun reviews the original event's head. Trigger a new supported event to
review a different head. If an upstream change breaks execution, report the run
URL and shared implementation SHA to the workflow maintainer and follow the
upstream recovery guide.
