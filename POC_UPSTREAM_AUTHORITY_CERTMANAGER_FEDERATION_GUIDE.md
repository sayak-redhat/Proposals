# UpstreamAuthority (cert-manager) + Cross-Cluster Federation POC

**What this proves (PR #113):**
1. SPIRE’s CA is signed by cert-manager → logs show `self_signed=false`
2. Two OpenShift clusters trust each other → mTLS shows `Verify return code: 0 (ok)`

**How to use this guide:**
1. Open **2 terminals** (Cluster1 and Cluster2)
2. Run **Step 0** in **both** terminals first — it sets `KUBECONFIG1` / `KUBECONFIG2` for the rest of the guide
3. For every step: click the **copy** icon on the grey `bash` code box (top-right of the block) — no need to highlight/select text
4. Paste into the terminal with Ctrl+Shift+V (or right-click Paste)
5. Run steps **in order**; only continue when **Expected** matches (`READY` / `RouteAvailable=True` / `LE SECRET READY`)

> Every command box re-exports `KUBECONFIG` from `$KUBECONFIG1` or `$KUBECONFIG2` (set once in Step 0). One click copy → paste → Enter.

**Recording-safe rule (read this):**
> On Cluster1, create the Let's Encrypt TLS **secret first**, and only then deploy SpireServer with `https_web`.  
> If you enable `https_web` before the secret exists, you will see a scary (but expected)  
> `RouteAvailable=False` / `secrets "spire-server-federation-tls" not found`.  
> This guide’s order avoids that on camera.
>
> **PAUSE instruction (important for recording):**  
> After Step 4, **pause the recording until the terminal prints `LE SECRET READY`**.  
> Do **not** start Step 6 (Cluster1 SpireServer) until you see that line.

**Prerequisites:**
- Two OpenShift 4.x clusters with `cluster-admin` access
- `oc` CLI installed and configured
- `openssl`, `jq`, and `curl` available locally
- Both clusters must have publicly routable `*.apps.<baseDomain>` (for Let's Encrypt HTTP-01)

**Tested with:**
- ZTWIM with `upstreamAuthority` CRD field
- Hybrid federation: Cluster1 `https_web` (Let's Encrypt) + Cluster2 `https_spiffe`

---

## Easy concepts (read once)

### What is UpstreamAuthority?

Without it, SPIRE creates its own CA (`self_signed=true`).  
With it, **cert-manager** signs SPIRE’s CA (`self_signed=false`).

```
cert-manager Root CA ("SPIRE Upstream CA")
        │ signs
        ▼
SPIRE Intermediate CA
        │ signs
        ▼
Workload SVID (pod identity certificate)
```

### Why a CA chain in cert-manager?

cert-manager is only a certificate robot. It needs a stamp (CA key) before it can sign SPIRE’s CA.

We create 3 objects:
1. `selfsigned-issuer` — bootstrap tool (creates the first root)
2. `spire-ca` Certificate — the root CA (`isCA: true`)
3. `spire-ca-issuer` — day-to-day issuer SPIRE calls

### Why Let's Encrypt?

Only for Cluster1 federation **website TLS** (`https_web`), so the other cluster can fetch the trust bundle without `curl -k`.

Let's Encrypt **cannot** sign SPIRE’s CA (it only issues website leaf certs).

**Recording order:** create the LE Certificate/secret in Step 4 **before** Cluster1 SpireServer `https_web` in Step 6.

### Federation profiles in this POC

| Cluster | Profile | Federation TLS |
|---------|---------|----------------|
| Cluster1 | `https_web` | Let's Encrypt |
| Cluster2 | `https_spiffe` | SPIRE self-signed |

### Architecture

```
CLUSTER1                              CLUSTER2
cert-manager CA                       cert-manager CA
   │                                     │
   ▼                                     ▼
SPIRE (self_signed=false)             SPIRE (self_signed=false)
Federation: https_web + LE            Federation: https_spiffe
   │                                     │
   └──────── exchange trust bundles ─────┘
                     │
         mTLS client (C2) → server (C1)
              Verify return code: 0
```

### Happy-path order for recording (no failures)

```
Step 1  cert-manager (both)
Step 2  Upstream CA chain (both)
Step 3  ZTWIM operator + namespace (both)
Step 4  Let's Encrypt Issuer + Certificate on Cluster1  ← WAIT until secret Ready
Step 5  Deploy Cluster2 SpireServer (https_spiffe)      ← no LE needed
Step 6  Deploy Cluster1 SpireServer (https_web)         ← only after Step 4 gate
Step 7  Verify UpstreamAuthority
Step 8  Federation bootstrap
Step 9  Cross-cluster mTLS
```

Do **not** apply Cluster1 `https_web` SpireServer until Step 4 prints `LE SECRET READY`.

> **For recording:** pause the video at the end of Step 4 until you see:
> ```
> LE SECRET READY
> ```
> Only then continue to Step 5 (Cluster2) and Step 6 (Cluster1).

---

## Step 0 — Fill your cluster details (ONCE)

> **Edit the two paths below** to point to your actual kubeconfig files, then
> one-click **copy** the box (icon at top-right of the code block). Paste into **both** terminals once.

```bash
########################################################################
# EDIT THESE TWO LINES — point to your own kubeconfig files
########################################################################
export KUBECONFIG1="/path/to/cluster1.kubeconfig"
export KUBECONFIG2="/path/to/cluster2.kubeconfig"
########################################################################

export KUBECONFIG="$KUBECONFIG1"
echo "Cluster1: $(oc whoami) API=$(oc config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
export CLUSTER1_APP_DOMAIN="apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')"
echo "CLUSTER1_APP_DOMAIN=$CLUSTER1_APP_DOMAIN"

export KUBECONFIG="$KUBECONFIG2"
echo "Cluster2: $(oc whoami) API=$(oc config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
export CLUSTER2_APP_DOMAIN="apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')"
echo "CLUSTER2_APP_DOMAIN=$CLUSTER2_APP_DOMAIN"
```

**Expected:** `system:admin` on both, real domains (not `apps.` alone), API hosts are your clusters.

---

## Step 1 — Install cert-manager (BOTH clusters)

**What:** Installs the cert-manager operator.  
**Why:** We need it to create the CA chain and (on Cluster1) talk to Let's Encrypt.

> One-click **copy** each box. First box → Cluster1 terminal. Second box → Cluster2 terminal.

### Cluster1

```bash
export KUBECONFIG="$KUBECONFIG1"
oc apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: cert-manager-operator
  namespace: cert-manager-operator
spec:
  targetNamespaces:
  - cert-manager-operator
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-cert-manager-operator
  namespace: cert-manager-operator
spec:
  channel: stable-v1
  installPlanApproval: Automatic
  name: openshift-cert-manager-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

echo "Waiting for cert-manager pods..."
for i in $(seq 1 36); do
  READY=$(oc get pods -n cert-manager --no-headers 2>/dev/null | grep -c '1/1' || true)
  echo "  [$i/36] Ready pods in cert-manager: $READY"
  [ "$READY" -ge 3 ] 2>/dev/null && break
  sleep 5
done
oc get pods -n cert-manager
```

### Cluster2

```bash
export KUBECONFIG="$KUBECONFIG2"
oc apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: cert-manager-operator
  namespace: cert-manager-operator
spec:
  targetNamespaces:
  - cert-manager-operator
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-cert-manager-operator
  namespace: cert-manager-operator
spec:
  channel: stable-v1
  installPlanApproval: Automatic
  name: openshift-cert-manager-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

echo "Waiting for cert-manager pods..."
for i in $(seq 1 36); do
  READY=$(oc get pods -n cert-manager --no-headers 2>/dev/null | grep -c '1/1' || true)
  echo "  [$i/36] Ready pods in cert-manager: $READY"
  [ "$READY" -ge 3 ] 2>/dev/null && break
  sleep 5
done
oc get pods -n cert-manager
```

**Expected (both):** cert-manager pods `1/1 Running`.

---

## Step 2 — Create CA chain (BOTH clusters)

**What:** Creates the 3-object CA chain (bootstrap issuer → root CA cert → CA issuer).  
**Why:** This is the stamp SPIRE will use for UpstreamAuthority.

> One-click **copy** each box.

### Cluster1

```bash
export KUBECONFIG="$KUBECONFIG1"
oc apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: spire-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: "SPIRE Upstream CA"
  secretName: spire-ca-secret
  duration: 87600h
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: spire-ca-issuer
  namespace: cert-manager
spec:
  ca:
    secretName: spire-ca-secret
EOF
sleep 10
oc get clusterissuer selfsigned-issuer
oc get certificate spire-ca -n cert-manager
oc get issuer spire-ca-issuer -n cert-manager
```

### Cluster2

```bash
export KUBECONFIG="$KUBECONFIG2"
oc apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: spire-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: "SPIRE Upstream CA"
  secretName: spire-ca-secret
  duration: 87600h
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: spire-ca-issuer
  namespace: cert-manager
spec:
  ca:
    secretName: spire-ca-secret
EOF
sleep 10
oc get clusterissuer selfsigned-issuer
oc get certificate spire-ca -n cert-manager
oc get issuer spire-ca-issuer -n cert-manager
```

**Expected (both):** all show `READY=True`.

> Important: `spire-ca-issuer` must be type `ca:` (not `selfSigned`).  
> SPIRE points to this with `issuerKind: Issuer`.

---

## Step 3 — Install ZTWIM operator (BOTH clusters)

**What:** Installs Zero Trust Workload Identity Manager operator.  
**Why:** This operator creates/manages SPIRE Server, Agent, CSI Driver, OIDC.

> One-click **copy** each box.  
> **Important:** OLM needs ~30–90s after Subscription create before the controller Deployment exists.  
> Do **not** run `oc wait` immediately — wait for CSV `Succeeded` first (included below).  
> Otherwise you may see: `deployments.apps "…controller-manager" not found` (harmless timing; install still proceeds).

### Cluster1

```bash
export KUBECONFIG="$KUBECONFIG1"
oc apply -f - <<'EOF'
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

echo "Waiting for ZTWIM CSV Succeeded (OLM install)..."
for i in $(seq 1 60); do
  PHASE=$(oc get csv -n zero-trust-workload-identity-manager -o jsonpath='{range .items[*]}{.metadata.name}{"="}{.status.phase}{"\n"}{end}' 2>/dev/null | grep '^zero-trust' | head -1 | cut -d= -f2)
  echo "  [$i/60] CSV phase=${PHASE:-<none>}"
  [ "$PHASE" = "Succeeded" ] && break
  sleep 5
done

echo "Waiting for controller-manager Deployment..."
for i in $(seq 1 36); do
  if oc get deploy zero-trust-workload-identity-manager-controller-manager -n zero-trust-workload-identity-manager >/dev/null 2>&1; then
    break
  fi
  echo "  [$i/36] deployment not created yet..."
  sleep 5
done

oc wait --for=condition=Available deployment/zero-trust-workload-identity-manager-controller-manager \
  -n zero-trust-workload-identity-manager --timeout=5m
oc get csv,pods -n zero-trust-workload-identity-manager
oc get crd spireservers.operator.openshift.io -o yaml | grep -c upstreamAuthority
```

### Cluster2

```bash
export KUBECONFIG="$KUBECONFIG2"
oc apply -f - <<'EOF'
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

echo "Waiting for ZTWIM CSV Succeeded (OLM install)..."
for i in $(seq 1 60); do
  PHASE=$(oc get csv -n zero-trust-workload-identity-manager -o jsonpath='{range .items[*]}{.metadata.name}{"="}{.status.phase}{"\n"}{end}' 2>/dev/null | grep '^zero-trust' | head -1 | cut -d= -f2)
  echo "  [$i/60] CSV phase=${PHASE:-<none>}"
  [ "$PHASE" = "Succeeded" ] && break
  sleep 5
done

echo "Waiting for controller-manager Deployment..."
for i in $(seq 1 36); do
  if oc get deploy zero-trust-workload-identity-manager-controller-manager -n zero-trust-workload-identity-manager >/dev/null 2>&1; then
    break
  fi
  echo "  [$i/36] deployment not created yet..."
  sleep 5
done

oc wait --for=condition=Available deployment/zero-trust-workload-identity-manager-controller-manager \
  -n zero-trust-workload-identity-manager --timeout=5m
oc get csv,pods -n zero-trust-workload-identity-manager
```

**Expected:** CSV `Succeeded`, controller-manager `1/1 Running`; Cluster1 `upstreamAuthority` count `> 0`.

---

## Step 4 — Let's Encrypt TLS secret FIRST (Cluster1 only) — HARD GATE

**What:** Get a public TLS cert for `federation.<CLUSTER1_APP_DOMAIN>` **before** any Cluster1 SpireServer with `https_web`.  
**Why:** `https_web` + `externalSecretRef` needs secret `spire-server-federation-tls` already present.

> Click **copy** on the box below (one click). Paste into **Cluster1** terminal.  
> **PAUSE recording** until you see `LE SECRET READY`. Then continue to Step 5/6.

```bash
export KUBECONFIG="$KUBECONFIG1"
export CLUSTER1_APP_DOMAIN="apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')"
echo "Will request cert for: federation.${CLUSTER1_APP_DOMAIN}"

oc apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-http01
  namespace: zero-trust-workload-identity-manager
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
      - http01:
          ingress:
            ingressClassName: openshift-default
EOF

sleep 10
oc get issuer letsencrypt-http01 -n zero-trust-workload-identity-manager

oc apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: spire-server-federation-tls
  namespace: zero-trust-workload-identity-manager
spec:
  secretName: spire-server-federation-tls
  commonName: federation.${CLUSTER1_APP_DOMAIN}
  dnsNames:
    - federation.${CLUSTER1_APP_DOMAIN}
  usages:
    - server auth
  issuerRef:
    kind: Issuer
    name: letsencrypt-http01
EOF

echo "Waiting for Let's Encrypt (up to ~5 minutes)..."
for i in $(seq 1 60); do
  READY=$(oc get certificate spire-server-federation-tls -n zero-trust-workload-identity-manager -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null || true)
  if [ "$READY" = "True" ] && oc get secret spire-server-federation-tls -n zero-trust-workload-identity-manager >/dev/null 2>&1; then
    echo "LE SECRET READY"
    break
  fi
  echo "  [$i/60] waiting..."
  sleep 5
done

oc get certificate spire-server-federation-tls -n zero-trust-workload-identity-manager
oc get secret spire-server-federation-tls -n zero-trust-workload-identity-manager

READY=$(oc get certificate spire-server-federation-tls -n zero-trust-workload-identity-manager -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')
if [ "$READY" != "True" ]; then
  echo "STOP: Let's Encrypt Certificate is not Ready. Do NOT deploy Cluster1 SpireServer yet."
  oc describe certificate spire-server-federation-tls -n zero-trust-workload-identity-manager | tail -30
  exit 1
fi
echo "Gate passed — safe to deploy Cluster1 https_web in Step 6."
```

**Expected:** line `LE SECRET READY` and Certificate `Ready=True`.

---

## Step 5 — Deploy Cluster2 first (https_spiffe + UpstreamAuthority)

**What:** Deploy SPIRE on Cluster2 with UpstreamAuthority and `https_spiffe` federation.  
**Why:** Cluster2 needs **no** external TLS secret.

> One-click **copy** the box → paste in **Cluster2** terminal.

```bash
export KUBECONFIG="$KUBECONFIG2"
export CLUSTER2_APP_DOMAIN="apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')"
export KUBECONFIG="$KUBECONFIG1"
export CLUSTER1_APP_DOMAIN="apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')"
export KUBECONFIG="$KUBECONFIG2"

echo "CLUSTER1_APP_DOMAIN=$CLUSTER1_APP_DOMAIN"
echo "CLUSTER2_APP_DOMAIN=$CLUSTER2_APP_DOMAIN"
echo "Using API: $(oc config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
if [ "$CLUSTER1_APP_DOMAIN" = "apps." ] || [ "$CLUSTER2_APP_DOMAIN" = "apps." ]; then
  echo "STOP: domain invalid — check kubeconfig paths"
  exit 1
fi

oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: ZeroTrustWorkloadIdentityManager
metadata:
  name: cluster
spec:
  trustDomain: ${CLUSTER2_APP_DOMAIN}
  clusterName: cluster2
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: "${CLUSTER2_APP_DOMAIN}"
    country: "US"
    organization: "RH"
  jwtIssuer: https://oidc-discovery.${CLUSTER2_APP_DOMAIN}
  persistence:
    type: pvc
    size: "1Gi"
    accessMode: ReadWriteOnce
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
  upstreamAuthority:
    certManager:
      namespace: cert-manager
      issuerName: spire-ca-issuer
      issuerKind: Issuer
  federation:
    bundleEndpoint:
      profile: https_spiffe
    managedRoute: "true"
    federatesWith:
    - trustDomain: ${CLUSTER1_APP_DOMAIN}
      bundleEndpointUrl: https://federation.${CLUSTER1_APP_DOMAIN}
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
  jwtIssuer: https://oidc-discovery.${CLUSTER2_APP_DOMAIN}
EOF

echo "Waiting for Cluster2 SpireServer Ready..."
for i in $(seq 1 36); do
  R=$(oc get spireserver cluster -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null || true)
  RA=$(oc get spireserver cluster -o jsonpath='{.status.conditions[?(@.type=="RouteAvailable")].status}' 2>/dev/null || true)
  if [ "$R" = "True" ] && [ "$RA" = "True" ]; then
    echo "Cluster2 Ready + RouteAvailable"
    break
  fi
  echo "  [$i/36] Ready=$R RouteAvailable=$RA"
  sleep 5
done

oc get pods -n zero-trust-workload-identity-manager
oc get routes -n zero-trust-workload-identity-manager
oc get spireserver cluster -o jsonpath='Ready={.status.conditions[?(@.type=="Ready")].status} RouteAvailable={.status.conditions[?(@.type=="RouteAvailable")].status}{"\n"}'
```

**Expected:** `Ready=True` `RouteAvailable=True`, federation route `passthrough`.

---

## Step 6 — Deploy Cluster1 (https_web + UpstreamAuthority) — only after Step 4

> Only after Step 4 printed `LE SECRET READY`.  
> One-click **copy** the box → paste in **Cluster1** terminal.

**What:** Deploy SPIRE on Cluster1 with UpstreamAuthority + `https_web` using the ready LE secret.

```bash
export KUBECONFIG="$KUBECONFIG1"
export CLUSTER1_APP_DOMAIN="apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')"
export KUBECONFIG="$KUBECONFIG2"
export CLUSTER2_APP_DOMAIN="apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')"
export KUBECONFIG="$KUBECONFIG1"

echo "CLUSTER1_APP_DOMAIN=$CLUSTER1_APP_DOMAIN"
echo "CLUSTER2_APP_DOMAIN=$CLUSTER2_APP_DOMAIN"
echo "Using API: $(oc config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
if [ "$CLUSTER1_APP_DOMAIN" = "apps." ] || [ "$CLUSTER2_APP_DOMAIN" = "apps." ]; then
  echo "STOP: domain invalid — check kubeconfig paths"
  exit 1
fi

oc get certificate spire-server-federation-tls -n zero-trust-workload-identity-manager
oc get secret spire-server-federation-tls -n zero-trust-workload-identity-manager
READY=$(oc get certificate spire-server-federation-tls -n zero-trust-workload-identity-manager -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')
if [ "$READY" != "True" ]; then
  echo "STOP: go back to Step 4. Do not apply Cluster1 SpireServer yet."
  exit 1
fi
echo "Gate OK — applying Cluster1 SpireServer with https_web"

oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: ZeroTrustWorkloadIdentityManager
metadata:
  name: cluster
spec:
  trustDomain: ${CLUSTER1_APP_DOMAIN}
  clusterName: cluster1
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: "${CLUSTER1_APP_DOMAIN}"
    country: "US"
    organization: "RH"
  jwtIssuer: https://oidc-discovery.${CLUSTER1_APP_DOMAIN}
  persistence:
    type: pvc
    size: "1Gi"
    accessMode: ReadWriteOnce
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
  upstreamAuthority:
    certManager:
      namespace: cert-manager
      issuerName: spire-ca-issuer
      issuerKind: Issuer
  federation:
    bundleEndpoint:
      profile: https_web
      httpsWeb:
        servingCert:
          externalSecretRef: spire-server-federation-tls
    managedRoute: "true"
    federatesWith:
    - trustDomain: ${CLUSTER2_APP_DOMAIN}
      bundleEndpointUrl: https://federation.${CLUSTER2_APP_DOMAIN}
      bundleEndpointProfile: https_spiffe
      endpointSpiffeId: spiffe://${CLUSTER2_APP_DOMAIN}/spire/server
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
  jwtIssuer: https://oidc-discovery.${CLUSTER1_APP_DOMAIN}
EOF

echo "Waiting for Cluster1 SpireServer Ready + federation route..."
for i in $(seq 1 48); do
  R=$(oc get spireserver cluster -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null || true)
  RA=$(oc get spireserver cluster -o jsonpath='{.status.conditions[?(@.type=="RouteAvailable")].status}' 2>/dev/null || true)
  if [ "$R" = "True" ] && [ "$RA" = "True" ] && oc get route spire-server-federation -n zero-trust-workload-identity-manager >/dev/null 2>&1; then
    echo "Cluster1 Ready + federation route present"
    break
  fi
  MSG=$(oc get spireserver cluster -o jsonpath='{.status.conditions[?(@.type=="RouteAvailable")].message}' 2>/dev/null || true)
  echo "  [$i/48] Ready=$R RouteAvailable=$RA msg=$MSG"
  sleep 5
done

oc get pods -n zero-trust-workload-identity-manager
oc get routes -n zero-trust-workload-identity-manager
oc get role,rolebinding -n zero-trust-workload-identity-manager | grep external-cert || true
oc get spireserver cluster -o jsonpath='Ready={.status.conditions[?(@.type=="Ready")].status} RouteAvailable={.status.conditions[?(@.type=="RouteAvailable")].status}{"\n"}'

curl -sS "https://federation.${CLUSTER1_APP_DOMAIN}" | jq '.keys | length'
echo | openssl s_client -connect "federation.${CLUSTER1_APP_DOMAIN}:443" -servername "federation.${CLUSTER1_APP_DOMAIN}" 2>/dev/null | openssl x509 -noout -issuer
```

**Expected:** `Ready=True`, federation `reencrypt`, curl keys without `-k`, issuer Let's Encrypt.

---

## Step 7 — Verify UpstreamAuthority (BOTH clusters)

**What:** Confirm SPIRE used cert-manager, not self-signed (`self_signed=false`).  
> One-click **copy** → paste in either terminal (checks both clusters).

```bash
echo "========== Cluster1 =========="
export KUBECONFIG="$KUBECONFIG1"
oc logs spire-server-0 -n zero-trust-workload-identity-manager -c spire-server | grep -iE "self_signed|X509 CA activated|cert-manager" | tail -5
oc get cm spire-bundle -n zero-trust-workload-identity-manager -o jsonpath='{.data.bundle\.crt}' | openssl x509 -noout -issuer -subject

echo "========== Cluster2 =========="
export KUBECONFIG="$KUBECONFIG2"
oc logs spire-server-0 -n zero-trust-workload-identity-manager -c spire-server | grep -iE "self_signed|X509 CA activated|cert-manager" | tail -5
oc get cm spire-bundle -n zero-trust-workload-identity-manager -o jsonpath='{.data.bundle\.crt}' | openssl x509 -noout -issuer -subject
```

**Expected (both):** `self_signed=false` and `issuer=CN=SPIRE Upstream CA`.

---

## Step 8 — Federation bootstrap

**What:** Fetch bundles + create `ClusterFederatedTrustDomain` on both sides + verify.  
> One-click **copy** the whole box → paste once (uses both kubeconfigs).

```bash
export KUBECONFIG="$KUBECONFIG1"
export CLUSTER1_APP_DOMAIN="apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')"
export KUBECONFIG="$KUBECONFIG2"
export CLUSTER2_APP_DOMAIN="apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')"

echo "CLUSTER1_APP_DOMAIN=$CLUSTER1_APP_DOMAIN"
echo "CLUSTER2_APP_DOMAIN=$CLUSTER2_APP_DOMAIN"
if [ "$CLUSTER1_APP_DOMAIN" = "apps." ] || [ "$CLUSTER2_APP_DOMAIN" = "apps." ]; then
  echo "STOP: domain invalid"
  exit 1
fi

curl -sS "https://federation.${CLUSTER1_APP_DOMAIN}" -o /tmp/cluster1-bundle.json
echo "Cluster1 keys: $(jq '.keys | length' /tmp/cluster1-bundle.json)"
curl -sk "https://federation.${CLUSTER2_APP_DOMAIN}" -o /tmp/cluster2-bundle.json
echo "Cluster2 keys: $(jq '.keys | length' /tmp/cluster2-bundle.json)"

BUNDLE_C2=$(cat /tmp/cluster2-bundle.json)
export KUBECONFIG="$KUBECONFIG1"
oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: federation-to-cluster2
spec:
  trustDomain: ${CLUSTER2_APP_DOMAIN}
  bundleEndpointURL: https://federation.${CLUSTER2_APP_DOMAIN}
  bundleEndpointProfile:
    type: https_spiffe
    endpointSPIFFEID: spiffe://${CLUSTER2_APP_DOMAIN}/spire/server
  className: zero-trust-workload-identity-manager-spire
  trustDomainBundle: |
$(echo "$BUNDLE_C2" | sed 's/^/    /')
EOF
oc get clusterfederatedtrustdomain

BUNDLE_C1=$(cat /tmp/cluster1-bundle.json)
export KUBECONFIG="$KUBECONFIG2"
oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: federation-to-cluster1
spec:
  trustDomain: ${CLUSTER1_APP_DOMAIN}
  bundleEndpointURL: https://federation.${CLUSTER1_APP_DOMAIN}
  bundleEndpointProfile:
    type: https_web
  className: zero-trust-workload-identity-manager-spire
  trustDomainBundle: |
$(echo "$BUNDLE_C1" | sed 's/^/    /')
EOF
oc get clusterfederatedtrustdomain

sleep 15
echo "========== bundle list Cluster1 =========="
export KUBECONFIG="$KUBECONFIG1"
oc exec spire-server-0 -n zero-trust-workload-identity-manager -c spire-server -- \
  /spire-server bundle list -socketPath /tmp/spire-server/private/api.sock | head -20

echo "========== bundle list Cluster2 =========="
export KUBECONFIG="$KUBECONFIG2"
oc exec spire-server-0 -n zero-trust-workload-identity-manager -c spire-server -- \
  /spire-server bundle list -socketPath /tmp/spire-server/private/api.sock | head -20
```

**Expected:** each side lists the other trust domain.

---

## Step 9 — Cross-cluster mTLS test

**What:** Deploy server on Cluster1 and client on Cluster2, then openssl mTLS handshake.  
**Why:** Final end-to-end proof that federated SPIFFE identities work across clusters.

### 9A. Cluster1 — deploy mTLS server

```bash
export KUBECONFIG=$KUBECONFIG1
export CLUSTER1_APP_DOMAIN=apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
export CLUSTER2_APP_DOMAIN=apps.$(KUBECONFIG=$KUBECONFIG2 oc get dns cluster -o jsonpath='{.spec.baseDomain}')

oc create namespace federation-test --dry-run=client -o yaml | oc apply -f -
oc create serviceaccount mtls-server -n federation-test --dry-run=client -o yaml | oc apply -f -

oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: mtls-server-workload
spec:
  className: zero-trust-workload-identity-manager-spire
  spiffeIDTemplate: "spiffe://{{ .TrustDomain }}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  podSelector:
    matchLabels:
      spiffe.io/spiffe-id: "true"
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: federation-test
  federatesWith:
    - "${CLUSTER2_APP_DOMAIN}"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: spiffe-helper-config
  namespace: federation-test
data:
  helper.conf: |
    agent_address = "/spiffe-workload-api/spire-agent.sock"
    cmd = ""
    cert_dir = "/certs"
    svid_file_name = "svid.pem"
    svid_key_file_name = "svid_key.pem"
    svid_bundle_file_name = "bundle.pem"
    renew_signal = ""
---
apiVersion: v1
kind: Pod
metadata:
  name: mtls-server
  namespace: federation-test
  labels:
    app: mtls-server
    spiffe.io/spiffe-id: "true"
spec:
  serviceAccountName: mtls-server
  containers:
  - name: server
    image: registry.access.redhat.com/ubi9/ubi:latest
    command: ["/bin/bash", "-c"]
    args:
      - |
        echo "Waiting for certificates..."
        while [ ! -f /certs/svid.pem ]; do sleep 2; done
        echo "Certificates found! Starting mTLS server..."
        openssl s_server -accept 8443 -cert /certs/svid.pem -key /certs/svid_key.pem -CAfile /certs/bundle.pem -Verify 1 -www
    ports:
    - containerPort: 8443
    volumeMounts:
    - name: certs
      mountPath: /certs
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      runAsNonRoot: true
      seccompProfile:
        type: RuntimeDefault
  - name: spiffe-helper
    image: ghcr.io/spiffe/spiffe-helper:0.8.0
    args: ["-config", "/etc/spiffe-helper/helper.conf"]
    volumeMounts:
    - name: spiffe-workload-api
      mountPath: /spiffe-workload-api
      readOnly: true
    - name: certs
      mountPath: /certs
    - name: spiffe-helper-config
      mountPath: /etc/spiffe-helper
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      runAsNonRoot: true
      seccompProfile:
        type: RuntimeDefault
  volumes:
  - name: spiffe-workload-api
    csi:
      driver: csi.spiffe.io
      readOnly: true
  - name: certs
    emptyDir: {}
  - name: spiffe-helper-config
    configMap:
      name: spiffe-helper-config
---
apiVersion: v1
kind: Service
metadata:
  name: mtls-server
  namespace: federation-test
spec:
  selector:
    app: mtls-server
  ports:
  - port: 8443
    targetPort: 8443
    name: https
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: mtls-server
  namespace: federation-test
spec:
  host: mtls-server.${CLUSTER1_APP_DOMAIN}
  to:
    kind: Service
    name: mtls-server
  port:
    targetPort: https
  tls:
    termination: passthrough
EOF

oc wait --for=condition=Ready pod/mtls-server -n federation-test --timeout=120s
oc get pods -n federation-test
```

**Expected:** `mtls-server  2/2  Running`

### 9B. Cluster2 — deploy mTLS client

```bash
export KUBECONFIG=$KUBECONFIG2
export CLUSTER1_APP_DOMAIN=apps.$(KUBECONFIG=$KUBECONFIG1 oc get dns cluster -o jsonpath='{.spec.baseDomain}')

oc create namespace federation-test --dry-run=client -o yaml | oc apply -f -
oc create serviceaccount mtls-client -n federation-test --dry-run=client -o yaml | oc apply -f -

oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: mtls-client-workload
spec:
  className: zero-trust-workload-identity-manager-spire
  spiffeIDTemplate: "spiffe://{{ .TrustDomain }}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  podSelector:
    matchLabels:
      spiffe.io/spiffe-id: "true"
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: federation-test
  federatesWith:
    - "${CLUSTER1_APP_DOMAIN}"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: spiffe-helper-config
  namespace: federation-test
data:
  helper.conf: |
    agent_address = "/spiffe-workload-api/spire-agent.sock"
    cmd = ""
    cert_dir = "/certs"
    svid_file_name = "svid.pem"
    svid_key_file_name = "svid_key.pem"
    svid_bundle_file_name = "bundle.pem"
    renew_signal = ""
---
apiVersion: v1
kind: Pod
metadata:
  name: mtls-client
  namespace: federation-test
  labels:
    app: mtls-client
    spiffe.io/spiffe-id: "true"
spec:
  serviceAccountName: mtls-client
  containers:
  - name: client
    image: registry.access.redhat.com/ubi9/ubi:latest
    command: ["sleep", "infinity"]
    volumeMounts:
    - name: certs
      mountPath: /certs
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      runAsNonRoot: true
      seccompProfile:
        type: RuntimeDefault
  - name: spiffe-helper
    image: ghcr.io/spiffe/spiffe-helper:0.8.0
    args: ["-config", "/etc/spiffe-helper/helper.conf"]
    volumeMounts:
    - name: spiffe-workload-api
      mountPath: /spiffe-workload-api
      readOnly: true
    - name: certs
      mountPath: /certs
    - name: spiffe-helper-config
      mountPath: /etc/spiffe-helper
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      runAsNonRoot: true
      seccompProfile:
        type: RuntimeDefault
  volumes:
  - name: spiffe-workload-api
    csi:
      driver: csi.spiffe.io
      readOnly: true
  - name: certs
    emptyDir: {}
  - name: spiffe-helper-config
    configMap:
      name: spiffe-helper-config
EOF

oc wait --for=condition=Ready pod/mtls-client -n federation-test --timeout=120s
oc get pods -n federation-test
```

**Expected:** `mtls-client  2/2  Running`

### 9C. Build trust store with intermediate + upstream root (REQUIRED for openssl demo)

**What:** Extract SPIRE intermediate from server `svid.pem` and combine with upstream roots.  
**Why:** With UpstreamAuthority:
- federation/`bundle.pem` has **Upstream root only**
- workload SVID is signed by SPIRE **intermediate**
- openssl often receives only the leaf from `s_server`
- so client CAfile must include the intermediate or verify fails with code `21`

```bash
# --- on Cluster1 terminal / shared filesystem ---
export KUBECONFIG=$KUBECONFIG1

oc exec -n federation-test mtls-server -c server -- cat /certs/svid.pem > /tmp/c1-server-svid.pem
oc exec -n federation-test mtls-server -c server -- cat /certs/bundle.pem > /tmp/c1-server-bundle.pem

rm -f /tmp/svid-cert-*.pem
awk 'BEGIN{c=0} /BEGIN CERTIFICATE/{c++} {print > ("/tmp/svid-cert-" c ".pem")}' /tmp/c1-server-svid.pem

echo "=== cert 1 (leaf) ==="
openssl x509 -in /tmp/svid-cert-1.pem -noout -subject -issuer
echo "=== cert 2 (intermediate) ==="
openssl x509 -in /tmp/svid-cert-2.pem -noout -subject -issuer

# intermediate + Cluster1 upstream root
cat /tmp/svid-cert-2.pem /tmp/c1-server-bundle.pem > /tmp/c1-trust.pem
```

**Expected cert 2:**
```
subject=...CN=<CLUSTER1_APP_DOMAIN>...
issuer=CN=SPIRE Upstream CA
```

```bash
# --- on Cluster2 terminal (same machine so /tmp is shared) ---
export KUBECONFIG=$KUBECONFIG2
export CLUSTER1_APP_DOMAIN=apps.$(KUBECONFIG=$KUBECONFIG1 oc get dns cluster -o jsonpath='{.spec.baseDomain}')
export CLUSTER2_APP_DOMAIN=apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')

curl -sk "https://federation.${CLUSTER2_APP_DOMAIN}" | \
  jq -r '.keys[] | select(.use=="x509-svid") | .x5c[]' | while read b64; do
    echo "$b64" | base64 -d | openssl x509 -inform DER
  done > /tmp/c2-fed-all.pem

cat /tmp/c1-trust.pem /tmp/c2-fed-all.pem > /tmp/combined-bundle.pem
grep -c "BEGIN CERTIFICATE" /tmp/combined-bundle.pem

oc cp /tmp/combined-bundle.pem federation-test/mtls-client:/certs/combined-bundle.pem -c client
```

### 9D. Run the mTLS test (Cluster2)

```bash
export KUBECONFIG=$KUBECONFIG2
export CLUSTER1_APP_DOMAIN=apps.$(KUBECONFIG=$KUBECONFIG1 oc get dns cluster -o jsonpath='{.spec.baseDomain}')

oc exec -n federation-test mtls-client -c client -- ls -l /certs

oc exec -n federation-test mtls-client -c client -- bash -c "
echo 'Q' | timeout 15 openssl s_client \
  -connect mtls-server.${CLUSTER1_APP_DOMAIN}:443 \
  -servername mtls-server.${CLUSTER1_APP_DOMAIN} \
  -cert /certs/svid.pem \
  -key /certs/svid_key.pem \
  -CAfile /certs/combined-bundle.pem \
  2>&1 | grep -E '(Verify return code|CONNECTED|subject|issuer|verify error)'
"
```

**SUCCESS:**
```
CONNECTED(00000003)
Verify return code: 0 (ok)
```

---

## Scorecard

| # | Check | Pass? |
|---|-------|-------|
| 1 | CRD has `upstreamAuthority` | |
| 2 | Step 4: LE Certificate Ready + secret exists **before** Cluster1 SpireServer | |
| 3 | Cluster2 `Ready=True` + federation route (https_spiffe) | |
| 4 | Cluster1 `Ready=True` + `RouteAvailable=True` (https_web) | |
| 5 | Cluster1 + Cluster2 `self_signed=false`, issuer `SPIRE Upstream CA` | |
| 6 | Cluster1 federation curl works **without** `-k` + Let's Encrypt issuer | |
| 7 | Cluster2 federation curl works **with** `-k` | |
| 8 | Both clusters list each other's trust domain | |
| 9 | **mTLS Verify return code: 0 (ok)** | |

---

## Cleanup

```bash
export KUBECONFIG=$KUBECONFIG1
oc delete namespace federation-test --ignore-not-found
oc delete clusterspiffeid mtls-server-workload --ignore-not-found
oc delete clusterfederatedtrustdomain federation-to-cluster2 --ignore-not-found

export KUBECONFIG=$KUBECONFIG2
oc delete namespace federation-test --ignore-not-found
oc delete clusterspiffeid mtls-client-workload --ignore-not-found
oc delete clusterfederatedtrustdomain federation-to-cluster1 --ignore-not-found
```

---

## Troubleshooting (only if something went off-script)

| Symptom | Cause | Fix |
|---------|-------|-----|
| `RouteAvailable=False` / `secrets "spire-server-federation-tls" not found` on Cluster1 | SpireServer `https_web` applied **before** LE secret was Ready | Finish Step 4 gate, wait — operator self-heals once secret exists. Do not panic-create routes mid-recording. |
| LE Certificate stuck `Ready=False` | HTTP-01 / DNS / rate limit | `oc describe certificate spire-server-federation-tls -n zero-trust-workload-identity-manager` and `oc get challenges -A`. Pause recording until Ready. |
| `export KUBECONFIG=$KUBECONFIG1` points to wrong cluster | Stale variable or new terminal | Re-run Step 0 in that terminal to set `KUBECONFIG1`/`KUBECONFIG2` |
| `Unable to connect ... no such host` while applying SpireServer | `KUBECONFIG1`/`KUBECONFIG2` unset in that shell → oc used old `~/.kube/config` | Re-run Step 0 in that terminal; confirm domains are not `apps.` |
| SpireServer `federatesWith` / URL ends with `apps.` only | Other-cluster domain lookup failed | Patch federatesWith with correct peer `apps.<baseDomain>` |
| `deployments.apps "zero-trust-…-controller-manager" not found` right after Subscription | OLM still installing; Deployment not created yet | Wait for CSV `Succeeded` then `oc wait` (Step 3 already does this). Re-run wait/get pods — not a failed install. |
| `/opt/spire/bin/spire-server` not found | Image binary path | Use `/spire-server` |
| mTLS `Verify return code: 21` with only federation roots | Missing SPIRE intermediate in client trust store | Step 9C: add cert #2 from server `svid.pem` |
| Manual `edge` federation route → `Client sent an HTTP request to an HTTPS server` | Wrong TLS mode | Do **not** create manual edge routes. Let the operator manage `reencrypt` + `externalCertificate`. |

### Emergency only (skip during a clean recording)

If you somehow applied Cluster1 `https_web` before the secret and status is still stuck after the secret is Ready:

```bash
export KUBECONFIG=$KUBECONFIG1
# Confirm secret exists first
oc get secret spire-server-federation-tls -n zero-trust-workload-identity-manager
oc annotate spireserver cluster force-reconcile="$(date +%s)" --overwrite
# Wait ~30s, then re-check RouteAvailable
oc get spireserver cluster -o jsonpath='RouteAvailable={.status.conditions[?(@.type=="RouteAvailable")].status}{"\n"}'
```

Only if that still fails, create a **reencrypt** route (never `edge`):

```bash
export KUBECONFIG=$KUBECONFIG1
export CLUSTER1_APP_DOMAIN=apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')

oc apply -f - <<EOF
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: spire-server-federation
  namespace: zero-trust-workload-identity-manager
spec:
  host: federation.${CLUSTER1_APP_DOMAIN}
  to:
    kind: Service
    name: spire-server
    weight: 100
  port:
    targetPort: federation
  tls:
    termination: reencrypt
    externalCertificate:
      name: spire-server-federation-tls
    insecureEdgeTerminationPolicy: Redirect
EOF
```

---

## Notes for demo / QE report

1. **UpstreamAuthority (PR #113)** worked on both clusters: `self_signed=false`, issuer `CN=SPIRE Upstream CA`.
2. **Hybrid federation** worked: Cluster1 Let's Encrypt (`https_web`) + Cluster2 `https_spiffe`.
3. **Recording tip:** issue LE secret **before** Cluster1 `https_web` SpireServer (Step 4 → Step 6). Operator then creates federation route + `spire-server-external-cert-reader` RBAC automatically.
4. **openssl demo detail:** with UpstreamAuthority, include **intermediate + upstream root** in the client CAfile. This is a demo/openssl limitation, not an UpstreamAuthority failure.
5. Real SPIFFE SDKs usually consume the Workload API and handle chain/federation more completely than raw openssl.
6. AWS PCA multi-cluster guide used `https_spiffe` on both sides — it never hits this LE/`https_web` route readiness path.
