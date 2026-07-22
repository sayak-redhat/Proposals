# End-to-End POC Guide: SPIRE + OSSM + OPA (Multi-Cluster)

**Audience:** Beginners  
**Goal:** Run a working multi-cluster POC where SPIRE proves identity, OSSM carries mTLS traffic, and OPA decides allow/deny.  
**Style:** Concepts first, then only the **correct working steps** (no failed attempts).

**Upstream idea:** [SPIFFE.io — OPA Authorization with Envoy and X.509-SVIDs](https://spiffe.io/docs/latest/microservices/envoy-opa/readme/)

**Validated on:** OpenShift 4.22, Red Hat OpenShift Service Mesh 3.4.0 (Istio 1.30.1), ZTWIM Operator, OPA `0.70.0-envoy`

---

## Table of Contents

1. [Motive of this POC](#1-motive-of-this-poc)
2. [High-level workflow (read this first)](#2-high-level-workflow-read-this-first)
3. [Concepts (beginner friendly)](#3-concepts-beginner-friendly)
4. [What we prove](#4-what-we-prove)
5. [Architecture](#5-architecture)
6. [Prerequisites](#6-prerequisites)
7. [Environment variables](#7-environment-variables)
8. [Phase 1 — Install ZTWIM + SPIRE on both clusters](#8-phase-1--install-ztwim--spire-on-both-clusters)
9. [Phase 2 — Federate the two SPIRE trust domains](#9-phase-2--federate-the-two-spire-trust-domains)
10. [Phase 3 — Install Red Hat Service Mesh + IstioCNI](#10-phase-3--install-red-hat-service-mesh--istiocni)
11. [Phase 4 — Deploy Istio with SPIRE](#11-phase-4--deploy-istio-with-spire)
12. [Phase 5 — Federated identities + East-West gateway](#12-phase-5--federated-identities--east-west-gateway)
13. [Phase 6 — Exchange remote secrets](#13-phase-6--exchange-remote-secrets)
14. [Phase 7 — Prove cross-cluster SPIRE mTLS](#14-phase-7--prove-cross-cluster-spire-mtls)
15. [Phase 8 — Add OPA authorization](#15-phase-8--add-opa-authorization)
16. [Phase 9 — Final success tests](#16-phase-9--final-success-tests)
17. [Quick map: who runs where](#17-quick-map-who-runs-where)
18. [Success checklist](#18-success-checklist)

---

## 1. Motive of this POC

Modern services need two answers for every request:

| Question | Layer | Tool in this POC |
|---|---|---|
| **Who are you?** (authentication) | Cryptographic identity | **SPIRE** (SPIFFE IDs / X.509 certs) |
| **Are you allowed?** (authorization) | Policy decision | **OPA** (Rego policy) |
| **How do services talk securely across clusters?** | Service mesh | **OSSM / Istio** (Envoy sidecars + mTLS) |

**Motive:** Show that on OpenShift you can:

1. Give every workload a strong SPIFFE identity with SPIRE (ZTWIM Operator).
2. Trust identities across **two clusters** (SPIRE federation).
3. Carry that identity in mesh mTLS with Red Hat Service Mesh.
4. Let **OPA** allow or deny requests using the caller’s SPIFFE ID — even when the caller is on another cluster.

This is zero-trust style: never trust by network location alone; trust identity + policy.

---

## 2. High-level workflow (read this first)

Same style as a nested-SPIRE story, but for **this** multi-cluster SPIRE + OSSM + OPA POC.

```text
PHASE 1: Identity Plane on Each Cluster (SPIRE / ZTWIM)
  1. Install ZTWIM Operator on Cluster 1 and Cluster 2
  2. Create SPIRE CRs on each cluster:
     - ZeroTrustWorkloadIdentityManager (trust domain + cluster name)
     - SpireServer (with federation.enabled=true, port 8443)
     - SpireAgent, SpiffeCSIDriver, SpireOIDCDiscoveryProvider
  3. SPIRE Server starts with its own Root CA for that trust domain
  4. OpenShift Route exposes federation endpoint:
     https://federation.apps.<cluster-domain>  (passthrough → :8443)
  5. Local Agents attest nodes (k8s_psat) and wait to issue workload SVIDs

PHASE 2: Cross-Cluster Trust (SPIRE Federation)
  6. On Cluster 1 create ClusterFederatedTrustDomain pointing to Cluster 2
     (className: zero-trust-workload-identity-manager-spire)
  7. On Cluster 2 create ClusterFederatedTrustDomain pointing to Cluster 1
  8. Controller creates federation relationship in each SPIRE Server
  9. Bootstrap peer bundles (https_spiffe needs an initial CA copy):
     curl peer federation URL → spire-server bundle set -format spiffe
 10. Result: each SPIRE trusts the other trust domain's CA
     Cluster 1 knows Cluster 2 Root CA (and vice versa)

PHASE 3: Mesh Control Plane + SPIRE Hookup (OSSM)
 11. Install Red Hat OpenShift Service Mesh Operator on both clusters
 12. Deploy IstioCNI on both clusters (node traffic redirect for sidecars)
 13. Deploy Istio CR on each cluster with:
     - meshConfig.trustDomain = that cluster's SPIRE trust domain
     - WORKLOAD_IDENTITY_SOCKET_FILE = spire-agent.sock
     - caCertificates for BOTH federation URLs
     - injection templates: spire (apps) + spireGw (gateways)
 14. Istiod starts; sidecars will get certs from SPIRE SDS (not Istio CA)

PHASE 4: Multi-Cluster Data Path Ready
 15. Create ClusterSPIFFEID with federatesWith for istio-system (+ sample later)
     so workloads get peer CA in ROOTCA bundle
 16. Install East-West gateway (Helm) with inject templates gateway,spireGw
 17. Create Gateway CR: TLS AUTO_PASSTHROUGH on port 15443, hosts "*.local"
 18. Exchange remote secrets (istioctl create-remote-secret both ways)
 19. Istiod on each side shows peer cluster STATUS=synced
 20. Mesh can discover remote services and send mTLS through EW gateways

PHASE 5: Prove AuthN Across Clusters (SPIRE mTLS smoke test)
 21. Label sample namespace for injection; create sample ClusterSPIFFEID
 22. Deploy helloworld on Cluster 2 (annotation: sidecar,spire)
 23. Deploy sleep + helloworld Service stub on Cluster 1 (sidecar,spire)
 24. From sleep: curl helloworld.sample:5000/hello
 25. Success = "Hello version: v1..."
     Meaning: Cluster 1 Envoy presented SPIRE SVID,
              Cluster 2 Envoy validated it using federated CA

PHASE 6: Add AuthZ with OPA (on the backend cluster)
 26. On Cluster 2 create OPA ConfigMap (Rego):
      default deny;
      allow GET only if SPIFFE ID == .../sa/frontend (Cluster 1)
 27. Deploy backend with 3 containers:
      - backend app
      - opa (envoy plugin, gRPC :8182)
      - istio-proxy (injected sidecar,spire)
 28. Create ServiceEntry opa-local → 127.0.0.1:8182
      + DestinationRule tls.mode=DISABLE
      (Istio CUSTOM authz needs a registry host; cannot use bare "localhost")
 29. Patch Istio extensionProviders:
      name opa-ext-authz-grpc → opa-local.sample.svc.cluster.local:8182
 30. Create AuthorizationPolicy action=CUSTOM on backend
      → Envoy must ask OPA before forwarding

PHASE 7: Callers + End-to-End Proof
 31. On Cluster 1 deploy:
      - frontend   (sa/frontend)     → should be ALLOWED
      - frontend-2 (sa/frontend-2)   → should be DENIED
      - backend Service stub (DNS only; pod lives on Cluster 2)
 32. Test from Cluster 1:
      frontend   GET  /headers → HTTP 200
      frontend-2 GET  /headers → HTTP 403
      frontend   POST /headers → HTTP 403
 33. OPA decision logs on Cluster 2 show:
      result:true  + URI=.../sa/frontend
      result:false + URI=.../sa/frontend-2
      result:false + method POST
 34. POC success:
      AuthN = SPIRE (who you are, across clusters)
      Mesh  = OSSM  (how traffic is carried with mTLS)
      AuthZ = OPA   (whether you may do this request)
```

### One request, end-to-end (after setup)

```text
frontend (Cluster 1)
  │  plain HTTP to backend.sample:8080
  ▼
frontend Envoy
  │  mTLS with SVID: spiffe://A/.../sa/frontend
  │  (cert from local SPIRE Agent SDS)
  ▼
Cluster 2 East-West Gateway :15443 (AUTO_PASSTHROUGH)
  ▼
backend Envoy (Cluster 2)
  │  validates client cert using federated Root CA from Cluster 1
  │  sets XFCC header with URI=spiffe://A/.../sa/frontend
  │  ext_authz gRPC → OPA sidecar :8182
  ▼
OPA Rego
  │  allow? method==GET AND SPIFFE==.../sa/frontend
  ├── YES → Envoy forwards to backend app → HTTP 200
  └── NO  → Envoy returns HTTP 403
```

---

## 3. Concepts (beginner friendly)

### AuthN vs AuthZ (read this — easy to confuse)

These two words look similar but mean different things. This whole POC is built on both.

| Short name | Full name | Question | In this POC |
|---|---|---|---|
| **AuthN** | **Authentication** | **Who are you?** | **SPIRE** (SPIFFE ID / certificate) |
| **AuthZ** | **Authorization** | **What are you allowed to do?** | **OPA** (Rego allow/deny) |

**AuthN (Authentication)** proves identity.

- SPIRE issues a short-lived certificate with a SPIFFE ID, for example:  
  `spiffe://apps....clu1.../ns/sample/sa/frontend`
- Envoy checks that certificate during mTLS.
- Result: “This request really is from `frontend` on Cluster 1.”

**AuthZ (Authorization)** decides permission **after** identity is known.

- OPA reads the caller’s SPIFFE ID (from the XFCC header).
- OPA applies the Rego policy (method + identity).
- Examples in this POC:
  - `frontend` + GET → **allow** → HTTP **200**
  - `frontend-2` + GET → **deny** → HTTP **403**
  - `frontend` + POST → **deny** → HTTP **403**

**Order of checks on a request:**

```text
Request arrives
    │
    ▼
AuthN (SPIRE + mTLS)     "Are you really who you claim?"     → identity
    │
    ▼
AuthZ (OPA)              "Is that identity allowed to do this?" → allow / deny
```

**Why you need both:**

- AuthN without AuthZ → any workload with a valid SPIRE cert could call anything.
- AuthZ without AuthN → policy has no trustworthy identity to decide on.

**Memory tip:**

```text
AuthN → iNentity (who)
AuthZ → aUthorize (allowed or not)
```

In one line for this POC:

```text
SPIRE  → AuthN → “This request is from sa/frontend on Cluster 1”
OPA    → AuthZ → “sa/frontend may GET; sa/frontend-2 may not”
OSSM   → carries the mTLS traffic that makes AuthN possible across clusters
```

### SPIFFE / SPIRE

- **SPIFFE ID** = a URI identity for a workload, for example:  
  `spiffe://apps.cluster1.example.com/ns/sample/sa/frontend`
- **SPIRE** = the system that issues short-lived certificates (SVIDs) containing that SPIFFE ID. This is the **AuthN** engine.
- **Trust domain** = the “root name” of identities on a cluster (often the OpenShift apps domain).
- **ZTWIM** = Red Hat Zero Trust Workload Identity Manager Operator that installs/manages SPIRE on OpenShift.

### SPIRE federation

Each cluster has its own SPIRE CA. Federation means:

- Cluster 1 learns Cluster 2’s CA bundle.
- Cluster 2 learns Cluster 1’s CA bundle.

Then Envoy on Cluster 2 can validate a certificate issued by Cluster 1’s SPIRE (AuthN still works across clusters).

### OSSM (OpenShift Service Mesh / Istio)

- **Istiod** = control plane (config + discovery).
- **Sidecar (Envoy)** = data plane proxy next to each app.
- **IstioCNI** = node helper so sidecars can intercept traffic on OpenShift.
- **East-West gateway** = entry point for cross-cluster mesh traffic (port `15443`).
- **Remote secret** = lets Istiod on one cluster discover services on the other.

In this POC, Envoy gets certificates from **SPIRE SDS**, not from Istio’s own CA:

```text
WORKLOAD_IDENTITY_SOCKET_FILE=spire-agent.sock
```

OSSM does not replace AuthN/AuthZ — it is the **transport path** that carries SPIRE identities with mTLS and then asks OPA for AuthZ.

### OPA (Open Policy Agent)

- OPA is the **AuthZ** engine: answers allow/deny using a **Rego** policy.
- Envoy asks OPA over gRPC (ext_authz) before forwarding to the app.
- In Istio this is wired with:
  - `meshConfig.extensionProviders`
  - `AuthorizationPolicy` with `action: CUSTOM`

OPA does **not** issue identities. It only decides after SPIRE/Envoy have already authenticated the caller.
---

## 4. What we prove

| Gate | Proof |
|---|---|
| SPIRE up | SPIRE pods Running on both clusters |
| Federation | Each SPIRE has the peer trust bundle |
| Mesh + SPIRE | Cross-cluster `sleep → helloworld` works |
| OPA allow | `frontend` GET → **HTTP 200** |
| OPA deny (wrong identity) | `frontend-2` GET → **HTTP 403** |
| OPA deny (wrong method) | `frontend` POST → **HTTP 403** |
| Decision evidence | OPA logs show `result: true/false` + SPIFFE IDs |

---

## 5. Architecture

```text
CLUSTER 1 (A)                                      CLUSTER 2 (B)
Trust domain: apps....clu1...                      Trust domain: apps....clu2...

┌────────────────────────────┐                     ┌─────────────────────────────────┐
│ frontend (sa/frontend)     │   mTLS via EW GW    │ backend pod                     │
│ frontend-2 (sa/frontend-2) │ ──────────────────► │  Envoy ──asks──► OPA sidecar    │
│ sleep (mTLS smoke test)    │                     │                 │               │
│                            │                     │                 ▼               │
│ SPIRE Agent (SDS)          │                     │            Rego allow/deny      │
└────────────────────────────┘                     │  Envoy ──if allow──► backend app│
                                                   │ SPIRE Agent (SDS)               │
                                                   └─────────────────────────────────┘
         ▲                                                       ▲
         └──────── SPIRE Servers exchange trust bundles ─────────┘
```

**Request path (OPA test):**

1. `frontend` on Cluster 1 calls `http://backend.sample:8080/headers`
2. Frontend Envoy starts mTLS with SPIRE cert (`sa/frontend`)
3. Traffic goes to Cluster 2 East-West gateway → backend Envoy
4. Backend Envoy sets `x-forwarded-client-cert` (XFCC) with caller SPIFFE URI
5. Backend Envoy asks OPA: allow?
6. OPA returns allow → backend app answers; deny → HTTP 403

---

## 6. Prerequisites

- Two OpenShift clusters with `system:admin` access
- `oc`, `helm`, `istioctl`, `jq`, `curl` on your laptop
- Clusters can reach each other’s federation Routes and East-West LoadBalancers
- OperatorHub `redhat-operators` available

Example kubeconfigs used in the validated POC:

```bash
export CLUSTER1=/path/to/cluster1/aws/auth/kubeconfig   # Cluster A
export CLUSTER2=/path/to/cluster2/aws/auth/kubeconfig   # Cluster B
```

---

## 7. Environment variables

Run this on your laptop once. Replace domains if yours differ.

```bash
export CLUSTER1=/path/to/cluster1/kubeconfig
export CLUSTER2=/path/to/cluster2/kubeconfig

export ZTWIM_NS=zero-trust-workload-identity-manager
export OSSM_NS=istio-system
export SAMPLE_NS=sample

# Discover domains
export CLUSTER_A_BASE_DOMAIN=$(oc get ingresses.config/cluster -o jsonpath='{.spec.domain}' --kubeconfig="$CLUSTER1")
export CLUSTER_B_BASE_DOMAIN=$(oc get ingresses.config/cluster -o jsonpath='{.spec.domain}' --kubeconfig="$CLUSTER2")

export CLUSTER_A_TRUST_DOMAIN="${CLUSTER_A_BASE_DOMAIN}"
export CLUSTER_B_TRUST_DOMAIN="${CLUSTER_B_BASE_DOMAIN}"

export CLUSTER_A=cluster-a
export CLUSTER_B=cluster-b
export NETWORK_A=network-a
export NETWORK_B=network-b

export FEDERATION_ENDPOINT_A="https://federation.${CLUSTER_A_BASE_DOMAIN}"
export FEDERATION_ENDPOINT_B="https://federation.${CLUSTER_B_BASE_DOMAIN}"
export JWT_ISSUER_A="https://oidc-discovery.${CLUSTER_A_BASE_DOMAIN}"
export JWT_ISSUER_B="https://oidc-discovery.${CLUSTER_B_BASE_DOMAIN}"

echo "A=$CLUSTER_A_TRUST_DOMAIN"
echo "B=$CLUSTER_B_TRUST_DOMAIN"
```

**Validated POC domains (example):**

```text
A = apps.<your-prefix>-clu1.qe.devcluster.openshift.com
B = apps.<your-prefix>-clu2.qe.devcluster.openshift.com
```

---

## 8. Phase 1 — Install ZTWIM + SPIRE on both clusters

### 8.1 Install ZTWIM Operator (do on Cluster 1, then Cluster 2)

**What it does:** Installs the operator that will create SPIRE components.

```bash
export KUBECONFIG="$CLUSTER1"   # then repeat with $CLUSTER2

oc apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: zero-trust-workload-identity-manager
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
  channel: tech-preview-v0.2
  name: openshift-zero-trust-workload-identity-manager
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

oc get csv -n ${ZTWIM_NS}
oc get po -n ${ZTWIM_NS}
```

**Expected:**

```text
CSV ... Succeeded
zero-trust-workload-identity-manager-controller-manager-...  1/1  Running
```

### 8.2 Deploy SPIRE with federation enabled

**Cluster 1** (use A variables):

```bash
export KUBECONFIG="$CLUSTER1"

oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: ZeroTrustWorkloadIdentityManager
metadata:
  name: cluster
spec:
  trustDomain: ${CLUSTER_A_TRUST_DOMAIN}
  clusterName: ${CLUSTER_A}
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  trustDomain: ${CLUSTER_A_TRUST_DOMAIN}
  clusterName: ${CLUSTER_A}
  caSubject:
    commonName: ${CLUSTER_A_TRUST_DOMAIN}
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
  jwtIssuer: ${JWT_ISSUER_A}
  federation:
    enabled: true
    bundleEndpoint:
      address: "0.0.0.0"
      port: 8443
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireAgent
metadata:
  name: cluster
spec:
  trustDomain: ${CLUSTER_A_TRUST_DOMAIN}
  clusterName: ${CLUSTER_A}
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
  trustDomain: ${CLUSTER_A_TRUST_DOMAIN}
  jwtIssuer: ${JWT_ISSUER_A}
EOF
```

**Cluster 2:** same YAML with `CLUSTER_B_*` / `JWT_ISSUER_B`.

Wait:

```bash
oc rollout status statefulset/spire-server -n ${ZTWIM_NS} --timeout=300s
oc rollout status daemonset/spire-agent -n ${ZTWIM_NS} --timeout=300s
oc rollout status daemonset/spire-spiffe-csi-driver -n ${ZTWIM_NS} --timeout=300s
oc wait --for=condition=Available deployment/spire-spiffe-oidc-discovery-provider -n ${ZTWIM_NS} --timeout=300s
oc get po -n ${ZTWIM_NS}
```

**Expected pods (each cluster):**

```text
spire-server-0                               2/2  Running
spire-agent-...                              1/1  Running   (one per node)
spire-spiffe-csi-driver-...                  2/2  Running
spire-spiffe-oidc-discovery-provider-...     1/1  Running
controller-manager-...                       1/1  Running
```

**Expected federation Route:**

```bash
oc get route -n ${ZTWIM_NS} | grep federation
```

```text
spire-server-federation   federation.apps.<your-domain>   ...   passthrough
```

---

## 9. Phase 2 — Federate the two SPIRE trust domains

### 9.1 Create ClusterFederatedTrustDomain (CFTD)

**What it does:** Tells local SPIRE “trust the other cluster and fetch its bundle.”

> Important: always set `className: zero-trust-workload-identity-manager-spire` or the controller ignores the CR.

**Cluster 1 → points to Cluster 2:**

```bash
export KUBECONFIG="$CLUSTER1"

oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: federation-to-cluster-b
spec:
  className: zero-trust-workload-identity-manager-spire
  trustDomain: ${CLUSTER_B_TRUST_DOMAIN}
  bundleEndpointURL: ${FEDERATION_ENDPOINT_B}
  bundleEndpointProfile:
    type: https_spiffe
    endpointSPIFFEID: spiffe://${CLUSTER_B_TRUST_DOMAIN}/spire/server
EOF
```

**Cluster 2 → points to Cluster 1:**

```bash
export KUBECONFIG="$CLUSTER2"

oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: federation-to-cluster-a
spec:
  className: zero-trust-workload-identity-manager-spire
  trustDomain: ${CLUSTER_A_TRUST_DOMAIN}
  bundleEndpointURL: ${FEDERATION_ENDPOINT_A}
  bundleEndpointProfile:
    type: https_spiffe
    endpointSPIFFEID: spiffe://${CLUSTER_A_TRUST_DOMAIN}/spire/server
EOF
```

**Expected:**

```bash
oc get clusterfederatedtrustdomain
```

```text
NAME                      TRUST DOMAIN        ENDPOINT URL
federation-to-cluster-b   apps....clu2...     https://federation.apps....clu2...
```

```bash
oc exec -n ${ZTWIM_NS} spire-server-0 -c spire-server -- \
  /spire-server federation list -socketPath /tmp/spire-server/private/api.sock
```

```text
Found 1 federation relationship
Trust domain : apps....<peer>...
```

> Note: SPIRE binary path in this image is `/spire-server` (not on `$PATH`).

### 9.2 Bootstrap peer bundles (required for `https_spiffe`)

**What it does:** Seeds the peer CA so SPIFFE auth to the federation endpoint can start.

**On Cluster 1:**

```bash
export KUBECONFIG="$CLUSTER1"

oc exec -n ${ZTWIM_NS} spire-server-0 -c spire-server -- \
  curl -sk -o /tmp/cluster2-bundle.json "${FEDERATION_ENDPOINT_B}"

oc exec -n ${ZTWIM_NS} spire-server-0 -c spire-server -- \
  /spire-server bundle set \
  -id ${CLUSTER_B_TRUST_DOMAIN} \
  -path /tmp/cluster2-bundle.json \
  -format spiffe \
  -socketPath /tmp/spire-server/private/api.sock

oc exec -n ${ZTWIM_NS} spire-server-0 -c spire-server -- \
  /spire-server bundle list -socketPath /tmp/spire-server/private/api.sock
```

**On Cluster 2:** same with Cluster 1 endpoint / trust domain.

**Expected:**

```text
bundle set.
****************************************
* apps....<peer-domain>...
****************************************
-----BEGIN CERTIFICATE-----
...
```

---

## 10. Phase 3 — Install Red Hat Service Mesh + IstioCNI

### 10.1 Service Mesh Operator (both clusters)

**What it does:** Installs the operator that can create `Istio` / `IstioCNI` CRs. It does **not** create the mesh yet.

```bash
export KUBECONFIG="$CLUSTER1"   # then $CLUSTER2

oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: servicemeshoperator3
  namespace: openshift-operators
spec:
  channel: stable
  name: servicemeshoperator3
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

oc get csv -n openshift-operators | grep -i servicemesh
```

**Expected:**

```text
servicemeshoperator3.v3.4.0   Red Hat OpenShift Service Mesh 3   ...   Succeeded
```

### 10.2 IstioCNI (both clusters)

**What it does:** Node DaemonSet that configures pod networking so sidecars can intercept traffic.

```bash
export KUBECONFIG="$CLUSTER1"   # then $CLUSTER2

oc new-project istio-cni 2>/dev/null || true

oc apply -f - <<EOF
apiVersion: sailoperator.io/v1
kind: IstioCNI
metadata:
  name: default
spec:
  namespace: istio-cni
  version: v1.30.1
EOF

until oc get daemonset/istio-cni-node -n istio-cni &>/dev/null; do sleep 3; done
oc rollout status daemonset/istio-cni-node -n istio-cni --timeout=300s
oc get po -n istio-cni
```

**Expected:**

```text
istio-cni-node-...   1/1  Running   (one per node)
```

---

## 11. Phase 4 — Deploy Istio with SPIRE

**What it does:** Creates Istiod and configures the mesh to:

- use SPIRE socket for workload certs
- trust both SPIRE federation bundle URLs
- provide injection templates `spire` and `spireGw`

### Cluster 1 Istio CR

```bash
export KUBECONFIG="$CLUSTER1"

export EXTRA_ROOT_CA_A="$(oc get secret oidc-serving-cert -n ${ZTWIM_NS} -o json | \
  jq -r '.data."tls.crt"' | base64 -d | sed 's/^/        /')"

oc new-project istio-system 2>/dev/null || true

cat <<EOF | oc apply -f -
apiVersion: sailoperator.io/v1
kind: Istio
metadata:
  name: default
spec:
  namespace: istio-system
  version: v1.30.1
  updateStrategy:
    type: InPlace
  values:
    meshConfig:
      trustDomain: ${CLUSTER_A_TRUST_DOMAIN}
      defaultConfig:
        proxyMetadata:
          WORKLOAD_IDENTITY_SOCKET_FILE: "spire-agent.sock"
      caCertificates:
        - spiffeBundleUrl: ${FEDERATION_ENDPOINT_A}
          trustDomains:
            - ${CLUSTER_A_TRUST_DOMAIN}
        - spiffeBundleUrl: ${FEDERATION_ENDPOINT_B}
          trustDomains:
            - ${CLUSTER_B_TRUST_DOMAIN}
    global:
      meshID: mesh1
      multiCluster:
        clusterName: ${CLUSTER_A}
      network: ${NETWORK_A}
    pilot:
      jwksResolverExtraRootCA: |
${EXTRA_ROOT_CA_A}
      env:
        ENABLE_CA_SERVER: "true"
    sidecarInjectorWebhook:
      templates:
        spire: |
          spec:
            initContainers:
            - name: istio-proxy
              volumeMounts:
              - name: workload-socket
                mountPath: /run/secrets/workload-spiffe-uds
                readOnly: true
            volumes:
              - name: workload-socket
                csi:
                  driver: "csi.spiffe.io"
                  readOnly: true
        spireGw: |
          spec:
            containers:
            - name: istio-proxy
              volumeMounts:
              - name: workload-socket
                mountPath: /run/secrets/workload-spiffe-uds
                readOnly: true
            volumes:
              - name: workload-socket
                csi:
                  driver: "csi.spiffe.io"
                  readOnly: true
EOF

until oc get deployment istiod -n istio-system &>/dev/null; do sleep 3; done
oc wait --for=condition=Available deployment/istiod -n istio-system --timeout=300s
oc get po -n istio-system
```

### Cluster 2 Istio CR

Same structure with B trust domain / network / `EXTRA_ROOT_CA_B`, and `caCertificates` order can start with B then A.

**Expected:**

```text
istiod-...   1/1  Running
```

---

## 12. Phase 5 — Federated identities + East-West gateway

### 12.1 ClusterSPIFFEID for `istio-system` and later `sample`

**What it does:** Makes matching workloads receive the **peer** trust domain in their CA bundle (`federatesWith`).

**Cluster 1:**

```bash
export KUBECONFIG="$CLUSTER1"

oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: istio-system-federation
spec:
  className: zero-trust-workload-identity-manager-spire
  spiffeIDTemplate: "spiffe://{{ .TrustDomain }}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: istio-system
  federatesWith:
    - ${CLUSTER_B_TRUST_DOMAIN}
EOF
```

**Cluster 2:** same with `federatesWith: [${CLUSTER_A_TRUST_DOMAIN}]`.

### 12.2 East-West gateway

**What it does:** Exposes port `15443` for cross-cluster mTLS passthrough.

On laptop once:

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

**Cluster 1:**

```bash
export KUBECONFIG="$CLUSTER1"

# Let Helm create the ServiceAccount, then grant SCC
helm upgrade --install istio-eastwestgateway istio/gateway \
  -n istio-system \
  --set-json 'podAnnotations={"inject.istio.io/templates":"gateway,spireGw"}' \
  --set name=istio-eastwestgateway \
  --set networkGateway=network-a \
  --kubeconfig="$KUBECONFIG"

oc adm policy add-scc-to-user anyuid -z istio-eastwestgateway -n istio-system

oc apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: cross-network-gateway
  namespace: istio-system
spec:
  selector:
    istio: eastwestgateway
  servers:
  - port:
      number: 15443
      name: tls
      protocol: TLS
    tls:
      mode: AUTO_PASSTHROUGH
    hosts:
    - "*.local"
EOF

oc wait --for=condition=Available deployment/istio-eastwestgateway -n istio-system --timeout=300s
oc get po,svc -n istio-system
```

**Cluster 2:** same with `networkGateway=network-b`.

**Expected:**

```text
istio-eastwestgateway-...   1/1  Running
istio-eastwestgateway   LoadBalancer   ...   <EXTERNAL-HOSTNAME>   15443:...
```

---

## 13. Phase 6 — Exchange remote secrets

**What it does:** Each Istiod can watch the other cluster’s API for service discovery.

From laptop:

```bash
istioctl create-remote-secret \
  --kubeconfig="$CLUSTER1" \
  --name=cluster-a \
  --istioNamespace=istio-system | \
  oc apply --kubeconfig="$CLUSTER2" -f -

istioctl create-remote-secret \
  --kubeconfig="$CLUSTER2" \
  --name=cluster-b \
  --istioNamespace=istio-system | \
  oc apply --kubeconfig="$CLUSTER1" -f -

istioctl remote-clusters --kubeconfig="$CLUSTER1"
istioctl remote-clusters --kubeconfig="$CLUSTER2"
```

**Expected:**

```text
NAME        SECRET                                      STATUS   ISTIOD
cluster-a                                               synced   istiod-...
cluster-b   istio-system/istio-remote-secret-cluster-b  synced   istiod-...
```

(and the mirror on Cluster 2)

---

## 14. Phase 7 — Prove cross-cluster SPIRE mTLS

### 14.1 Sample namespace + federated ClusterSPIFFEID

**Both clusters:**

```bash
export KUBECONFIG="$CLUSTER1"   # then $CLUSTER2

oc new-project sample 2>/dev/null || true
oc label namespace sample istio-injection=enabled --overwrite
```

**Cluster 1 sample SPIFFEID** (`federatesWith` = B):

```bash
export KUBECONFIG="$CLUSTER1"
oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: sample-federation
spec:
  className: zero-trust-workload-identity-manager-spire
  spiffeIDTemplate: "spiffe://{{ .TrustDomain }}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: sample
  federatesWith:
    - ${CLUSTER_B_TRUST_DOMAIN}
EOF
```

**Cluster 2 sample SPIFFEID** (`federatesWith` = A): mirror with A trust domain.

### 14.2 Workloads

**Cluster 2 — helloworld server:**

```bash
export KUBECONFIG="$CLUSTER2"

oc apply -n sample -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/helloworld/helloworld.yaml -l service=helloworld
oc apply -n sample -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/helloworld/helloworld.yaml -l version=v1
oc patch deploy helloworld-v1 -n sample --type='merge' \
  -p '{"spec":{"template":{"metadata":{"annotations":{"inject.istio.io/templates":"sidecar,spire"}}}}}'
oc rollout status deploy/helloworld-v1 -n sample --timeout=300s
```

**Cluster 1 — sleep client + helloworld Service stub:**

```bash
export KUBECONFIG="$CLUSTER1"

oc apply -n sample -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/helloworld/helloworld.yaml -l service=helloworld
oc apply -n sample -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/sleep/sleep.yaml
oc patch deploy sleep -n sample --type='merge' \
  -p '{"spec":{"template":{"metadata":{"annotations":{"inject.istio.io/templates":"sidecar,spire"}}}}}'
oc rollout status deploy/sleep -n sample --timeout=300s
```

### 14.3 mTLS smoke test

```bash
export KUBECONFIG="$CLUSTER1"
oc exec deploy/sleep -n sample -c sleep -- curl -sS helloworld.sample:5000/hello
```

**Expected:**

```text
Hello version: v1, instance: helloworld-v1-...
```

**Also expect pods `2/2`** (app + istio-proxy) on both sides.

Only after this works, continue to OPA.

---

## 15. Phase 8 — Add OPA authorization

All OPA control-plane pieces run on **Cluster 2**. Callers run on **Cluster 1**.

### 15.1 OPA ConfigMap + Rego (Cluster 2)

**What it does:** Deny by default; allow only GET from Cluster 1 `sa/frontend`.

Replace the SPIFFE domain in the policy with your Cluster 1 trust domain.

```bash
export KUBECONFIG="$CLUSTER2"

oc apply -n sample -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: opa-policy
data:
  opa-config.yaml: |
    decision_logs:
      console: true
    plugins:
      envoy_ext_authz_grpc:
        addr: :8182
        query: data.envoy.authz.allow
  opa-policy.rego: |
    package envoy.authz

    import input.attributes.request.http as http_request

    default allow = false

    allow {
      http_request.method == "GET"
      # Replace with your Cluster 1 trust domain (echo $CLUSTER_A_TRUST_DOMAIN)
      svc_spiffe_id == "spiffe://<CLUSTER_A_TRUST_DOMAIN>/ns/sample/sa/frontend"
    }

    # XFCC looks like: By=...;Hash=...;Subject="...";URI=spiffe://...
    svc_spiffe_id = id {
      xfcc := http_request.headers["x-forwarded-client-cert"]
      id := regex.find_all_string_submatch_n(`URI=([^;]+)`, xfcc, 1)[0][1]
    }
EOF
```
### 15.2 Backend + OPA sidecar (Cluster 2)

**What it does:** Runs app + OPA + Istio/SPIRE sidecar in one pod.

```bash
export KUBECONFIG="$CLUSTER2"

oc apply -n sample -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  labels:
    app: backend
spec:
  ports:
    - name: http
      port: 8080
      targetPort: 8080
  selector:
    app: backend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      annotations:
        inject.istio.io/templates: "sidecar,spire"
      labels:
        app: backend
    spec:
      serviceAccountName: backend
      containers:
        - name: backend
          image: docker.io/mccutchen/go-httpbin:v2.15.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
        - name: opa
          image: openpolicyagent/opa:0.70.0-envoy
          imagePullPolicy: IfNotPresent
          ports:
            - name: opa-envoy
              containerPort: 8182
            - name: opa-api
              containerPort: 8181
          args:
            - "run"
            - "--server"
            - "--config-file=/run/opa/opa-config.yaml"
            - "/run/opa/opa-policy.rego"
          volumeMounts:
            - name: opa-policy
              mountPath: /run/opa
              readOnly: true
      volumes:
        - name: opa-policy
          configMap:
            name: opa-policy
EOF

oc wait --for=condition=Available deployment/backend -n sample --timeout=300s
oc get pods -l app=backend -n sample
```

**Expected:**

```text
backend-...   3/3  Running
```

(containers: `backend`, `opa`, `istio-proxy`)

### 15.3 Register OPA in Istio service registry (Cluster 2)

**What it does:** Istio CUSTOM authz needs a real registry host. Sidecar OPA is reached as `127.0.0.1` via ServiceEntry.

```bash
export KUBECONFIG="$CLUSTER2"

oc apply -n sample -f - <<EOF
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: opa-local
spec:
  hosts:
  - opa-local.sample.svc.cluster.local
  ports:
  - number: 8182
    name: grpc
    protocol: GRPC
  location: MESH_INTERNAL
  resolution: STATIC
  endpoints:
  - address: 127.0.0.1
---
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: opa-local
spec:
  host: opa-local.sample.svc.cluster.local
  trafficPolicy:
    tls:
      mode: DISABLE
EOF
```

### 15.4 Extension provider + AuthorizationPolicy (Cluster 2)

```bash
export KUBECONFIG="$CLUSTER2"

oc patch istio default --type=merge -p '{
  "spec": {
    "values": {
      "meshConfig": {
        "extensionProviders": [
          {
            "name": "opa-ext-authz-grpc",
            "envoyExtAuthzGrpc": {
              "service": "opa-local.sample.svc.cluster.local",
              "port": 8182
            }
          }
        ]
      }
    }
  }
}'

oc apply -n sample -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: opa-ext-authz
spec:
  selector:
    matchLabels:
      app: backend
  action: CUSTOM
  provider:
    name: opa-ext-authz-grpc
  rules:
    - to:
        - operation:
            paths: ["/*"]
EOF

oc wait --for=condition=Available deployment/istiod -n istio-system --timeout=120s
oc rollout restart deployment/backend -n sample
oc rollout status deployment/backend -n sample --timeout=300s
```

**Expected:**

```bash
oc get authorizationpolicy -n sample
```

```text
NAME            ACTION   AGE
opa-ext-authz   CUSTOM   ...
```

```bash
oc get istio default -o jsonpath='{.spec.values.meshConfig.extensionProviders}' | python3 -m json.tool
```

```json
[
  {
    "name": "opa-ext-authz-grpc",
    "envoyExtAuthzGrpc": {
      "service": "opa-local.sample.svc.cluster.local",
      "port": 8182
    }
  }
]
```

### 15.5 Frontends + backend Service stub (Cluster 1)

```bash
export KUBECONFIG="$CLUSTER1"

oc apply -n sample -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: backend
  labels:
    app: backend
spec:
  ports:
    - name: http
      port: 8080
      targetPort: 8080
  selector:
    app: backend
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: frontend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      annotations:
        inject.istio.io/templates: "sidecar,spire"
      labels:
        app: frontend
    spec:
      serviceAccountName: frontend
      terminationGracePeriodSeconds: 0
      containers:
        - name: frontend
          image: curlimages/curl:8.16.0
          command: ["/bin/sh", "-c", "sleep inf"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: frontend-2
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend-2
  template:
    metadata:
      annotations:
        inject.istio.io/templates: "sidecar,spire"
      labels:
        app: frontend-2
    spec:
      serviceAccountName: frontend-2
      terminationGracePeriodSeconds: 0
      containers:
        - name: frontend-2
          image: curlimages/curl:8.16.0
          command: ["/bin/sh", "-c", "sleep inf"]
EOF

oc wait --for=condition=Available deployment/frontend -n sample --timeout=300s
oc wait --for=condition=Available deployment/frontend-2 -n sample --timeout=300s
oc get po -n sample
```

**Expected:**

```text
frontend-...     2/2  Running
frontend-2-...   2/2  Running
```

---

## 16. Phase 9 — Final success tests

### 16.1 Authorization tests (Cluster 1)

```bash
export KUBECONFIG="$CLUSTER1"

oc exec deploy/frontend -n sample -c frontend -- \
  curl -s -o /dev/null -w "frontend GET → HTTP %{http_code}\n" \
  http://backend.sample:8080/headers

oc exec deploy/frontend-2 -n sample -c frontend-2 -- \
  curl -s -o /dev/null -w "frontend-2 GET → HTTP %{http_code}\n" \
  http://backend.sample:8080/headers

oc exec deploy/frontend -n sample -c frontend -- \
  curl -s -o /dev/null -w "frontend POST → HTTP %{http_code}\n" \
  -X POST http://backend.sample:8080/headers
```

**Expected output:**

```text
frontend GET → HTTP 200
frontend-2 GET → HTTP 403
frontend POST → HTTP 403
```

### 16.2 OPA decision logs (Cluster 2)

```bash
export KUBECONFIG="$CLUSTER2"
oc logs deploy/backend -n sample -c opa --tail=40
```

**Expected evidence in logs:**

| Call | SPIFFE URI in XFCC | `result` |
|---|---|---|
| frontend GET | `.../sa/frontend` | `true` |
| frontend-2 GET | `.../sa/frontend-2` | `false` |
| frontend POST | `.../sa/frontend` | `false` |

Example (abbreviated):

```text
... "method":"GET" ... "URI=spiffe://.../sa/frontend" ... "result":true
... "method":"GET" ... "URI=spiffe://.../sa/frontend-2" ... "result":false
... "method":"POST" ... "URI=spiffe://.../sa/frontend" ... "result":false
```

You will also see:

- `source.principal: spiffe://<cluster-a>/ns/sample/sa/frontend...`
- `destination.principal: spiffe://<cluster-b>/ns/sample/sa/backend`

That confirms **SPIRE AuthN across clusters** + **OPA AuthZ**.

---

## 17. Quick map: who runs where

| Component | Cluster 1 | Cluster 2 |
|---|---|---|
| ZTWIM + SPIRE | yes | yes |
| CFTD pointing to peer | yes | yes |
| Service Mesh Operator + IstioCNI + Istio | yes | yes |
| East-West gateway | yes | yes |
| Remote secret of peer | receives B | receives A |
| helloworld / backend+OPA | — | yes |
| sleep / frontend / frontend-2 | yes | — |
| OPA policy + AuthzPolicy | — | yes |

---

## 18. Success checklist

Use this as your “POC passed” gate:

- [ ] SPIRE pods Running on both clusters
- [ ] `federation list` shows 1 relationship on both
- [ ] `bundle list` shows peer trust domain + certificate on both
- [ ] Service Mesh CSV `Succeeded`; IstioCNI DaemonSet Ready; Istiod Ready
- [ ] East-West gateway Ready with EXTERNAL-IP/hostname and port 15443
- [ ] `istioctl remote-clusters` shows peer **synced** on both
- [ ] `curl helloworld.sample:5000/hello` from sleep returns Hello v1
- [ ] Backend pod is **3/3**
- [ ] OPA tests: **200 / 403 / 403**
- [ ] OPA logs show matching `result: true/false` with correct SPIFFE IDs

When all boxes are checked, the POC has demonstrated:

> **Multi-cluster SPIRE identity + OSSM mTLS + OPA policy enforcement.**

---

## One-line mental model

```text
SPIRE says who you are
OSSM carries that identity safely across clusters
OPA decides what you may do
```
