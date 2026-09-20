---
name: epic-software-release
description: Produces an ePIC software release. Use when the user wants to cut, tag, or publish a new release of ePIC software components.
---

# ePIC Software Release

Skill for producing ePIC software releases. Built collaboratively with the user, step by step.

## Interaction policy

At every user-visible artifact the agent produces, it MUST stop, present the URL, and wait for **explicit user confirmation** before proceeding. This applies to:

- Each published release (EICrecon, epic, and any subsequent components) — present `https://github.com/<org>/<repo>/releases/tag/<VERSION>`.
- Each pull request opened (eic/eic-spack, eic/containers, …) — present the PR URL.

Do not chain these actions without an intervening confirmation.

## Steps

### 1. Determine the next EICrecon version

List recent releases and identify the latest tag:

```bash
gh release list --repo eic/EICrecon --limit 10
```

The latest release is the one marked `Latest` in the second column. Compare its date to today (`date`):

- If the latest release is in the **current calendar month**, the next release is typically a **patch** bump (e.g. `v1.39.2` → `v1.39.3`).
- If the latest release is from a **previous calendar month**, the next release is typically a **minor** bump (e.g. `v1.39.2` → `v1.40.0`).

Example: on 2026-08-11, latest release `v1.39.2` (2026-07-17) is from the previous month → next version is **`v1.40.0`** (minor).

Record this as `EICRECON_VERSION` (with `v`) and `EICRECON_VERSION_NO_V` (without `v`, used by spack).

Also record the branch the release will be tagged from as `EICRECON_BRANCH`:
- **Minor** bump → the default branch `main`.
- **Patch** bump → the existing stable branch `vX.Y` (e.g. `v1.39`).

### 2. Determine the next epic (geometry) version

```bash
gh release list --repo eic/epic --limit 10
```

Note: `eic/epic` uses **CalVer** (`YY.MM.patch`), not SemVer. Versioning rule:

- If the latest release is in the **current** calendar month → next release is a patch bump (e.g. `26.08.0` → `26.08.1`).
- If the latest release is from a **previous** month → next release starts a new month at `.0` (e.g. `26.07.2` → `26.08.0`).

Example: on 2026-08-11, latest `26.07.2` → next version **`26.08.0`**.

Record this as `EPIC_VERSION`.

Also record the branch the release will be tagged from as `EPIC_BRANCH`:
- **New month** (`.0`) → the default branch `main`.
- **Patch** bump → the existing stable branch `YY.MM` (e.g. `26.07`).

### 3. Determine the software stack (containers) stable version

The containers stable release follows CalVer with a `v` prefix and `-stable` suffix (see step 18):

- Stable branch: `vYY.MM-stable` (e.g. `v26.08-stable`).
- Release tag: `vYY.MM.0-stable` (e.g. `v26.08.0-stable`).

The `YY.MM` normally matches the epic (geometry) release month. Record this as `STACK_VERSION` (e.g. `v26.08.0-stable`).

### 4. Confirm versions with the user

Present all three determined versions together and **stop for explicit user confirmation** before creating any release, branch, or PR:

```
Planned release versions (tagged from branch):
- epic software stack (containers): <STACK_VERSION>
- geometry (eic/epic):              <EPIC_VERSION>    from <EPIC_BRANCH>
- EICrecon (eic/EICrecon):          <EICRECON_VERSION> from <EICRECON_BRANCH>

Confirm you are okay releasing these versions from these branches before I proceed.
```

Do not create any release, branch, or PR until the user confirms. If the user wants different versions or branches, update the recorded values and re-confirm.

### 5. Create the EICrecon release with auto-generated notes

Use `gh` to tag and publish the release; let GitHub generate the release notes from merged PRs:

```bash
gh release create <EICRECON_VERSION> \
    --repo eic/EICrecon \
    --target <EICRECON_BRANCH> \
    --title <EICRECON_VERSION> \
    --generate-notes
```

Example (minor bump `v1.40.0`, so `EICRECON_BRANCH` is `main`; a patch would use the stable branch `vX.Y` instead):

```bash
gh release create v1.40.0 --repo eic/EICrecon --target main --title v1.40.0 --generate-notes
```

### 6. Verify the generated release notes

Inspect the published release notes:

```bash
gh release view <EICRECON_VERSION> --repo eic/EICrecon
```

Check that:
- Notes are categorized (Tracking, Calorimetry, Infrastructure, etc.) via `.github/release.yml`.
- The `Full Changelog` link compares against the correct previous tag (for a minor bump, this is the previous minor `X.Y.0`, not the last patch).
- No obviously missing PRs or malformed entries.

### 7. Prune bot noise from release notes

Remove auto-generated bot entries (pre-commit.ci autoupdates, dependabot bumps) which are not user-facing:

```bash
gh release view <EICRECON_VERSION> --repo eic/EICrecon --json body -q .body > /tmp/notes.md
grep -vE '^\* \[pre-commit\.ci\]|@dependabot' /tmp/notes.md > /tmp/notes.new.md
diff /tmp/notes.md /tmp/notes.new.md   # review what will be removed
gh release edit <EICRECON_VERSION> --repo eic/EICrecon --notes-file /tmp/notes.new.md
```

### 8. Create and clean the eic/epic release

Same procedure as EICrecon (steps 5–7), just with `--repo eic/epic` and `<EPIC_VERSION>`:

```bash
gh release create <EPIC_VERSION> --repo eic/epic --target <EPIC_BRANCH> --title <EPIC_VERSION> --generate-notes

gh release view <EPIC_VERSION> --repo eic/epic --json body -q .body > /tmp/epic-notes.md
grep -vE '^\* \[pre-commit\.ci\]|@dependabot' /tmp/epic-notes.md > /tmp/epic-notes.new.md
diff /tmp/epic-notes.md /tmp/epic-notes.new.md
gh release edit <EPIC_VERSION> --repo eic/epic --notes-file /tmp/epic-notes.new.md
```

### 9. Create stable branches and backport labels

Once both releases (EICrecon and epic) are published, create a stable branch pointing at each release tag, and a matching backport label in each repo.

Branch naming:
- `eic/EICrecon`: branch `vX.Y` (e.g. `v1.40`) from tag `vX.Y.0`
- `eic/epic`: branch `YY.MM` (e.g. `26.08`) from tag `YY.MM.0`

Label naming: `backport <BRANCH>` with description `Backport into <BRANCH>` and color `#0fafaa`.

```bash
# EICrecon
EICRECON_SHA=$(gh api repos/eic/EICrecon/commits/<EICRECON_VERSION> -q .sha)
gh api -X POST repos/eic/EICrecon/git/refs \
    -f ref=refs/heads/v<X.Y> -f sha=$EICRECON_SHA
gh label create "backport v<X.Y>" --repo eic/EICrecon \
    --description "Backport into v<X.Y>" --color "0fafaa"

# epic
EPIC_SHA=$(gh api repos/eic/epic/commits/<EPIC_VERSION> -q .sha)
gh api -X POST repos/eic/epic/git/refs \
    -f ref=refs/heads/<YY.MM> -f sha=$EPIC_SHA
gh label create "backport <YY.MM>" --repo eic/epic \
    --description "Backport into <YY.MM>" --color "0fafaa"
```

Note: `gh api repos/.../commits/<tag>` dereferences annotated tags to their target commit SHA, which is what a branch ref needs.

### 10. Clone eic-spack to a temporary location

```bash
TMPDIR=$(mktemp -d)
git clone git@github.com:eic/eic-spack.git "$TMPDIR/eic-spack"
cd "$TMPDIR/eic-spack"
```

Remember the path — subsequent steps will update package recipes in this checkout.

### 11. Compute spack checksums for the new tarballs

Run `spack checksum` inside the `eicweb/eic_ci:nightly` container (Docker or Singularity/Apptainer). Note that **spack versions drop the leading `v`** from git tags: git tag `v1.40.0` → spack version `1.40.0`. The `epic` package already uses the plain CalVer string (`26.08.0`).

```bash
docker run --rm eicweb/eic_ci:nightly bash -lc '
    spack checksum epic <EPIC_VERSION> &&
    echo === &&
    spack checksum eicrecon <EICRECON_VERSION_NO_V>
'
```

Example:

```bash
docker run --rm eicweb/eic_ci:nightly bash -lc '
    spack checksum epic 26.08.0 && echo === && spack checksum eicrecon 1.40.0
'
```

Each invocation prints a line like:

```
version("1.40.0", sha256="d5ac2bbe17093941f69e819aaa70987166a3ad838561319366da05a33cccee78")
```

Record both sha256 values — they will be added to the spack recipes next.

### 12. Add version lines to spack recipes

Edit the two package recipes in the eic-spack checkout:

- `spack_repo/eic/packages/epic/package.py`
- `spack_repo/eic/packages/eicrecon/package.py`

Insert the new `version(...)` line immediately **after** the `version("main", branch="main")` line and **above** the previous latest release, so versions stay in descending order.

Example resulting block (`epic`):

```python
    version("main", branch="main")
    version("26.08.0", sha256="2fc6d39db5a84be0ddbc6ad7fdeaac603d080924e4a3035a0500251fc93b6f9c")
    version("26.07.2", sha256="42876a5330754d0aedcd4f511be43cf454f9f73304d0c0a3f404080e91c30feb")
    ...
```

And (`eicrecon`, note no `v` prefix):

```python
    version("main", branch="main")
    version("1.40.0", sha256="d5ac2bbe17093941f69e819aaa70987166a3ad838561319366da05a33cccee78")
    version("1.39.2", sha256="9068425ba5f1a778efab2930fad8b37c81171e34d764b7f93e70fe9bd56cb4b6")
    ...
```

### 13. Push a feature branch to eic/eic-spack and open a PR

Push **directly** to a feature branch on `eic/eic-spack` (do NOT use a personal fork). Do not run `gh repo fork` — it will silently create a fork you may not want.

```bash
git checkout -b release-YYYY-MM
git -c user.name=<you> -c user.email=<you>@users.noreply.github.com \
    commit -am "feat: add epic <EPIC_VERSION> and eicrecon <EICRECON_VERSION_NO_V>"
git push origin release-YYYY-MM

gh pr create --repo eic/eic-spack \
    --title "feat: add epic <EPIC_VERSION> and eicrecon <EICRECON_VERSION_NO_V>" \
    --body "Adds new versions:
- epic <EPIC_VERSION> (https://github.com/eic/epic/releases/tag/<EPIC_VERSION>)
- eicrecon <EICRECON_VERSION_NO_V> (https://github.com/eic/EICrecon/releases/tag/v<EICRECON_VERSION_NO_V>)" \
    --head release-YYYY-MM
```

After the PR URL is printed, **stop and present it to the user for confirmation** (per Interaction policy) before proceeding.

### 14. Clone eic/containers to a temporary location

```bash
git clone git@github.com:eic/containers.git "$TMPDIR/containers"
cd "$TMPDIR/containers"
```

### 15. Bump EICSPACK_VERSION in eic-spack.sh

`eic-spack.sh` pins the `eic/eic-spack` commit consumed by the container build:

```bash
EICSPACK_ORGREPO="eic/eic-spack"
EICSPACK_VERSION="<sha>"
```

Decide which SHA to write:

- If the eic-spack PR from step 13 **is merged** → use its **merge commit** on `eic/eic-spack@main`.
- If the PR is **not yet merged** → temporarily point at the **PR head commit** (the tip of the feature branch on `eic/eic-spack`). This must be replaced with the merge commit before the containers PR is merged.

Only bump if the new SHA is strictly newer than the currently pinned one; verify with:

```bash
OLD=$(grep -oP 'EICSPACK_VERSION="\K[^"]+' eic-spack.sh)
NEW=<merge-or-head-sha>
gh api repos/eic/eic-spack/compare/$OLD...$NEW -q '{status,ahead_by,behind_by}'
# expect: status=ahead, behind_by=0
```

Then update the file:

```bash
sed -i "s|EICSPACK_VERSION=\"$OLD\"|EICSPACK_VERSION=\"$NEW\"|" eic-spack.sh
```

Get the merge commit SHA for a merged PR with:

```bash
gh pr view <PR> --repo eic/eic-spack --json mergeCommit -q .mergeCommit.oid
```

### 16. Update EICrecon version in spack-environment/packages.yaml

In `spack-environment/packages.yaml`, locate the `eicrecon:` block and bump the pinned version (no `v` prefix, matching spack). The line is tagged with `# EICRECON_VERSION`:

```yaml
  eicrecon:
    require:
    - '@<EICRECON_VERSION_NO_V>' # EICRECON_VERSION
```

```bash
sed -i "s|- '@<OLD>' # EICRECON_VERSION|- '@<NEW>' # EICRECON_VERSION|" \
    spack-environment/packages.yaml
```

### 17. Update pinned epic versions in per-flavor spack.yaml files

For each `spack-environment/*/epic/spack.yaml`:

- If the `specs:` list contains **only** `epic@main # EPIC_VERSION` (or an unpinned `epic ...`), leave it alone.
- If the list enumerates concrete `epic@X.Y.Z` versions (currently `cuda/` and `xl/`), then:
  - **Add** the new release: `- epic@<EPIC_VERSION>`
  - **Remove** all entries for the oldest month still present, **including patch releases** (e.g. drop `epic@26.04.0` and `epic@26.04.1`).

Keep the list in ascending order after `epic@main`.

Example before/after (adding `26.08.0`, dropping `26.04.*`):

```yaml
  - epic@main # EPIC_VERSION
- - epic@26.04.0
- - epic@26.04.1
  - epic@26.05.0
  ...
  - epic@26.07.2
+ - epic@26.08.0
```

### 18. Push a feature branch to eic/containers and open a PR

Same direct-branch pattern as the eic-spack PR (no personal fork):

```bash
cd "$TMPDIR/containers"
git checkout -b release-YYYY-MM
git -c user.name=<you> -c user.email=<you>@users.noreply.github.com \
    commit -am "feat: bump to epic <EPIC_VERSION> and eicrecon <EICRECON_VERSION_NO_V>"
git push origin release-YYYY-MM

gh pr create --repo eic/containers \
    --title "feat: bump to epic <EPIC_VERSION> and eicrecon <EICRECON_VERSION_NO_V>" \
    --body "- Bump EICSPACK_VERSION to eic/eic-spack@<sha>
- eicrecon <OLD> -> <EICRECON_VERSION_NO_V> in spack-environment/packages.yaml
- Add epic@<EPIC_VERSION> and drop the oldest epic@YY.MM.* in cuda/ and xl/ flavors

Releases:
- https://github.com/eic/epic/releases/tag/<EPIC_VERSION>
- https://github.com/eic/EICrecon/releases/tag/v<EICRECON_VERSION_NO_V>" \
    --head release-YYYY-MM
```

After the PR URL is printed, **stop and present it to the user for confirmation** before proceeding.

#### Note the picked-up eic-spack commits

When bumping `EICSPACK_VERSION`, list any **notable** commits picked up along the way in both the commit message and the PR body. Get the range with:

```bash
gh api repos/eic/eic-spack/compare/<OLD>...<NEW> \
    -q '.commits[] | "\(.sha[0:7]) \(.commit.message | split("\n")[0])"'
```

Include commits that could affect the produced container (new package versions, runtime/plugin discovery changes, etc.). Skip CI-only / repo-hygiene commits. Reference them as `eic/eic-spack#<PR>` so GitHub cross-links.

Example body section:

```
Notable eic-spack commits picked up:
- eic/eic-spack#1000 g4occt: add versions
- eic/eic-spack#1006 simphony: make the DD4hep plugins discoverable at runtime
```

### 19. Cut the containers stable branch and release

Once the containers PR is merged, cut the stable branch and tag+release from the **merge commit** (default branch on `eic/containers` is `master`). Use the `STACK_VERSION` confirmed in step 4.

Naming (containers convention, note the leading `v` in both, unlike the `epic` repo):
- Stable branch: `vYY.MM-stable` (e.g. `v26.08-stable`) — note the `v` prefix and the two-component `YY.MM`.
- Release tag: `vYY.MM.0-stable` (e.g. `v26.08.0-stable`) — this is `STACK_VERSION`.

```bash
SHA=$(gh pr view <PR> --repo eic/containers --json mergeCommit -q .mergeCommit.oid)

# Stable branch
gh api -X POST repos/eic/containers/git/refs \
    -f ref=refs/heads/v<YY.MM>-stable -f sha=$SHA

# Release
gh release create <STACK_VERSION> --repo eic/containers \
    --target $SHA --title <STACK_VERSION> --generate-notes
```

Apply the same bot-noise pruning (step 7) to the release notes if needed.

Stop and present the release URL for user confirmation.

### 20. Point the user at the container pipelines

After the stable release is published, the container image build runs on the internal GitLab mirror. Direct the user to monitor:

https://eicweb.phy.anl.gov/containers/eic_container/-/pipelines

This is the end of the automated portion of the release procedure.
