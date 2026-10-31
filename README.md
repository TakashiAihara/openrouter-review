# openrouter-review

A reusable GitHub Actions workflow that reviews each pull request with [opencode](https://github.com/anomalyco/opencode) on an [OpenRouter](https://openrouter.ai/) model and posts the review as one comment in Japanese.

- The calling repository names the model; there is no default. Every request is sent with `data_collection: deny`, so OpenRouter refuses it rather than route it to a provider that may train on it.
- The agent can only read: it runs as opencode's `plan` agent with no shell, no file edits, no web fetch or search, and no paths outside the repository except the PR's diff, which a step writes beforehand. The repository's own opencode config (`opencode.json`, `.opencode/`) is not loaded, so a PR cannot widen these.
- Drafts are reviewed too. The latest of a burst of pushes gets a new review (earlier runs are cancelled); an earlier comment is not edited.
- Each run keeps opencode's sessions (the prompt, replies, tool calls, the model's reasoning when it gives any, and subagent sessions) and log as the artifact `openrouter-review-logs` for 14 days, to tune the prompt from. opencode's own system prompt is not in it, because opencode does not save it: it is a file opencode picks by model family, and the plan-mode text it adds to the prompt is not saved either; both are in opencode's source. The repository's root `AGENTS.md` is not loaded; one in a subdirectory is attached when the agent reads a file there. opencode's log holds little: it is written in batches, and at v1.19.4 the kept file covered only the first seconds of a run. Artifacts of a public repository can be downloaded by anyone signed in to GitHub; the bundle is dropped, and the job fails, if it contains the API key or the GitHub token.
- Pull requests from forks, and runs started by a bot, are skipped: forks and Dependabot get no secrets, and opencode refuses bot actors.

This is [opencode-review](https://github.com/TakashiAihara/opencode-review) with the model provider swapped from opencode Zen to OpenRouter.

## Use

1. Create an OpenRouter API key and add it as the repository secret `OPENROUTER_API_KEY`. Give the key a credit limit: it is what a leaked key can spend. At $0 only free models answer; a paid model such as the example below needs a limit above $0 and a balance.
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

`@main` follows this repository, so a prompt or permission change here reaches every caller at once. Pin a commit instead if you want to review each change before it runs.

`opencode-version` picks the [opencode release](https://github.com/anomalyco/opencode/releases) to review with, as `X.Y.Z`. It defaults to the version this repository was checked with, and a new upstream release does not change it. Only an immutable release is accepted, and the download is checked against the sha256 digest GitHub recorded when it was published; a non-immutable release fails the job. opencode's own GitHub action is not used, because it takes no version and installs whatever release is marked latest.

`model` is an OpenRouter model id without the `openrouter/` prefix; `z-ai/glm-5.3-flash` above is an example, not a default: it is the paid counterpart of the free model that found both bugs below, and every endpoint OpenRouter lists for it does not train on prompts (2026-10-27). It was not measured here. A model whose endpoints all may train on prompts fails, because of `data_collection: deny`.

## Review instructions per path

To give the reviewer guidance for some files, add `.github/openrouter-review.yaml`. It takes the same shape as CodeRabbit's [`reviews.path_instructions`](https://docs.coderabbit.ai/reference/configuration), so entries can be copied from a `.coderabbit.yaml` as long as their globs avoid the syntax listed below:

```yaml
reviews:
  path_instructions:
    - path: "**/*.ts"
      instructions: |
        Point out a try/catch that swallows an error.
    - path: ".github/workflows/**"
      instructions: |
        Check that third-party actions are pinned to a commit SHA.
```

- Only entries whose `path` glob matches a file changed by the pull request are passed to the reviewer, with the matched files listed. For those files they decide what the reviewer checks over the general review prompt, so an entry can ask for checks the prompt leaves out, such as style. The output format and the read-only rules stay.
- Globs are matched by git's `:(glob)` pathspec, not CodeRabbit's minimatch. `**/*.ts` matches `a.ts` at the root too, and `src` matches everything under `src/`. Braces (`**/*.{ts,js}`), extglob (`+(a|b)`), and a leading `!` or `#` are not supported and fail the job; split or rewrite such an entry. A `{` is refused even inside `[...]`.
- The file is read from the base branch, not the pull request, so a pull request cannot edit it to change its own review. An edit to the file takes effect once it is merged.
- The job fails when the file has no `reviews.path_instructions` list (a misspelt `reviews` or `path_instructions`, or `path_instructions` at the top level), when an entry lacks `path` or `instructions` or has instructions over 20000 characters, and when the file is not valid YAML. Other keys are ignored, so a misspelt key next to the list is not caught. The run log says how many entries were read and which matched.
- This is separate from `AGENTS.md`, which the reviewer does not load at the repository root. An `AGENTS.md` or `CONTEXT.md` in a subdirectory, including one the pull request adds, is still attached when the reviewer reads a file there; opencode has no setting to stop that. `CLAUDE.md` is not loaded.

## Limits

- OpenRouter caps free models at 20 requests a minute and 50 a day per account, or 1000 a day once the account has bought at least 10 credits ([limits](https://openrouter.ai/docs/api/reference/limits)). One review of this repository's first pull request took 15 to 18 requests with `cohere/north-mini-code:free` (runs 40321742692 and 40322027041; the agent reads files one call at a time), and every repository using the same account shares the cap. More accounts do not raise it, and the [terms](https://openrouter.ai/terms) forbid making them for that.
- Public repositories only for now. opencode fetches the PR branch without credentials.
- When a model is withdrawn, the job fails until the caller changes `model`.
- The model reads the PR's diff and messages, so text in them can steer what the review says. With no shell and no edits, the comment is the only thing it writes.

## Free models tried (2026-10)

Each model reviewed a replay of a pull request with two known bugs, which opencode-review's `opencode/glm-5.3-flash-free` found. Both bugs sit in a file outside the diff that the agent can read. The runs used this workflow with `data_collection: deny` removed, so that models that train on prompts could answer.

| Model | Trains on prompts | Known bugs found |
|---|---|---|
| `nvidia/nemotron-3-ultra-550b-a55b:free` | yes | 0 of 2 |
| `nvidia/nemotron-3.5-lightning:free` | yes | 0 of 2 |
| `cohere/north-mini-code:free` | no | 0 of 2, and reported a bug that is not there |
| `google/gemma-4-31b-it:free` | no | not measured: rate-limited upstream on every run |

The NVIDIA free endpoints also come under NVIDIA's [API Trial terms](https://assets.ngc.nvidia.com/products/api-catalog/legal/NVIDIA%20API%20Trial%20Terms%20of%20Service.pdf), which allow trial and evaluation only, not production use.
