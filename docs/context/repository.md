# Repository

How this repository checks and releases itself, where it differs from its sibling repositories.

## The release runs the action from its own checkout and pushes only a tag

**Id:** 55c59756-d47f-4884-b012-6bfd54e719bb
**Type:** decision
**Status:** superseded
**Evidence:** confirmed
**Source:** conversation 2026-11-22
**Superseded by:** bb7bb9f2-691b-4039-9a07-3defe0ceb20c

The release workflow calls the action as `./`, and its `version` job checks out without a deploy key, has `contents: write` and pushes only the new tag with `GITHUB_TOKEN`.

**Reason:** The repository has no version file, so a release never commits to `main`, and its rulesets restrict only `main` and the deletion or moving of tags, so `GITHUB_TOKEN` may create a tag and no deploy key is needed.

**Consequence:** Adding a version file would need a deploy key that bypasses the `main` ruleset, as the sibling repositories have.

## The quality gate lints the action's scripts with shellcheck

**Id:** 615de468-d5b1-4746-80c3-52ba09431c57
**Type:** decision
**Status:** active
**Evidence:** confirmed
**Source:** conversation 2026-11-22

`mise run check` runs actionlint and then shellcheck on the `run:` blocks that `yq` extracts from `action.yml`; there are no behavior tests.

**Reason:** actionlint does not lint the scripts of a composite action, and static checks reach the Bash the action consists of without a test harness.

**Rejected alternative:** bats tests that run the `run:` blocks in throwaway Git repositories.

**Rejected alternative:** A CI-only self-test of the action.

**Consequence:** A change to the version logic is first exercised by a release, its own or a consuming repository's.

## The action moved to a new repository instead of renaming next-version

**Id:** f4d234a9-7293-4151-8b56-58f530a9073d
**Type:** decision
**Type:** constraint
**Status:** active
**Evidence:** confirmed
**Source:** conversation 2026-11-23
**Verification:** contradicted — the name Release from Commit, which b1b0ff0d-b377-4bb8-8405-1dc3348944b9 replaces with Release from Conventional Commits
**See:** repository.md#the-action-is-named-release-from-conventional-commits — b1b0ff0d-b377-4bb8-8405-1dc3348944b9 — as of 2026-11-24

The action lives in `lennartmoeller/release-from-commit`, named after its action name "Release from Commit", while `lennartmoeller/next-version` keeps serving the workflows still pinned to it until each switches, and is deleted then.

**Reason:** GitHub redirects git and web URLs of a renamed repository but not calls to the action it hosts, so every workflow still naming `next-version` would fail its release between a rename and its own switch. The name is "Release from Commit" because the action also commits, tags and creates the GitHub release, and that name is free on GitHub Marketplace, where names such as "Next Version", "Bump Version" and "Release Version" are taken.

**Rejected alternative:** Renaming `next-version` in place, which keeps its settings but fails consumer releases until each switches.

**Rejected alternative:** Keeping the name `next-version`, which no longer says that the action commits, pushes and releases.

**Consequence:** The repository is set up from scratch through the shared settings script, and the name `next-version` becomes free once that repository is deleted.

## The repository starts without next-version's history, at 0.1.0

**Id:** 6ac984e7-ad3b-4f46-b1ac-11664de85613
**Type:** decision
**Status:** active
**Evidence:** unknown
**Source:** conversation 2026-11-23
**See:** repository.md#the-action-moved-to-a-new-repository-instead-of-renaming-next-version — f4d234a9-7293-4151-8b56-58f530a9073d — as of 2026-11-23

The repository begins with one commit holding the whole action and no tags, so its first release is `v0.1.0`, instead of carrying over `next-version`'s commits and its tag `v1.0.0` and continuing at `v2.0.0`.

**Reason:** unknown; the maintainer chose to start fresh without stating why.

**Rejected alternative:** Carrying over `next-version`'s history and tags.

**Consequence:** `next-version`'s history survives only in local clones once it is deleted, and while the action is at `0.x`, the consuming repositories' Renovate does not automerge its minor updates, which it automerges only from `1.0`.

## The action is named Release from Conventional Commits

**Id:** b1b0ff0d-b377-4bb8-8405-1dc3348944b9
**Type:** decision
**Status:** needs-review
**Evidence:** confirmed
**Source:** conversation 2026-11-24
**Revisit when:** the GitHub repository is renamed to release-from-conventional-commits
**See:** repository.md#the-action-moved-to-a-new-repository-instead-of-renaming-next-version — f4d234a9-7293-4151-8b56-58f530a9073d — as of 2026-11-24
**See:** versioning.md#a-release-covers-every-commit-since-the-last-version-so-the-workflows-triggers-decide-when-to-release — 33407527-e9d2-4596-a6fc-6f3e3e727dd8 — as of 2026-11-24

The action's `name` and the README carry "Release from Conventional Commits", replacing "Release from Commit", while the GitHub repository is still `lennartmoeller/release-from-commit` until it is renamed to `release-from-conventional-commits`.

**Reason:** The maintainer finds the name truer, and the plural correct since the bump comes from every commit since the last version rather than from one commit. On 2026-11-24 the name was free on GitHub Marketplace, as an account and as a repository of the maintainer.

**Rejected alternative:** Keeping "Release from Commit", which reads as releasing one commit and does not say that the commits are read as Conventional Commits.

**Consequence:** Until the repository is renamed, its name differs from the action's and the README's `uses:` line names a repository that does not exist yet. The repository can be renamed without breaking a release only while no workflow calls the action, since calls to an action do not follow a repository rename.

## The release writes its tag into the README and pushes with a deploy key

**Id:** bb7bb9f2-691b-4039-9a07-3defe0ceb20c
**Type:** decision
**Status:** active
**Evidence:** confirmed
**Source:** conversation 2026-11-24
**See:** repository.md#the-action-is-named-release-from-conventional-commits — b1b0ff0d-b377-4bb8-8405-1dc3348944b9 — as of 2026-11-24

The release workflow calls the action as `./` with a `write_version` script that sets the version in the README's usage example to `v<version>` and fails when that line is missing, so every release commits to `main`, and its `version` job checks out with the `RELEASE_DEPLOY_KEY` deploy key.

**Reason:** The maintainer wants the usage example to name the latest release as a tag. A push to `main` has to bypass its ruleset, which `GITHUB_TOKEN` cannot, so the checkout uses a deploy key as the sibling repositories do.

**Rejected alternative:** Pinning the usage example to a commit SHA, since the maintainer wants the tag in the README.

**Consequence:** Until the repository has a deploy key that may bypass the `main` ruleset, stored as the `RELEASE_DEPLOY_KEY` secret, every release fails before it pushes anything.

## The repository releases only on a manual run

**Id:** 55e8f263-99b0-4c77-a46c-f7d5427e6ddf
**Type:** decision
**Status:** active
**Evidence:** confirmed
**Source:** conversation 2026-11-24
**See:** repository.md#the-release-writes-its-tag-into-the-readme-and-pushes-with-a-deploy-key — bb7bb9f2-691b-4039-9a07-3defe0ceb20c — as of 2026-11-24
**See:** versioning.md#a-release-covers-every-commit-since-the-last-version-so-the-workflows-triggers-decide-when-to-release — 33407527-e9d2-4596-a6fc-6f3e3e727dd8 — as of 2026-11-24

The release workflow has only a `workflow_dispatch` trigger, so a merge to `main` releases nothing until the maintainer runs it, and that run releases every commit since the last version.

**Reason:** The maintainer does not want to release every smaller change that goes into `main`, and prefers to collect several into one release.

**Rejected alternative:** Releasing on every push to `main`, the trigger the workflow had before.

**Consequence:** Merged changes reach consumers only after a manual run, which `CONTRIBUTING.md` describes as the release step.
