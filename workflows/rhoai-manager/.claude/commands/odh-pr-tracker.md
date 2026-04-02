# /odh-pr-tracker - Check if ODH PRs are in the RHOAI Build

Check whether one or more ODH (Open Data Hub) pull requests have been pulled into the latest RHOAI build.

## Purpose

When developers merge changes into `opendatahub-io/odh-dashboard` (upstream), those changes don't automatically appear in RHOAI images. The RHOAI team periodically syncs upstream commits into their fork and pins a specific commit in the build config. This command tells you whether a given ODH PR has made it through that pipeline.

## How It Works

ODH changes flow like this:
1. PR merged into `opendatahub-io/odh-dashboard` (upstream)
2. RHOAI team syncs upstream into `red-hat-data-services/odh-dashboard` (fork)
3. Build config (`red-hat-data-services/RHOAI-Build-Config`) is updated with the pinned commit
4. Konflux builds the image from that pinned commit

"Is my PR in RHOAI?" = is the PR's merge commit an ancestor of the commit currently pinned in the RHOAI build config?

## Prerequisites

- `gh` CLI authenticated with access to `red-hat-data-services` org

## Steps

For each PR URL provided by the user (e.g. `https://github.com/opendatahub-io/odh-dashboard/pull/6959`):

### 1. Get the PR merge commit

```bash
gh pr view <PR_NUMBER> --repo opendatahub-io/odh-dashboard \
  --json mergeCommit,mergedAt,state,title
```

If `state` is not `"MERGED"`, report it as unmerged and skip further checks.

### 2. Get the RHOAI-pinned ODH commit

```bash
curl -sf https://raw.githubusercontent.com/red-hat-data-services/RHOAI-Build-Config/rhoai-3.4/catalog/catalog_build_args.map \
  | grep ODH_DASHBOARD_GIT_COMMIT
```

Extract the SHA from `ODH_DASHBOARD_GIT_COMMIT=<sha>`.

### 3. Compare the two commits

```bash
gh api "repos/red-hat-data-services/odh-dashboard/compare/<PR_MERGE_COMMIT>...<RHOAI_PINNED_COMMIT>" \
  --jq '{status: .status, behind_by: .behind_by}'
```

Interpret the result:
- `status: "ahead"` and `behind_by: 0` → PR commit IS an ancestor of the RHOAI commit → **included** ✅
- `status: "diverged"` or `behind_by > 0` → PR is NOT yet in the RHOAI build → **not included** ❌
- `status: "behind"` → RHOAI is behind the PR commit → **not included** ❌
- `status: "identical"` → same commit → **included** ✅

The merge commit SHA is the same in both repos because the fork mirrors upstream commits directly (not rebased).

### 4. Output a clear summary

For each PR:

```
PR #<number>: <title>
  Merged:           <mergedAt>
  RHOAI build at:   <rhoai_commit_short> (rhoai-3.4 branch)
  Status:           ✅ Included in latest RHOAI build
                    — or —
                    ❌ NOT yet in RHOAI build
```

If multiple PRs were provided, check all of them and summarize together.

## Notes

- The `rhoai-3.4` branch is the active release branch as of early 2026. If it no longer exists, check `https://github.com/red-hat-data-services/RHOAI-Build-Config` for the current branch and use that instead.
- This checks what's in the **build config**, not what's on a specific cluster. To check a deployed cluster, also compare the cluster's running image against the build config.

## Example Usage

**User**: `/odh-pr-tracker https://github.com/opendatahub-io/odh-dashboard/pull/6959`

**Claude**:
1. Gets merge commit `f754568f` for PR #6959
2. Fetches `ODH_DASHBOARD_GIT_COMMIT=297a39d8` from rhoai-3.4 build config
3. Compares: status `ahead`, `behind_by: 0` → included
4. Reports: ✅ PR #6959 is included in the latest RHOAI build

**User**: `/odh-pr-tracker https://github.com/opendatahub-io/odh-dashboard/pull/6959 https://github.com/opendatahub-io/odh-dashboard/pull/6800`

Claude checks both PRs in sequence and reports status for each.
