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
      DEEPSEEK_API_KEY: ${{ secrets.DEEPSEEK_API_KEY }}
      SHOPWARE_CLI_ACCOUNT_CLIENT_ID: ${{ secrets.SHOPWARE_CLI_ACCOUNT_CLIENT_ID }}
      SHOPWARE_CLI_ACCOUNT_CLIENT_SECRET: ${{ secrets.SHOPWARE_CLI_ACCOUNT_CLIENT_SECRET }}
```

The workflow uses [communique](https://github.com/jdx/communique) with DeepSeek's
OpenAI-compatible API (`https://api.deepseek.com`) to generate a JSON draft from
Git history and pull request context. `DEEPSEEK_API_KEY` is passed to communique
as `OPENAI_API_KEY`.
Communique selects the previous reachable version tag as its baseline, falling
back to the initial commit when there are no tags. The workflow prepends
an entry to `CHANGELOG.md` using shopware-cli's `# <version>` format and converts
category headings in the draft's `changelog` field to bold labels for compatibility.
The GitHub release uses `release_title` and `release_body` unchanged. The draft,
including its coverage review, is saved as the `release-draft` artifact. It builds and
validates the extension, uploads it to the Shopware Store, and creates a version
tag and GitHub release with the ZIP attached. The changelog commit is included in
the release tag; the selected branch is not updated. Bump the extension version
before dispatching; an existing version tag stops the workflow before upload.

Configure the three repository secrets shown above. `GITHUB_TOKEN` is supplied
automatically. The extension name is detected with `shopware-cli extension get-name .`.
Optional `with` inputs are `model` (defaults to `deepseek-flash`) and `previousRef` (auto-detected when
omitted). For example, set `previousRef: '3.6.0'` to generate changes since that
release for the version declared by the extension. Communique is pinned to 1.6.2.
The reusable workflow also exposes
`workflow_dispatch` for repositories containing it directly.

The build and release steps use shopware-cli directly because
[`store-release@main`](https://github.com/shopware/github-actions/blob/main/store-release/action.yml)
delegates to a build action whose ZIP renaming currently fails with `disableGit`.
Building from the working tree is needed to include the generated changelog.
