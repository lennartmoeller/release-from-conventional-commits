# Versioning

How the action derives the last version and the bump type from Git tags and commits, and the repository workflow it assumes.

## The action assumes a squash-merge trunk where every push to main is released

**Id:** c70dedce-8f2b-47a7-bda0-07740506af1c
**Type:** constraint
**Status:** superseded
**Evidence:** confirmed
**Source:** conversation 2026-11-22
**Revisit when:** a consumer merges with merge commits, pushes several commits at once, or releases from maintenance branches
**Superseded by:** 429d49a6-0511-4ed5-9174-0e3f3f697c1b

The action is meant only for repositories that squash-merge every pull request into main and release every push.

**Reason:** Under that workflow each release corresponds to exactly one commit on main, and main always carries the highest tag, so the HEAD commit's subject and the highest `vX.Y.Z` tag in the repository are enough.

**Rejected alternative:** Deriving the last version from `git tag --merged HEAD` only. It lost because no release branch diverges from main under a squash-only trunk, so the highest tag overall is already the one HEAD builds on.

**Rejected alternative:** Deriving the bump type from every commit since the last tag. It lost because each push to main is a single squash commit, so the HEAD commit is the only one since the last release.

**Consequence:** Merge commits always yield a patch bump, a `feat` behind a later commit is missed, and a tag on an unmerged branch is picked up.

## Only a ! in the subject marks a breaking change

**Id:** 2502007e-c68d-4435-82a6-e3607c435d05
**Type:** decision
**Status:** active
**Evidence:** confirmed
**Source:** conversation 2026-11-22
**See:** versioning.md#a-release-covers-every-commit-since-the-last-version-so-the-workflows-triggers-decide-when-to-release — 33407527-e9d2-4596-a6fc-6f3e3e727dd8 — as of 2026-11-23

The action bumps the major version only when the latest commit's subject carries `!` before the colon, and no longer searches the commit body for a `BREAKING CHANGE:` footer.

**Reason:** A squash commit's body is the pull request description, and Renovate's descriptions quote upstream release notes verbatim, so a `BREAKING CHANGE:` line from another project's changelog would release a new major version of a repository that merely updated that dependency.

**Rejected alternative:** Renovate leaving release notes out of its pull request bodies.

**Rejected alternative:** The pull request title alone as squash commit message, which drops the description from the history.

**Rejected alternative:** Accepting the risk of a stray major release.

**Consequence:** A breaking change written only as a footer releases as a minor or patch version, so the title must carry the `!`.

## The action assumes a squash-merge trunk where each push to main may release

**Id:** 429d49a6-0511-4ed5-9174-0e3f3f697c1b
**Type:** constraint
**Status:** superseded
**Evidence:** confirmed
**Source:** conversation 2026-11-22 and 2026-11-23
**Revisit when:** a consumer merges with merge commits, pushes several commits at once, or releases from maintenance branches
**See:** versioning.md#commits-whose-type-ships-nothing-release-nothing — 804eb5fc-1675-4b70-abf8-ce772e92f06c — as of 2026-11-23
**Superseded by:** 33407527-e9d2-4596-a6fc-6f3e3e727dd8

The action is meant only for repositories that squash-merge every pull request into main and run the action on every push to main.

**Reason:** Each push to main is one squash commit, and main always carries the highest tag, so the HEAD commit's subject and the highest `vX.Y.Z` tag in the repository are enough. Commits since the last tag that released nothing could only have bumped the patch version, which the next release's bump always covers. The first part was stated on 2026-11-22; the second was agreed on 2026-11-23 together with `no_release_types`.

**Rejected alternative:** Deriving the last version from `git tag --merged HEAD` only. It lost because no release branch diverges from main under a squash-only trunk, so the highest tag overall is already the one HEAD builds on.

**Rejected alternative:** Deriving the bump type from every commit since the last tag. It lost because each push to main is a single squash commit, and the commits since the last release that released nothing could only have bumped the patch version.

**Consequence:** Merge commits always yield a patch bump, a `feat` behind a later commit is missed, and a tag on an unmerged branch is picked up.

## Commits whose type ships nothing release nothing

**Id:** 804eb5fc-1675-4b70-abf8-ce772e92f06c
**Type:** decision
**Status:** active
**Evidence:** confirmed
**Source:** conversation 2026-11-23
**See:** versioning.md#a-release-covers-every-commit-since-the-last-version-so-the-workflows-triggers-decide-when-to-release — 33407527-e9d2-4596-a6fc-6f3e3e727dd8 — as of 2026-11-23

A commit whose type is in the `no_release_types` input, by default `build ci docs refactor style test`, adds no bump unless its subject carries `!`, so when no commit since the last version bumps it, the action releases nothing: it skips writing, pushing and the GitHub release, and leaves the version outputs empty.

**Reason:** Every push to main released at least a patch version, so pull requests that changed only documentation, CI or tests cut versions that shipped nothing. Listing the types that release nothing, rather than the ones that release, keeps every other subject releasing, including one that is not a Conventional Commit, so a fix with an unusual title still ships. The list is an input so that each calling repository can set its own.

**Rejected alternative:** Also skipping `chore`. Renovate labels updates `chore(deps)` that change what ships, such as a Docker base image or the bundler, and finance-tracker's nginx base image reaches its deployment only through a release, so a skipped `chore` would hold a shipped update on main without notice. It can join the list once each repository's Renovate labels shipped updates `fix`.

**Rejected alternative:** A fixed list without an input, since every calling repository would use the same one. It lost because the maintainer wants callers to configure it themselves.

**Consequence:** A calling workflow's later jobs that use the version outputs need an `if:` on `next_version`, and a commit that released nothing ships with the next release, whose generated notes list its pull request.

## A release covers every commit since the last version, so the workflow's triggers decide when to release

**Id:** 33407527-e9d2-4596-a6fc-6f3e3e727dd8
**Type:** decision
**Status:** active
**Evidence:** inferred
**Source:** conversation 2026-11-23
**See:** versioning.md#commits-whose-type-ships-nothing-release-nothing — 804eb5fc-1675-4b70-abf8-ce772e92f06c — as of 2026-11-23
**See:** versioning.md#only-a--in-the-subject-marks-a-breaking-change — 2502007e-c68d-4435-82a6-e3607c435d05 — as of 2026-11-23
**See:** versioning.md#the-action-keeps-inputs-and-outputs-that-only-outside-callers-may-need — 74fe3fd2-b11d-4d43-9794-34bf0d2578ce — as of 2026-11-24

The action takes the last version from the highest `vX.Y.Z` tag in the history of HEAD and the bump from the highest bump among the subjects of all commits after that tag, so a run releases everything since the previous release, whether its workflow runs on every push to main or only on a manual run. The `last_version` input is gone.

**Reason:** The maintainer wants a release when the release workflow runs, not on every commit to main, while a `push` trigger on main still releases every push. A run that does not follow every push has to see every commit since the last release, and that range starts at a version tag, so a version passed by hand as `last_version` left no commits to measure the bump from. Taking the tag from the history of HEAD rather than from the whole repository follows from reading the commits after it as a range, which needs the tag to be an ancestor of HEAD. That last part is inferred from the range; the rest was stated by the maintainer.

**Rejected alternative:** The latest commit's subject alone, which a manual run after several merges would release with only the last one's bump.

**Consequence:** A run without a new commit since the last version releases nothing unless `bump_type` is given, a merge commit's own subject counts as a patch, and a version tag created by hand counts as the last release.

## The action has no bump override and outputs only the next version and tag

**Id:** b3a227dd-b2d6-4e9a-84c6-f6d3ede7baed
**Type:** decision
**Status:** superseded
**Evidence:** inferred
**Source:** conversation 2026-11-24
**Revisit when:** a workflow needs to force a bump, or to read the last version or the bump type
**See:** versioning.md#a-release-covers-every-commit-since-the-last-version-so-the-workflows-triggers-decide-when-to-release — 33407527-e9d2-4596-a6fc-6f3e3e727dd8 — as of 2026-11-24
**Superseded by:** 74fe3fd2-b11d-4d43-9794-34bf0d2578ce

The `bump_type` input and the `last_version` and `bump_type` outputs are gone, so the bump always comes from the commits since the last version, and the action outputs only `next_version` and `next_tag`.

**Reason:** No calling workflow set `bump_type` or read either output, and the maintainer holds the action strictly to the principle that every exposed input and output needs a consumer or a decision behind it. The absence of callers is read from the release workflows of agentic, finance-tracker, github-sso-keeper and pr-policy-gate on 2026-11-24, so that part is inferred.

**Rejected alternative:** Keeping `bump_type` to force a bump on a manual run, which no workflow did.

**Consequence:** A release can no longer force a bump or release without a new commit.

## The action keeps inputs and outputs that only outside callers may need

**Id:** 74fe3fd2-b11d-4d43-9794-34bf0d2578ce
**Type:** decision
**Status:** active
**Evidence:** confirmed
**Source:** conversation 2026-11-24
**See:** versioning.md#a-release-covers-every-commit-since-the-last-version-so-the-workflows-triggers-decide-when-to-release — 33407527-e9d2-4596-a6fc-6f3e3e727dd8 — as of 2026-11-24

The `bump_type` input and the `last_version` and `bump_type` outputs stay, although none of the maintainer's repositories sets or reads them.

**Reason:** The action is public, so an input or output earns its place by what any caller may need, not only by the maintainer's own workflows; that none of them uses it does not mean nobody else does.

**Rejected alternative:** Removing them because no workflow of the maintainer's repositories used them, as a strict reading of the principle that every exposed input and output needs a consumer asked for.

**Consequence:** A manual run can force a bump with `bump_type`, also without a new commit since the last version.
