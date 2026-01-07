# Manual Federation Testing Guide

**Document Version:** 1.0  
**Last Updated:** January 2026  
**Author:** Sayak Das  
**Product:** Zero Trust Workload Identity Manager (ZTWIM)

---

## Overview

This guide provides **step-by-step `oc` commands** for setting up SPIRE federation between **2 OpenShift clusters** using **3 different federation profile scenarios**:

| Scenario | Cluster 1 Profile | Cluster 2 Profile | Use Case |
|----------|------------------|------------------|----------|
| **Scenario 1** | `https_spiffe` | `https_spiffe` | Internal clusters, testing |
| **Scenario 2** | `https_spiffe` | `https_web` (ACME) | Mixed: internal + public |
| **Scenario 3** | `https_spiffe` | `https_web` (cert-manager) | Enterprise PKI |

> **📌 Note: 3-Cluster Setup**  
> If you need to set up federation across **3 clusters** (with mixed profiles including ACME), refer to the automated script:  
> 🔗 [3-Cluster Federation Setup Document (with ACME)](https://gist.github.com/rausingh-rh/305cd0f9ae7be9e522e10341fa8b6647) - created by **Raushan Kumar Singh**

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   CLUSTER 1                                    CLUSTER 2                    │
│   Trust Domain: $APP_DOMAIN1                   Trust Domain: $APP_DOMAIN2   │
│                                                                             │
│   ┌─────────────────────────┐                 ┌─────────────────────────┐  │
│   │      SPIRE Server       │◄───────────────►│      SPIRE Server       │  │
│   │                         │   Federation    │                         │  │
│   │  Profile: https_spiffe  │     Trust       │  Profile: varies        │  │
│   │  (self-signed cert)     │    Exchange     │  (per scenario)         │  │
│   └─────────────────────────┘                 └─────────────────────────┘  │
│                                                                             │
│   Federation Endpoint:                        Federation Endpoint:          │
│   https://federation.$APP_DOMAIN1             https://federation.$APP_DOMAIN2│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

### Required Tools

```bash
oc version
curl --version
openssl version
```

### Required Access

- 2 OpenShift clusters (4.20)
- cluster-admin on both clusters
- Kubeconfig files for both clusters

---

## Initial Setup (All Scenarios)

### Step 0.1: Open Two Terminals

```
Terminal 1 → Cluster 1
Terminal 2 → Cluster 2
```

### Step 0.2: Set Environment Variables

**Terminal 1 (Cluster 1):**

```bash
# Set kubeconfig
export KUBECONFIG=/path/to/cluster1/kubeconfig

# Set variables
export SPIRE_NS="zero-trust-workload-identity-manager"
export BASE_DOMAIN=$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
export APP_DOMAIN="apps.${BASE_DOMAIN}"
export JWT_ISSUER="oidc-discovery.${APP_DOMAIN}"
export CLUSTER_NAME="cluster1"

# Display
echo "=========================================="
echo "CLUSTER 1 CONFIGURATION"
echo "=========================================="
echo "APP_DOMAIN:   ${APP_DOMAIN}"
echo "JWT_ISSUER:   https://${JWT_ISSUER}"
echo "CLUSTER_NAME: ${CLUSTER_NAME}"
echo "=========================================="
```

**Terminal 2 (Cluster 2):**

```bash
# Set kubeconfig
export KUBECONFIG=/path/to/cluster2/kubeconfig

# Set variables
export SPIRE_NS="zero-trust-workload-identity-manager"
export BASE_DOMAIN=$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
export APP_DOMAIN="apps.${BASE_DOMAIN}"
export JWT_ISSUER="oidc-discovery.${APP_DOMAIN}"
export CLUSTER_NAME="cluster1"

# Display
echo "=========================================="
echo "CLUSTER 2 CONFIGURATION"
echo "=========================================="
echo "APP_DOMAIN:   ${APP_DOMAIN}"
echo "JWT_ISSUER:   https://${JWT_ISSUER}"
echo "CLUSTER_NAME: ${CLUSTER_NAME}"
echo "=========================================="
```

### Step 0.3: Exchange Domain Names

**Copy Cluster 2's APP_DOMAIN to Terminal 1:**

```bash
# Terminal 1 - Set Cluster 2's domain
export REMOTE_DOMAIN="<PASTE_CLUSTER_2_APP_DOMAIN_HERE>"
echo "Will federate with: ${REMOTE_DOMAIN}"
```

**Copy Cluster 1's APP_DOMAIN to Terminal 2:**

```bash
# Terminal 2 - Set Cluster 1's domain
export REMOTE_DOMAIN="<PASTE_CLUSTER_1_APP_DOMAIN_HERE>"
echo "Will federate with: ${REMOTE_DOMAIN}"
```

### Step 0.4: Install Operator (Both Clusters)

**Run on BOTH terminals:**

```bash
echo "[INFO] Installing Zero Trust Workload Identity Manager operator..."

oc apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: zero-trust-workload-identity-manager
  labels:
    openshift.io/cluster-monitoring: "true"
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: zero-trust-workload-identity-manager-og
  namespace: zero-trust-workload-identity-manager
spec:
  upgradeStrategy: Default
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-zero-trust-workload-identity-manager
  namespace: zero-trust-workload-identity-manager
spec:
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  name: openshift-zero-trust-workload-identity-manager
  channel: stable-v1
EOF

echo "[INFO] Waiting for operator to be ready..."
oc wait --timeout=5m -n ${SPIRE_NS} \
  deployment/zero-trust-workload-identity-manager-controller-manager \
  --for=condition=Available

echo "[INFO] ✅ Operator installed successfully"
oc get pods -n ${SPIRE_NS}
```

---

# Scenario 1: https_spiffe ↔ https_spiffe

**Both clusters use self-signed certificates managed by SPIRE**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   CLUSTER 1                                    CLUSTER 2                    │
│   ┌─────────────────────────┐                 ┌─────────────────────────┐  │
│   │  Profile: https_spiffe  │◄───────────────►│  Profile: https_spiffe  │  │
│   │  (self-signed cert)     │                 │  (self-signed cert)     │  │
│   │                         │                 │                         │  │
│   │  curl -k required       │                 │  curl -k required       │  │
│   │  endpointSpiffeId ✓     │                 │  endpointSpiffeId ✓     │  │
│   └─────────────────────────┘                 └─────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Step 1.1: Deploy SPIRE on Cluster 1 (https_spiffe)

**Terminal 1:**

```bash
echo "[INFO] Deploying SPIRE resources on Cluster 1 with https_spiffe profile..."

oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: ZeroTrustWorkloadIdentityManager
metadata:
  name: cluster
spec:
  trustDomain: ${APP_DOMAIN}
  clusterName: ${CLUSTER_NAME}
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: ${APP_DOMAIN}
    country: "US"
    organization: "RH"
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
    maxOpenConns: 100
    maxIdleConns: 2
    connMaxLifetime: 3600
  jwtIssuer: https://${JWT_ISSUER}
  federation:
    bundleEndpoint:
      profile: https_spiffe
      refreshHint: 300
    managedRoute: "true"
    federatesWith:
    - trustDomain: ${REMOTE_DOMAIN}
      bundleEndpointUrl: https://federation.${REMOTE_DOMAIN}
      bundleEndpointProfile: https_spiffe
      endpointSpiffeId: spiffe://${REMOTE_DOMAIN}/spire/server
---
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
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpiffeCSIDriver
metadata:
  name: cluster
spec: {}
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireOIDCDiscoveryProvider
metadata:
  name: cluster
spec:
  jwtIssuer: https://${JWT_ISSUER}
EOF

echo "[INFO] ✅ Cluster 1 SPIRE resources deployed"
```

## Step 1.2: Deploy SPIRE on Cluster 2 (https_spiffe)

**Terminal 2:**

```bash
echo "[INFO] Deploying SPIRE resources on Cluster 2 with https_spiffe profile..."

oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: ZeroTrustWorkloadIdentityManager
metadata:
  name: cluster
spec:
  trustDomain: ${APP_DOMAIN}
  clusterName: ${CLUSTER_NAME}
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: ${APP_DOMAIN}
    country: "US"
    organization: "RH"
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
    maxOpenConns: 100
    maxIdleConns: 2
    connMaxLifetime: 3600
  jwtIssuer: https://${JWT_ISSUER}
  federation:
    bundleEndpoint:
      profile: https_spiffe
      refreshHint: 300
    managedRoute: "true"
    federatesWith:
    - trustDomain: ${REMOTE_DOMAIN}
      bundleEndpointUrl: https://federation.${REMOTE_DOMAIN}
      bundleEndpointProfile: https_spiffe
      endpointSpiffeId: spiffe://${REMOTE_DOMAIN}/spire/server
---
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
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpiffeCSIDriver
metadata:
  name: cluster
spec: {}
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireOIDCDiscoveryProvider
metadata:
  name: cluster
spec:
  jwtIssuer: https://${JWT_ISSUER}
EOF

echo "[INFO] ✅ Cluster 2 SPIRE resources deployed"
```

## Step 1.3: Wait for SPIRE Servers (Both Clusters)

**Run on BOTH terminals:**

```bash
echo "[INFO] Waiting for SPIRE server to be ready..."

# Wait for pod to exist
while ! oc get pod spire-server-0 -n ${SPIRE_NS} &>/dev/null; do
  echo "[INFO] Waiting for spire-server-0 pod to be created..."
  sleep 10
done

# Wait for pod to be ready
oc wait --timeout=5m -n ${SPIRE_NS} pod/spire-server-0 --for=condition=Ready

echo "[INFO] ✅ SPIRE server is ready"
oc get pods -n ${SPIRE_NS}
```

## Step 1.4: Wait for Federation Endpoints (Both Clusters)

**Run on BOTH terminals:**

```bash
echo "[INFO] Waiting for federation endpoint to be accessible..."

FEDERATION_URL="https://federation.${APP_DOMAIN}"
MAX_WAIT=300
ELAPSED=0

while [[ $ELAPSED -lt $MAX_WAIT ]]; do
  if curl -sSLk --max-time 10 --fail "$FEDERATION_URL" -o /dev/null 2>/dev/null; then
    echo "[INFO] ✅ Federation endpoint is ready: $FEDERATION_URL"
    break
  fi
  echo "[INFO] Waiting... ($ELAPSED seconds)"
  sleep 10
  ELAPSED=$((ELAPSED + 10))
done

if [[ $ELAPSED -ge $MAX_WAIT ]]; then
  echo "[ERROR] Timeout waiting for federation endpoint"
  echo "[ERROR] Check: oc get route spire-server-federation -n ${SPIRE_NS}"
fi
```

## Step 1.5: Verify Federation (Both Clusters)

**Run on BOTH terminals:**

```bash
echo "=========================================="
echo "      FEDERATION VERIFICATION            "
echo "=========================================="

echo ""
echo "[INFO] 1. Federation Route:"
oc get route spire-server-federation -n ${SPIRE_NS}

echo ""
echo "[INFO] 2. Test Local Bundle Endpoint:"
curl -k -s https://federation.${APP_DOMAIN} | head -c 200
echo ""

echo ""
echo "[INFO] 3. Test Remote Bundle Endpoint:"
curl -k -s https://federation.${REMOTE_DOMAIN} | head -c 200
echo ""

echo ""
echo "[INFO] 4. SPIRE Server Bundle List (should show REMOTE domain):"
oc exec -n ${SPIRE_NS} spire-server-0 -c spire-server -- /spire-server bundle list | grep "^\*"

echo ""
echo "[INFO] 5. Bundle Refresh Logs:"
oc logs -n ${SPIRE_NS} spire-server-0 -c spire-server --tail=10 | grep -i "bundle"

echo "=========================================="
```

**Expected Output:**
```
==========================================
      FEDERATION VERIFICATION            
==========================================

[INFO] 1. Federation Route:
NAME                      HOST/PORT                           TERMINATION
spire-server-federation   federation.apps.xxx.example.com     passthrough

[INFO] 2. Test Local Bundle Endpoint:
{"keys":[{"use":"x509-svid","kty":"RSA",...

[INFO] 3. Test Remote Bundle Endpoint:
{"keys":[{"use":"x509-svid","kty":"RSA",...

[INFO] 4. SPIRE Server Bundle List (should show REMOTE domain):
* apps.remote-cluster.example.com

[INFO] 5. Bundle Refresh Logs:
time="..." level=info msg="Bundle refreshed" trust_domain=apps.remote-cluster.example.com
==========================================
```

---

# Scenario 2: https_spiffe ↔ https_web (ACME)

**Cluster 1 uses self-signed cert, Cluster 2 uses Let's Encrypt (ACME)**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   CLUSTER 1                                    CLUSTER 2                    │
│   ┌─────────────────────────┐                 ┌─────────────────────────┐  │
│   │  Profile: https_spiffe  │◄───────────────►│  Profile: https_web     │  │
│   │  (self-signed cert)     │                 │  (Let's Encrypt cert)   │  │
│   │                         │                 │                         │  │
│   │  curl -k required       │                 │  curl OK (no -k)        │  │
│   │  endpointSpiffeId ✓     │                 │  NO endpointSpiffeId    │  │
│   └─────────────────────────┘                 └─────────────────────────┘  │
│                                                                             │
│   Note: Cluster 2 needs public DNS and internet access for ACME            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Step 2.1: Deploy SPIRE on Cluster 1 (https_spiffe)

**Terminal 1:**

```bash
echo "[INFO] Deploying SPIRE resources on Cluster 1 with https_spiffe profile..."

oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: ZeroTrustWorkloadIdentityManager
metadata:
  name: cluster
spec:
  trustDomain: ${APP_DOMAIN}
  clusterName: ${CLUSTER_NAME}
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: ${APP_DOMAIN}
    country: "US"
    organization: "RH"
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
    maxOpenConns: 100
    maxIdleConns: 2
    connMaxLifetime: 3600
  jwtIssuer: https://${JWT_ISSUER}
  federation:
    bundleEndpoint:
      profile: https_spiffe
      refreshHint: 300
    managedRoute: "true"
    federatesWith:
    - trustDomain: ${REMOTE_DOMAIN}
      bundleEndpointUrl: https://federation.${REMOTE_DOMAIN}
      bundleEndpointProfile: https_web
---
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
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpiffeCSIDriver
metadata:
  name: cluster
spec: {}
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireOIDCDiscoveryProvider
metadata:
  name: cluster
spec:
  jwtIssuer: https://${JWT_ISSUER}
EOF

echo "[INFO] ✅ Cluster 1 SPIRE resources deployed (https_spiffe)"
```

## Step 2.2: Deploy SPIRE on Cluster 2 (https_web with ACME)

**Terminal 2:**

```bash
# Set ACME email
export ACME_EMAIL="admin@example.com"

echo "[INFO] Deploying SPIRE resources on Cluster 2 with https_web (ACME) profile..."
echo "[INFO] Using ACME email: ${ACME_EMAIL}"

oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: ZeroTrustWorkloadIdentityManager
metadata:
  name: cluster
spec:
  trustDomain: ${APP_DOMAIN}
  clusterName: ${CLUSTER_NAME}
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: ${APP_DOMAIN}
    country: "US"
    organization: "RH"
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
    maxOpenConns: 100
    maxIdleConns: 2
    connMaxLifetime: 3600
  jwtIssuer: https://${JWT_ISSUER}
  federation:
    bundleEndpoint:
      profile: https_web
      httpsWeb:
        acme:
          directoryUrl: "https://acme-v02.api.letsencrypt.org/directory"
          domainName: "federation.${APP_DOMAIN}"
          email: "${ACME_EMAIL}"
          tosAccepted: "true"
      refreshHint: 300
    managedRoute: "true"
    federatesWith:
    - trustDomain: ${REMOTE_DOMAIN}
      bundleEndpointUrl: https://federation.${REMOTE_DOMAIN}
      bundleEndpointProfile: https_spiffe
      endpointSpiffeId: spiffe://${REMOTE_DOMAIN}/spire/server
---
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
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpiffeCSIDriver
metadata:
  name: cluster
spec: {}
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireOIDCDiscoveryProvider
metadata:
  name: cluster
spec:
  jwtIssuer: https://${JWT_ISSUER}
EOF

echo "[INFO] ✅ Cluster 2 SPIRE resources deployed (https_web with ACME)"
echo "[INFO] Note: ACME certificate may take 2-5 minutes to be issued"
```

## Step 2.3: Wait for SPIRE Servers and ACME Certificate

**Terminal 1 (Cluster 1):**

```bash
echo "[INFO] Waiting for SPIRE server on Cluster 1..."

while ! oc get pod spire-server-0 -n ${SPIRE_NS} &>/dev/null; do
  sleep 10
done
oc wait --timeout=5m -n ${SPIRE_NS} pod/spire-server-0 --for=condition=Ready

echo "[INFO] ✅ Cluster 1 SPIRE server is ready"
```

**Terminal 2 (Cluster 2):**

```bash
echo "[INFO] Waiting for SPIRE server on Cluster 2..."

while ! oc get pod spire-server-0 -n ${SPIRE_NS} &>/dev/null; do
  sleep 10
done
oc wait --timeout=5m -n ${SPIRE_NS} pod/spire-server-0 --for=condition=Ready

echo "[INFO] ✅ Cluster 2 SPIRE server is ready"

echo ""
echo "[INFO] Waiting for ACME certificate (may take 2-5 minutes)..."
sleep 180

echo "[INFO] Checking ACME status in logs:"
oc logs -n ${SPIRE_NS} spire-server-0 -c spire-server --tail=30 | grep -i "acme\|cert"
```

## Step 2.4: Verify Federation

**Terminal 1 (Cluster 1 - https_spiffe):**

```bash
echo "[INFO] Verifying federation on Cluster 1..."

echo ""
echo "[INFO] Test Cluster 2's ACME endpoint (no -k needed!):"
curl -s https://federation.${REMOTE_DOMAIN} | head -c 200
echo ""

echo ""
echo "[INFO] Bundle List (should show Cluster 2):"
oc exec -n ${SPIRE_NS} spire-server-0 -c spire-server -- /spire-server bundle list | grep "^\*"
```

**Terminal 2 (Cluster 2 - https_web ACME):**

```bash
echo "[INFO] Verifying federation on Cluster 2..."

echo ""
echo "[INFO] Test Cluster 1's endpoint (need -k for self-signed):"
curl -k -s https://federation.${REMOTE_DOMAIN} | head -c 200
echo ""

echo ""
echo "[INFO] Bundle List (should show Cluster 1):"
oc exec -n ${SPIRE_NS} spire-server-0 -c spire-server -- /spire-server bundle list | grep "^\*"
```

---

# Scenario 3: https_spiffe ↔ https_web (cert-manager)

**Cluster 1 uses self-signed cert, Cluster 2 uses cert-manager managed certificate**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   CLUSTER 1                                    CLUSTER 2                    │
│   ┌─────────────────────────┐                 ┌─────────────────────────┐  │
│   │  Profile: https_spiffe  │◄───────────────►│  Profile: https_web     │  │
│   │  (self-signed cert)     │                 │  (cert-manager cert)    │  │
│   │                         │                 │                         │  │
│   │  curl -k required       │                 │  curl -k (if self-signed)│  │
│   │  endpointSpiffeId ✓     │                 │  NO endpointSpiffeId    │  │
│   └─────────────────────────┘                 └─────────────────────────┘  │
│                                                                             │
│   Note: Cluster 2 needs cert-manager operator installed                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Step 3.1: Deploy SPIRE on Cluster 1 (https_spiffe)

**Terminal 1:**

```bash
echo "[INFO] Deploying SPIRE resources on Cluster 1 with https_spiffe profile..."

oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: ZeroTrustWorkloadIdentityManager
metadata:
  name: cluster
spec:
  trustDomain: ${APP_DOMAIN}
  clusterName: ${CLUSTER_NAME}
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: ${APP_DOMAIN}
    country: "US"
    organization: "RH"
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
    maxOpenConns: 100
    maxIdleConns: 2
    connMaxLifetime: 3600
  jwtIssuer: https://${JWT_ISSUER}
  federation:
    bundleEndpoint:
      profile: https_spiffe
      refreshHint: 300
    managedRoute: "true"
    federatesWith:
    - trustDomain: ${REMOTE_DOMAIN}
      bundleEndpointUrl: https://federation.${REMOTE_DOMAIN}
      bundleEndpointProfile: https_web
---
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
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpiffeCSIDriver
metadata:
  name: cluster
spec: {}
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireOIDCDiscoveryProvider
metadata:
  name: cluster
spec:
  jwtIssuer: https://${JWT_ISSUER}
EOF

echo "[INFO] ✅ Cluster 1 SPIRE resources deployed (https_spiffe)"
```

## Step 3.2: Setup cert-manager on Cluster 2

**Terminal 2:**

```bash
echo "[INFO] Setting up cert-manager resources on Cluster 2..."

# Step 3.2.1: Create ClusterIssuer (self-signed for testing)
echo "[INFO] Creating ClusterIssuer..."
oc apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
EOF

# Step 3.2.2: Create Certificate for federation endpoint
echo "[INFO] Creating Certificate for federation endpoint..."
oc apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: spire-federation-cert
  namespace: ${SPIRE_NS}
spec:
  secretName: spire-federation-tls
  duration: 2160h
  renewBefore: 360h
  subject:
    organizations:
      - "My Company"
  commonName: federation.${APP_DOMAIN}
  dnsNames:
    - federation.${APP_DOMAIN}
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
EOF

# Wait for certificate
echo "[INFO] Waiting for certificate to be ready..."
sleep 30
oc get certificate -n ${SPIRE_NS}

echo "[INFO] ✅ cert-manager resources created"
```

## Step 3.3: Deploy SPIRE on Cluster 2 (https_web with cert-manager)

**Terminal 2:**

```bash
echo "[INFO] Deploying SPIRE resources on Cluster 2 with https_web (cert-manager) profile..."

oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: ZeroTrustWorkloadIdentityManager
metadata:
  name: cluster
spec:
  trustDomain: ${APP_DOMAIN}
  clusterName: ${CLUSTER_NAME}
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: ${APP_DOMAIN}
    country: "US"
    organization: "RH"
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
    maxOpenConns: 100
    maxIdleConns: 2
    connMaxLifetime: 3600
  jwtIssuer: https://${JWT_ISSUER}
  federation:
    bundleEndpoint:
      profile: https_web
      httpsWeb:
        servingCert:
          secretName: spire-federation-tls
      refreshHint: 300
    managedRoute: "true"
    federatesWith:
    - trustDomain: ${REMOTE_DOMAIN}
      bundleEndpointUrl: https://federation.${REMOTE_DOMAIN}
      bundleEndpointProfile: https_spiffe
      endpointSpiffeId: spiffe://${REMOTE_DOMAIN}/spire/server
---
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
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpiffeCSIDriver
metadata:
  name: cluster
spec: {}
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireOIDCDiscoveryProvider
metadata:
  name: cluster
spec:
  jwtIssuer: https://${JWT_ISSUER}
EOF

echo "[INFO] ✅ Cluster 2 SPIRE resources deployed (https_web with cert-manager)"
```

## Step 3.4: Wait and Verify

**Run on BOTH terminals:**

```bash
echo "[INFO] Waiting for SPIRE server..."

while ! oc get pod spire-server-0 -n ${SPIRE_NS} &>/dev/null; do
  sleep 10
done
oc wait --timeout=5m -n ${SPIRE_NS} pod/spire-server-0 --for=condition=Ready

echo "[INFO] ✅ SPIRE server is ready"

# Wait for federation endpoint
sleep 60

echo ""
echo "[INFO] Verifying federation..."
echo ""
echo "[INFO] Bundle List:"
oc exec -n ${SPIRE_NS} spire-server-0 -c spire-server -- /spire-server bundle list | grep "^\*"
```

---

# Complete Verification Script

Run this on **BOTH clusters** after any scenario setup:

```bash
echo "=========================================="
echo "   COMPLETE FEDERATION VERIFICATION      "
echo "=========================================="

echo ""
echo "[1/7] Pods Status:"
oc get pods -n ${SPIRE_NS}

echo ""
echo "[2/7] SPIRE Server Health:"
oc exec -n ${SPIRE_NS} spire-server-0 -c spire-server -- /spire-server healthcheck

echo ""
echo "[3/7] Federation Route:"
oc get route spire-server-federation -n ${SPIRE_NS} -o wide

echo ""
echo "[4/7] Local Bundle Endpoint Test:"
curl -k -s https://federation.${APP_DOMAIN} | head -c 200
echo ""

echo ""
echo "[5/7] Remote Bundle Endpoint Test:"
curl -k -s https://federation.${REMOTE_DOMAIN} | head -c 200
echo ""

echo ""
echo "[6/7] SPIRE Bundle List (should show both domains):"
oc exec -n ${SPIRE_NS} spire-server-0 -c spire-server -- /spire-server bundle list | grep "^\*"

echo ""
echo "[7/7] Bundle Refresh Logs:"
oc logs -n ${SPIRE_NS} spire-server-0 -c spire-server --tail=15 | grep -i "bundle"

echo ""
echo "=========================================="
echo "              SUMMARY                    "
echo "=========================================="
echo "Local Domain:  ${APP_DOMAIN}"
echo "Remote Domain: ${REMOTE_DOMAIN}"
echo ""
echo "Federation endpoints:"
echo "  Local:  https://federation.${APP_DOMAIN}"
echo "  Remote: https://federation.${REMOTE_DOMAIN}"
echo "=========================================="
```

---

# Cleanup

## Remove SPIRE Resources (Both Clusters)

```bash
echo "[INFO] Cleaning up SPIRE resources..."

# Delete operands
oc delete zerotrustworkloadidentitymanager cluster --ignore-not-found
oc delete spireserver cluster --ignore-not-found
oc delete spireagent cluster --ignore-not-found
oc delete spiffecsidriver cluster --ignore-not-found
oc delete spireoidcdiscoveryprovider cluster --ignore-not-found

# Delete PVC
oc delete pvc -n ${SPIRE_NS} -l app.kubernetes.io/name=spire-server --ignore-not-found

# For cert-manager scenario
oc delete certificate spire-federation-cert -n ${SPIRE_NS} --ignore-not-found
oc delete clusterissuer selfsigned-issuer --ignore-not-found

echo "[INFO] ✅ SPIRE resources cleaned up"
```

## Remove Operator (Both Clusters)

```bash
echo "[INFO] Removing operator..."

oc delete subscription openshift-zero-trust-workload-identity-manager -n ${SPIRE_NS} --ignore-not-found
oc delete csv -n ${SPIRE_NS} --all --ignore-not-found
oc delete namespace ${SPIRE_NS} --ignore-not-found

echo "[INFO] ✅ Operator removed"
```

---

# Quick Reference

## Profile Configuration Summary

| Profile | bundleEndpointProfile | endpointSpiffeId | curl flag |
|---------|----------------------|------------------|-----------|
| `https_spiffe` | `https_spiffe` | ✅ Required | `-k` |
| `https_web` (ACME) | `https_web` | ❌ Not needed | None |
| `https_web` (cert-manager) | `https_web` | ❌ Not needed | `-k` (if self-signed) |

## Key Commands

| Action | Command |
|--------|---------|
| Check SPIRE health | `oc exec -n ${SPIRE_NS} spire-server-0 -c spire-server -- /spire-server healthcheck` |
| List bundles | `oc exec -n ${SPIRE_NS} spire-server-0 -c spire-server -- /spire-server bundle list` |
| Show entries | `oc exec -n ${SPIRE_NS} spire-server-0 -c spire-server -- /spire-server entry show` |
| Check route | `oc get route spire-server-federation -n ${SPIRE_NS}` |
| Test endpoint | `curl -k https://federation.${APP_DOMAIN}` |
| Check logs | `oc logs -n ${SPIRE_NS} spire-server-0 -c spire-server --tail=50` |

## Troubleshooting

### Pods Not Starting (Proxy Error)

```bash
# Create trusted CA bundle
oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: trusted-ca-bundle
  namespace: zero-trust-workload-identity-manager
  labels:
    config.openshift.io/inject-trusted-cabundle: "true"
data: {}
EOF

# Patch subscription
oc patch subscription openshift-zero-trust-workload-identity-manager \
  -n zero-trust-workload-identity-manager \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/config", "value": {"env": [{"name": "TRUSTED_CA_BUNDLE_CONFIGMAP", "value": "trusted-ca-bundle"}]}}]'
```

### Bundle Not Refreshing

```bash
# Check for errors
oc logs -n ${SPIRE_NS} spire-server-0 -c spire-server --tail=50 | grep -i "error\|failed"

# Verify remote endpoint is reachable
curl -kv https://federation.${REMOTE_DOMAIN}
```

### ACME Certificate Not Issued

```bash
# Check ACME logs
oc logs -n ${SPIRE_NS} spire-server-0 -c spire-server --tail=50 | grep -i "acme"

# Common issues:
# 1. DNS not resolving
# 2. Port 443 not accessible from internet
# 3. Rate limits (use staging URL for testing)
```

---

## Document History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | Jan 2026 | Initial release |
| 2.0 | Jan 2026 | Updated with inline federatesWith and 3 scenarios |

---

*End of Document*
