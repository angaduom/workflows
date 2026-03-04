# /rhoai-version - Detect RHOAI Version and Build Information

Detect the Red Hat OpenShift AI (RHOAI) version and build information installed on the currently connected OpenShift cluster.

## Purpose

This command provides comprehensive version information about the RHOAI installation including operator version, component status, and deployed image digests.

## Prerequisites

- Must be logged into an OpenShift cluster (use `/oc-login` first if needed)
- RHOAI must be installed on the cluster

## Steps

### 1. Verify OpenShift Login

Run `oc whoami` and `oc whoami --show-server` to confirm you are logged into an OpenShift cluster. If not logged in, stop and inform the user they need to authenticate first with `/oc-login`.

### 2. Detect RHOAI Operator Subscription

Run the following to extract subscription details directly (avoid `-o yaml` as it produces excessive output):

```bash
oc get subscriptions.operators.coreos.com rhods-operator -n redhat-ods-operator -o jsonpath='Channel: {.spec.channel}{"\n"}Source: {.spec.source}{"\n"}Approval: {.spec.installPlanApproval}{"\n"}Current CSV: {.status.currentCSV}{"\n"}Installed CSV: {.status.installedCSV}{"\n"}Starting CSV: {.spec.startingCSV}{"\n"}' 2>/dev/null
```

If no subscription found in `redhat-ods-operator`, check `openshift-operators`:

```bash
oc get subscriptions.operators.coreos.com -n openshift-operators -o jsonpath='{range .items[?(@.spec.name=="rhods-operator")]}Channel: {.spec.channel}{"\n"}Source: {.spec.source}{"\n"}Approval: {.spec.installPlanApproval}{"\n"}Current CSV: {.status.currentCSV}{"\n"}{end}' 2>/dev/null
```

### 3. Check ClusterServiceVersion (CSV)

Get only the RHOAI operator CSV (filter by name to avoid noisy output):

```bash
oc get csv -n redhat-ods-operator -o custom-columns=NAME:.metadata.name,DISPLAY:.spec.displayName,VERSION:.spec.version,PHASE:.status.phase 2>/dev/null | grep -E 'NAME|rhods-operator'
```

### 4. Check DataScienceCluster

**Do NOT use `-o yaml` for the full DSC resource** — it is very large and the jsonpath for nested dynamic component keys does not extract cleanly.

Instead, use these targeted commands:

**Get component managementState values:**
```bash
oc get datasciencecluster default-dsc -o json 2>/dev/null | python3 -c "
import sys, json
dsc = json.load(sys.stdin)
comps = dsc.get('spec', {}).get('components', {})
for name, cfg in sorted(comps.items()):
    state = cfg.get('managementState', 'Unknown') if isinstance(cfg, dict) else 'Unknown'
    print(f'  {name}: {state}')
"
```

**Get status conditions:**
```bash
oc get datasciencecluster default-dsc -o jsonpath='{range .status.conditions[*]}{.type}: {.status} ({.reason}){"\n"}{end}' 2>/dev/null
```

### 5. Check DSCInitialization

```bash
oc get dscinitializations default-dsci -o jsonpath='Name: {.metadata.name}{"\n"}Monitoring: {.spec.monitoring.managementState}{"\n"}' 2>/dev/null
```

### 6. Extract Operator Image

```bash
oc get deployment rhods-operator -n redhat-ods-operator -o jsonpath='{.spec.template.spec.containers[*].image}' 2>/dev/null
```

If not found, try the ODH deployment name:
```bash
oc get deployment opendatahub-operator-controller-manager -n openshift-operators -o jsonpath='{.spec.template.spec.containers[*].image}' 2>/dev/null
```

### 7. Get Component Images (Always Run)

Collect all deployed component images from `redhat-ods-applications`. This is NOT optional — always include this table.

```bash
oc get deployments -n redhat-ods-applications -o custom-columns='COMPONENT:.metadata.name,IMAGE:.spec.template.spec.containers[0].image' 2>/dev/null
```

Parse each image to extract a short image name and the `sha256` digest. Present as a markdown table:

```
| Component                        | Image                                              | Digest (short) |
|----------------------------------|------------------------------------------------------|-----------------|
| rhods-dashboard                  | odh-dashboard-rhel9                                  | sha256:db295f.. |
| kserve-controller-manager        | odh-kserve-controller-rhel9                          | sha256:e83b4b.. |
| ...                              | ...                                                  | ...             |
```

To build this table:
- **Component** = the deployment name
- **Image** = the portion after the last `/` and before `@sha256:` (e.g., `odh-dashboard-rhel9`)
- **Digest (short)** = first 8 characters of the sha256 hash

### 8. Present Summary

Output a clear summary in this format:

```
== RHOAI Version Summary ==

Cluster:          <server URL>
Logged in as:     <username>

Operator:
  Name:           <CSV name>
  Version:        <version>
  Phase:          <phase>
  Channel:        <subscription channel>
  Source:         <catalog source>
  Approval:       <install plan approval>
  Operator Image: <image reference>

DataScienceCluster:
  Name:           default-dsc
  Status:         <Ready/Not Ready> (<conditions summary>)
  Components:
    - <component>: <Managed|Removed>
    ...

DSCInitialization:
  Name:           default-dsci
  Monitoring:     <Managed/Removed>

== Component Images (redhat-ods-applications) ==

| Component | Image | Digest (short) |
|-----------|-------|-----------------|
| ...       | ...   | ...             |
```

If any resource is not found, note it clearly (e.g., "Not installed" or "Namespace not found") rather than failing silently.

## Important Notes

- **Do NOT use `oc get datasciencecluster -o yaml`** — the output is extremely large (hundreds of lines) and jsonpath with dynamic component keys fails to extract cleanly. Use the `python3 -c` approach in Step 4 or targeted jsonpath for conditions.
- **Do NOT use `-o yaml` for subscriptions** — use targeted jsonpath to extract only the fields you need.
- **The DSC resource name is `default-dsc`** and DSCI is `default-dsci` on standard RHOAI installs. Always reference by name for reliable extraction.
- **Component keys in `spec.components` are dynamic** and vary by RHOAI version. Do not hardcode a list — iterate over whatever keys exist.
- **Status conditions have changed across versions** — older versions used `Available/Progressing/Degraded/Upgradeable`, newer versions (3.x) use per-component `*Ready` conditions plus `Ready`, `ProvisioningSucceeded`, `ComponentsReady`. Handle both.

## Example Usage

**User**: `/rhoai-version`

**Claude**:
1. Checks if user is logged into cluster
2. Queries RHOAI operator subscription and CSV
3. Checks DataScienceCluster and DSCInitialization status
4. Lists all component images with digests
5. Presents formatted summary with all version information

## Integration with Other Commands

This command is useful:
- Before running `/rhoai-update` to know current version
- After running `/rhoai-update` to verify the new version
- For troubleshooting RHOAI installations
- For documenting the current cluster state
