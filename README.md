# openrouter-review

A reusable GitHub Actions workflow that reviews each pull request with [opencode](https://github.com/anomalyco/opencode) on an [OpenRouter](https://openrouter.ai/) model and posts the review as one comment in Japanese.

- The calling repository names the model; there is no default. Every request is sent with `data_collection: deny`, so OpenRouter refuses it rather than route it to a provider that may train on it.
- The agent can only read: it runs as opencode's `plan` agent with no shell, no file edits, no web fetch or search, and no paths outside the repository except the PR's diff, which a step writes beforehand. The repository's own opencode config (`opencode.json`, `.opencode/`) is not loaded, so a PR cannot widen these.
- Drafts are reviewed too. The latest of a burst of pushes gets a new review (earlier runs are cancelled); an earlier comment is not edited.
- Pull requests from forks, and runs started by a bot, are skipped: forks and Dependabot get no secrets, and opencode refuses bot actors.

This is [opencode-review](https://github.com/TakashiAihara/opencode-review) with the model provider swapped from opencode Zen to OpenRouter.

## Use

1. Create an OpenRouter API key and add it as the repository secret `OPENROUTER_API_KEY`. Set the key's credit limit to $0: free models still answer, and a leaked key cannot spend the account's balance on paid ones.
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
    with:
      model: z-ai/glm-5.3-flash
    secrets:
      OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
```

`@main` follows this repository, so a model or prompt change here reaches every caller at once. Pin a commit instead if you want to review each change before it runs.

`model` is an OpenRouter model id without the `openrouter/` prefix; `z-ai/glm-5.3-flash` above is an example, not a default: it is the paid counterpart of the free model that found both bugs below. A model whose endpoints all may train on prompts fails, because of `data_collection: deny`. A paid model needs a balance and a key credit limit above $0.

## Limits

- OpenRouter caps free models at 20 requests a minute and 50 a day per account, or 1000 a day once the account has bought at least 10 credits ([limits](https://openrouter.ai/docs/api/reference/limits)). One review took 15 to 18 requests here (the agent reads files one call at a time), and every repository using the same account shares the cap. More accounts do not raise it, and the [terms](https://openrouter.ai/terms) forbid making them for that.
- Public repositories only for now. opencode fetches the PR branch without credentials, and the opencode binary the action installs is the latest release, not a pinned one.
- When a model is withdrawn, the job fails until the caller changes `model`.
- The model reads the PR's diff and messages, so text in them can steer what the review says. With no shell and no edits, the comment is the only thing it writes.

## Free models tried (2026-10)

Each model reviewed a replay of a pull request with two known bugs, which opencode-review's `opencode/glm-5.3-flash-free` found. Both bugs sit in a file outside the diff that the agent can read.

| Model | Trains on prompts | Known bugs found |
|---|---|---|
| `nvidia/nemotron-3-ultra-550b-a55b:free` | yes | 0 of 2 |
| `nvidia/nemotron-3.5-lightning:free` | yes | 0 of 2 |
| `cohere/north-mini-code:free` | no | 0 of 2, and reported a bug that is not there |
| `google/gemma-4-31b-it:free` | no | not measured: rate-limited upstream on every run |

The NVIDIA free endpoints also come under NVIDIA's [API Trial terms](https://assets.ngc.nvidia.com/products/api-catalog/legal/NVIDIA%20API%20Trial%20Terms%20of%20Service.pdf), which allow trial and evaluation only, not production use.
