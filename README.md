# Release from Conventional Commits

A GitHub Action that calculates the next semantic version from Git tags and Conventional Commits, writes it into your version files with a commit and push, and creates a GitHub release.

## Usage

```yaml
on:
  # manually run to release
  workflow_dispatch:
  # or release on every commit to main
  push:
    branches: [main]

# only one release at a time
concurrency:
  group: release
  cancel-in-progress: false

jobs:
  version:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - name: Check out repository
        uses: actions/checkout@v7
        # read the tags and the commits since the last version
        with:
          fetch-depth: 0
      - name: Release next version
        uses: lennartmoeller/release-from-conventional-commits@v0.1.1
        with:
          write_version: yq -i '.version = strenv(VERSION)' package.json
```

> [!IMPORTANT]
> The action releases only from the repository's default branch and fails on a run from any other branch.

### Inputs

| Input | Requirement | Description |
|---|---|---|
| `write_version` | Optional | Bash script that writes `$VERSION`, the next version without a leading `v`, into the repository's files for the release commit. When empty, the release commits nothing and pushes only the tag. |
| `bump_type` | Optional | Bump to apply, one of `major`, `minor` or `patch`. When empty, [derived](#how-the-version-is-calculated) from the subjects of the commits since the last version. |
| `create_github_release` | Optional, default `true` | Whether to create the GitHub release for the pushed tag, with generated notes. Set it to `false` when a later job creates the release with files. |
| `no_release_types` | Optional, default `build ci docs refactor style test` | Space-separated Conventional Commit types whose commits [add no bump](#how-the-version-is-calculated) when `bump_type` is empty, unless marked breaking with `!`. Set it empty so that every commit bumps at least the patch version. |

### Outputs

| Output | Description |
|---|---|
| `last_version` | Version to bump, [from the highest `vX.Y.Z` tag](#how-the-version-is-calculated) in the history or `0.0.0`, without a leading `v`, such as `1.2.3`. |
| `next_version` | Last version with the bump applied, without a leading `v`, such as `1.3.0`. Empty when nothing is released. |
| `next_tag` | Next version with a leading `v`, such as `v1.3.0`. Empty when nothing is released. |
| `bump_type` | Bump that was applied, one of `major`, `minor` or `patch`. Empty when nothing is released. |

## How the version is calculated

The last version is the highest `vX.Y.Z` tag in the history of the checked-out commit, or `0.0.0` when there is none.
The bump is the highest bump among the subjects of the commits since that tag:

| Commit subject | Bump |
|---|---|
| `!` before the colon | major |
| `feat:` | minor |
| A type in `no_release_types` | none |
| Anything else | patch |

The next version is the last version with that bump applied, so `1.2.3` becomes `2.0.0`, `1.3.0` or `1.2.4`.
With squash merges, a pull request's title sets the bump for the whole pull request.
Without a bump, the action releases nothing: it skips `write_version`, pushes nothing and creates no GitHub release, and the commits ship with the next release.

## Contributing

See [`CONTRIBUTING.md`](./CONTRIBUTING.md) to set up, check and release a change.

## License

The action is licensed under the [MIT License](./LICENSE).
