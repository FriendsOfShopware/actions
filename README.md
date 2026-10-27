# workflows
Collection of reusable workflows

## Publish to Store

Add `.github/workflows/publish-to-store.yml` to your extension repository:

```yaml
name: Publish to Store

on:
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: read

jobs:
  publish:
    uses: FriendsOfShopware/actions/.github/workflows/publish-to-store.yml@main
    secrets:
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
      SHOPWARE_CLI_ACCOUNT_CLIENT_ID: ${{ secrets.SHOPWARE_CLI_ACCOUNT_CLIENT_ID }}
      SHOPWARE_CLI_ACCOUNT_CLIENT_SECRET: ${{ secrets.SHOPWARE_CLI_ACCOUNT_CLIENT_SECRET }}
```

The workflow uses [communique](https://github.com/jdx/communique) with an
OpenAI-compatible API to generate a JSON draft from Git history and pull request
context. It defaults to DeepSeek (`https://api.deepseek.com`) and passes the
`OPENAI_API_KEY` secret to communique.
When `previousRef` is omitted, the workflow checks version tags in descending
version order. It selects the first tag whose commit or parent is reachable from
`HEAD`, including tags on separate build commits created by the Store release
workflow. If none match, it falls back to the initial commit. It validates the
reference and passes it explicitly to communique. The workflow prepends
an entry to `CHANGELOG.md` using shopware-cli's `# <version>` format and converts
category headings in the draft's `changelog` field to bold labels for compatibility.
The GitHub release uses `release_title` and `release_body` unchanged. The draft,
including its coverage review, is saved as the `release-draft` artifact. It builds and
validates the extension, uploads it to the Shopware Store, and creates a version
tag and GitHub release with the ZIP attached. The release tag contains the extracted
build, including compiled assets and the generated changelog, with `vendor` removed.
The selected branch is not updated. A commit message containing `[store update]`
also triggers a Store page update when the calling event supplies `head_commit`. Bump the extension version
before dispatching; an existing version tag stops the workflow before upload.

Configure the three repository secrets shown above. `GITHUB_TOKEN` is supplied
automatically. The extension name is detected with `shopware-cli extension get-name .`.
Optional `with` inputs are `cliVersion` (defaults to `latest`), `model` (defaults to `deepseek-flash`), `base-url`
(defaults to `https://api.deepseek.com`), and `previousRef` (auto-detected when
omitted). For a different OpenAI-compatible API, set `base-url` and `model` and
provide its API key through the `OPENAI_API_KEY` secret.
For example, set `previousRef: '3.6.0'` to generate changes since that
release for the version declared by the extension. Communique is pinned to 1.6.2.
The reusable workflow also exposes
`workflow_dispatch` for repositories containing it directly.

The publish job uses the `store` GitHub environment with the deployment URL
`https://store.shopware.com`. Configure protection rules on this environment in
the calling extension repository under **Settings → Environments → store**.
Any required approval applies before the publish job starts.

The build and release steps use shopware-cli directly because
[`store-release@main`](https://github.com/shopware/github-actions/blob/main/store-release/action.yml)
delegates to a build action whose ZIP renaming currently fails with `disableGit`.
Building from the working tree is needed to include the generated changelog.
