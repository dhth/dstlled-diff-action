# dstlled-diff action

✨ Overview
---

`dstlled-diff` (short for "distilled-diff") is a GitHub action that processes a
specific git revision range and generates a diff that only includes changes in
the signatures of various code constructs (such as classes, functions, and
objects, etc.). It is powered by [dstll][1].

The main goal of `dstlled-diff` is to simplify the review of structural code
changes by removing diff components that do not alter signatures.

![pr-comment](https://github.com/user-attachments/assets/808b43eb-76a7-4832-98e2-e6c85b3ddfe0)

⚡️ Usage
---

```yaml
name: dstlled-diff

on:
  pull_request:

jobs:
  dstlled-diff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - id: get-dstlled-diff
        uses: dhth/dstlled-diff-action@v0.1.0
        with:
          starting-commit: ${{ github.event.pull_request.base.sha }}
          ending-commit: ${{ github.event.pull_request.head.sha }}
          post-comment-on-pr: 'true'
```

The "dstlled-diff" will be available as an output of the action, via
`steps.get-dstlled-diff.outputs.diff`.

🔡 Inputs
---

Following inputs can be used as `step.with` keys:

| Name                 | Type   | Default | Description                                                        |
|----------------------|--------|---------|--------------------------------------------------------------------|
| `starting-commit`    | String |         | Starting commit for the git revision range                         |
| `ending-commit`      | String |         | Ending commit for the git revision range                           |
| `pattern`            | String | `*`     | Pattern to run dstlled-diff on                                     |
| `directory`          | String | `.`     | Working directory (below repository root)                          |
| `post-comment-on-pr` | Bool   | `false` | Post comment containing dstlled-diff to corresponding pull request |
| `save-diff-to-file`  | Bool   | `false` | Save diff to a local file called `diff.patch`                      |

⚙️ Use cases
---

### Web Interface

The output of `dstlled-diff` can be rendered in a web view, as seen in the image
below. Code for this can be found [here](./.github/workflows/web-demo.yml).

![web-demo](https://github.com/user-attachments/assets/7e7e75ab-1e50-450f-82d6-a5d5fa7d834f)

[1]: https://github.com/dhth/dstll
