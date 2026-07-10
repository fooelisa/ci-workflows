# ci-workflows

Reusable workflows for `fooelisa/*` repos, used from both Forgejo Actions and GitHub Actions.

Currently one workflow: **`ai-review`** — AI code reviewer using Claude (via the [claude-action-runner](https://github.com/fooelisa/claude-action-runner) image).

## Opting a repo into AI review

### GitHub-hosted repo

1. **Set an org secret** (one-time): `ANTHROPIC_API_KEY` at `https://github.com/organizations/fooelisa/settings/secrets/actions`. Generate a fresh key at [console.anthropic.com/settings/keys](https://console.anthropic.com/settings/keys) — dedicated to CI, not a personal-use key.
2. **Add a caller workflow** to the consumer repo:

    ```yaml
    # .github/workflows/ai-review.yaml
    on:
      pull_request:
        types: [opened, synchronize, reopened]
    # Required: caller must grant what the reusable workflow needs.
    # GHA rejects the run at parse time otherwise.
    permissions:
      contents: read
      pull-requests: write
      issues: write
    jobs:
      review:
        uses: fooelisa/ci-workflows/.github/workflows/ai-review.yaml@main
        secrets: inherit
    ```

3. Open a PR → review comment appears within 30–90s.

### Forgejo-hosted repo

1. **Create the `claude-reviewer` Forgejo user** (one-time):

    ```
    kubectl -n forgejo exec deploy/forgejo -- \
      forgejo admin user create --username claude-reviewer \
        --email claude-reviewer@dyinggiraffe.org --random-password
    ```

    Log in as that user in the Forgejo UI, generate a PAT with scopes `read:repository` + `write:issue`.

2. **Set two org secrets** at `https://forgejo.motmot-carp.ts.net/org/fooelisa/-/settings/actions/secrets`:
    - `ANTHROPIC_API_KEY`: same value as the GitHub org secret
    - `FORGEJO_REVIEW_TOKEN`: the PAT from step 1

3. **Add a caller workflow** to the consumer repo:

    ```yaml
    # .forgejo/workflows/ai-review.yaml
    on:
      pull_request:
        types: [opened, synchronize, reopened]
    jobs:
      review:
        uses: fooelisa/ci-workflows/.forgejo/workflows/ai-review.yaml@main
        secrets: inherit
    ```

## What lands on the PR

One markdown comment with four severity buckets (Critical / Warnings / Suggestions / Nits) plus a one-sentence summary. Re-pushes update the same comment (matched by an HTML marker) rather than posting a new one.

Diff filtering (lockfiles, `dist/`, `build/`, `node_modules/`, `vendor/`, `*.min.*`, `*.generated.*`) happens before the model sees anything. Diffs >150K chars post a "diff too large" comment and skip the review.

## Tuning the prompt

The system prompt lives in [claude-action-runner](https://github.com/fooelisa/claude-action-runner)'s `system-prompt.md`. Edit + push there; the next image build rolls forward when this workflow's image pin bumps.

## Where it runs

- **Forgejo callers**: on `pik8s04` (the forgejo-runner pod). Docker memory limit: 512 MiB.
- **GitHub callers**: on github-hosted runners. Zero impact on your cluster.
