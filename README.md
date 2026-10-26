# openrouter-review

A reusable GitHub Actions workflow that reviews each pull request with [opencode](https://github.com/anomalyco/opencode) on a free [OpenRouter](https://openrouter.ai/) model and posts the review as one comment in Japanese.

- The model is `google/gemma-4-31b-it:free`. Its free endpoint (Google AI Studio) is listed by OpenRouter as not training on prompts and keeping them 55 days. Every request is sent with `data_collection: deny`, so OpenRouter refuses it rather than route it to a provider that may train on it.
- The agent can only read: it runs as opencode's `plan` agent with no shell, no file edits, no web fetch or search, and no paths outside the repository except the PR's diff, which a step writes beforehand. The repository's own opencode config (`opencode.json`, `.opencode/`) is not loaded, so a PR cannot widen these.
- Drafts are reviewed too. The latest of a burst of pushes gets a new review (earlier runs are cancelled); an earlier comment is not edited.
- Pull requests from forks, and runs started by a bot, are skipped: forks and Dependabot get no secrets, and opencode refuses bot actors.

This is [opencode-review](https://github.com/TakashiAihara/opencode-review) with the model provider swapped from opencode Zen to OpenRouter.

## Use

1. Create an OpenRouter API key and add it as the repository secret `OPENROUTER_API_KEY`. Keep the account at no balance and no auto top-up: a leaked key then reaches the free models only.
2. Add `.github/workflows/openrouter-review.yml`:

```yaml
name: openrouter-review

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read

jobs:
  review:
    permissions:
      contents: read
      pull-requests: write
      issues: write
    uses: TakashiAihara/openrouter-review/.github/workflows/review.yml@main
    secrets:
      OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
```

`@main` follows this repository, so a model or prompt change here reaches every caller at once. Pin a commit instead if you want to review each change before it runs.

To use another model, pass `with: { model: <OpenRouter model id> }`, for example `cohere/north-mini-code:free`. A model whose free endpoints all may train on prompts fails, because of `data_collection: deny`.

## Limits

- OpenRouter caps free models at 20 requests a minute and 50 a day per account, or 1000 a day once the account has bought at least 10 credits ([limits](https://openrouter.ai/docs/api/reference/limits)). One review is several requests (the agent reads files one call at a time), and every repository using the same account shares the cap. More accounts do not raise it, and the [terms](https://openrouter.ai/terms) forbid making them for that.
- Public repositories only for now. opencode fetches the PR branch without credentials, and the opencode binary the action installs is the latest release, not a pinned one.
- When a free model is withdrawn, the job fails until `model` is changed here.
- The model reads the PR's diff and messages, so text in them can steer what the review says. With no shell and no edits, the comment is the only thing it writes.
