# PR Team Reviewer Re-request

**File:** `.github/workflows/pr_review.yml`

## The problem

When any member of a requested reviewer team submits a review (approved or changes requested), GitHub automatically removes that team from the PR's "Requested Reviewers" list. GitHub treats one member's review as the whole team having reviewed. The result: remaining team members no longer see the PR in their review queue.

There is no native GitHub setting to change this behavior. The only built-in workaround is manually re-adding the team from the PR sidebar after each review.

This workflow re-adds the relevant teams automatically.

## What the workflow does

1. Triggers on `pull_request_review` events where the review state is `approved` or `changes_requested`.
2. Checks out the repo to read the `CODEOWNERS` file.
3. Fetches the PR's changed files via `gh pr view --json files`.
4. Applies GitHub's last-match-wins CODEOWNERS resolution to find only the teams that own the changed files.
5. Generates a short-lived GitHub App token scoped to the repo (see [Why a GitHub App token is required](#why-a-github-app-token-is-required)).
6. Re-requests each matched team, then polls `gh pr view --json reviewRequests` to confirm the team is present — retrying up to 6 times with 5-second delays (30 seconds max per team) to handle the race condition where GitHub removes the team again immediately while review event processing is still settling.

## Why a GitHub App token is required

`GITHUB_TOKEN` cannot add team reviewers on org-owned repos. A GitHub App installed on the org with `pull_requests: write` and `members: read` permissions is required. The token is scoped to the specific repo (not the whole org) using the `owner` and `repositories` parameters on `actions/create-github-app-token`.

## Setup

### 1. Create a GitHub App

In your org's GitHub settings, create a new GitHub App with the following permissions:

| Permission | Level |
|---|---|
| Pull requests | Read & write |
| Members | Read-only |

Install the app on the **org** (not just a single repo). Note the numeric **App ID** shown on the app's settings page.

Generate a **private key** (PEM format) from the app's settings page.

### 2. Add repo secrets

Add these two secrets to the repository (Settings > Secrets and variables > Actions):

| Secret | Value |
|---|---|
| `APP_ID` | The numeric app ID |
| `APP_PRIVATE_KEY` | The full PEM private key, including the `-----BEGIN/END-----` lines |

### 3. Add a CODEOWNERS file

The workflow looks for a `CODEOWNERS` file in these locations, in order:

- `CODEOWNERS`
- `.github/CODEOWNERS`
- `docs/CODEOWNERS`

Only `@org/team` entries are processed. Individual user entries (`@username`) are ignored.

### Setup checklist

- [ ] GitHub App created with `Pull requests: Read & write` and `Members: Read-only`
- [ ] App installed on the org
- [ ] `APP_ID` secret added to the repo
- [ ] `APP_PRIVATE_KEY` secret added to the repo
- [ ] `CODEOWNERS` file present with at least one `@org/team` entry

## CODEOWNERS matching

The workflow implements GitHub's **last-match-wins** rule: for each changed file, every rule in `CODEOWNERS` is evaluated top to bottom, and the last matching rule wins. The workflow collects the winning team for each changed file and deduplicates across all files.

### Example

```
*                    @org/platform
src/payments/**      @org/payments
src/payments/ui/**   @org/frontend
```

| Changed files | Teams re-requested |
|---|---|
| `src/payments/ui/Button.tsx` | `frontend` |
| `src/payments/api.ts` | `payments` |
| `README.md` | `platform` |
| `src/payments/api.ts` and `README.md` | `payments`, `platform` |

## Known limitations

- **3000-file cap.** `gh pr view --json files` returns up to 3000 files per PR (GitHub's GraphQL API hard limit). Files beyond that cap are not evaluated against CODEOWNERS.
- **Retry timeout.** The retry loop handles the race condition where GitHub removes a team immediately after re-adding during event processing. If GitHub's processing takes longer than 30 seconds, the workflow logs a failure for that team but does not fail the run.
- **User reviewers ignored.** Only `@org/team` entries in CODEOWNERS are re-requested. Individual `@username` entries are not processed.
- **`commented` state not handled.** The workflow only triggers on `approved` and `changes_requested` — the two states where GitHub removes the team. A `commented` review does not remove the team, so no re-request is needed.
