# /rhoai-install - Install RHOAI on OpenShift Cluster

Install Red Hat OpenShift AI (RHOAI) on an OpenShift cluster using OLM (Operator Lifecycle Manager).

## Command Usage

### Development/Nightly Builds (default)
```bash
/rhoai-install                                    # Latest dev catalog (3.4 beta)
/rhoai-install channel=beta                       # Explicit beta channel
/rhoai-install image=quay.io/modh/rhoai-catalog:latest-release-3.5  # Custom image
```

### GA Production Releases
```bash
/rhoai-install catalog=redhat-operators           # GA catalog, stable channel
/rhoai-install catalog=redhat-operators channel=fast     # GA catalog, fast channel
/rhoai-install catalog=redhat-operators channel=stable   # GA catalog, stable channel
```

### Combined Parameters
```bash
/rhoai-install catalog=rhoai-catalog-dev channel=beta image=quay.io/modh/rhoai-catalog:custom
```

## Catalog Types

| Catalog | Description | Use Case |
|---------|-------------|----------|
| `rhoai-catalog-dev` (default) | Development nightly builds | Testing EA/nightly builds |
| `redhat-operators` | Red Hat certified GA releases | Production deployments |

## Available Channels

| Channel | Description | Catalog Type |
|---------|-------------|--------------|
| `beta` (default) | Latest EA/nightly builds | rhoai-catalog-dev |
| `fast` | Early GA releases | redhat-operators |
| `stable` | Stable GA releases | redhat-operators |

## Prerequisites

Before running this command:
1. **Cluster access**: Logged into OpenShift cluster with cluster-admin privileges (use `/oc-login`)
2. **Tools installed**: `oc` CLI and `jq` must be available
3. **No existing RHOAI**: This command is for fresh installations only

## Process

### Step 1: Parse Input Arguments

```bash
# Default values
CATALOG_SOURCE="rhoai-catalog-dev"
CATALOG_IMAGE=""
CHANNEL="beta"
CUSTOM_IMAGE_OVERRIDE=""

# Parse key=value arguments
for arg in "$@"; do
  case "$arg" in
    catalog=*)
      CATALOG_SOURCE="${arg#*=}"
      ;;
    channel=*)
      CHANNEL="${arg#*=}"
      ;;
    image=*)
      CUSTOM_IMAGE_OVERRIDE="${arg#*=}"
      ;;
    *)
      echo "⚠️  Unknown parameter: $arg (expected: catalog=, channel=, or image=)"
      ;;
  esac
done

# Smart defaults based on catalog type
if [[ "$CATALOG_SOURCE" == "rhoai-catalog-dev" ]]; then
  # Development catalog - use custom image or default
  if [[ -n "$CUSTOM_IMAGE_OVERRIDE" ]]; then
    CATALOG_IMAGE="$CUSTOM_IMAGE_OVERRIDE"
  else
    CATALOG_IMAGE="quay.io/modh/rhoai-catalog:latest-release-3.4"
  fi
  CATALOG_NAMESPACE="openshift-marketplace"
  USE_CUSTOM_CATALOG=true

  echo "📦 Catalog: Development (rhoai-catalog-dev)"
  echo "   Image: $CATALOG_IMAGE"
  echo "   Channel: $CHANNEL"

elif [[ "$CATALOG_SOURCE" == "redhat-operators" ]]; then
  # GA catalog - uses built-in Red Hat operators catalog
  CATALOG_IMAGE=""
  CATALOG_NAMESPACE="openshift-marketplace"
  USE_CUSTOM_CATALOG=false

  echo "📦 Catalog: GA Production (redhat-operators)"
  echo "   Channel: $CHANNEL"

  if [[ -n "$CUSTOM_IMAGE_OVERRIDE" ]]; then
    echo "⚠️  WARNING: image parameter ignored for redhat-operators catalog (uses built-in catalog)"
  fi

else
  echo "❌ ERROR: Unknown catalog '$CATALOG_SOURCE'"
  echo "   Supported: rhoai-catalog-dev, redhat-operators"
  exit 1
fi
```

**Parameter Summary:**
- `catalog` - Catalog source to use (default: `rhoai-catalog-dev`)
- `channel` - Subscription channel (default: `beta`)
- `image` - Custom catalog image (only for rhoai-catalog-dev)

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

### Step 3: Create Operator Namespace

```bash
OPERATOR_NAMESPACE="redhat-ods-operator"

# Create namespace if it doesn't exist
if ! oc get namespace "$OPERATOR_NAMESPACE" &>/dev/null; then
  oc create namespace "$OPERATOR_NAMESPACE"
  echo "✅ Created namespace: $OPERATOR_NAMESPACE"
else
  echo "✅ Namespace already exists: $OPERATOR_NAMESPACE"
fi
```

### Step 4: Create CatalogSource (if using custom catalog)

```bash
if [[ "$USE_CUSTOM_CATALOG" == "true" ]]; then
  echo "Creating custom CatalogSource: $CATALOG_SOURCE"

  cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: $CATALOG_SOURCE
  namespace: $CATALOG_NAMESPACE
spec:
  displayName: "Red Hat OpenShift AI Dev Catalog"
  image: $CATALOG_IMAGE
  publisher: Red Hat
  sourceType: grpc
  updateStrategy:
    registryPoll:
      interval: 30m
EOF

  echo "✅ CatalogSource created: $CATALOG_SOURCE"

  # Wait for catalog to be ready
  echo "Waiting for CatalogSource to be ready..."
  TIMEOUT=300
  INTERVAL=10
  ELAPSED=0

  while [[ $ELAPSED -lt $TIMEOUT ]]; do
    CATALOG_STATE=$(oc get catalogsource "$CATALOG_SOURCE" -n "$CATALOG_NAMESPACE" \
      -o jsonpath='{.status.connectionState.lastObservedState}' 2>/dev/null || echo "")

    if [[ "$CATALOG_STATE" == "READY" ]]; then
      echo "✅ CatalogSource is READY"
      break
    fi

    sleep "$INTERVAL"
    ELAPSED=$((ELAPSED + INTERVAL))
    echo "   CatalogSource state: ${CATALOG_STATE:-Unknown} (${ELAPSED}s/${TIMEOUT}s)"
  done

  [[ "$CATALOG_STATE" == "READY" ]] || echo "⚠️  WARNING: CatalogSource not READY after ${TIMEOUT}s"
else
  echo "Using built-in catalog: $CATALOG_SOURCE"
fi
```

### Step 5: Create OperatorGroup

```bash
# Create OperatorGroup in operator namespace
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: rhods-operator
  namespace: $OPERATOR_NAMESPACE
spec:
  targetNamespaces:
  - $OPERATOR_NAMESPACE
EOF

echo "✅ OperatorGroup created"
```

### Step 6: Create Subscription

```bash
# Create Subscription
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhods-operator
  namespace: $OPERATOR_NAMESPACE
spec:
  channel: $CHANNEL
  installPlanApproval: Automatic
  name: rhods-operator
  source: $CATALOG_SOURCE
  sourceNamespace: $CATALOG_NAMESPACE
EOF

echo "✅ Subscription created"
echo "   Channel: $CHANNEL"
echo "   Source: $CATALOG_SOURCE"

sleep 5
```

This creates:
- **Namespace**: `redhat-ods-operator`
- **CatalogSource**: Custom catalog (if using dev catalog) or uses built-in `redhat-operators`
- **Subscription**: `rhods-operator` pointing to the chosen catalog
- **OperatorGroup**: For the operator namespace

### Step 7: Wait for Operator CSV

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

### Step 8: Create DataScienceCluster

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

### Step 9: Configure DSC Components

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

### Step 10: Wait for DSC Ready

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

### Step 11: Wait for Dashboard

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

### Step 12: Configure Dashboard Features

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

### Step 13: Verify Installation

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

The command creates a report at `artifacts/rhoai-manager/reports/install-report-[timestamp].md` with:
- Installation parameters (catalog source, channel, image)
- Operator CSV details
- DataScienceCluster status
- Configured components
- Dashboard URL
- Feature flags enabled

## Usage Examples

### Development/Testing
```bash
# Install latest dev build (default: beta channel, dev catalog)
/rhoai-install

# Install from dev catalog with custom image
/rhoai-install image=quay.io/modh/rhoai-catalog:latest-release-3.5

# Install from dev catalog with specific channel
/rhoai-install channel=beta
```

### Production GA
```bash
# Install from GA catalog (stable channel)
/rhoai-install catalog=redhat-operators channel=stable

# Install from GA catalog (fast channel for early GA releases)
/rhoai-install catalog=redhat-operators channel=fast

# Install from GA catalog with default stable channel
/rhoai-install catalog=redhat-operators
```

Or simply ask:
- "Install RHOAI from dev catalog"
- "Install RHOAI from production catalog"
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
