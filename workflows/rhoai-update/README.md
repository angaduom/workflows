# RHOAI Nightly Update Workflow

Automated workflow for updating Red Hat OpenShift AI to latest nightly builds and managing test results.

## Overview

This workflow provides an AI-powered pipeline for:
- Updating RHOAI to latest nightly builds
- Running automated test suites
- Collecting and analyzing test results
- Updating JIRA issues with outcomes

## Structure

```
workflows/rhoai-update/
├── .ambient/
│   └── ambient.json          # Workflow configuration
├── .claude/
│   └── commands/
│       └── rhoai-update.md   # RHOAI update command
└── README.md                 # This file
```

## Commands

### /rhoai-update

Updates RHOAI to the latest nightly build.

**What it does:**
- Checks current RHOAI version
- Updates the OLM catalog source to latest nightly
- Monitors the operator upgrade process
- Verifies component reconciliation
- Reports final status

**Usage:**
```
/rhoai-update
```

Or simply ask:
- "Update RHOAI to latest nightly"
- "Upgrade to RHOAI 3.4 nightly"
- "What version of RHOAI is installed?"

## Prerequisites

- OpenShift cluster with RHOAI installed via OLM
- `oc` CLI configured with cluster-admin access
- RHOAI installed from `rhoai-catalog-dev` catalog

## Output Artifacts

All artifacts are stored in `artifacts/rhoai-update/`:

- `reports/*.md` - Update reports and version changes
- `tests/*.log` - Test execution results
- `jira/*.md` - JIRA update summaries
- `logs/*.log` - Detailed execution logs

## GitHub Actions Integration

This workflow is designed to run via GitHub Actions with Ambient:

```yaml
- name: Update RHOAI to Latest Nightly
  uses: ambient-code/ambient-action@v0.0.2
  with:
    api-token: ${{ secrets.AMBIENT_API_TOKEN }}
    workflow: workflows/rhoai-update
    prompt: Update RHOAI to the latest nightly build
```

## Future Enhancements

- [ ] Automated test suite execution
- [ ] Test result parsing and analysis
- [ ] JIRA integration for issue updates
- [ ] Slack/email notifications
- [ ] Rollback capabilities
- [ ] Pre-upgrade validation checks
