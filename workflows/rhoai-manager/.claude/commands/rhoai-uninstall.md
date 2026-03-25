# /rhoai-uninstall - Uninstall RHOAI from Cluster

Completely uninstall Red Hat OpenShift AI (RHOAI) from an OpenShift cluster, removing all related resources.

## Purpose

This command performs a comprehensive cleanup of RHOAI, removing the operator, custom resources, CRDs, and all related namespaces.

## Prerequisites

- Must be logged into an OpenShift cluster (use `/oc-login` first if needed)
- Cluster admin permissions required
- RHOAI must be installed on the cluster

## Command Usage

- `/rhoai-uninstall` - Standard uninstall (forceful cleanup)
- `/rhoai-uninstall graceful` - Graceful uninstall followed by forceful cleanup
- `/rhoai-uninstall keep-crds` - Uninstall but keep CRDs
- `/rhoai-uninstall keep-all` - Keep CRDs and user resources (projects, models, etc.)

## Uninstall Options

### Standard Uninstall (Default)
Forcefully removes all RHOAI resources including:
- Operator and subscriptions
- Custom resources (DSC, DSCInitialization, etc.)
- CRDs
- Namespaces
- User resources (data science projects, models, workbenches)

### Graceful Uninstall
Attempts graceful removal first, then forceful cleanup:
- Allows RHOAI to clean up resources in proper order
- Runs finalizers correctly
- Falls back to forceful cleanup if graceful fails

### Keep CRDs
Removes RHOAI but keeps the CRDs installed

### Keep All
Keeps both CRDs and user resources:
- Data science projects remain
- User models, workbenches, connections preserved
- Useful for reinstalling RHOAI without losing user work

## Uninstall Process

### Step 1: Verify Cluster Access

Check that you're logged into the cluster with admin permissions:

```bash
# Verify login
oc whoami

# Verify admin permissions
oc auth can-i delete namespace
```

If not logged in or lacking permissions, stop and inform the user.

### Step 2: Check Current RHOAI Installation

Verify RHOAI is installed:

```bash
# Check for RHOAI operator namespace
oc get namespace redhat-ods-operator 2>/dev/null

# Check for RHOAI operator
oc get csv -n redhat-ods-operator | grep rhods-operator

# Check for DataScienceCluster
oc get datasciencecluster -A
```

Report what's found and confirm with user before proceeding.

### Step 3: Graceful Uninstall (if requested)

If graceful uninstall is requested:

```bash
# Create the deletion ConfigMap
oc create configmap delete-self-managed-odh -n redhat-ods-operator

# Label it to trigger graceful deletion
oc label configmap/delete-self-managed-odh \
  api.openshift.com/addon-managed-odh-delete=true \
  -n redhat-ods-operator

# Wait for redhat-ods-applications namespace to be removed (up to 5 minutes)
echo "Waiting for graceful deletion to complete (max 5 minutes)..."
if oc wait --for=delete --timeout=300s namespace redhat-ods-applications 2>/dev/null; then
  echo "✅ Graceful deletion completed successfully"
else
  echo "⚠️  Graceful deletion timed out or failed, proceeding with forceful cleanup"
fi

# Clean up the ConfigMap
oc delete configmap delete-self-managed-odh -n redhat-ods-operator --ignore-not-found
```

### Step 4: Delete RHOAI Custom Resources

Remove all RHOAI custom resources before deleting CRDs:

```bash
# Delete DataScienceCluster
echo "Deleting DataScienceCluster resources..."
oc get datasciencecluster -A -o custom-columns=:metadata.name,:metadata.namespace --no-headers | \
  while read name namespace; do
    oc patch datasciencecluster $name -n $namespace --type=merge -p '{"metadata":{"finalizers":null}}' 2>/dev/null || true
    oc delete datasciencecluster $name -n $namespace --timeout=60s --ignore-not-found
  done

# Delete DSCInitialization
echo "Deleting DSCInitialization resources..."
oc get dscinitialization -A -o custom-columns=:metadata.name,:metadata.namespace --no-headers | \
  while read name namespace; do
    oc patch dscinitialization $name -n $namespace --type=merge -p '{"metadata":{"finalizers":null}}' 2>/dev/null || true
    oc delete dscinitialization $name -n $namespace --timeout=60s --ignore-not-found
  done

# Delete Notebooks (they often have finalizers)
echo "Deleting Notebook resources..."
oc get notebooks.kubeflow.org -A -o custom-columns=:metadata.name,:metadata.namespace --no-headers | \
  while read name namespace; do
    oc patch notebooks.kubeflow.org $name -n $namespace --type=merge -p '{"metadata":{"finalizers":null}}' 2>/dev/null || true
    oc delete notebooks.kubeflow.org $name -n $namespace --timeout=60s --ignore-not-found
  done

# Delete InferenceServices
echo "Deleting InferenceService resources..."
oc delete inferenceservices.serving.kserve.io --all -A --ignore-not-found --timeout=60s

# Delete ServingRuntimes
echo "Deleting ServingRuntime resources..."
oc delete servingruntimes.serving.kserve.io --all -A --ignore-not-found --timeout=60s

# Delete DataSciencePipelinesApplications
echo "Deleting DataSciencePipelinesApplication resources..."
oc delete datasciencepipelinesapplications.datasciencepipelinesapplications.opendatahub.io --all -A --ignore-not-found --timeout=60s
```

### Step 5: Delete Webhooks

Remove validating and mutating webhooks that may block deletion:

```bash
# Delete RHOAI-related validating webhooks
echo "Deleting validating webhooks..."
oc get validatingwebhookconfiguration -o json | \
  jq -r '.items[] | select(.metadata.name | test("odh|rhods|opendatahub|kserve")) | .metadata.name' | \
  xargs -r oc delete validatingwebhookconfiguration

# Delete RHOAI-related mutating webhooks
echo "Deleting mutating webhooks..."
oc get mutatingwebhookconfiguration -o json | \
  jq -r '.items[] | select(.metadata.name | test("odh|rhods|opendatahub|kserve")) | .metadata.name' | \
  xargs -r oc delete mutatingwebhookconfiguration
```

### Step 6: Delete RHOAI Operator

Remove the operator subscription and CSV:

```bash
# Delete subscription
echo "Deleting RHOAI operator subscription..."
oc delete subscription rhods-operator -n redhat-ods-operator --ignore-not-found --timeout=60s

# Delete CSV
echo "Deleting ClusterServiceVersion..."
CSV_NAME=$(oc get csv -n redhat-ods-operator -o custom-columns=:metadata.name --no-headers | grep rhods-operator)
if [ -n "$CSV_NAME" ]; then
  oc delete csv $CSV_NAME -n redhat-ods-operator --ignore-not-found --timeout=60s
fi

# Delete catalog source if it's a dev catalog
echo "Checking for dev catalog sources..."
if oc get catalogsource rhoai-catalog-dev -n openshift-marketplace &>/dev/null; then
  echo "Deleting rhoai-catalog-dev..."
  oc delete catalogsource rhoai-catalog-dev -n openshift-marketplace --ignore-not-found
fi
```

### Step 7: Delete Namespaces

Remove all RHOAI-related namespaces:

```bash
# List of RHOAI namespaces
NAMESPACES="redhat-ods-operator redhat-ods-applications redhat-ods-applications-auth-provider redhat-ods-monitoring rhods-notebooks rhoai-model-registries"

for ns in $NAMESPACES; do
  if oc get namespace $ns &>/dev/null; then
    echo "Deleting namespace: $ns"

    # Delete all resources in the namespace first
    oc delete all --all -n $ns --ignore-not-found --timeout=30s 2>/dev/null || true

    # Delete the namespace
    oc delete namespace $ns --ignore-not-found --timeout=60s || true

    # If stuck in Terminating, remove finalizers
    if oc get namespace $ns -o jsonpath='{.status.phase}' 2>/dev/null | grep -q "Terminating"; then
      echo "  Namespace stuck in Terminating, removing finalizers..."
      oc patch namespace $ns -p '{"spec":{"finalizers":[]}}' --type=merge 2>/dev/null || true
    fi
  fi
done
```

### Step 8: Delete CRDs (unless keep-crds or keep-all)

If user didn't request to keep CRDs:

```bash
echo "Deleting RHOAI CRDs..."

# Core RHOAI CRDs
oc delete crd datascienceclusters.datasciencecluster.opendatahub.io --ignore-not-found
oc delete crd dscinitializations.dscinitialization.opendatahub.io --ignore-not-found
oc delete crd acceleratorprofiles.dashboard.opendatahub.io --ignore-not-found
oc delete crd hardwareprofiles.dashboard.opendatahub.io --ignore-not-found
oc delete crd odhapplications.dashboard.opendatahub.io --ignore-not-found
oc delete crd odhdashboardconfigs.opendatahub.io --ignore-not-found
oc delete crd odhdocuments.dashboard.opendatahub.io --ignore-not-found
oc delete crd modelregistries.modelregistry.opendatahub.io --ignore-not-found

# KServe CRDs
oc delete crd inferenceservices.serving.kserve.io --ignore-not-found
oc delete crd servingruntimes.serving.kserve.io --ignore-not-found
oc delete crd inferencegraphs.serving.kserve.io --ignore-not-found

# Notebook CRDs (remove finalizers first)
oc get notebooks.kubeflow.org -A -o custom-columns=:metadata.name,:metadata.namespace --no-headers | \
  while read name namespace; do
    oc patch notebooks.kubeflow.org $name -n $namespace --type=merge -p '{"metadata":{"finalizers":null}}' 2>/dev/null || true
  done
oc delete crd notebooks.kubeflow.org --ignore-not-found

# DataSciencePipelinesApplications
oc delete crd datasciencepipelinesapplications.datasciencepipelinesapplications.opendatahub.io --ignore-not-found

# All CRDs labeled by RHOAI operator
oc delete crd -l operators.coreos.com/rhods-operator.redhat-ods-operator --ignore-not-found

# Ray CRDs
oc delete crd rayclusters.ray.io --ignore-not-found
oc delete crd rayjobs.ray.io --ignore-not-found
oc delete crd rayservices.ray.io --ignore-not-found

# CodeFlare CRDs
oc delete crd appwrappers.workload.codeflare.dev --ignore-not-found

# TrustyAI CRDs
oc delete crd trustyaiservices.trustyai.opendatahub.io --ignore-not-found
```

### Step 9: Clean Up User Resources (if keep-all not requested)

Remove user data science projects and resources:

```bash
# Find and delete data science project namespaces
echo "Looking for user data science projects..."
USER_PROJECTS=$(oc get namespaces -l opendatahub.io/dashboard=true -o custom-columns=:metadata.name --no-headers)

if [ -n "$USER_PROJECTS" ]; then
  echo "Found user projects: $USER_PROJECTS"
  for project in $USER_PROJECTS; do
    echo "  Deleting project: $project"
    oc delete namespace $project --ignore-not-found --timeout=60s || true
  done
else
  echo "No user data science projects found"
fi
```

### Step 10: Verify Cleanup

Check that all resources have been removed:

```bash
# Check for remaining RHOAI namespaces
echo "Checking for remaining RHOAI namespaces..."
REMAINING_NS=$(oc get namespaces | grep -E "redhat-ods|rhods|rhoai" || echo "")
if [ -n "$REMAINING_NS" ]; then
  echo "⚠️  Some namespaces still exist:"
  echo "$REMAINING_NS"
else
  echo "✅ All RHOAI namespaces removed"
fi

# Check for RHOAI CRDs
echo "Checking for remaining RHOAI CRDs..."
REMAINING_CRDS=$(oc get crd | grep -E "opendatahub|kubeflow|kserve" || echo "")
if [ -n "$REMAINING_CRDS" ]; then
  echo "⚠️  Some CRDs still exist:"
  echo "$REMAINING_CRDS"
else
  echo "✅ All RHOAI CRDs removed"
fi

# Check for operator
echo "Checking for RHOAI operator..."
REMAINING_CSV=$(oc get csv -A | grep rhods-operator || echo "")
if [ -n "$REMAINING_CSV" ]; then
  echo "⚠️  RHOAI operator still exists:"
  echo "$REMAINING_CSV"
else
  echo "✅ RHOAI operator removed"
fi
```

### Step 11: Report Summary

Provide a summary of what was removed:

```
✅ RHOAI Uninstall Complete!

Removed:
- RHOAI Operator
- DataScienceCluster and DSCInitialization
- All RHOAI namespaces
- Custom Resources (notebooks, inference services, etc.)
[- CRDs (if not kept)]
[- User data science projects (if not kept)]

The cluster is now clean and ready for a fresh RHOAI installation if needed.
```

## Important Warnings

**Before running this command, warn the user:**

1. **⚠️  Data Loss Warning**
   - This will DELETE all RHOAI resources including user workbenches, models, and data
   - User should backup any important work first
   - Cannot be undone

2. **⚠️  Cluster Access Required**
   - Requires cluster-admin permissions
   - Will modify cluster-wide resources (CRDs, webhooks)

3. **⚠️  Downtime Warning**
   - Any running workloads will be terminated
   - Data science pipelines will be stopped
   - Active model servers will be shut down

## Example Interactions

### Example 1: Standard Uninstall

**User**: `/rhoai-uninstall`

**Claude**:
1. Checks cluster access and RHOAI installation
2. Warns about data loss and asks for confirmation
3. Deletes custom resources
4. Removes webhooks
5. Deletes operator
6. Removes namespaces
7. Deletes CRDs
8. Reports: "✅ RHOAI completely removed from cluster"

### Example 2: Graceful Uninstall

**User**: `/rhoai-uninstall graceful`

**Claude**:
1. Creates deletion ConfigMap
2. Waits for graceful deletion (up to 5 minutes)
3. If graceful succeeds, cleans up remaining resources
4. If graceful fails/times out, proceeds with forceful cleanup
5. Reports final status

### Example 3: Keep User Resources

**User**: `/rhoai-uninstall keep-all`

**Claude**:
1. Removes RHOAI operator and core resources
2. Keeps CRDs installed
3. Preserves user data science projects
4. Reports: "✅ RHOAI operator removed. CRDs and user projects preserved."

## Troubleshooting

### Issue 1: Namespaces Stuck in Terminating

**Cause**: Finalizers or webhooks blocking deletion

**Solution**:
```bash
# Remove finalizers
oc patch namespace <ns-name> -p '{"spec":{"finalizers":[]}}' --type=merge

# Delete blocking webhooks
oc delete validatingwebhookconfiguration --all
oc delete mutatingwebhookconfiguration --all
```

### Issue 2: CRDs Won't Delete

**Cause**: Custom resources still exist

**Solution**: Delete all custom resources first, remove finalizers if needed

### Issue 3: Permission Denied

**Cause**: Insufficient permissions

**Solution**: Must be cluster-admin. Check with `oc auth can-i delete namespace`

## Integration with Other Commands

Typical workflow:
```
/oc-login              # Login to cluster
/rhoai-version         # Check what's installed
/rhoai-uninstall       # Remove RHOAI
```

## Success Criteria

Uninstall is successful when:
- ✅ All RHOAI namespaces deleted
- ✅ RHOAI operator removed
- ✅ CRDs deleted (unless kept)
- ✅ No RHOAI webhooks remain
- ✅ `oc get csv -A | grep rhods-operator` returns nothing
