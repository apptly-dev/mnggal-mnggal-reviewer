# mnggal-mnggal-reviewer

A composite GitHub Action that wraps
[open-code-review](https://github.com/alibaba/open-code-review) (OCR) with the
compute3 LLM backend preconfigured and its own **mnggal-mnggal** GitHub App
identity. Adding the reviewer to a repo becomes a few lines, not a copy of every
`llm_*` input.

The heavy lifting is off-box: the action runs on the self-hosted **runner1**
guest, which calls **compute3's** Ollama endpoint over the LAN. See
`../runner1.md` and `../compute3.md`.

## Usage

Drop one workflow into each repo to be reviewed. The trigger, fork guard and
`runs-on` stay in the caller (a composite action can't set them); everything else
— the LLM backend and the App identity — is preconfigured, so the caller passes
nothing:

```yaml
name: OCR Review
on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]
concurrency:
  group: ocr-review-${{ github.event.pull_request.number }}
  cancel-in-progress: true
jobs:
  ocr-review:
    if: |
      github.actor != 'renovate[bot]' &&
      !github.event.pull_request.draft &&
      github.event.pull_request.head.repo.full_name == github.repository
    runs-on: [self-hosted, linux, x64, runner1]
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: apptly-dev/mnggal-mnggal-reviewer@v1
```

Override a default only for a different backend, e.g.
`with: { llm_model: some-other-model }`.

## Identity — the mnggal-mnggal GitHub App

Comments post under the App (e.g. `mnggal-mnggal[bot]`) instead of
`github-actions[bot]`. Create the App once, then install it on every org whose
repos use the reviewer.

One-time, in the GitHub UI:

1. Create a GitHub App named **mnggal-mnggal** (owner: `apptly-dev`; make it
   public so other orgs can install it). Repository permissions: **Pull
   requests: read & write**, **Contents: read**. No webhook.
2. Note its **App ID** and generate a **private key** (PEM).
3. Install the App on each org, all repositories (or a chosen set).

Then set `app_id`'s default in `action.yml` to the App ID shown on the App's
settings page (this one is `4290632` — <https://github.com/apps/mnggal-mnggal>).

### The private key lives on the runner

Because the reviewer only ever runs on runner1, the App key lives on the guest,
not in GitHub — the action reads it and callers pass nothing. One place to
rotate, zero repo secrets.

Place the PEM at `/etc/mnggal-mnggal-reviewer/app.pem`, readable by the runner's
service user:

```sh
# as root on runner1:
install -d -m 0750 -o root -g ocr-runner /etc/mnggal-mnggal-reviewer
install -m 0640 -o root -g ocr-runner app.pem /etc/mnggal-mnggal-reviewer/app.pem
```

Override the location with the `key_path` input if needed. The trade-off: any job
on runner1 can read the file — acceptable under the same fork-PR guard the whole
runner relies on, and the minted installation token is scoped to the App's
permissions (PR write, contents read).

## Pinning

The action pins `alibaba/open-code-review@v1.7.8` and defaults `ocr_version` to
the matching npm release, so a review is reproducible. Bump both together.
