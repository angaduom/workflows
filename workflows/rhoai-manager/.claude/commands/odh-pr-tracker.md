# /odh-pr-tracker - Check if ODH PRs are in the RHOAI Build

Check whether one or more ODH (Open Data Hub) pull requests have been pulled into the latest RHOAI build.

## Purpose

When developers merge changes into an `opendatahub-io/<repo>` upstream, those changes don't automatically appear in RHOAI images. The RHOAI team periodically syncs upstream commits into their `red-hat-data-services/<repo>` fork and pins a specific commit in the build config. This command tells you whether a given ODH PR has made it through that pipeline.

Works for any component tracked in the RHOAI build config — odh-dashboard, eval-hub, or anything else.

## How It Works

ODH changes flow like this:
1. PR merged into `opendatahub-io/<repo>` (upstream)
2. RHOAI team syncs upstream into `red-hat-data-services/<repo>` (fork)
3. Build config (`red-hat-data-services/RHOAI-Build-Config`) is updated with the pinned commit
4. Konflux builds the image from that pinned commit

"Is my PR in RHOAI?" = is the PR's merge commit an ancestor of the commit currently pinned in the RHOAI build config?

## Prerequisites

- `gh` CLI authenticated with access to `red-hat-data-services` org

## Steps

For each PR URL provided by the user (e.g. `https://github.com/opendatahub-io/eval-hub/pull/123`):

### 1. Get the PR merge commit

Parse the PR URL to extract the upstream org/repo and PR number, then:

```bash
gh pr view <PR_NUMBER> --repo <upstream_org>/<repo> \
  --json mergeCommit,mergedAt,state,title
```

If `state` is not `"MERGED"`, report it as unmerged and skip further checks.

### 2. Find the RHOAI-pinned commit for this repo

Fetch the full build config map:

```bash
curl -sf https://raw.githubusercontent.com/red-hat-data-services/RHOAI-Build-Config/rhoai-3.4/catalog/catalog_build_args.map
```

The fork URL is almost always `red-hat-data-services/<repo>` (same repo name, different org). Find the line:

```
<SOMETHING>_GIT_URL=https://github.com/red-hat-data-services/<repo>
```

There may be multiple components pointing to the same repo (e.g. odh-dashboard has several modular-arch entries). Pick the one most relevant — for dashboard use `ODH_DASHBOARD_GIT_URL`, otherwise take the first match. Then swap `_GIT_URL` → `_GIT_COMMIT` to get the pinned SHA.

Example for eval-hub:
```
ODH_EVAL_HUB_GIT_URL=https://github.com/red-hat-data-services/eval-hub
ODH_EVAL_HUB_GIT_COMMIT=1aad0fe1...
```

### 3. Compare the two commits

```bash
gh api "repos/red-hat-data-services/<repo>/compare/<PR_MERGE_COMMIT>...<RHOAI_PINNED_COMMIT>" \
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
PR #<number>: <title>  [<upstream_org>/<repo>]
  Merged:           <mergedAt>
  RHOAI build at:   <rhoai_commit_short> (rhoai-3.4 branch)
  Status:           ✅ Included in latest RHOAI build
                    — or —
                    ❌ NOT yet in RHOAI build
```

If multiple PRs were provided, check all of them and summarize together.

## Notes

- The `rhoai-3.4` branch is the active release branch as of early 2026. If it no longer exists, check `https://github.com/red-hat-data-services/RHOAI-Build-Config` for the current branch and use that instead.
- If the repo name differs between upstream and the RH fork, the `_GIT_URL` lookup will still find it — just grep for the fork URL directly.
- This checks what's in the **build config**, not what's on a specific cluster. To check a deployed cluster, also compare the cluster's running image against the build config.

## Example Usage

**User**: `/odh-pr-tracker https://github.com/opendatahub-io/odh-dashboard/pull/6959`

**Claude**:
1. Gets merge commit `f754568f` for PR #6959 in `opendatahub-io/odh-dashboard`
2. Finds `ODH_DASHBOARD_GIT_URL=.../odh-dashboard` → grabs `ODH_DASHBOARD_GIT_COMMIT=297a39d8`
3. Compares: status `ahead`, `behind_by: 0` → included
4. Reports: ✅ PR #6959 is included in the latest RHOAI build

**User**: `/odh-pr-tracker https://github.com/opendatahub-io/eval-hub/pull/42`

**Claude**:
1. Gets merge commit for PR #42 in `opendatahub-io/eval-hub`
2. Finds `ODH_EVAL_HUB_GIT_URL=.../eval-hub` → grabs `ODH_EVAL_HUB_GIT_COMMIT=1aad0fe1`
3. Compares commits in `red-hat-data-services/eval-hub`
4. Reports result

**User**: `/odh-pr-tracker https://github.com/opendatahub-io/odh-dashboard/pull/6959 https://github.com/opendatahub-io/eval-hub/pull/42`

Claude checks both PRs and reports status for each.
