# SPIRE / OSSM / Zero-Trust Proposals & Guides

End-to-end testing guides and POC documentation for **SPIRE federation**, **Red Hat OpenShift Service Mesh (OSSM)**, and **OPA-based authorization** on OpenShift multi-cluster environments.

## Guides

| Guide | Description |
|---|---|
| [Manual Federation Testing Guide](Manual-Federation-Testing-Guide.md) | Step-by-step manual testing guide for SPIRE-to-SPIRE federation across OpenShift clusters. Covers trust domain setup, bundle exchange, and federation verification. |
| [SPIRE Federation with PostgreSQL Test Guide](SPIRE_Federation_PostgreSQL_Test_Guide.md) | Tests SPIRE federation between two OpenShift clusters using **PostgreSQL 15** as the backend datastore and **https_web** federation profile with ACME/Let's Encrypt certificates. |
| [SPIRE + OSSM + OPA E2E POC Guide](ossm-spire-opa-e2e-poc-guide.md) | Complete multi-cluster POC where SPIRE proves workload identity, OSSM carries mTLS traffic across clusters, and OPA enforces policy-based authorization using SPIFFE IDs. Beginner-friendly with concepts-first approach. |
| [UpstreamAuthority (cert-manager) + Federation POC](POC_UPSTREAM_AUTHORITY_CERTMANAGER_FEDERATION_GUIDE.md) | Proves SPIRE's CA is signed by cert-manager (`self_signed=false`) and two OpenShift clusters federate with hybrid profiles (`https_web` + `https_spiffe`) for cross-cluster mTLS. |

## Stack

- **SPIRE / SPIFFE** — Workload identity (X.509 SVIDs, trust domains, federation)
- **ZTWIM Operator** — Red Hat Zero Trust Workload Identity Manager for OpenShift
- **cert-manager** — Certificate lifecycle management (UpstreamAuthority CA chain, Let's Encrypt)
- **Red Hat OpenShift Service Mesh 3.x** — Istio / Envoy-based service mesh with mTLS
- **OPA (Open Policy Agent)** — Rego-based authorization using caller SPIFFE IDs
- **OpenShift 4.x** — Kubernetes platform

## Author

**Sayak Das** — Red Hat QE
