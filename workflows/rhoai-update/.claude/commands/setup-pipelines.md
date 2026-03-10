# /setup-pipelines - Setup AI Pipeline Server on RHOAI

Deploy and configure a Data Science Pipelines server on Red Hat OpenShift AI with MinIO object storage.

## Command Usage

- `/setup-pipelines` - Setup AI pipeline server with default configuration
- `/setup-pipelines verify` - Verify existing installation without making changes

## When to Use This Command

This command is triggered when the user runs:
- `/setup-pipelines` - Setup the AI pipeline server infrastructure
- Or when asked to "setup pipeline server", "configure pipelines", "deploy data science pipelines", etc.

## Prerequisites

**Required:**
1. Red Hat OpenShift AI (RHOAI) installed on the cluster
2. Logged into OpenShift cluster with admin permissions (use `/oc-login`)
3. `oc` CLI installed and configured

**Recommended:**
- At least 50Gi of available storage for pipeline artifacts
- Cluster has default StorageClass configured (e.g., `gp3-csi` for AWS)

## What This Command Sets Up

1. **minio-storage namespace**: MinIO S3-compatible object storage
2. **ai-pipelines namespace**: Data Science Pipelines Application server
3. **Object Storage**: MinIO with two buckets (`pipelines` and `data`)
4. **Pipeline Server**: Complete DSP infrastructure with MariaDB database
5. **Data Connections**: S3 connection secrets for RHOAI dashboard integration

## How It Works

### Step 1: Check Prerequisites

First, verify prerequisites and check if components already exist:

```bash
# Check if logged into cluster
if ! oc whoami &> /dev/null; then
  echo "❌ Not logged into OpenShift cluster. Run /oc-login first."
  exit 1
fi

# Check if RHOAI is installed
if ! oc get crd datasciencepipelinesapplications.datasciencepipelinesapplications.opendatahub.io &> /dev/null; then
  echo "❌ RHOAI is not installed on this cluster"
  exit 1
fi

echo "✅ Prerequisites verified"
```

**If prerequisites fail:**
- Report which requirement is missing
- Provide guidance on how to resolve (e.g., "Run /oc-login first")
- Do not proceed

### Step 2: Create Namespaces

Create the required namespaces if they don't exist:

```bash
# Create minio-storage namespace
if oc get namespace minio-storage &> /dev/null; then
  echo "ℹ️  Namespace minio-storage already exists"
else
  oc create namespace minio-storage
  oc label namespace minio-storage opendatahub.io/dashboard=true
  echo "✅ Created namespace: minio-storage"
fi

# Create ai-pipelines namespace
if oc get namespace ai-pipelines &> /dev/null; then
  echo "ℹ️  Namespace ai-pipelines already exists"
else
  oc create namespace ai-pipelines
  oc label namespace ai-pipelines opendatahub.io/dashboard=true
  echo "✅ Created namespace: ai-pipelines"
fi
```

**Key points:**
- Check if namespace exists before creating
- Label with `opendatahub.io/dashboard=true` for RHOAI dashboard visibility
- Report status for each namespace

### Step 3: Deploy MinIO Object Storage

Deploy MinIO in the `minio-storage` namespace:

```bash
# Generate random MinIO credentials if not already set
if ! oc get secret minio-secret -n minio-storage &> /dev/null 2>&1; then
  echo "🔐 Generating MinIO credentials..."
  MINIO_ACCESS_KEY="minio"
  MINIO_SECRET_KEY=$(openssl rand -base64 16 | tr -d '\n')
  echo "Generated MinIO credentials (save these):"
  echo "  Access Key: $MINIO_ACCESS_KEY"
  echo "  Secret Key: $MINIO_SECRET_KEY"
else
  echo "ℹ️  MinIO secret already exists, retrieving credentials..."
  MINIO_ACCESS_KEY=$(oc get secret minio-secret -n minio-storage -o jsonpath='{.data.accesskey}' | base64 -d)
  MINIO_SECRET_KEY=$(oc get secret minio-secret -n minio-storage -o jsonpath='{.data.secretkey}' | base64 -d)
fi

# Create MinIO deployment
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: minio-secret
  namespace: minio-storage
type: Opaque
stringData:
  accesskey: \${MINIO_ACCESS_KEY}
  secretkey: \${MINIO_SECRET_KEY}
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-pvc
  namespace: minio-storage
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: gp3-csi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: minio-storage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: quay.io/minio/minio:latest
        command:
        - /bin/bash
        - -c
        args:
        - minio server /data --console-address :9001
        env:
        - name: MINIO_ROOT_USER
          valueFrom:
            secretKeyRef:
              name: minio-secret
              key: accesskey
        - name: MINIO_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: minio-secret
              key: secretkey
        ports:
        - containerPort: 9000
          name: api
        - containerPort: 9001
          name: console
        volumeMounts:
        - name: storage
          mountPath: /data
        livenessProbe:
          httpGet:
            path: /minio/health/live
            port: 9000
          initialDelaySeconds: 30
          periodSeconds: 20
        readinessProbe:
          httpGet:
            path: /minio/health/ready
            port: 9000
          initialDelaySeconds: 10
          periodSeconds: 10
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: minio-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: minio-storage
spec:
  type: ClusterIP
  ports:
  - port: 9000
    targetPort: 9000
    protocol: TCP
    name: api
  - port: 9001
    targetPort: 9001
    protocol: TCP
    name: console
  selector:
    app: minio
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: minio-console
  namespace: minio-storage
spec:
  port:
    targetPort: console
  to:
    kind: Service
    name: minio
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
EOF
```

**What happens:**
- Checks if MinIO deployment already exists
- Generates random credentials or retrieves existing from secret
- Creates secret with generated credentials
- Creates 20Gi PVC for storage
- Deploys MinIO server
- Exposes MinIO console via OpenShift route

### Step 4: Wait for MinIO to be Ready

Wait for MinIO pod to be running:

```bash
echo "Waiting for MinIO to be ready..."
for i in {1..20}; do
  STATUS=$(oc get deployment minio -n minio-storage -o jsonpath='{.status.readyReplicas}' 2>/dev/null || echo "0")
  if [ "$STATUS" = "1" ]; then
    echo "✅ MinIO is ready"
    break
  fi
  echo "Check $i/20: MinIO not ready yet..."
  sleep 5
done

# Verify MinIO is actually running
if ! oc get deployment minio -n minio-storage -o jsonpath='{.status.readyReplicas}' | grep -q "1"; then
  echo "❌ MinIO failed to start. Check pod logs:"
  oc get pods -n minio-storage -l app=minio
  exit 1
fi
```

**Timeout handling:**
- Wait up to 100 seconds for MinIO to be ready
- If timeout, show pod status and logs
- Do not proceed if MinIO is not ready

### Step 5: Create S3 Buckets in MinIO

Create required S3 buckets:

```bash
# Create buckets for pipelines and data
oc exec -n minio-storage deployment/minio -- sh -c "
  mc alias set local http://localhost:9000 \$MINIO_ROOT_USER \$MINIO_ROOT_PASSWORD && \
  mc mb local/pipelines --ignore-existing && \
  mc mb local/data --ignore-existing && \
  mc ls local/
"
```

**Buckets created:**
- `pipelines` - For pipeline run artifacts and results
- `data` - For training datasets and ML data

**If buckets already exist:**
- `--ignore-existing` flag prevents errors
- Report that buckets already exist and are ready

### Step 6: Deploy Data Science Pipelines Application

Create data connections and pipeline server in `ai-pipelines` namespace:

```yaml
# Create S3 data connections
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: pipeline-artifacts
  namespace: ai-pipelines
  labels:
    opendatahub.io/dashboard: "true"
    opendatahub.io/managed: "true"
  annotations:
    opendatahub.io/connection-type: "s3"
    openshift.io/display-name: "Pipeline Artifacts Storage"
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: \${MINIO_ACCESS_KEY}
  AWS_SECRET_ACCESS_KEY: \${MINIO_SECRET_KEY}
  AWS_S3_ENDPOINT: http://minio.minio-storage.svc.cluster.local:9000
  AWS_S3_BUCKET: pipelines
  AWS_DEFAULT_REGION: us-east-1
---
apiVersion: v1
kind: Secret
metadata:
  name: pipeline-data
  namespace: ai-pipelines
  labels:
    opendatahub.io/dashboard: "true"
    opendatahub.io/managed: "true"
  annotations:
    opendatahub.io/connection-type: "s3"
    openshift.io/display-name: "Pipeline Training Data"
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: \${MINIO_ACCESS_KEY}
  AWS_SECRET_ACCESS_KEY: \${MINIO_SECRET_KEY}
  AWS_S3_ENDPOINT: http://minio.minio-storage.svc.cluster.local:9000
  AWS_S3_BUCKET: data
  AWS_DEFAULT_REGION: us-east-1
---
apiVersion: datasciencepipelinesapplications.opendatahub.io/v1
kind: DataSciencePipelinesApplication
metadata:
  name: pipelines
  namespace: ai-pipelines
spec:
  apiServer:
    deploy: true
    enableSamplePipeline: false
    argoLauncherImage: "quay.io/modh/argoexec@sha256:0edc3c5c9ba3c6f665d9c1aeddfed4e6dada66d891d4fe8e2ee6b6c37d13f02c"
    argoDriverImage: "quay.io/modh/argoexec@sha256:0edc3c5c9ba3c6f665d9c1aeddfed4e6dada66d891d4fe8e2ee6b6c37d13f02c"
  database:
    mariaDB:
      deploy: true
      pipelineDBName: mlpipeline
      pvcSize: 10Gi
  objectStorage:
    externalStorage:
      bucket: pipelines
      host: minio.minio-storage.svc.cluster.local:9000
      port: ""
      region: us-east-1
      s3CredentialsSecret:
        accessKey: AWS_ACCESS_KEY_ID
        secretKey: AWS_SECRET_ACCESS_KEY
        secretName: pipeline-artifacts
      scheme: http
  persistenceAgent:
    deploy: true
  scheduledWorkflow:
    deploy: true
EOF
```

**Components deployed:**
- Data connections (secrets) for S3 access
- DataSciencePipelinesApplication custom resource
- Embedded MariaDB database (10Gi)
- API Server with OAuth enabled
- Persistence Agent for artifact tracking
- Scheduled Workflow controller
- Workflow Controller (Argo Workflows)

### Step 7: Wait for Pipeline Server to be Ready

Monitor the pipeline server deployment:

```bash
echo "Waiting for pipeline server components to be ready..."
for i in {1..30}; do
  echo "Check $i/30:"
  oc get pods -n ai-pipelines -l component=data-science-pipelines 2>/dev/null | grep -v "Completed" || echo "No pods yet..."

  READY=$(oc get dspa pipelines -n ai-pipelines -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null)
  if [ "$READY" = "True" ]; then
    echo "✅ Pipeline server is ready!"
    break
  fi
  sleep 10
done

# Verify final status
READY=$(oc get dspa pipelines -n ai-pipelines -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null)
if [ "$READY" != "True" ]; then
  echo "❌ Pipeline server failed to become ready"
  echo "Checking component status:"
  oc get dspa pipelines -n ai-pipelines -o yaml | grep -A 20 "status:"
  exit 1
fi
```

**What's being monitored:**
- All 7 pipeline component pods (API server, persistence agent, workflow controller, etc.)
- MariaDB database readiness
- Overall DataSciencePipelinesApplication status

**Timeout:** 5 minutes (30 checks × 10 seconds)

### Step 8: Display Setup Summary

Provide complete information about the deployed infrastructure:

```bash
echo ""
echo "═══════════════════════════════════════════════════"
echo "✅ AI Pipeline Server Setup Complete"
echo "═══════════════════════════════════════════════════"
echo ""
echo "📦 MinIO Object Storage (minio-storage namespace):"
echo "  - Deployment: $(oc get deployment minio -n minio-storage -o jsonpath='{.status.readyReplicas}')/1 ready"
echo "  - Storage: 20Gi PVC"
echo "  - Buckets: pipelines, data"
echo "  - Access Key: $MINIO_ACCESS_KEY"
echo "  - Secret Key: $MINIO_SECRET_KEY"
MINIO_CONSOLE=$(oc get route minio-console -n minio-storage -o jsonpath='{.spec.host}')
echo "  - Console: https://$MINIO_CONSOLE"
echo ""
echo "🔄 Pipeline Server (ai-pipelines namespace):"
oc get dspa pipelines -n ai-pipelines
echo ""
PIPELINE_API=$(oc get route ds-pipeline-pipelines -n ai-pipelines -o jsonpath='{.spec.host}' 2>/dev/null || echo "Not found")
echo "  - API Endpoint: https://$PIPELINE_API"
echo "  - Database: MariaDB (10Gi embedded)"
echo "  - Components:"
oc get pods -n ai-pipelines -l component=data-science-pipelines --no-headers | wc -l | xargs echo "    - Pipeline pods:"
echo ""
echo "🔗 Data Connections:"
echo "  - pipeline-artifacts (S3 connection for pipeline results)"
echo "  - pipeline-data (S3 connection for training data)"
echo ""
echo "📊 Access from RHOAI Dashboard:"
echo "  1. Navigate to Data Science Projects"
echo "  2. Select 'ai-pipelines' project"
echo "  3. Go to 'Pipelines' tab"
echo "  4. You can now import and run pipelines"
echo ""
echo "🔍 Verify Installation:"
echo "  oc get dspa pipelines -n ai-pipelines"
echo "  oc get pods -n ai-pipelines"
echo "  oc get pods -n minio-storage"
echo ""
```

## Handling Different Scenarios

### Scenario A: Fresh Installation

**Situation:** No components exist

**Actions:**
1. Create both namespaces
2. Deploy MinIO completely
3. Deploy pipeline server from scratch
4. Report: "✅ AI Pipeline Server setup complete (fresh installation)"

### Scenario B: Partial Installation (MinIO exists)

**Situation:** MinIO already deployed but no pipeline server

**Actions:**
1. Skip MinIO deployment
2. Verify MinIO is healthy
3. Verify buckets exist (create if missing)
4. Deploy pipeline server
5. Report: "✅ AI Pipeline Server setup complete (using existing MinIO)"

### Scenario C: Partial Installation (Namespaces exist)

**Situation:** Namespaces exist but deployments don't

**Actions:**
1. Use existing namespaces
2. Deploy MinIO if not present
3. Deploy pipeline server if not present
4. Report: "✅ AI Pipeline Server setup complete (using existing namespaces)"

### Scenario D: Complete Installation Exists

**Situation:** Everything already deployed

**Actions:**
1. Verify all components are healthy
2. Check pipeline server status
3. Verify MinIO connectivity
4. Report: "ℹ️  AI Pipeline Server already exists and is healthy"
5. Display current status and endpoints

### Scenario E: Failed Previous Installation

**Situation:** Resources exist but in failed state

**Actions:**
1. Detect failed components
2. Report current status
3. Ask user: "Previous installation found in failed state. Delete and reinstall? (yes/no)"
4. If yes: Delete DSPA, wait for cleanup, redeploy
5. If no: Exit and show troubleshooting steps

### Scenario F: Verify Mode

**Situation:** User runs `/setup-pipelines verify`

**Actions:**
1. Check if all components exist
2. Verify health status
3. Test connectivity between components
4. Report detailed status
5. Do not make any changes

## Verification and Health Checks

After setup, verify the installation:

```bash
# Check pipeline server status
READY_STATUS=$(oc get dspa pipelines -n ai-pipelines -o jsonpath='{.status.conditions[?(@.type=="Ready")]}' | jq .)

# Check object storage connectivity
STORAGE_STATUS=$(oc get dspa pipelines -n ai-pipelines -o jsonpath='{.status.conditions[?(@.type=="ObjectStoreAvailable")]}' | jq .)

# Check database connectivity
DB_STATUS=$(oc get dspa pipelines -n ai-pipelines -o jsonpath='{.status.conditions[?(@.type=="DatabaseAvailable")]}' | jq .)

# Test MinIO access
oc exec -n minio-storage deployment/minio -- mc ls local/

# Test pipeline API
PIPELINE_URL="https://$(oc get route ds-pipeline-pipelines -n ai-pipelines -o jsonpath='{.spec.host}')"
curl -sk "$PIPELINE_URL/apis/v2beta1/healthz"
```

**Health criteria:**
- ✅ Pipeline server Ready condition is True
- ✅ ObjectStore connection verified
- ✅ Database connection verified
- ✅ All 7 pipeline component pods running
- ✅ MinIO pod running and accessible
- ✅ Pipeline API responds to health check

## Troubleshooting Common Issues

### Issue 1: MinIO Pod Stuck in ContainerCreating

**Cause:** PVC not bound due to missing StorageClass or insufficient storage

**Solution:**
```bash
# Check PVC status
oc get pvc minio-pvc -n minio-storage

# Check available StorageClasses
oc get storageclass

# If gp3-csi doesn't exist, update the YAML to use available StorageClass
```

### Issue 2: Pipeline Server Database Not Ready

**Cause:** MariaDB pod failing to start

**Solution:**
```bash
# Check MariaDB pod
oc get pods -n ai-pipelines | grep mariadb

# Check MariaDB logs
oc logs -n ai-pipelines deployment/mariadb-pipelines

# Common fix: Delete and recreate DSPA
oc delete dspa pipelines -n ai-pipelines
# Wait 30 seconds, then reapply DSPA YAML
```

### Issue 3: Object Store Connection Failed

**Cause:** MinIO service not reachable from pipeline server

**Solution:**
```bash
# Verify MinIO service exists
oc get svc minio -n minio-storage

# Test connectivity from pipeline namespace
oc run test -n ai-pipelines --image=curlimages/curl --rm -it --restart=Never -- curl -v http://minio.minio-storage.svc.cluster.local:9000/minio/health/live
```

### Issue 4: Pipeline Server Components in CrashLoopBackOff

**Cause:** Configuration error or resource constraints

**Solution:**
```bash
# Check pod logs
oc logs -n ai-pipelines deployment/ds-pipeline-pipelines

# Check resource availability
oc describe nodes | grep -A 5 "Allocated resources"

# Delete and recreate with adjusted resources if needed
```

## Security Considerations

1. **MinIO Credentials**
   - Randomly generated on first installation
   - Credentials stored in Secret `minio-secret` in namespace `minio-storage`
   - Retrieved from secret if already exists (idempotent)
   - Access key: `minio` (static)
   - Secret key: Randomly generated 16-byte base64 string

2. **Network Access**
   - MinIO API: ClusterIP (internal only)
   - MinIO Console: Route with TLS (external access)
   - Pipeline API: Route with OAuth (external access via RHOAI dashboard)

3. **Storage**
   - MinIO PVC: 20Gi (adjust based on expected artifact volume)
   - MariaDB PVC: 10Gi (stores pipeline metadata)
   - Both use default StorageClass

4. **Namespace Isolation**
   - `minio-storage`: Only MinIO components
   - `ai-pipelines`: Only pipeline server components
   - Models/workloads in separate namespaces can access via service URLs

## Integration with Other Services

### Accessing Models from Pipelines

Pipelines can access models deployed in other namespaces:

```python
# In your pipeline code
llm_url = "http://llama-predictor.llm-models.svc.cluster.local:80"
embedding_url = "https://granite-embedding.llm-models.svc.cluster.local:8443"
llamastack_url = "http://llama-stack.llm-models.svc.cluster.local:8321"
```

### Using External S3 Instead of MinIO

To use AWS S3 or other external storage, update data connection secrets:

```yaml
stringData:
  AWS_ACCESS_KEY_ID: <your-aws-key>
  AWS_SECRET_ACCESS_KEY: <your-aws-secret>
  AWS_S3_ENDPOINT: https://s3.amazonaws.com
  AWS_S3_BUCKET: my-pipeline-bucket
  AWS_DEFAULT_REGION: us-east-1
```

## Success Criteria

The setup is successful when:
- ✅ `minio-storage` namespace exists with label `opendatahub.io/dashboard=true`
- ✅ `ai-pipelines` namespace exists with label `opendatahub.io/dashboard=true`
- ✅ MinIO deployment running (1/1 ready)
- ✅ MinIO buckets `pipelines` and `data` exist
- ✅ DataSciencePipelinesApplication `pipelines` has Ready=True status
- ✅ All 7 pipeline component pods running
- ✅ Pipeline API accessible and responds to health check
- ✅ Data connections visible in RHOAI dashboard

## Output Format

Always provide:
1. **Setup Progress** - What's being deployed/verified at each step
2. **Component Status** - Health of each deployed component
3. **Access Information** - URLs and credentials for accessing services
4. **Next Steps** - How to use the pipeline server from RHOAI dashboard
5. **Troubleshooting** - Any warnings or issues detected

Keep the user informed throughout the multi-step deployment process.
