# SPIRE Federation with PostgreSQL Test Guide

## 📋 Test Overview

This document provides step-by-step instructions to test SPIRE-to-SPIRE federation between two OpenShift clusters using **PostgreSQL 15** as the backend datastore and **https_web** federation profile with ACME/Let's Encrypt certificates.

---

## 🎯 Test Objectives

| Objective | Description |
|-----------|-------------|
| **PostgreSQL Datastore** | Verify SPIRE Server can use PostgreSQL as persistent storage |
| **Federation Setup** | Establish trust between two separate SPIRE deployments |
| **Workload Identity** | Verify workloads receive SPIFFE identities (SVIDs) |
| **Resilience** | Verify data persists across PostgreSQL restarts |

---

## 📦 Prerequisites

### Required Components
- Two OpenShift clusters (any cloud provider: AWS, GCP, Azure, etc.)
- `oc` CLI tool installed and configured
- Zero Trust Workload Identity Manager Operator installed on both clusters
- Network connectivity between clusters (federation endpoints must be reachable)

### Cluster Requirements
| Requirement | Details |
|-------------|---------|
| OpenShift Version | 4.20 |
| Storage | Default StorageClass for PVC provisioning |

---

## 🔧 Test Environment Variables

Set these variables before starting. Replace with your actual cluster domains:

```bash
# Cluster 1 Configuration
export CLUSTER1_KUBECONFIG="/path/to/cluster1/kubeconfig"
export CLUSTER1_DOMAIN="apps.<cluster1-name>.<domain>"

# Cluster 2 Configuration  
export CLUSTER2_KUBECONFIG="/path/to/cluster2/kubeconfig"
export CLUSTER2_DOMAIN="apps.<cluster2-name>.<domain>"

# Example:
# export CLUSTER1_DOMAIN="apps.sayadas-aws-1.qe.devcluster.openshift.com"
# export CLUSTER2_DOMAIN="apps.ci-ln-3ml0mvb-72292.gcp-2.ci.openshift.org"
```

---

## 📝 Test Execution Steps

---

### STEP 1: Verify Operator Installation

**Purpose:** Confirm the Zero Trust Workload Identity Manager Operator is installed on both clusters.

#### Cluster 1:
```bash
export KUBECONFIG=$CLUSTER1_KUBECONFIG

echo "=== Verifying Operator on Cluster 1 ==="
oc get pods -n zero-trust-workload-identity-manager
```

#### Cluster 2:
```bash
export KUBECONFIG=$CLUSTER2_KUBECONFIG

echo "=== Verifying Operator on Cluster 2 ==="
oc get pods -n zero-trust-workload-identity-manager
```

#### ✅ Expected Result:
```
NAME                                                              READY   STATUS    RESTARTS   AGE
zero-trust-workload-identity-manager-controller-manager-xxxxx     1/1     Running   0          XXm
```

---

### STEP 2: Clean Up Existing SPIRE Configuration (If Any)

**Purpose:** Remove any existing SPIRE configuration to start fresh.

#### Cluster 1:
```bash
export KUBECONFIG=$CLUSTER1_KUBECONFIG

echo "=== Cleaning up Cluster 1 ==="
oc delete spireagent cluster --ignore-not-found
oc delete spireserver cluster --ignore-not-found
oc delete clusterfederatedtrustdomain --all --ignore-not-found
sleep 30
oc get pods -n zero-trust-workload-identity-manager
echo "✅ Cluster 1: Cleanup complete"
```

#### Cluster 2:
```bash
export KUBECONFIG=$CLUSTER2_KUBECONFIG

echo "=== Cleaning up Cluster 2 ==="
oc delete spireagent cluster --ignore-not-found
oc delete spireserver cluster --ignore-not-found
oc delete clusterfederatedtrustdomain --all --ignore-not-found
sleep 30
oc get pods -n zero-trust-workload-identity-manager
echo "✅ Cluster 2: Cleanup complete"
```

#### ✅ Expected Result:
Only the controller-manager pod should remain running.

---

### STEP 3: Deploy PostgreSQL Database

**Purpose:** Deploy PostgreSQL 15 as the SPIRE Server datastore.

#### Cluster 1:
```bash
export KUBECONFIG=$CLUSTER1_KUBECONFIG

echo "=== Deploying PostgreSQL on Cluster 1 ==="

# Create postgres namespace
oc create namespace postgres --dry-run=client -o yaml | oc apply -f -

# Deploy PostgreSQL with PVC
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgresql-pvc
  namespace: postgres
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql
  namespace: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
      - name: postgresql
        image: registry.redhat.io/rhel9/postgresql-15:latest
        env:
        - name: POSTGRESQL_USER
          value: "spire"
        - name: POSTGRESQL_PASSWORD
          value: "spirepassword"
        - name: POSTGRESQL_DATABASE
          value: "spire"
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/pgsql/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: postgresql-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgresql
  namespace: postgres
spec:
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: postgresql
EOF

# Wait for PostgreSQL to be ready
echo "Waiting for PostgreSQL to be ready..."
oc wait --for=condition=available deployment/postgresql -n postgres --timeout=180s

# Verify PostgreSQL
oc get pods -n postgres
echo "✅ Cluster 1: PostgreSQL deployed"
```

#### Cluster 2:
```bash
export KUBECONFIG=$CLUSTER2_KUBECONFIG

echo "=== Deploying PostgreSQL on Cluster 2 ==="

# Create postgres namespace
oc create namespace postgres --dry-run=client -o yaml | oc apply -f -

# Deploy PostgreSQL with PVC
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgresql-pvc
  namespace: postgres
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql
  namespace: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
      - name: postgresql
        image: registry.redhat.io/rhel9/postgresql-15:latest
        env:
        - name: POSTGRESQL_USER
          value: "spire"
        - name: POSTGRESQL_PASSWORD
          value: "spirepassword"
        - name: POSTGRESQL_DATABASE
          value: "spire"
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/pgsql/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: postgresql-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgresql
  namespace: postgres
spec:
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: postgresql
EOF

# Wait for PostgreSQL to be ready
echo "Waiting for PostgreSQL to be ready..."
oc wait --for=condition=available deployment/postgresql -n postgres --timeout=180s

# Verify PostgreSQL
oc get pods -n postgres
echo "✅ Cluster 2: PostgreSQL deployed"
```

#### ✅ Expected Result:
```
NAME                          READY   STATUS    RESTARTS   AGE
postgresql-xxxxxxxxx-xxxxx    1/1     Running   0          XXs
✅ Cluster X: PostgreSQL deployed
```

---

### STEP 4: Deploy SPIRE Server with PostgreSQL and Federation

**Purpose:** Deploy SpireServer CR with PostgreSQL datastore and https_web federation.

#### Cluster 1:
```bash
export KUBECONFIG=$CLUSTER1_KUBECONFIG

echo "=== Deploying SPIRE Server on Cluster 1 ==="

cat <<EOF | oc apply -f -
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: ${CLUSTER1_DOMAIN}
    country: "US"
    organization: "RedHat"
  persistence:
    type: pvc
    size: "1Gi"
    accessMode: ReadWriteOnce
  datastore:
    databaseType: postgres
    connectionString: "host=postgresql.postgres.svc.cluster.local port=5432 user=spire password=spirepassword dbname=spire sslmode=disable"
    maxOpenConns: 100
    maxIdleConns: 10
    connMaxLifetime: 3600
  jwtIssuer: https://oidc-discovery.${CLUSTER1_DOMAIN}
  federation:
    bundleEndpoint:
      profile: https_web
      httpsWeb:
        acme:
          directoryUrl: "https://acme-v02.api.letsencrypt.org/directory"
          domainName: "federation.${CLUSTER1_DOMAIN}"
          email: "admin@redhat.com"
          tosAccepted: "true"
      refreshHint: 300
    managedRoute: "true"
    federatesWith:
    - trustDomain: ${CLUSTER2_DOMAIN}
      bundleEndpointUrl: https://federation.${CLUSTER2_DOMAIN}
      bundleEndpointProfile: https_web
EOF

echo "Waiting for SPIRE Server (90s)..."
sleep 90

oc get pods -n zero-trust-workload-identity-manager | grep spire-server
oc get spireserver cluster
echo "✅ Cluster 1: SPIRE Server deployed"
```

#### Cluster 2:
```bash
export KUBECONFIG=$CLUSTER2_KUBECONFIG

echo "=== Deploying SPIRE Server on Cluster 2 ==="

cat <<EOF | oc apply -f -
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: ${CLUSTER2_DOMAIN}
    country: "US"
    organization: "RedHat"
  persistence:
    type: pvc
    size: "1Gi"
    accessMode: ReadWriteOnce
  datastore:
    databaseType: postgres
    connectionString: "host=postgresql.postgres.svc.cluster.local port=5432 user=spire password=spirepassword dbname=spire sslmode=disable"
    maxOpenConns: 100
    maxIdleConns: 10
    connMaxLifetime: 3600
  jwtIssuer: https://oidc-discovery.${CLUSTER2_DOMAIN}
  federation:
    bundleEndpoint:
      profile: https_web
      httpsWeb:
        acme:
          directoryUrl: "https://acme-v02.api.letsencrypt.org/directory"
          domainName: "federation.${CLUSTER2_DOMAIN}"
          email: "admin@redhat.com"
          tosAccepted: "true"
      refreshHint: 300
    managedRoute: "true"
    federatesWith:
    - trustDomain: ${CLUSTER1_DOMAIN}
      bundleEndpointUrl: https://federation.${CLUSTER1_DOMAIN}
      bundleEndpointProfile: https_web
EOF

echo "Waiting for SPIRE Server (90s)..."
sleep 90

oc get pods -n zero-trust-workload-identity-manager | grep spire-server
oc get spireserver cluster
echo "✅ Cluster 2: SPIRE Server deployed"
```

#### ✅ Expected Result:
```
spire-server-0    2/2     Running   0          XXs
NAME      AGE
cluster   XXs
✅ Cluster X: SPIRE Server deployed
```

---

### STEP 5: Deploy SPIRE Agent

**Purpose:** Deploy SpireAgent CR with Kubernetes attestors.

#### Cluster 1:
```bash
export KUBECONFIG=$CLUSTER1_KUBECONFIG

echo "=== Deploying SPIRE Agent on Cluster 1 ==="

cat <<EOF | oc apply -f -
apiVersion: operator.openshift.io/v1alpha1
kind: SpireAgent
metadata:
  name: cluster
spec:
  nodeAttestor:
    k8sPSATEnabled: "true"
  workloadAttestors:
    k8sEnabled: "true"
    workloadAttestorsVerification:
      type: "auto"
EOF

echo "Waiting for SPIRE Agent (60s)..."
sleep 60

oc get pods -n zero-trust-workload-identity-manager | grep spire-agent
oc get spireagent cluster
echo "✅ Cluster 1: SPIRE Agent deployed"
```

#### Cluster 2:
```bash
export KUBECONFIG=$CLUSTER2_KUBECONFIG

echo "=== Deploying SPIRE Agent on Cluster 2 ==="

cat <<EOF | oc apply -f -
apiVersion: operator.openshift.io/v1alpha1
kind: SpireAgent
metadata:
  name: cluster
spec:
  nodeAttestor:
    k8sPSATEnabled: "true"
  workloadAttestors:
    k8sEnabled: "true"
    workloadAttestorsVerification:
      type: "auto"
EOF

echo "Waiting for SPIRE Agent (60s)..."
sleep 60

oc get pods -n zero-trust-workload-identity-manager | grep spire-agent
oc get spireagent cluster
echo "✅ Cluster 2: SPIRE Agent deployed"
```

#### ✅ Expected Result:
```
spire-agent-xxxxx   1/1     Running   0          XXs
spire-agent-xxxxx   1/1     Running   0          XXs
spire-agent-xxxxx   1/1     Running   0          XXs
NAME      AGE
cluster   XXs
✅ Cluster X: SPIRE Agent deployed
```

---

### STEP 6: Verify Federation is Working

**Purpose:** Confirm that trust bundles are synchronized between clusters.

#### Cluster 1:
```bash
export KUBECONFIG=$CLUSTER1_KUBECONFIG

echo "=== Verifying Federation on Cluster 1 ==="

echo "--- Check Federation Route ---"
oc get route -n zero-trust-workload-identity-manager | grep federation

echo ""
echo "--- Check Trust Bundles in PostgreSQL ---"
oc exec -it deploy/postgresql -n postgres -- psql -U spire -d spire -c "SELECT trust_domain FROM bundles;"

echo ""
echo "--- Check SPIRE Server Logs for Bundle Refresh ---"
oc logs spire-server-0 -n zero-trust-workload-identity-manager -c spire-server --tail=20 | grep -i "bundle"

echo "✅ Cluster 1: Federation verification complete"
```

#### Cluster 2:
```bash
export KUBECONFIG=$CLUSTER2_KUBECONFIG

echo "=== Verifying Federation on Cluster 2 ==="

echo "--- Check Federation Route ---"
oc get route -n zero-trust-workload-identity-manager | grep federation

echo ""
echo "--- Check Trust Bundles in PostgreSQL ---"
oc exec -it deploy/postgresql -n postgres -- psql -U spire -d spire -c "SELECT trust_domain FROM bundles;"

echo ""
echo "--- Check SPIRE Server Logs for Bundle Refresh ---"
oc logs spire-server-0 -n zero-trust-workload-identity-manager -c spire-server --tail=20 | grep -i "bundle"

echo "✅ Cluster 2: Federation verification complete"
```

#### ✅ Expected Result:

**Route should exist:**
```
spire-server-federation   federation.apps.<domain>   spire-server   federation   passthrough/Redirect   None
```

**PostgreSQL should show TWO trust domains:**
```
                       trust_domain                       
----------------------------------------------------------
 spiffe://apps.<cluster1-domain>    ← Local
 spiffe://apps.<cluster2-domain>    ← Federated!
(2 rows)
```

**Logs should show bundle refresh:**
```
level=info msg="Bundle refreshed" trust_domain=apps.<remote-cluster>
```

---

### STEP 7: Verify PostgreSQL Tables

**Purpose:** Confirm SPIRE created all required database tables.

#### On Either Cluster:
```bash
export KUBECONFIG=$CLUSTER1_KUBECONFIG

echo "=== Checking PostgreSQL Tables ==="
oc exec -it deploy/postgresql -n postgres -- psql -U spire -d spire -c "\dt"
```

#### ✅ Expected Result:
```
 Schema |              Name               | Type  | Owner 
--------+---------------------------------+-------+-------
 public | attested_node_entries           | table | spire
 public | attested_node_entries_events    | table | spire
 public | bundles                         | table | spire
 public | ca_journals                     | table | spire
 public | dns_names                       | table | spire
 public | federated_registration_entries  | table | spire
 public | federated_trust_domains         | table | spire
 public | join_tokens                     | table | spire
 public | migrations                      | table | spire
 public | node_resolver_map_entries       | table | spire
 public | registered_entries              | table | spire
 public | registered_entries_events       | table | spire
 public | selectors                       | table | spire
(13 rows)
```

---

### STEP 8: Deploy Test Workloads

**Purpose:** Deploy test workloads that receive SPIFFE identities.

#### Cluster 1:
```bash
export KUBECONFIG=$CLUSTER1_KUBECONFIG

echo "=== Deploying Test Workload on Cluster 1 ==="

# Create test namespace
oc create namespace spire-test --dry-run=client -o yaml | oc apply -f -

# Create ClusterSPIFFEID
cat <<EOF | oc apply -f -
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: test-workload
spec:
  spiffeIDTemplate: "spiffe://${CLUSTER1_DOMAIN}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  podSelector:
    matchLabels:
      spiffe.io/spire-managed-identity: "true"
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: spire-test
  federatesWith:
    - "spiffe://${CLUSTER2_DOMAIN}"
EOF

# Deploy test pod
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: test-workload
  namespace: spire-test
---
apiVersion: v1
kind: Pod
metadata:
  name: test-client
  namespace: spire-test
  labels:
    app: test-client
    spiffe.io/spire-managed-identity: "true"
spec:
  serviceAccountName: test-workload
  containers:
  - name: test-client
    image: registry.access.redhat.com/ubi9/ubi:latest
    command: ["sleep", "infinity"]
    volumeMounts:
    - name: spiffe-workload-api
      mountPath: /spiffe-workload-api
      readOnly: true
  volumes:
  - name: spiffe-workload-api
    csi:
      driver: csi.spiffe.io
      readOnly: true
EOF

echo "Waiting for pod to be ready..."
sleep 30
oc get pods -n spire-test
echo "✅ Cluster 1: Test workload deployed"
```

#### Cluster 2:
```bash
export KUBECONFIG=$CLUSTER2_KUBECONFIG

echo "=== Deploying Test Workload on Cluster 2 ==="

# Create test namespace
oc create namespace spire-test --dry-run=client -o yaml | oc apply -f -

# Create ClusterSPIFFEID
cat <<EOF | oc apply -f -
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: test-workload
spec:
  spiffeIDTemplate: "spiffe://${CLUSTER2_DOMAIN}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  podSelector:
    matchLabels:
      spiffe.io/spire-managed-identity: "true"
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: spire-test
  federatesWith:
    - "spiffe://${CLUSTER1_DOMAIN}"
EOF

# Deploy test pod
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: test-workload
  namespace: spire-test
---
apiVersion: v1
kind: Pod
metadata:
  name: test-server
  namespace: spire-test
  labels:
    app: test-server
    spiffe.io/spire-managed-identity: "true"
spec:
  serviceAccountName: test-workload
  containers:
  - name: test-server
    image: registry.access.redhat.com/ubi9/ubi:latest
    command: ["sleep", "infinity"]
    volumeMounts:
    - name: spiffe-workload-api
      mountPath: /spiffe-workload-api
      readOnly: true
  volumes:
  - name: spiffe-workload-api
    csi:
      driver: csi.spiffe.io
      readOnly: true
EOF

echo "Waiting for pod to be ready..."
sleep 30
oc get pods -n spire-test
echo "✅ Cluster 2: Test workload deployed"
```

#### ✅ Expected Result:
```
NAME          READY   STATUS    RESTARTS   AGE
test-client   1/1     Running   0          XXs
✅ Cluster X: Test workload deployed
```

---

### STEP 9: Verify SVID Issuance

**Purpose:** Confirm workloads received X509-SVID certificates.

#### Cluster 1:
```bash
export KUBECONFIG=$CLUSTER1_KUBECONFIG

echo "=== Verifying SVID Issuance on Cluster 1 ==="

echo "--- Check SPIRE Agent Logs ---"
AGENT_POD=$(oc get pods -n zero-trust-workload-identity-manager -o name | grep spire-agent | head -1)
oc logs -n zero-trust-workload-identity-manager $AGENT_POD --tail=30 | grep -i "Creating X509-SVID\|spire-test"

echo ""
echo "--- Check Registered Entries in PostgreSQL ---"
oc exec -it deploy/postgresql -n postgres -- psql -U spire -d spire -c "SELECT spiffe_id FROM registered_entries WHERE spiffe_id LIKE '%spire-test%';"

echo "✅ Cluster 1: SVID verification complete"
```

#### Cluster 2:
```bash
export KUBECONFIG=$CLUSTER2_KUBECONFIG

echo "=== Verifying SVID Issuance on Cluster 2 ==="

echo "--- Check SPIRE Agent Logs ---"
AGENT_POD=$(oc get pods -n zero-trust-workload-identity-manager -o name | grep spire-agent | head -1)
oc logs -n zero-trust-workload-identity-manager $AGENT_POD --tail=30 | grep -i "Creating X509-SVID\|spire-test"

echo ""
echo "--- Check Registered Entries in PostgreSQL ---"
oc exec -it deploy/postgresql -n postgres -- psql -U spire -d spire -c "SELECT spiffe_id FROM registered_entries WHERE spiffe_id LIKE '%spire-test%';"

echo "✅ Cluster 2: SVID verification complete"
```

#### ✅ Expected Result:

**Agent logs should show SVID creation:**
```
level=info msg="Creating X509-SVID" spiffe_id="spiffe://apps.<domain>/ns/spire-test/sa/test-workload"
```

**PostgreSQL should show registered entry:**
```
                              spiffe_id                              
---------------------------------------------------------------------
 spiffe://apps.<domain>/ns/spire-test/sa/test-workload
(1 row)
```

---

### STEP 10: Test PostgreSQL Resilience

**Purpose:** Verify SPIRE data persists across PostgreSQL restarts.

#### Cluster 1:
```bash
export KUBECONFIG=$CLUSTER1_KUBECONFIG

echo "=== Testing PostgreSQL Resilience on Cluster 1 ==="

echo "--- Before Restart ---"
oc exec -it deploy/postgresql -n postgres -- psql -U spire -d spire -c "SELECT trust_domain FROM bundles;"
oc exec -it deploy/postgresql -n postgres -- psql -U spire -d spire -c "SELECT COUNT(*) as entry_count FROM registered_entries;"

echo ""
echo "--- Restarting PostgreSQL ---"
oc delete pod -n postgres -l app=postgresql
echo "Waiting for PostgreSQL to restart (45s)..."
sleep 45
oc wait --for=condition=available deployment/postgresql -n postgres --timeout=120s

echo ""
echo "--- After Restart ---"
oc exec -it deploy/postgresql -n postgres -- psql -U spire -d spire -c "SELECT trust_domain FROM bundles;"
oc exec -it deploy/postgresql -n postgres -- psql -U spire -d spire -c "SELECT COUNT(*) as entry_count FROM registered_entries;"

echo ""
echo "--- Check SPIRE Server Recovery ---"
oc logs spire-server-0 -n zero-trust-workload-identity-manager -c spire-server --tail=10 | grep -i "bundle"

echo "✅ Cluster 1: PostgreSQL resilience test complete"
```

#### ✅ Expected Result:

**Bundles should be preserved (same count before and after):**
```
                       trust_domain                       
----------------------------------------------------------
 spiffe://apps.<cluster1-domain>
 spiffe://apps.<cluster2-domain>
(2 rows)
```

**Entry count should be preserved:**
```
 entry_count 
-------------
           3
(1 row)
```

**SPIRE Server should recover and show bundle refresh:**
```
level=info msg="Bundle refreshed" trust_domain=apps.<remote-cluster>
```

---

## 📊 Test Results Summary Template

Use this template to record your test results:

| Test Step | Cluster 1 | Cluster 2 | Pass/Fail |
|-----------|-----------|-----------|-----------|
| Step 1: Operator Installed | | | |
| Step 2: Cleanup Complete | | | |
| Step 3: PostgreSQL Deployed | | | |
| Step 4: SPIRE Server Deployed | | | |
| Step 5: SPIRE Agent Deployed | | | |
| Step 6: Federation Working (2 bundles) | | | |
| Step 7: PostgreSQL Tables Created (13) | | | |
| Step 8: Test Workload Deployed | | | |
| Step 9: SVID Issued | | | |
| Step 10: PostgreSQL Resilience | | | |

---

## 🧹 Cleanup

To remove all test resources:

```bash
# Cluster 1
export KUBECONFIG=$CLUSTER1_KUBECONFIG
oc delete namespace spire-test --ignore-not-found
oc delete clusterspiffeid test-workload --ignore-not-found
oc delete spireagent cluster --ignore-not-found
oc delete spireserver cluster --ignore-not-found
oc delete namespace postgres --ignore-not-found

# Cluster 2
export KUBECONFIG=$CLUSTER2_KUBECONFIG
oc delete namespace spire-test --ignore-not-found
oc delete clusterspiffeid test-workload --ignore-not-found
oc delete spireagent cluster --ignore-not-found
oc delete spireserver cluster --ignore-not-found
oc delete namespace postgres --ignore-not-found
```

---

## 📚 Key Configuration Reference

### SpireServer CR Fields for PostgreSQL + Federation

| Field | Description | Example Value |
|-------|-------------|---------------|
| `spec.persistence.type` | Storage type for local data | `pvc` |
| `spec.persistence.size` | PVC size | `1Gi` |
| `spec.datastore.databaseType` | Database type | `postgres` |
| `spec.datastore.connectionString` | PostgreSQL connection | `host=postgresql.postgres.svc.cluster.local...` |
| `spec.federation.bundleEndpoint.profile` | Federation profile | `https_web` |
| `spec.federation.bundleEndpoint.httpsWeb.acme` | ACME configuration | See example above |
| `spec.federation.federatesWith` | Remote trust domains | Array of trust domain configs |

### SpireAgent CR Fields

| Field | Description | Example Value |
|-------|-------------|---------------|
| `spec.nodeAttestor.k8sPSATEnabled` | Use K8s PSAT for node attestation | `"true"` |
| `spec.workloadAttestors.k8sEnabled` | Use K8s for workload attestation | `"true"` |

---

## 🔍 Troubleshooting

### Common Issues

| Issue | Symptom | Solution |
|-------|---------|----------|
| SpireServer fails to create | `spec.persistence: Required value` | Add `persistence` block to SpireServer CR |
| SpireAgent crashes | `WorkloadAttestor constraint not satisfied` | Add `nodeAttestor` and `workloadAttestors` to SpireAgent CR |
| Federation not working | Only 1 bundle in PostgreSQL | Check federation route, verify ACME certificate provisioned |
| https_web fails | `httpsWeb is required` | Add `httpsWeb.acme` block to bundleEndpoint |

### Useful Debug Commands

```bash
# Check SPIRE Server logs
oc logs spire-server-0 -n zero-trust-workload-identity-manager -c spire-server --tail=50

# Check SPIRE Agent logs
oc logs -n zero-trust-workload-identity-manager -l app=spire-agent --tail=50

# Check Operator logs
oc logs -n zero-trust-workload-identity-manager deployment/zero-trust-workload-identity-manager-controller-manager --tail=50

# Check PostgreSQL connectivity
oc exec -it deploy/postgresql -n postgres -- psql -U spire -d spire -c "SELECT 1;"

# Check all SPIRE resources
oc get spireserver,spireagent,clusterspiffeid,clusterfederatedtrustdomain
```

---

## 🧪 Test Scenarios Executed with Results

The following test scenarios were executed on **January 12, 2026** with two OpenShift clusters:
- **Cluster 1:** AWS (`apps.sayadas-aws-1.qe.devcluster.openshift.com`)
- **Cluster 2:** GCP (`apps.ci-ln-3ml0mvb-72292.gcp-2.ci.openshift.org`)

---

### Test Scenario 1: PostgreSQL Datastore Configuration

| Test Case | Description | Expected Result | Actual Result | Status |
|-----------|-------------|-----------------|---------------|--------|
| TC-1.1 | Deploy PostgreSQL 15 with PVC | Pod running, PVC bound | Pod `postgresql-xxx` running (1/1), PVC bound | ✅ PASS |
| TC-1.2 | Configure SpireServer with PostgreSQL connection string | SpireServer CR created | SpireServer `cluster` created successfully | ✅ PASS |
| TC-1.3 | SPIRE creates database tables automatically | 13 tables created | 13 tables created (bundles, registered_entries, etc.) | ✅ PASS |
| TC-1.4 | SPIRE Server connects to PostgreSQL | No connection errors | Server running 2/2, logs show successful DB operations | ✅ PASS |

**Evidence - PostgreSQL Tables Created:**
```
 Schema |              Name               | Type  | Owner 
--------+---------------------------------+-------+-------
 public | attested_node_entries           | table | spire
 public | attested_node_entries_events    | table | spire
 public | bundles                         | table | spire
 public | ca_journals                     | table | spire
 public | dns_names                       | table | spire
 public | federated_registration_entries  | table | spire
 public | federated_trust_domains         | table | spire
 public | join_tokens                     | table | spire
 public | migrations                      | table | spire
 public | node_resolver_map_entries       | table | spire
 public | registered_entries              | table | spire
 public | registered_entries_events       | table | spire
 public | selectors                       | table | spire
(13 rows)
```

---

### Test Scenario 2: Federation Profile Testing

| Test Case | Description | Expected Result | Actual Result | Status |
|-----------|-------------|-----------------|---------------|--------|
| TC-2.1 | Configure `https_spiffe` profile | Federation established | Failed - "local copy of bundle not found" | ❌ FAIL |
| TC-2.2 | Bootstrap `https_spiffe` with ClusterFederatedTrustDomain | Bundle inserted into PostgreSQL | Operator did not process trustDomainBundle field | ❌ FAIL |
| TC-2.3 | Configure `https_web` profile with ACME | Federation established | ACME ToS accepted, certificates provisioned | ✅ PASS |
| TC-2.4 | Federation route created | Route accessible externally | Route `spire-server-federation` created with passthrough TLS | ✅ PASS |

**Finding - `https_spiffe` Limitation:**
```
The Zero Trust Workload Identity Manager Operator v1.0.0 does NOT implement 
the trustDomainBundle bootstrap functionality. The CRD schema includes the 
field, but the operator does not process it to insert bundles into the datastore.

Workaround: Use https_web profile with ACME/Let's Encrypt instead.
```

**Evidence - ACME Certificate Provisioning:**
```
level=info msg="ACME Terms of Service accepted" 
  directory_url="https://acme-v02.api.letsencrypt.org/directory" 
  email=admin@redhat.com 
  tos_url="https://letsencrypt.org/documents/LE-SA-v1.6-August-18-2025.pdf"
```

---

### Test Scenario 3: Cross-Cluster Trust Bundle Synchronization

| Test Case | Description | Expected Result | Actual Result | Status |
|-----------|-------------|-----------------|---------------|--------|
| TC-3.1 | Cluster 1 fetches Cluster 2's bundle | 2 bundles in PostgreSQL | 2 bundles present | ✅ PASS |
| TC-3.2 | Cluster 2 fetches Cluster 1's bundle | 2 bundles in PostgreSQL | 2 bundles present | ✅ PASS |
| TC-3.3 | Bundle refresh occurs periodically | "Bundle refreshed" in logs | Logs show periodic refresh | ✅ PASS |

**Evidence - Trust Bundles in PostgreSQL (Cluster 1):**
```sql
SELECT trust_domain FROM bundles;
                       trust_domain                       
----------------------------------------------------------
 spiffe://apps.sayadas-aws-1.qe.devcluster.openshift.com    ← Local
 spiffe://apps.ci-ln-3ml0mvb-72292.gcp-2.ci.openshift.org   ← Federated
(2 rows)
```

**Evidence - Trust Bundles in PostgreSQL (Cluster 2):**
```sql
SELECT trust_domain FROM bundles;
                       trust_domain                       
----------------------------------------------------------
 spiffe://apps.ci-ln-3ml0mvb-72292.gcp-2.ci.openshift.org   ← Local
 spiffe://apps.sayadas-aws-1.qe.devcluster.openshift.com    ← Federated
(2 rows)
```

**Evidence - Bundle Refresh Logs:**
```
level=info msg="Bundle refreshed" subsystem_name=bundle_client 
  trust_domain=apps.ci-ln-3ml0mvb-72292.gcp-2.ci.openshift.org
```

---

### Test Scenario 4: SPIRE Agent Deployment and Node Attestation

| Test Case | Description | Expected Result | Actual Result | Status |
|-----------|-------------|-----------------|---------------|--------|
| TC-4.1 | Deploy SpireAgent with empty spec | Agent fails | CrashLoopBackOff - "WorkloadAttestor constraint not satisfied" | ✅ Expected |
| TC-4.2 | Deploy SpireAgent with k8sPSAT attestor | Agents running | 3 agents running (1/1) on each cluster | ✅ PASS |
| TC-4.3 | Node attestation succeeds | "Successfully reattested node" in logs | Logs confirm successful attestation | ✅ PASS |

**Evidence - SpireAgent Configuration Required:**
```yaml
spec:
  nodeAttestor:
    k8sPSATEnabled: "true"    # Required!
  workloadAttestors:
    k8sEnabled: "true"         # Required!
    workloadAttestorsVerification:
      type: "auto"
```

**Evidence - Node Attestation Success:**
```
level=info msg="Successfully reattested node" 
  spiffe_id="spiffe://apps.sayadas-aws-1.qe.devcluster.openshift.com/spire/agent/k8s_psat/test01/ddcddde4-2774-4ad2-badc-5bd90c259ff3"
```

---

### Test Scenario 5: Workload Identity (SVID) Issuance

| Test Case | Description | Expected Result | Actual Result | Status |
|-----------|-------------|-----------------|---------------|--------|
| TC-5.1 | Create ClusterSPIFFEID for test namespace | CRD created | ClusterSPIFFEID `test-workload` created | ✅ PASS |
| TC-5.2 | Deploy pod with SPIFFE CSI driver | Pod mounts workload API | Pod running with `/spiffe-workload-api/spire-agent.sock` | ✅ PASS |
| TC-5.3 | Workload registered in PostgreSQL | Entry in registered_entries | Entry found with correct SPIFFE ID | ✅ PASS |
| TC-5.4 | X509-SVID issued to workload | "Creating X509-SVID" in agent logs | Logs confirm SVID creation | ✅ PASS |

**Evidence - Registered Entry in PostgreSQL:**
```sql
SELECT spiffe_id FROM registered_entries WHERE spiffe_id LIKE '%spire-test%';
                              spiffe_id                              
---------------------------------------------------------------------
 spiffe://apps.sayadas-aws-1.qe.devcluster.openshift.com/ns/spire-test/sa/test-workload
(1 row)
```

**Evidence - SVID Issuance Log:**
```
level=info msg="Creating X509-SVID" 
  entry_id=test01.b5f686a6-c4d5-49b1-a2f6-c18ed3b5cc9c 
  spiffe_id="spiffe://apps.sayadas-aws-1.qe.devcluster.openshift.com/ns/spire-test/sa/test-workload"
```

---

### Test Scenario 6: PostgreSQL Resilience

| Test Case | Description | Expected Result | Actual Result | Status |
|-----------|-------------|-----------------|---------------|--------|
| TC-6.1 | Delete PostgreSQL pod | New pod starts, data preserved | New pod started, PVC reattached | ✅ PASS |
| TC-6.2 | Trust bundles survive restart | 2 bundles still in DB | 2 bundles confirmed after restart | ✅ PASS |
| TC-6.3 | Registered entries survive restart | Entry count unchanged | 3 entries confirmed after restart | ✅ PASS |
| TC-6.4 | SPIRE Server recovers connection | "Bundle refreshed" after recovery | Federation sync resumed | ✅ PASS |

**Evidence - Data Preserved After PostgreSQL Restart:**
```
--- Before Restart ---
 trust_domain: 2 rows
 entry_count: 3

--- After Restart ---
 trust_domain: 2 rows  ← Preserved!
 entry_count: 3        ← Preserved!
```

**Evidence - SPIRE Server Recovery:**
```
level=error msg="Failed to list entries" error="datastore-sql: read tcp ... connection reset by peer"
  ↑ Transient error during restart (expected)

level=info msg="Bundle refreshed" trust_domain=apps.ci-ln-3ml0mvb-72292.gcp-2.ci.openshift.org
  ↑ Recovery successful!
```

---

## 📊 Overall Test Results Summary

| Category | Total Tests | Passed | Failed | Pass Rate |
|----------|-------------|--------|--------|-----------|
| PostgreSQL Datastore | 4 | 4 | 0 | 100% |
| Federation Profile | 4 | 2 | 2 | 50% |
| Trust Bundle Sync | 3 | 3 | 0 | 100% |
| SPIRE Agent | 3 | 3 | 0 | 100% |
| Workload Identity | 4 | 4 | 0 | 100% |
| PostgreSQL Resilience | 4 | 4 | 0 | 100% |
| **TOTAL** | **22** | **20** | **2** | **91%** |

---

## 🐛 Known Issues Discovered

| Issue ID | Description | Severity | Workaround |
|----------|-------------|----------|------------|
| ISSUE-001 | `https_spiffe` profile requires bootstrap but operator doesn't implement `trustDomainBundle` processing | High | Use `https_web` profile instead |
| ISSUE-002 | SpireAgent with empty spec crashes with "WorkloadAttestor constraint not satisfied" | Medium | Always specify `nodeAttestor` and `workloadAttestors` in SpireAgent CR |

---

## ✅ Test Conclusion

**SPIRE Federation with PostgreSQL 15 as the backend datastore is FUNCTIONAL** using the `https_web` federation profile with ACME/Let's Encrypt certificates on the Zero Trust Workload Identity Manager Operator v1.0.0.

**Key Findings:**
1. ✅ PostgreSQL works as a reliable datastore for SPIRE
2. ✅ Data persists across PostgreSQL restarts (PVC-backed)
3. ✅ Cross-cluster federation works with `https_web` profile
4. ✅ Workload identities (SVIDs) are issued correctly
5. ⚠️ `https_spiffe` profile requires operator enhancement for bootstrap support

---

## 📝 Document Information

| Field | Value |
|-------|-------|
| **Document Version** | 1.1 |
| **Test Date** | January 12, 2026 |
| **PostgreSQL Version** | 15 (RHEL9) |
| **SPIRE Version** | 1.13.3-dev |
| **Operator Version** | Zero Trust Workload Identity Manager v1.0.0 |
| **Federation Profile** | https_web with ACME/Let's Encrypt |
| **Test Environment** | AWS (Cluster 1) + GCP (Cluster 2) |
| **Tester** | Sayak Das |

---

**End of Test Guide**

