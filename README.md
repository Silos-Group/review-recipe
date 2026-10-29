# review-recipe

A Claude review on **every push** to every pull request, in any repo of any Silos silo. A review comes back
after each push, checks whether earlier findings were fixed, and never posts the same finding twice.

This repo holds only the recipe: the review command, a reusable GitHub workflow and a Claude Code plugin for
local sessions. It has no secrets and no knowledge of any silo; each silo supplies its own configuration. The
design is Silos ADR-019 ("A full review sees the whole stack, in any silo") and its review standard.

## What runs

- **The review command** (`plugins/silos-review/commands/review.md`): Anthropic's `code-review` command with
  one change of behaviour. It does not skip a pull request it has already reviewed. Each re-review lists earlier
  findings as fixed or still open, posts only new ones inline, and records the commit it reviewed. Separate agents
  still double-check every finding before it is posted, as upstream.
- **The reusable workflow** (`.github/workflows/review.yml`): runs that command with `anthropics/claude-code-action`
  on the calling repo's runners, posts with that workflow's `GITHUB_TOKEN`, and reaches Claude through Claude
  Console workload identity federation, so no API key is stored anywhere.

## Add it to a repo

1. In the Claude Console, add a workload-identity **federation rule** for the repo. Match GitHub's immutable OIDC
   subject (`repo:<org>@<org-id>/<repo>@<repo-id>:pull_request`) plus the claims `event_name = pull_request` and
   `repository_id`, and target the silo's review service account.
2. Set three repository **variables** (identifiers, not secrets): `ANTHROPIC_FEDERATION_RULE_ID`,
   `ANTHROPIC_SERVICE_ACCOUNT_ID`, `ANTHROPIC_ORGANIZATION_ID`.
3. Add `.github/workflows/claude-review.yml`:

```yaml
name: claude-review
on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]
permissions:
  contents: read
  pull-requests: write
  issues: read
  id-token: write
concurrency:
  group: claude-review-${{ github.event.pull_request.number }}
  cancel-in-progress: false  # queue: a review killed mid-post leaves findings the next review cannot see
jobs:
  review:
    uses: Silos-Group/review-recipe/.github/workflows/review.yml@<commit>
    with:
      federation_rule_id: ${{ vars.ANTHROPIC_FEDERATION_RULE_ID }}
      service_account_id: ${{ vars.ANTHROPIC_SERVICE_ACCOUNT_ID }}
      organization_id: ${{ vars.ANTHROPIC_ORGANIZATION_ID }}
```

Pin a full commit of this repo, and move the pin on purpose. The workflow installs the review command from that same commit, which GitHub reports in the run's identity token.

## What the review may do

It reads the pull request (`gh pr view`, `gh pr diff`, `gh issue view`, `gh search`), posts one summary comment
(`gh pr comment`) and inline comments. It has no `gh api`, because `gh api` accepts write methods and the review
reads untrusted pull request text.

## Use it in a local Claude Code session

```
/plugin marketplace add Silos-Group/review-recipe
/plugin install silos-review@silos-review-recipe
/silos-review:review <owner>/<repo>/pull/<number>
```

## Settings a HIPAA-configured Claude org needs

The workflow sets `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS=1`, because the API rejects the `context_management` beta
for HIPAA-configured orgs. It also sets `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1`: otherwise the review's sub-agents
run in the background, the job ends before they report, and nothing is posted.

## Licence

Apache License 2.0. The review command is derived from Anthropic's `code-review` command, also Apache 2.0; see
`NOTICE` for what changed.
