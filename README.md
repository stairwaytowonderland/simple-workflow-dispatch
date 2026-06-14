# simple-workflow-dispatch

A composite GitHub Action that triggers a [`workflow_dispatch`](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#workflow_dispatch)
event on a target workflow. It wraps `gh workflow run` so callers only need to supply the workflow file, the target ref,
and any optional input fields.

> [!NOTE]
> `GITHUB_TOKEN` **can** trigger `workflow_dispatch` events. This is an [explicit exception](https://docs.github.com/en/actions/security-for-github-actions/security-guides/automatic-token-authentication#using-the-github_token-in-a-workflow)
> to the general rule that `GITHUB_TOKEN` cannot trigger new workflow runs — no PAT required for this event type.

## Inputs

| Name           | Description                                                          | Required | Default               |
| -------------- | -------------------------------------------------------------------- | -------- | --------------------- |
| `github-token` | Token with `repo` + `admin:org` scopes                               | No       | `${{ github.token }}` |
| `workflow`     | Workflow file to dispatch (e.g. `publish.yaml`)                      | Yes      | `''`                  |
| `tag`          | Tag passed as the `tag` field to the dispatched workflow             | No       | `${{ github.ref }}`   |
| `ref`          | Git ref to run the dispatched workflow on (SHA, branch, or tag name) | No       | `''`                  |
| `fields`       | Newline-separated `key=value` pairs forwarded as `--field` arguments | No       | `''`                  |

## Usage

### Basic — dispatch a workflow on the current ref

```yaml
- uses: stairwaytowonderland/simple-workflow-dispatch@main
  with:
    workflow: publish.yaml
```

### Dispatch on a specific ref

```yaml
- uses: stairwaytowonderland/simple-workflow-dispatch@main
  with:
    workflow: deploy.yaml
    ref: main
```

### Dispatch a tag workflow

```yaml
steps:

  - id: release
    uses: stairwaytowonderland/node-semantic-release@main
    with:
      github-token: ${{ secrets.GH_PAT }}

  - uses: stairwaytowonderland/simple-workflow-dispatch@main
    with:
      workflow: publish.yaml
      tag: ${{ github.ref_name }}
      fields: |
        notes-b64=${{ steps.release.outputs.new-release-notes-base64 }}
```

> [!NOTE]
> In the above example, the target workflow needs to have a `tag` input,
> as well as a `notes-b64` input.
>
> The target workflow
> would also need a step to decode any base64-encoded data:
>
> ```yaml
> - name: Decode base64 notes
>   if: inputs.notes-b64 != ''
>   run: printf '%s' "$NOTES_B64" | base64 -d > release-notes.md
>   env:
>     NOTES_B64: ${{ inputs.notes-b64 }}
> ```

## Permissions

The calling job must grant `actions: write` so that `GITHUB_TOKEN` can trigger the dispatch event:

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      actions: write   # required to trigger workflow_dispatch
      contents: write  # include if the job also creates commits or tags
    steps:
      - uses: stairwaytowonderland/simple-workflow-dispatch@main
        with:
          workflow: publish.yaml
```

> [!TIP]
> If your repository's default token permissions are set to **"Read repository contents and packages permissions"**
> (the restrictive default), you must add `permissions: actions: write` explicitly — it will not be inherited.

## How it works

1. A `tag` field is always forwarded to the target workflow (defaults to the calling workflow's `github.ref`).
2. Any additional `fields` are parsed line-by-line and appended as `--field` arguments to `gh workflow run`.
3. The workflow runs on `ref` when provided; otherwise it falls back to the `tag` value.
4. Authentication is handled automatically via `github.token` — no additional secrets are needed.
