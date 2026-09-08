# GitHub Actions & CI

Run adversarial reviews automatically on pull requests. The Action runs the
Claude lane; extra provider lanes are local-CLI only.

```yaml
name: Adversarial Code Review

on:
  pull_request:
    types: [opened, ready_for_review, reopened, labeled]

jobs:
  review:
    if: >-
      github.event.action != 'labeled' ||
      github.event.label.name == 'review'
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: emouty/adversarial-review@v2
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `claude_code_oauth_token` | Yes | — | Claude Code OAuth token |
| `pr_number` | No | Triggering PR | PR number to review |
| `mode` | No | `no-fix` | `no-fix` (report only) or `auto-fix`. Unrecognized values fall back to `no-fix` (fail-safe) |
| `paths` | No | — | Comma-separated paths/globs to scope the review (passed as `--paths`) |
| `allowed_tools` | No | — | Additional allowed tools (comma-separated) |
| `model` | No | — | Model override for the lead agent |
| `comment` | No | `false` | `true` posts findings to the PR as a pending review (passed as `--comment`) |

By default the Action produces the report and `.reviews/` artifacts only — these
die with the runner unless uploaded (`actions/upload-artifact`). To post
findings on the PR, set the `comment` input to `true`; this lands as an
**unpublished** pending review owned by the workflow's token identity, which
must publish it (e.g. `gh api` in a later step) before it's visible.

## Recommended triggers

Avoid `synchronize` — it fires on every push, and the review is slow and
expensive. Use `labeled` with a `review` label to re-run after pushing fixes.

## Fork PRs and prompt injection

Fork PRs don't receive repository secrets by default, so the review job won't run
on them under `pull_request`. Be deliberate with the label-gated re-run path,
though: applying the `review` label to a fork PR runs the full pipeline — with
`contents: write` and `issues: write` — over attacker-authored code and
comments.

The agent prompts treat the diff and PR content as **data, not instructions**,
but that is a mitigation, not a security boundary. **Read a fork's diff before
labeling it.**

## Release automation

Releases are managed by release-please. A `feat:` or `fix:` commit on `main`
opens or updates a release PR; once that PR merges, release-please creates the
GitHub release and the workflow moves the floating `v1` tag.

To let release PRs run required checks and auto-merge, configure a GitHub App
token for each repository that uses this workflow:

1. Create a GitHub App, such as `release-please-automerge-bot`.
2. Grant repository permissions: **Contents: Read and write**, **Issues: Read and
   write**, **Pull requests: Read and write**, and **Metadata: Read-only**.
3. Generate a private key for the app.
4. Install the app on the repository, or on all repositories that should use
   release auto-merge.
5. Add repository or organization variable `RELEASE_APP_ID` with the app id.
6. Add repository or organization secret `RELEASE_APP_PRIVATE_KEY` with the full
   PEM private key.
7. Enable repository auto-merge in GitHub settings, or run
   `gh api -X PATCH repos/OWNER/REPO -f allow_auto_merge=true`.

Without those settings, the workflow falls back to `GITHUB_TOKEN`. That can
create the release PR, but GitHub suppresses follow-on workflow runs from
bot-created events, so branch protection may leave the release PR waiting for a
manual merge.
