# Release

How the action writes, commits, tags and pushes a release, and what it leaves to the calling workflow.

## The action pushes the release commit and tag itself

**Id:** 2ed110d4-86bf-4d73-8a7b-4ebb73aad868
**Type:** decision
**Status:** active
**Evidence:** confirmed
**Source:** conversation 2026-11-23

After calculating the version, the action commits the changed files as `chore(release): <version> [skip ci]`, tags `v<version>` and pushes both; the checkout and the publish jobs stay in the calling workflow.

**Reason:** Every release workflow of the repositories that use the action ended in the same push script, byte for byte in six workflows, so it was copied release logic of the kind the action was made to end.

**Rejected alternative:** Also the checkout. It needs an `ssh_key` input, hides which credentials may push, and makes this repository's own release check out twice, since a local action needs a checkout before it runs.

**Rejected alternative:** Leaving the push script in each workflow, kept aligned through a shared template.

**Consequence:** The action always pushes, because a repository without version files releases with an empty script, so it has no compute-only use; that use ended with `next-version`, the repository this one replaced.

## A script input writes the version files

**Id:** 18c87eab-dd51-482c-a6b6-28b2f98ff6ee
**Type:** decision
**Status:** active
**Evidence:** confirmed
**Source:** conversation 2026-11-23
**See:** release.md#the-action-pushes-the-release-commit-and-tag-itself — 2ed110d4-86bf-4d73-8a7b-4ebb73aad868 — as of 2026-11-23

The caller passes `write_version`, a Bash script the action runs under `set -euo pipefail` with `$VERSION` between calculating and pushing; the action itself never writes a version file.

**Reason:** The repositories differ only in which files carry the version and which tool writes them, such as `yq` for JSON and YAML or `uv version` for `pyproject.toml` and `uv.lock`, and one script input keeps the release to a single step in each workflow that still names what it pushes.

**Rejected alternative:** A second action in this repository that pushes after the caller's own version-writing steps. It keeps those steps linted and the calculating action unchanged, but adds a second `uses:` to every workflow.

**Rejected alternative:** A JavaScript action whose post step pushes at the end of the job. It adds Node and a build to a Bash-only repository, and the push no longer shows in the calling workflow.

**Rejected alternative:** Writers built in per package manager, selected by inputs. They cannot cover a lock file that needs its own tool or a regenerated file without a version, and grow one branch per format.

**Consequence:** A tool the script needs is set up before the action, and a step that changes a file without the version runs before the action as an ordinary step, whose change `git commit --all` carries into the release commit.

## The version scripts go unlinted

**Id:** 9a4ab683-932f-460f-9ce3-90a6f9c04b16
**Type:** decision
**Status:** active
**Evidence:** confirmed
**Source:** conversation 2026-11-23

No quality gate shellchecks the `write_version` script a workflow passes, since actionlint shellchecks only `run:` scripts and a `with:` value is a plain string to it.

**Reason:** The scripts are one or two lines, and a mistake fails the release after the merge but before anything is committed or pushed, so it is fixed in the next pull request.

**Rejected alternative:** Extracting the input with `yq` and shellchecking it in every calling repository's gate, which adds a tool and a gate line to each.

**Rejected alternative:** A script file per calling repository, passed by path and shellchecked by its gate, which adds a file each and gives up the inline step.

**Consequence:** A quoting mistake that shellcheck would catch shows up only when the release runs.

## The action creates the GitHub release unless a later job attaches files

**Id:** 46669e3b-4e28-444e-a992-4a5a52811336
**Type:** decision
**Status:** active
**Evidence:** confirmed
**Source:** conversation 2026-11-23
**See:** release.md#the-action-pushes-the-release-commit-and-tag-itself — 2ed110d4-86bf-4d73-8a7b-4ebb73aad868 — as of 2026-11-23

After the push, the action creates the GitHub release for the new tag with generated notes, unless `create_github_release` is `false`, which a workflow sets when a later job creates the release with files built from the tag.

**Reason:** Three of the five releases that use the action attach no files, so their release job was one more copied command, while the files of the other two come from jobs that run after the action and cannot reach it.

**Rejected alternative:** Keeping the release job in every workflow.

**Rejected alternative:** Building the files in the job that runs the action and attaching them through an input. It runs third-party build code next to the deploy key that may push to `main`, builds before the tag exists, and makes an extension signed at addons.mozilla.org before its tag is pushed use up its version number if the push fails.

**Rejected alternative:** Creating every release in the action and uploading the files later. Immutable releases refuse files after publication, and a release published without them breaks a Firefox extension that reads `updates.json` from the latest release.

**Consequence:** GitHub releases are created two ways, by the action or by a release job, and a job that lets the action create the release needs `contents: write`.

## The action releases only from the default branch

**Id:** 2d869b39-e175-4f23-9898-4a3d86045818
**Type:** decision
**Status:** active
**Evidence:** confirmed
**Source:** conversation 2026-11-23
**See:** versioning.md#a-release-covers-every-commit-since-the-last-version-so-the-workflows-triggers-decide-when-to-release — 33407527-e9d2-4596-a6fc-6f3e3e727dd8 — as of 2026-11-23
**See:** release.md#the-action-pushes-the-release-commit-and-tag-itself — 2ed110d4-86bf-4d73-8a7b-4ebb73aad868 — as of 2026-11-23

The action fails unless the run's ref is the repository's default branch, read from the event as `github.event.repository.default_branch`, and pushes the release commit to that ref instead of a hard-coded `main`.

**Reason:** Once a manual run releases everything since the last version, the run can be started from any branch, while the release commit was always pushed to `main`, so a run from a feature branch would tag a commit on that branch and could move `main` to it. The maintainer chose the default branch over a fixed `main`.

**Rejected alternative:** Releasing on whichever branch the run is on, which tags commits outside the default branch and breaks the single line of versions the bump is measured along.

**Consequence:** A trigger whose event carries no repository fails the check, since the default branch then reads as empty.

## The release commit and tag are unsigned

**Id:** 4b0f8a86-96db-49ad-9681-ed3422afb21a
**Type:** constraint
**Status:** active
**Evidence:** confirmed
**Source:** conversation 2026-11-24
**Revisit when:** the action creates its release commit through the GitHub API, or a repository requires signed commits in a ruleset its release cannot bypass
**See:** release.md#the-action-pushes-the-release-commit-and-tag-itself — 2ed110d4-86bf-4d73-8a7b-4ebb73aad868 — as of 2026-11-24

The action commits and tags as `github-actions[bot]` with plain `git`, so its release commit and annotated tag carry no signature, and a repository that requires signed commits on its default branch accepts the release push only from a deploy key or other actor that bypasses that rule.

**Reason:** GitHub verifies a signature only against a key registered to the committer's account, which `github-actions[bot]` has none of, and only commits created through GitHub's API or interface are signed by GitHub itself. The maintainer's repositories therefore require signed commits in the ruleset their release deploy key bypasses.

**Rejected alternative:** Creating the release commit and tag through the GitHub API with a GitHub App that the rulesets let through, so that GitHub signs them, which changes the action and every release workflow and adds an App with its secret to each repository.

**Consequence:** Release commits stay unverified on the default branch, the only unsigned commits there.
