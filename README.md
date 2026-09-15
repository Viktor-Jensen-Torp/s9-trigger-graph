# S9 — does the trigger graph actually fire?

Echo-only workflows. **No Pi, no models, no cost.** They do exactly what the
real ones do to GitHub — mint App tokens, label, push, open pull requests,
comment — and nothing else. Every step prints what it saw.

The point is to settle the plumbing before spending a ten-minute model run on
the question "did the label event fire?".

## What it answers

1. Does `pull_request.labeled` fire a workflow when an **App token** applied the
   label? (`NOTES.md` §6 says App tokens chain. This is that claim, tested.)
2. Does pushing to a pull request branch re-trigger CI, and does the
   `workflow_run` that follows resolve the **right** pull request?
3. Does a pull request opened by an App get a CI run that actually **starts**,
   rather than one blocked pending approval?
4. Do label counters survive concurrent edits from two jobs?
5. **Does the merge-group branch really contain `pr-<n>`?** `review.yml` reads
   the pull request number out of it, and GitHub documents only the
   `gh-readonly-queue/{base}` prefix. This is the one place an assumption is
   written straight into a workflow.

## Setup

A throwaway repo in the org. Public, so the merge queue is available.

    gh repo create Viktor-Jensen-Torp/s9-trigger-graph --public --clone
    cd s9-trigger-graph
    git checkout -b develop && git push -u origin develop

Then:

1. Copy `workflows/*.yml` into `.github/workflows/`.
2. Ruleset on `develop`: require a pull request, require the check `ci`,
   **Require merge queue**. Name the branch exactly — a wildcard pattern cannot
   have a queue.
3. Register **one** GitHub App (this spike does not need three). Give it
   contents, issues and pull-requests write. Install it on this repo.
   Set `vars.APP_CLIENT_ID` and `secrets.APP_PRIVATE_KEY`.
4. Create the labels: `ready-to-develop`, `agent`, `needs:rework`,
   `needs:human`, `strike:1`, `strike:2`, `strike:3`, `force:red`.

## Driving it

    gh issue create --title "s9 probe" --body "nothing real"
    gh issue edit 1 --add-label ready-to-develop

Then watch, in order:

| Expect | Which question it answers |
|---|---|
| a branch `agent/issue-1` and a pull request | — |
| a CI run that **starts** without an approval banner | 3 |
| `echo-review` running after CI completes, naming PR #1 | 2 |
| a `VERDICT:` comment, then `echo-gatekeep` | — |
| the pull request entering the merge queue | — |
| `echo-ci` running again on a `gh-readonly-queue/...` ref | 5 |

Then the red path:

    gh pr edit 1 --add-label force:red      # makes echo-ci fail
    gh pr edit 1 --add-label needs:rework   # applied by YOU, not an App

The second one is the control: a label applied by a person always chains. Then
repeat it with the App token to answer question 1:

    gh workflow run relabel.yml -f pr=1

## What to record

Every answer goes in `NOTES.md` with how it was established. Especially
question 5 — write down the **literal** merge-group branch name you observe.
