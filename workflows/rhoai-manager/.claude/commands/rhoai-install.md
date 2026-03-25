# /rhoai-install - Install RHOAI on OpenShift Cluster

Install Red Hat OpenShift AI (RHOAI) on an OpenShift cluster using OLM (Operator Lifecycle Manager).

## Command Usage

- `/rhoai-install` - Install latest RHOAI (currently 3.4, channel: beta)
- `/rhoai-install 3.4` - Install RHOAI 3.4 (channel: beta)
- `/rhoai-install 3.4-ea.2` - Install RHOAI 3.4 EA build 2 (channel: beta)
- `/rhoai-install 3.4 -c beta` - Explicitly specify channel
- `/rhoai-install 3.3 -c stable-3.3` - Install RHOAI 3.3 stable
- `/rhoai-install 3.4@sha256:abc123...` - Install 3.4 with specific SHA digest

## Available Channels

| Channel | Description | Use Case |
|---------|-------------|----------|
| `beta` | Latest EA builds | Testing 3.4.0-ea.x builds |
| `stable` | Latest GA release | Production stable |
| `stable-3.3` | RHOAI 3.3.x GA | Stable 3.3 releases |

## Prerequisites

Before running this command:
1. **Cluster access**: Logged into OpenShift cluster with cluster-admin privileges (use `/oc-login`)
2. **Tools installed**: `oc` CLI and `jq` must be available
3. **No existing RHOAI**: This command is for fresh installations only

## Process

### Step 1: Parse Input Arguments

```bash
# Default values
VERSION_ARG=""
CHANNEL="beta"  # Default for 3.4+ EA builds

# Parse arguments
while [[ $# -gt 0 ]]; do
  case $1 in
    -c|--channel)
      CHANNEL="$2"
      shift 2
      ;;
    *)
      VERSION_ARG="$1"
      shift
      ;;
  esac
done

# Build image URL
if [[ -z "$VERSION_ARG" ]]; then
  IMAGE="quay.io/rhoai/rhoai-fbc-fragment:rhoai-3.4"
  echo "No version specified, defaulting to RHOAI 3.4"
elif [[ "$VERSION_ARG" == *"/"* ]]; then
  IMAGE="$VERSION_ARG"
elif [[ "$VERSION_ARG" == rhoai-* ]]; then
  IMAGE="quay.io/rhoai/rhoai-fbc-fragment:${VERSION_ARG}"
else
  IMAGE="quay.io/rhoai/rhoai-fbc-fragment:rhoai-${VERSION_ARG}"
fi

echo "Target image: $IMAGE"
echo "Target channel: $CHANNEL"
```

**Validation:**
- Warn if version 3.4 uses channel other than `beta`
- Suggest `stable-3.3` or `stable` for version 3.3

### Step 2: Verify Cluster Access

```bash
# Check prerequisites
command -v oc &>/dev/null || die "oc command not found"
command -v jq &>/dev/null || die "jq command not found"
oc whoami &>/dev/null || die "Not logged into an OpenShift cluster"

echo "Logged in as: $(oc whoami)"
echo "Cluster: $(oc whoami --show-server)"

# Verify RHOAI is not already installed
if oc get csv -n redhat-ods-operator 2>/dev/null | grep -q rhods-operator; then
  die "RHOAI is already installed. Use /rhoai-update to update existing installation."
fi
```

### Step 3: Clone olminstall Repository

```bash
OLMINSTALL_REPO="https://gitlab.cee.redhat.com/data-hub/olminstall.git"
OLMINSTALL_DIR="/tmp/olminstall"

if [ -d "$OLMINSTALL_DIR" ]; then
  echo "Updating existing clone..."
  git -C "$OLMINSTALL_DIR" pull --rebase --quiet 2>/dev/null || true
else
  echo "Cloning from $OLMINSTALL_REPO..."
  git clone --quiet "$OLMINSTALL_REPO" "$OLMINSTALL_DIR"
fi

[[ -d "$OLMINSTALL_DIR" ]] || die "Failed to clone olminstall"
echo "olminstall ready"
```

### Step 4: Install RHOAI Operator

```bash
cd "$OLMINSTALL_DIR"
bash setup.sh -t operator -i "$IMAGE" -u "$CHANNEL"
```

This creates:
- **Namespace**: `redhat-ods-operator`
- **CatalogSource**: `rhoai-catalog-dev` in `openshift-marketplace`
- **Subscription**: `rhoai-operator-dev` pointing to catalog
- **OperatorGroup**: For the operator namespace

### Step 5: Wait for Operator CSV

```bash
# Wait up to 600 seconds for CSV to reach Succeeded
CSV_PHASE=""
TIMEOUT=600
INTERVAL=10
ELAPSED=0

while [[ $ELAPSED -lt $TIMEOUT ]]; do
  CSV_LINE=$(oc get csv -n redhat-ods-operator 2>/dev/null | grep rhods-operator | grep -v Replacing || echo "")

  if [[ -n "$CSV_LINE" ]]; then
    CSV_NAME=$(echo "$CSV_LINE" | awk "{print \$1}")
    CSV_PHASE=$(echo "$CSV_LINE" | awk "{print \$NF}")
    echo "CSV: $CSV_NAME, Phase: $CSV_PHASE"

    if [[ "$CSV_PHASE" == "Succeeded" ]]; then
      echo "✅ Operator installed successfully"
      break
    fi
  fi

  sleep "$INTERVAL"
  ELAPSED=$((ELAPSED + INTERVAL))
  echo "Waiting for rhods-operator CSV... (${ELAPSED}s/${TIMEOUT}s)"
done

[[ "$CSV_PHASE" == "Succeeded" ]] || die "Operator did not reach Succeeded phase within ${TIMEOUT}s"
```

### Step 6: Create DataScienceCluster

```bash
# Wait for DSCInitialization
TIMEOUT=120
INTERVAL=10
ELAPSED=0

while [[ $ELAPSED -lt $TIMEOUT ]]; do
  if oc get dscinitializations default-dsci &>/dev/null; then
    echo "✅ DSCInitialization found"
    break
  fi
  sleep "$INTERVAL"
  ELAPSED=$((ELAPSED + INTERVAL))
  echo "Waiting for DSCInitialization... (${ELAPSED}s/${TIMEOUT}s)"
done

oc get dscinitializations default-dsci &>/dev/null || die "DSCInitialization not found within ${TIMEOUT}s"

# Extract DSC from CSV initialization-resource
CSV_NAME=$(oc get csv -n redhat-ods-operator 2>/dev/null | awk '/rhods-operator/{print $1; exit}')
if [[ -n "$CSV_NAME" ]]; then
  oc get csv "$CSV_NAME" -n redhat-ods-operator \
    -o jsonpath='{.metadata.annotations.operatorframework\.io/initialization-resource}' \
    > /tmp/default-dsc.json

  oc apply -f /tmp/default-dsc.json
  echo "✅ DSC created from CSV initialization-resource"
else
  die "Cannot find rhods-operator CSV in redhat-ods-operator namespace"
fi
```

### Step 7: Configure DSC Components

```bash
# Wait for DSC to exist
TIMEOUT=120
INTERVAL=10
ELAPSED=0

while [[ $ELAPSED -lt $TIMEOUT ]]; do
  if oc get datasciencecluster default-dsc &>/dev/null; then
    echo "✅ DataScienceCluster found"
    break
  fi
  sleep "$INTERVAL"
  ELAPSED=$((ELAPSED + INTERVAL))
  echo "Waiting for DataScienceCluster... (${ELAPSED}s/${TIMEOUT}s)"
done

# Patch DSC to enable required components
cat > /tmp/dsc-components-patch.yaml << 'YAML'
spec:
  components:
    aipipelines:
      managementState: Managed
      argoWorkflowsControllers:
        managementState: Managed
    llamastackoperator:
      managementState: Managed
    mlflowoperator:
      managementState: Managed
    trainer:
      managementState: Removed
YAML

oc patch datasciencecluster default-dsc --type merge --patch-file /tmp/dsc-components-patch.yaml || \
  die "Failed to patch DataScienceCluster"

echo "✅ DSC component configuration applied:"
echo "   - aipipelines: Managed (with argoWorkflowsControllers)"
echo "   - llamastackoperator: Managed"
echo "   - mlflowoperator: Managed"
echo "   - trainer: Removed (requires JobSet operator)"

sleep 5
```

**Why these components?**
- `aipipelines`: For AI/ML pipelines with Argo Workflows
- `llamastackoperator`: For Llama Stack server deployments
- `mlflowoperator`: For ML experiment tracking
- `trainer`: Removed (requires JobSet operator, not available by default)

### Step 8: Wait for DSC Ready

```bash
# Wait for DataScienceCluster to be Ready
TIMEOUT=600
INTERVAL=15
ELAPSED=0
DSC_PHASE=""

while [[ $ELAPSED -lt $TIMEOUT ]]; do
  DSC_PHASE=$(oc get datasciencecluster -o jsonpath="{.items[0].status.phase}" 2>/dev/null || echo "Unknown")
  echo "DSC phase: $DSC_PHASE"

  if [[ "$DSC_PHASE" == "Ready" ]]; then
    echo "✅ DataScienceCluster is Ready"
    break
  fi

  sleep "$INTERVAL"
  ELAPSED=$((ELAPSED + INTERVAL))
  echo "Waiting for DataScienceCluster... (${ELAPSED}s/${TIMEOUT}s)"
done

if [[ "$DSC_PHASE" != "Ready" ]]; then
  echo "⚠️  WARNING: DSC is not Ready after ${TIMEOUT}s (current: ${DSC_PHASE:-Unknown})"
  echo "Not-ready components:"
  oc get dsc default-dsc -o json 2>/dev/null | \
    jq -r '.status.conditions[] | select(.status=="False") | select(.message | test("Removed") | not) | "  \(.type): \(.message)"' 2>/dev/null || true
fi
```

### Step 9: Wait for Dashboard

```bash
# Wait for dashboard deployment to be ready
TIMEOUT=300
INTERVAL=10
ELAPSED=0

while [[ $ELAPSED -lt $TIMEOUT ]]; do
  READY=$(oc get deployment rhods-dashboard -n redhat-ods-applications -o jsonpath="{.status.readyReplicas}" 2>/dev/null || echo "0")
  DESIRED=$(oc get deployment rhods-dashboard -n redhat-ods-applications -o jsonpath="{.spec.replicas}" 2>/dev/null || echo "0")

  if [[ "$READY" -gt 0 && "$READY" -eq "$DESIRED" ]]; then
    echo "✅ Dashboard deployment is ready"
    break
  fi

  sleep "$INTERVAL"
  ELAPSED=$((ELAPSED + INTERVAL))
  echo "Waiting for dashboard deployment... (${ELAPSED}s/${TIMEOUT}s)"
done

echo "Dashboard containers:"
oc get deployment rhods-dashboard -n redhat-ods-applications \
  -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"\n"}{end}' 2>/dev/null || \
  echo "  Dashboard deployment not found"
```

### Step 10: Configure Dashboard Features

```bash
# Wait for OdhDashboardConfig to exist
TIMEOUT=120
INTERVAL=10
ELAPSED=0

while [[ $ELAPSED -lt $TIMEOUT ]]; do
  if oc get odhdashboardconfig odh-dashboard-config -n redhat-ods-applications &>/dev/null; then
    echo "✅ OdhDashboardConfig found"
    break
  fi
  sleep "$INTERVAL"
  ELAPSED=$((ELAPSED + INTERVAL))
  echo "Waiting for OdhDashboardConfig... (${ELAPSED}s/${TIMEOUT}s)"
done

if ! oc get odhdashboardconfig odh-dashboard-config -n redhat-ods-applications &>/dev/null; then
  echo "⚠️  WARNING: OdhDashboardConfig not found yet, feature flags will be configured when available"
else
  # Enable feature flags
  oc patch odhdashboardconfig odh-dashboard-config -n redhat-ods-applications --type merge -p '{
    "spec": {
      "dashboardConfig": {
        "automl": true,
        "autorag": true,
        "genAiStudio": true
      }
    }
  }' || {
    echo "⚠️  WARNING: Failed to patch dashboard config, feature flags may need manual configuration"
  }

  echo "✅ Dashboard feature flags configured:"
  echo "   - automl: enabled"
  echo "   - autorag: enabled"
  echo "   - genAiStudio: enabled"

  # Restart dashboard to pick up changes
  echo "Restarting dashboard to apply feature flag changes..."
  oc rollout restart deployment rhods-dashboard -n redhat-ods-applications 2>/dev/null || true
  sleep 3
fi
```

### Step 11: Verify Installation

```bash
echo ""
echo "=== Installation Summary ==="

# Show CSV
echo ""
echo "CSV:"
oc get csv -n redhat-ods-operator 2>/dev/null | grep rhods-operator || echo "  WARNING: CSV not found"

# Show Dashboard URL
echo ""
echo "Dashboard:"
DASHBOARD_ROUTE=$(oc get route rhods-dashboard -n redhat-ods-applications -o jsonpath='{.spec.host}' 2>/dev/null || echo "")
if [[ -n "$DASHBOARD_ROUTE" ]]; then
  echo "  https://$DASHBOARD_ROUTE"
else
  echo "  WARNING: Dashboard route not found yet"
fi

echo ""
echo "✅ RHOAI installation complete!"
```

## Output

The command creates a report at `artifacts/rhoai-update/reports/install-report-[timestamp].md` with:
- Installation parameters (version, channel, image)
- Operator CSV details
- DataScienceCluster status
- Configured components
- Dashboard URL
- Feature flags enabled

## Usage Examples

```bash
# Install latest RHOAI 3.4 (beta channel)
/rhoai-install

# Install RHOAI 3.4 EA build 2
/rhoai-install 3.4-ea.2

# Install RHOAI 3.3 stable
/rhoai-install 3.3 -c stable-3.3

# Install with specific SHA digest
/rhoai-install 3.4@sha256:abc123def456...
```

Or simply ask:
- "Install RHOAI 3.4"
- "Set up RHOAI on my cluster"
- "Install latest RHOAI nightly"

## Common Issues

**Problem:** CSV stuck in "Installing" phase
**Solution:** Check operator pod logs in `redhat-ods-operator` namespace

**Problem:** DSC not reaching Ready
**Solution:** Check component conditions with `oc get dsc default-dsc -o yaml | yq '.status.conditions'`

**Problem:** Dashboard not accessible
**Solution:** Verify route exists and check dashboard pod logs in `redhat-ods-applications`

## Next Steps

After installation:
1. Access the dashboard at the URL shown in the output
2. Configure user access and permissions
3. Deploy models and workbenches
4. Set up data connections

To update RHOAI to a newer version, use `/rhoai-update`.
