---
layout: default
title: "Complete Documentation"
description: "Official Installation, Testing & Architecture Guide for the NFL Stadium Wallet ecosystem — OpenShift, GitOps, Service Mesh 3, Kuadrant, Gateway API & Observability."
lang: en
---

<section class="hero-section">
  <div class="hero-badge-row">
    <img src="https://img.shields.io/badge/redhat-CC0000?style=for-the-badge&logo=redhat&logoColor=white" alt="Red Hat">
    <img src="https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white" alt="Kubernetes">
    <img src="https://img.shields.io/badge/helm-0db7ed?style=for-the-badge&logo=helm&logoColor=white" alt="Helm">
    <img src="https://img.shields.io/badge/ArgoCD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white" alt="ArgoCD">
    <img src="https://img.shields.io/badge/.NET%208-512BD4?style=for-the-badge&logo=dotnet&logoColor=white" alt=".NET 8">
    <img src="https://img.shields.io/badge/Vue.js-4FC08D?style=for-the-badge&logo=vuedotjs&logoColor=white" alt="Vue.js">
  </div>
  <h1>NFL Stadium <span>Wallet</span></h1>
  <p class="hero-sub">Official Installation, Testing &amp; Architecture Guide — Complete digital wallet ecosystem for NFL stadiums on Red Hat OpenShift.</p>
  <p class="hero-meta"><strong>Version:</strong> 2.0 &nbsp;|&nbsp; <strong>Owner:</strong> Maximiliano Pizarro</p>
</section>

<div class="toc" markdown="0">
  <h3>Table of Contents</h3>
  <ol>
    <li><a href="#executive-summary">Executive Summary</a></li>
    <li><a href="#architecture">Architecture &amp; Data Flows</a></li>
    <li><a href="#technology-stack">Technology Stack</a></li>
    <li><a href="#prerequisites">Infrastructure Prerequisites</a></li>
    <li><a href="#deployment">GitOps Installation Guide</a></li>
    <li><a href="#service-mesh">Service Mesh 3 (Ambient Mode)</a></li>
    <li><a href="#connectivity-link">Connectivity Link &amp; Gateway API</a></li>
    <li><a href="#security">Security: API Keys &amp; Policies</a>
      <ul>
        <li><a href="#namespace-isolation">Namespace Restriction (Test/Prod)</a></li>
        <li><a href="#gateway-namespace-policies">Cross-Namespace Policies with Gateway API &amp; Istio</a></li>
        <li><a href="#dns-failover">Failover with DNSPolicy &amp; Route 53</a></li>
      </ul>
    </li>
    <li><a href="#gitops">Multi-Cluster GitOps with ACM</a></li>
    <li><a href="#developer-hub">Red Hat Developer Hub (Kuadrant Plugin)</a></li>
    <li><a href="#observability">Observability</a></li>
    <li><a href="#screenshots">Screenshots</a></li>
    <li><a href="#canary">Canary / Blue-Green Deployments</a></li>
    <li><a href="#service-mesh-topology">Service Mesh — Topology &amp; Traffic</a></li>
    <li><a href="#testing">Test Plan &amp; Validation (QA)</a></li>
    <li><a href="#api-reference">API Reference</a></li>
    <li><a href="#troubleshooting">Troubleshooting</a></li>
    <li><a href="#artifact-hub">Publish to Artifact Hub</a></li>
  </ol>
</div>

---

# 1. Executive Summary {#executive-summary}

This document provides the definitive guide for deploying, configuring, and validating the **NFL Stadium Wallet** ecosystem. The platform adopts a modern approach based on **GitOps**, **Zero-Trust** security without sidecars via **OSSM3 (Ambient Mode)**, and comprehensive API lifecycle management through **Kuadrant** and **Red Hat Developer Hub**.

The system is composed of an **interactive frontend** (Vue.js) and **three core microservices** (.NET 8):

| Microservice | Function |
|--------------|----------|
| **api-customers** | Centralized identity and customer profile management |
| **api-bills** | Transactional logic for the Buffalo Bills venue |
| **api-raiders** | Transactional logic for the Las Vegas Raiders venue |

The microservices interact with external data sources (**ESPN API**) securely and auditably to obtain real-time sports data.

<div class="section-cards">
  <div class="section-card">
    <span class="card-icon">&#9881;</span>
    <h4>Declarative GitOps</h4>
    <p>Continuous synchronization with OpenShift GitOps (ArgoCD) — all state is defined in Git.</p>
  </div>
  <div class="section-card">
    <span class="card-icon">&#128274;</span>
    <h4>Zero-Trust without Sidecars</h4>
    <p>OSSM3 Ambient Mode: automatic mTLS without sidecar container injection.</p>
  </div>
  <div class="section-card">
    <span class="card-icon">&#128200;</span>
    <h4>Complete Observability</h4>
    <p>Grafana, Prometheus, Kiali, TempoStack and OpenTelemetry for full visibility.</p>
  </div>
  <div class="section-card">
    <span class="card-icon">&#127758;</span>
    <h4>Federated Multi-Cluster</h4>
    <p>Hub-and-Spoke topology with ACM, deployed across East and West clusters.</p>
  </div>
</div>

---

# 2. Architecture & Data Flows {#architecture}

## 2.1 Three-Tier Architecture

The solution is structured in a modern three-tier model:

| Tier | Component | Technology Stack | Function | Scalability |
|------|-----------|-----------------|----------|-------------|
| **Frontend** | webapp (SPA) | Vue 3, Vite, vue-router, Apache (UBI8 httpd-24) | UI for login, balance inquiries and QR code generation for payments | Stateless — OpenShift HPA |
| **Backend API** | 3 Independent Microservices | .NET 8.0 ASP.NET Core | ApiCustomers (identity), ApiWalletBuffaloBills (Bills transactions), ApiWalletLasVegasRaiders (Raiders transactions) | Independently deployable and scalable |
| **Data** | Persistent Storage | SQLite (customers.db, buffalobills.db, lasvegasraiders.db) | Local persistence per API | Strict data isolation |

> **Production Note:** For full production deployments, SQLite databases should be migrated to high-availability solutions such as PostgreSQL on OpenShift, potentially using the Crunchy Data operator.

## 2.2 Network Architecture & Service Mesh Diagram

<div class="diagram-box">
                    ┌─────────────────────────────────────────────┐
                    │         Management Plane                     │
                    │  ┌──────────────────┐ ┌──────────────────┐  │
                    │  │ Red Hat Developer │ │ OpenShift GitOps │  │
                    │  │ Hub (API Portal) │ │ (Sync Engine)    │  │
                    │  └──────────────────┘ └──────────────────┘  │
                    └─────────────────────────────────────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              ▼                       ▼                       ▼
    ┌──────────────────────────────────────────────────────────┐
    │  OpenShift Cluster (Namespace: nfl-wallet)                │
    │  ┌──────────────────────────────────────────────────────┐│
    │  │  Gateway API / Kuadrant Ingress                      ││
    │  │  ┌─────────────────────────────────┐                 ││
    │  │  │  OSSM3 Ambient Mesh (Zero-Trust)│                 ││
    │  │  │  ┌───────┐  ┌────────────────┐  │                 ││
    │  │  │  │ztunnel│  │Waypoint Proxy  │  │                 ││
    │  │  │  │ L4/mTLS│  │ L7 Auth/Route │  │                 ││
    │  │  │  └───┬───┘  └───────┬────────┘  │                 ││
    │  │  │      │              │            │                 ││
    │  │  │  ┌───┴──┐  ┌───────┴────────┐   │                 ││
    │  │  │  │webapp│  │api-customers   │   │                 ││
    │  │  │  │Vue.js│  │api-bills       │   │                 ││
    │  │  │  │:5173 │  │api-raiders     │   │                 ││
    │  │  │  └──────┘  │  .NET 8 :8080  │   │                 ││
    │  │  │            └────────┬───────┘   │                 ││
    │  │  └─────────────────────┼───────────┘                 ││
    │  └────────────────────────┼──────────────────────────────┘│
    └───────────────────────────┼────────────────────────────────┘
                                │ Egress Traffic
                                ▼
                    ┌───────────────────────┐
                    │  ESPN Public API       │
                    │  Scoreboards & Stats   │
                    └───────────────────────┘
</div>

## 2.3 Multi-Cluster Topology & Federation

The system utilizes a **Hub-and-Spoke** model, governed by Red Hat platform and management tools:

- **Hub Cluster (Control Plane):** Red Hat Advanced Cluster Management (ACM), OpenShift GitOps (ArgoCD), Centralized Observability Hub (Prometheus, Grafana, Kiali).
- **Data Clusters East/West (Spoke):** Application workloads, OSSM3 in Ambient Mode, federation via HBONE (HTTP/2-Based Encryption).

<div class="diagram-box">
                    Hub (OpenShift GitOps + ACM)
    ┌─────────────────────────────────────────────────────────┐
    │  app-nfl-wallet-acm.yaml                                 │
    │  ┌─────────────┐  ┌──────────────────────────────────┐  │
    │  │ Placement   │  │ GitOpsCluster                     │  │
    │  │ nfl-wallet- │  │ (creates east/west secrets)       │  │
    │  │ gitops-     │  │                                    │  │
    │  │ placement   │  └──────────────────────────────────┘  │
    │  └──────┬──────┘                                        │
    │         │                                                │
    │  app-nfl-wallet-acm-cluster-decision.yaml                │
    │  ┌──────────────────────────────────────────────────┐   │
    │  │ ApplicationSet (matrix)                           │   │
    │  │ clusterDecisionResource × list (dev, test, prod)   │   │
    │  └──────────────┬───────────────────────────────────┘   │
    │                 │                                        │
    │                 ▼                                        │
    │  Applications: nfl-wallet-&lt;namespace&gt;-&lt;clusterName&gt;      │
    └──────────────────────┬──────────────────────────────────┘
                            │
         ┌──────────────────┼──────────────────┐
         ▼                                     ▼
    Cluster East                          Cluster West
    ├── nfl-wallet-dev                    ├── nfl-wallet-dev
    ├── nfl-wallet-test                   ├── nfl-wallet-test
    └── nfl-wallet-prod                   └── nfl-wallet-prod
</div>

## 2.4 ESPN API Integration

The `api-bills` and `api-raiders` microservices require real-time sports data.

- **Endpoint:** `https://site.api.espn.com/apis/site/v2/sports/football/nfl/scoreboard`
- **Egress Management:** OSSM3 Ambient Mode intercepts outbound HTTP requests via the node's ztunnel, enabling latency monitoring and egress policy enforcement without the performance penalty of sidecar container injection.

---

# 3. Technology Stack {#technology-stack}

<div class="tech-badges">
  <img src="https://img.shields.io/badge/OpenShift-EE0000?style=for-the-badge&logo=redhatopenshift&logoColor=white" alt="OpenShift">
  <img src="https://img.shields.io/badge/ArgoCD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white" alt="ArgoCD">
  <img src="https://img.shields.io/badge/Istio-466BB0?style=for-the-badge&logo=istio&logoColor=white" alt="Istio">
  <img src="https://img.shields.io/badge/Helm-0F1689?style=for-the-badge&logo=helm&logoColor=white" alt="Helm">
  <img src="https://img.shields.io/badge/.NET%208-512BD4?style=for-the-badge&logo=dotnet&logoColor=white" alt=".NET 8">
  <img src="https://img.shields.io/badge/Vue.js-4FC08D?style=for-the-badge&logo=vuedotjs&logoColor=white" alt="Vue.js">
  <img src="https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white" alt="Grafana">
  <img src="https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white" alt="Prometheus">
</div>

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Frontend** | Vue 3, Vite, vue-router | SPA served by Apache (UBI8) |
| **Backend** | .NET 8.0 ASP.NET Core (x3) | Microservices: Customers, Bills, Raiders |
| **Data** | SQLite | One database per API |
| **Containers** | Podman / OpenShift | Build and images on Quay.io |
| **Orchestration** | OpenShift 4.20+, Kubernetes | Container platform |
| **GitOps** | OpenShift GitOps (ArgoCD) | Declarative synchronization |
| **Service Mesh** | OSSM 3.2 (Sail Operator, Ambient Mode) | Zero-Trust, mTLS, L7 routing |
| **Gateway** | Gateway API, Kuadrant | Ingress, Rate Limiting, Auth |
| **Observability** | Prometheus, Grafana, Kiali, TempoStack, OpenTelemetry | Metrics, traces, topology |
| **Multi-Cluster** | ACM (Advanced Cluster Management) | Hub-and-Spoke, federation |
| **Developer Portal** | Red Hat Developer Hub (RHDH) | API catalog, self-service |

---

# 4. Infrastructure Prerequisites {#prerequisites}

## 4.1 Cluster Requirements

| Requirement | Details | Rationale |
|-------------|---------|-----------|
| **OpenShift Container Platform** | Version 4.20 or newer, with cluster-admin privileges | Ensures compatibility with OSSM 3.2 and latest Kuadrant policies |
| **Topology** | Minimum of three distinct clusters: Hub (ACM/GitOps), East (Workloads), West (Workloads) | Essential for validating multi-cluster federation and resilience |
| **SNO (Single Node OpenShift)** | When deploying on SNO for PoC, increase `maxPods` (recommended minimum: 500) | Accommodates Service Mesh and Kuadrant control plane demands |

## 4.2 Required Operators

The following operators must be installed and configured by the Cluster Admin:

1. **OpenShift GitOps** — Declarative repository synchronization
2. **OpenShift Service Mesh 3 (Sail Operator)** — Istio Ambient Mode control plane
3. **Gateway API Operator** — Service routing and exposure
4. **Kuadrant Operator** — Rate Limiting and Auth Policies
5. **Red Hat Developer Hub (RHDH)** — API portal with Kuadrant plugin

## 4.3 Local Tooling

| Tool | Usage |
|------|-------|
| `oc` CLI | Login to all three cluster contexts |
| .NET 8.0 SDK + Node.js 20 | Local development and pre-deployment validation |
| Podman | Building, managing and local testing of UBI8 container images |
| Ansible | Multi-cluster initialization playbook execution |
| Helm 3 | `nfl-wallet` chart deployment |

---

# 5. GitOps Installation Guide {#deployment}

Installation is performed **declaratively** via OpenShift GitOps (ArgoCD), not through imperative commands.

## 5.1 Local Execution with Podman Compose

For local development, the full stack runs with **Podman Compose**:

```bash
# From the repo root
podman-compose up -d --build

# Access the application
# http://localhost:5160
```

Local services:
- **api-customers** — Port 5001
- **api-buffalo-bills** — Port 5002
- **api-las-vegas-raiders** — Port 5003
- **webapp** — Port 5160

[![Podman Compose]({{ '/images/podman.png' | relative_url }})]({{ '/images/podman.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Running the stack with Podman Compose: webapp and three APIs in local containers.</span>

## 5.2 Development with Red Hat OpenShift Dev Spaces

The repository includes a **devfile.yaml** for Red Hat OpenShift Dev Spaces, enabling development and testing in a cloud IDE without installing .NET or Node.js locally.

[![OpenShift Dev Spaces]({{ '/images/devspaces.png' | relative_url }})]({{ '/images/devspaces.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">OpenShift Dev Spaces workspace with the NFL-Wallet project.</span>

[![Dev Spaces Build]({{ '/images/devspaces2.png' | relative_url }})]({{ '/images/devspaces2.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Build and run in Dev Spaces: compile and start the webapp and APIs from the workspace.</span>

[![Dev Spaces App]({{ '/images/devspaces3.png' | relative_url }})]({{ '/images/devspaces3.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">App running from Dev Spaces: frontend and APIs served from the cloud workspace.</span>

## 5.3 Helm Chart Deployment

```bash
kubectl create namespace nfl-wallet

helm install nfl-wallet ./helm/nfl-wallet -n nfl-wallet
```

### Chart Values (Reference)

| Key | Description | Default |
|-----|-------------|---------|
| `global.imageRegistry` | Image registry | `quay.io` |
| `imageNamespace` | Registry namespace | `maximilianopizarro` |
| `apiCustomers.service.port` | Service port | `8080` |
| `apiBills.service.port` | Service port | `8081` |
| `apiRaiders.service.port` | Service port | `8082` |
| `webapp.service.port` | Service port | `5173` |
| `webapp.route.enabled` | Create OpenShift Route | `true` |
| `gateway.enabled` | Create Gateway + HTTPRoutes | `false` |
| `gateway.className` | GatewayClass | `istio` |
| `apiKeys.enabled` | Create Secret and inject API keys | `false` |
| `authorizationPolicy.enabled` | Istio AuthorizationPolicy for X-API-Key | `false` |
| `observability.rhobs.enabled` | RHOBS resources (ThanosQuerier, PodMonitor, UIPlugin) | `false` |

## 5.4 Apply the ArgoCD Root Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nfl-wallet-production
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: 'https://github.com/maximilianopizarro/nfl-wallet-gitops.git'
    targetRevision: HEAD
    path: helm/nfl-wallet
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: nfl-wallet
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

> Once applied, ArgoCD will deploy Deployments, Services, HTTPRoutes, and Kuadrant policies in order.

[![OpenShift Topology]({{ '/images/topology.png' | relative_url }})]({{ '/images/topology.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Topology view in OpenShift: webapp → api-customers, api-bills, api-raiders.</span>

---

# 6. Service Mesh 3 (Ambient Mode) {#service-mesh}

## 6.1 Zero-Sidecar Security Model

OSSM 3.2 in Ambient Mode separates L4 and L7 security functions into specialized components:

| Component | Layer | Function |
|-----------|-------|----------|
| **ztunnel** | L4 | Node-level security: mTLS for all East-West traffic, L4 telemetry, transport encryption |
| **Waypoint Proxy** | L7 | Dedicated per-service Envoy proxy: advanced L7 telemetry, complex HTTP routing, access control |

Waypoints are strategically deployed for `api-customers`, `api-bills`, and `api-raiders` without injecting sidecars into the application pods.

## 6.2 Ambient Mode Enrollment

The namespace enrolls in the mesh via a label, automatically applied by ArgoCD:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nfl-wallet
  labels:
    istio.io/dataplane-mode: ambient
```

**Validation:** Application pods do NOT have the `istio-proxy` container, but traffic is encrypted via mTLS managed by the ztunnel DaemonSet.

## 6.3 Waypoint Proxy

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: nfl-wallet-waypoint
  namespace: nfl-wallet
  labels:
    istio.io/waypoint-for: service
spec:
  gatewayClassName: istio-waypoint
  listeners:
  - name: mesh
    port: 15008
    protocol: HBONE
```

## 6.4 Federation & Trust

- **Shared Root CA:** A single Root CA securely distributed and trusted by all Data Clusters (East/West)
- **meshNetworks:** Configuration objects defining cross-cluster network reachability
- **East-West HBONE Gateways:** Secure L4 transport via HBONE for multi-primary service discovery and encrypted cross-region communication

---

# 7. Connectivity Link & Gateway API {#connectivity-link}

## 7.1 Ingress with HTTPRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nfl-wallet-frontend
spec:
  parentRefs:
  - name: nfl-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: webapp
      port: 5173
```

Four HTTPRoutes are created: webapp (`/`), api-customers (`/api-customers`), api-bills (`/api-bills`), api-raiders (`/api-raiders`), with URL rewrite to the backend.

## 7.2 Enable Gateway

```bash
helm install nfl-wallet ./helm/nfl-wallet -n nfl-wallet \
  --set gateway.enabled=true \
  --set gateway.className=openshift-gateway
```

## 7.3 Rate Limiting with Kuadrant

```yaml
apiVersion: kuadrant.io/v1beta2
kind: RateLimitPolicy
metadata:
  name: api-customers-limit
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: api-customers-route
  limits:
    "customer-api-standard":
      rates:
      - limit: 100
        duration: 1
        unit: minute
```

Enable Rate Limiting + Auth:

```bash
helm upgrade nfl-wallet ./helm/nfl-wallet -n nfl-wallet \
  --set gateway.enabled=true \
  --set gateway.rateLimitPolicy.enabled=true \
  --set gateway.authPolicy.enabled=true \
  --set gateway.authPolicy.bills.enabled=true
```

[![Connectivity Link]({{ '/images/connectivity-link.png' | relative_url }})]({{ '/images/connectivity-link.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Connectivity Link: Gateway API and HTTPRoutes exposing webapp and APIs.</span>

[![Connectivity Link with Auth]({{ '/images/connectivity-link-auth.png' | relative_url }})]({{ '/images/connectivity-link-auth.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Connectivity Link with Kuadrant AuthPolicy (X-API-Key) and RateLimitPolicy on /api-bills.</span>

---

# 8. Security: API Keys & Policies {#security}

## 8.1 Two Security Models

| Model | Location | CRD | Use Case |
|-------|----------|-----|----------|
| **Istio AuthorizationPolicy** | Service Mesh (workload) | `security.istio.io/v1` | Direct pod-level validation |
| **AuthPolicy with Authorino** | Gateway (Kuadrant) | `kuadrant.io/v1` | Gateway-level validation with custom 403 |

## 8.2 Istio AuthorizationPolicy

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: api-raiders-require-apikey
spec:
  selector:
    matchLabels:
      app: api-raiders
  action: ALLOW
  rules:
    - when:
        - key: request.headers[x-api-key]
          values: ["*"]
```

## 8.3 Kuadrant AuthPolicy (Gateway)

```yaml
apiVersion: kuadrant.io/v1
kind: AuthPolicy
metadata:
  name: nfl-wallet-api-bills-auth
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: nfl-wallet-api-bills
  rules:
    authorization:
      require-apikey:
        opa:
          rego: |
            allow = true {
              input.context.request.http.headers["x-api-key"] != ""
            }
    response:
      unauthorized:
        body:
          value: '{"error":"Forbidden","message":"Missing or invalid X-API-Key header."}'
        headers:
          content-type:
            value: application/json
```

## 8.4 Security by Environment

| Environment | API Key | AuthPolicy |
|-------------|---------|------------|
| **Dev** | Not required | No authentication — `istio-injection: enabled` (sidecar mode) |
| **Test** | `nfl-wallet-customers-key` | AuthPolicy + API keys + `istio.io/dataplane-mode: ambient` |
| **Prod** | `nfl-wallet-customers-key` | AuthPolicy + API keys + canary route + ambient mode |

## 8.5 Namespace Access Restriction (Test / Prod) {#namespace-isolation}

In a multi-environment setup on the same cluster, it is critical that **test** services cannot access **prod** services and vice versa. OSSM3 in Ambient Mode provides this isolation through Istio **AuthorizationPolicy** at the namespace level.

### Isolation principle

Each namespace (`nfl-wallet-test`, `nfl-wallet-prod`) applies an **AuthorizationPolicy** that only allows traffic originating from the same namespace and from the mesh system (gateways, waypoints):

```yaml
# Access restriction: only same-namespace traffic in PROD
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: restrict-cross-namespace
  namespace: nfl-wallet-prod
spec:
  action: ALLOW
  rules:
  # Allow traffic from the same namespace
  - from:
    - source:
        namespaces: ["nfl-wallet-prod"]
  # Allow traffic from the Istio gateway/waypoint
  - from:
    - source:
        namespaces: ["istio-system"]
  # Allow traffic from ztunnel (ambient mode)
  - from:
    - source:
        namespaces: ["ztunnel"]
```

```yaml
# Equivalent restriction for TEST
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: restrict-cross-namespace
  namespace: nfl-wallet-test
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        namespaces: ["nfl-wallet-test"]
  - from:
    - source:
        namespaces: ["istio-system"]
  - from:
    - source:
        namespaces: ["ztunnel"]
```

### Isolation result

| Source → Destination | Allowed | Mechanism |
|----------------------|---------|-----------|
| nfl-wallet-test → nfl-wallet-test | Yes | Same-namespace rule |
| nfl-wallet-prod → nfl-wallet-prod | Yes | Same-namespace rule |
| nfl-wallet-test → nfl-wallet-prod | **No** | Blocked by AuthorizationPolicy |
| nfl-wallet-prod → nfl-wallet-test | **No** | Blocked by AuthorizationPolicy |
| nfl-wallet-dev → nfl-wallet-prod | **No** | Blocked by AuthorizationPolicy |
| istio-system → nfl-wallet-prod | Yes | Gateway/Waypoint ingress |
| External (via Gateway) → nfl-wallet-prod | Yes | Traffic enters through istio-system |

### Applied via Kustomize

The AuthorizationPolicies are included in each Kustomize overlay:

```
nfl-wallet/overlays/test/restrict-cross-namespace.yaml
nfl-wallet/overlays/prod/restrict-cross-namespace.yaml
nfl-wallet/overlays/test-east/restrict-cross-namespace.yaml
nfl-wallet/overlays/prod-west/restrict-cross-namespace.yaml
```

ArgoCD automatically syncs these policies when deploying each environment.

> **Dev without restriction:** The dev environment (`nfl-wallet-dev`) intentionally does not apply this restriction to facilitate development and cross-service debugging.

## 8.5.1 Cross-Namespace Policies with Gateway API & Istio {#gateway-namespace-policies}

Beyond workload-level AuthorizationPolicy, the **Gateway API** and **Istio** offer additional mechanisms to control traffic between namespaces. These options operate at different layers (L4/L7) and provide varying granularity.

### Option 1: ReferenceGrant (native Gateway API)

The `ReferenceGrant` resource (formerly `ReferencePolicy`) from the Gateway API controls which namespaces can reference resources in another namespace. This is useful for restricting which HTTPRoutes can point to Services in another namespace:

```yaml
# Allow ONLY HTTPRoutes from nfl-wallet-prod
# to reference the gateway in istio-system
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-prod-to-gateway
  namespace: istio-system
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: nfl-wallet-prod
  to:
  - group: ""
    kind: Service
```

Without a corresponding `ReferenceGrant`, HTTPRoutes from `nfl-wallet-test` **cannot** reference the Gateway in another namespace, preventing accidental route exposure between environments.

### Option 2: Istio PeerAuthentication (strict mTLS per namespace)

`PeerAuthentication` enforces strict mTLS in a namespace, ensuring that only pods with a valid SPIFFE identity from the same trust domain can communicate:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: strict-mtls
  namespace: nfl-wallet-prod
spec:
  mtls:
    mode: STRICT
```

Combined with AuthorizationPolicy, this ensures that even if a rogue pod attempts to send traffic, the ztunnel will reject the connection if it doesn't have a valid mTLS certificate from the allowed namespace.

### Option 3: Sidecar Resource (egress control per namespace)

The Istio `Sidecar` resource (also functional in Ambient Mode through Waypoints) limits the hosts to which a namespace can send outbound traffic:

```yaml
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
  name: restrict-egress
  namespace: nfl-wallet-test
spec:
  egress:
  - hosts:
    # Can only communicate with services in its own namespace
    - "./nfl-wallet-test/*"
    # And with the Istio system
    - "istio-system/*"
```

This prevents test services from discovering or attempting to connect to production services, as they won't be visible in the service registry.

### Option 4: Gateway Listeners with allowedRoutes (namespace scoping)

Gateway Listeners can restrict which namespaces can create HTTPRoutes that reference them:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: nfl-wallet-gateway
  namespace: nfl-wallet-prod
spec:
  gatewayClassName: istio
  listeners:
  - name: prod-listener
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: prod-tls-cert
    allowedRoutes:
      namespaces:
        from: Same                   # Only HTTPRoutes from the same namespace
```

```yaml
# Shared Gateway that accepts routes from specific namespaces
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gateway
  namespace: istio-system
spec:
  gatewayClassName: istio
  listeners:
  - name: prod-only
    port: 443
    protocol: HTTPS
    hostname: "*.prod.nfl-wallet.com"
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            environment: production  # Only namespaces with this label
  - name: test-only
    port: 443
    protocol: HTTPS
    hostname: "*.test.nfl-wallet.com"
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            environment: test
```

### Option 5: Kuadrant RateLimitPolicy per namespace

Kuadrant allows applying `RateLimitPolicy` directly to the Gateway, with differentiated limits by source namespace. This prevents one environment from monopolizing shared resources:

```yaml
apiVersion: kuadrant.io/v1beta2
kind: RateLimitPolicy
metadata:
  name: per-namespace-limits
  namespace: istio-system
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: shared-gateway
  limits:
    "test-namespace-limit":
      rates:
      - limit: 50
        duration: 1
        unit: minute
      when:
      - selector: metadata.filter_metadata.istio_authn.source.namespace
        operator: eq
        value: nfl-wallet-test
    "prod-namespace-limit":
      rates:
      - limit: 500
        duration: 1
        unit: minute
      when:
      - selector: metadata.filter_metadata.istio_authn.source.namespace
        operator: eq
        value: nfl-wallet-prod
```

### Options Comparison

| Mechanism | Layer | What It Controls | When to Use |
|-----------|-------|-----------------|-------------|
| **AuthorizationPolicy** | L4/L7 | Who can send traffic to a workload | Basic namespace isolation |
| **ReferenceGrant** | API | Which namespaces can create routes to a Gateway/Service | Control which environments use which gateways |
| **PeerAuthentication** | L4 | Requires strict mTLS for all traffic | Guarantee cryptographic identity |
| **Sidecar (egress)** | L7 | Which hosts a namespace can send traffic to | Limit service discovery |
| **allowedRoutes** | API | Which namespaces can create HTTPRoutes on a listener | Scoping shared gateways |
| **RateLimitPolicy** | L7 | How many requests per namespace | Prevent one environment from abusing the gateway |

> **Recommendation:** For NFL Wallet, we combine **AuthorizationPolicy** (workload isolation), **PeerAuthentication STRICT** (mandatory mTLS), and **allowedRoutes** on the Gateway (route scoping per namespace). This combination provides defense in depth.

## 8.6 Multi-Cluster Failover with DNSPolicy & Route 53 {#dns-failover}

To achieve **geographic high availability** and automatic failover between East and West clusters, Kuadrant integrates **DNSPolicy** with **Amazon Route 53** (or other compatible DNS providers). This ensures that if one cluster fails, traffic is automatically redirected to the healthy cluster.

### DNS Failover Architecture

<div class="diagram-box">
                        Route 53 (DNS)
                    ┌─────────────────────┐
                    │  nfl-wallet.nfl.com  │
                    │                     │
                    │  Health Check East ──┤──▶ Cluster East
                    │  Health Check West ──┤──▶ Cluster West
                    │                     │
                    │  Routing Policy:     │
                    │  Failover / Weighted │
                    └─────────┬───────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼                               ▼
    Cluster East (Primary)          Cluster West (Secondary)
    nfl-wallet-gateway-istio        nfl-wallet-gateway-istio
    ├── webapp                      ├── webapp
    ├── api-customers               ├── api-customers
    ├── api-bills                   ├── api-bills
    └── api-raiders                 └── api-raiders
</div>

### DNSPolicy Definition with Kuadrant

Kuadrant provides the `DNSPolicy` CRD that binds to the Gateway and automatically manages DNS records:

```yaml
apiVersion: kuadrant.io/v1
kind: DNSPolicy
metadata:
  name: nfl-wallet-dns-failover
  namespace: nfl-wallet-prod
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: nfl-wallet-gateway
  providerRefs:
  - name: aws-route53-credentials    # Secret with Route 53 credentials
  routingStrategy: loadbalanced       # Strategy: loadbalanced or simple
  loadBalancing:
    geo: us-east-1                    # Geographic region for this cluster
    defaultGeo: true                  # Default if geo doesn't match
    weight: 120                       # Relative weight for weighted routing
```

### Provider Configuration (Route 53)

The AWS credentials Secret for Route 53:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: aws-route53-credentials
  namespace: nfl-wallet-prod
type: Opaque
data:
  AWS_ACCESS_KEY_ID: <base64>
  AWS_SECRET_ACCESS_KEY: <base64>
  AWS_REGION: <base64>               # us-east-1
```

### DNS Routing Strategies

| Strategy | Behavior | Use Case |
|----------|----------|----------|
| **simple** | Single A/CNAME record | Single cluster, no failover |
| **loadbalanced** | Multiple records with health checks | Multi-cluster with automatic failover |

### Failover with Health Checks

When using `routingStrategy: loadbalanced`, Kuadrant automatically configures:

1. **Route 53 Health Checks:** Verify that the Gateway endpoint responds in each cluster
2. **Weighted DNS Records:** Distribute traffic between East and West based on configured weights
3. **Automatic Failover:** If the East health check fails, Route 53 stops resolving to East and sends all traffic to West

```yaml
# DNSPolicy for Cluster East (Primary)
apiVersion: kuadrant.io/v1
kind: DNSPolicy
metadata:
  name: nfl-wallet-dns-east
  namespace: nfl-wallet-prod
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: nfl-wallet-gateway
  providerRefs:
  - name: aws-route53-credentials
  routingStrategy: loadbalanced
  loadBalancing:
    geo: us-east-1
    defaultGeo: true
    weight: 120
```

```yaml
# DNSPolicy for Cluster West (Secondary)
apiVersion: kuadrant.io/v1
kind: DNSPolicy
metadata:
  name: nfl-wallet-dns-west
  namespace: nfl-wallet-prod
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: nfl-wallet-gateway
  providerRefs:
  - name: aws-route53-credentials
  routingStrategy: loadbalanced
  loadBalancing:
    geo: us-west-2
    defaultGeo: false
    weight: 80
```

### Result: DNS Resolution

| Scenario | East Health | West Health | DNS Resolution |
|----------|------------|------------|----------------|
| Normal | Healthy | Healthy | 60% East / 40% West (by weights 120:80) |
| East Down | Unhealthy | Healthy | 100% West (automatic failover) |
| West Down | Healthy | Unhealthy | 100% East |
| Both Down | Unhealthy | Unhealthy | No resolution (alert) |

> **Note:** DNSPolicy requires the Kuadrant operator to have access to the Route 53 API (or configured DNS provider). Credentials should be managed with **External Secrets Operator** or **Sealed Secrets** in production.

---

# 9. Multi-Cluster GitOps with ACM {#gitops}

## 9.1 Deployment Modes

### With ACM (Hub + Managed Clusters East/West)

```bash
# 1. Placements + GitOpsCluster (creates east/west secrets in ArgoCD)
kubectl apply -f app-nfl-wallet-acm.yaml -n openshift-gitops

# 2. ApplicationSet (generates 6 Applications)
kubectl apply -f app-nfl-wallet-acm-cluster-decision.yaml -n openshift-gitops
```

### Without ACM (Independent East/West)

```bash
kubectl apply -f app-nfl-wallet-east.yaml -n openshift-gitops
kubectl apply -f app-nfl-wallet-west.yaml -n openshift-gitops
```

## 9.2 Environments and Namespaces

| Environment | Namespace |
|-------------|-----------|
| Dev | `nfl-wallet-dev` |
| Test | `nfl-wallet-test` |
| Prod | `nfl-wallet-prod` |

## 9.3 Kustomize Overlay Structure

| Path | Use |
|------|-----|
| `nfl-wallet/overlays/dev` | Single-cluster dev |
| `nfl-wallet/overlays/test` | Single-cluster test |
| `nfl-wallet/overlays/prod` | Single-cluster prod |
| `nfl-wallet/overlays/dev-east` | ACM: dev on east |
| `nfl-wallet/overlays/dev-west` | ACM: dev on west |
| `nfl-wallet/overlays/test-east` | ACM: test on east |
| `nfl-wallet/overlays/test-west` | ACM: test on west |
| `nfl-wallet/overlays/prod-east` | ACM: prod on east |
| `nfl-wallet/overlays/prod-west` | ACM: prod on west |

## 9.4 GitOps Repository Structure

```
.
├── app-nfl-wallet-acm.yaml                   # Placements + GitOpsCluster (ACM)
├── app-nfl-wallet-acm-cluster-decision.yaml  # ApplicationSet (list generator)
├── app-nfl-wallet-east.yaml                  # ApplicationSet east (no ACM)
├── app-nfl-wallet-west.yaml                  # ApplicationSet west (no ACM)
├── kuadrant.yaml                             # Kuadrant CR
├── nfl-wallet/                               # Kustomize (routes, AuthPolicy, API keys)
│   ├── base/                                 # gateway route
│   ├── base-canary/                          # canary route (prod)
│   └── overlays/                             # dev, test, prod + east/west
├── nfl-wallet-observability/                 # Grafana + ServiceMonitors
├── observability/                            # Grafana Operator base
├── docs/                                     # Documentation
└── scripts/                                  # force-sync-apps, test-apis, etc.
```

[![OpenShift GitOps]({{ '/images/gitops.png' | relative_url }})]({{ '/images/gitops.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">OpenShift GitOps (ArgoCD) — Applications and sync status.</span>

[![GitOps Applications]({{ '/images/gitops1.png' | relative_url }})]({{ '/images/gitops1.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">ArgoCD — Detail of Applications generated by the ApplicationSet.</span>

[![ACM Topology]({{ '/images/ACM3.png' | relative_url }})]({{ '/images/ACM3.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">ACM — Topology with hub and managed clusters (East, West).</span>

[![ACM Applications]({{ '/images/ACM4.png' | relative_url }})]({{ '/images/ACM4.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">ACM — ApplicationSet and the 6 generated Applications (dev/test/prod × east/west).</span>

[![ACM Apps Overview]({{ '/images/acm-apps.png' | relative_url }})]({{ '/images/acm-apps.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">ACM — Overview of applications deployed on managed clusters.</span>

---

# 10. Red Hat Developer Hub {#developer-hub}

API governance is centralized through **Kuadrant** on the backend and **RHDH** on the frontend, providing a self-service experience for developers.

## 10.1 Self-Service Flow

1. **Discovery:** In the RHDH catalog, locate `nfl-wallet-api-customers` (Type: API - OpenAPI, Lifecycle: production)
2. **Request Access:** Click **+ Request API Access**
3. **Tier Configuration:** Select `silver (500 per daily)`
4. **Use Case (optional):** Technical or business justification
5. **Approval & Provisioning:** Kuadrant orchestrates credential creation (API Key or OIDC Token)
6. **Enforcement:** Gateway API intercepts, validates the credential and enforces the 500 requests/day limit

[![RHDH Kuadrant Policies]({{ '/images/rhdh-kuadrant-policies.png' | relative_url }})]({{ '/images/rhdh-kuadrant-policies.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Red Hat Developer Hub — Kuadrant Plugin: Policies view for nfl-wallet-api-customers. PlanPolicy and AuthPolicy discovered. Effective tiers: gold (1000/day), silver (500/day), bronze (100/day).</span>

[![RHDH API Definition]({{ '/images/rhdh-kuadrant-api-definition.png' | relative_url }})]({{ '/images/rhdh-kuadrant-api-definition.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Red Hat Developer Hub — API Definition: NFL Wallet - Customers API v1 (OAS 3.0). Endpoints GET /Customers and GET /Customers/{id} with Authorize button for authentication.</span>

[![RHDH Request Access]({{ '/images/rhdh-kuadrant-request-access.png' | relative_url }})]({{ '/images/rhdh-kuadrant-request-access.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Red Hat Developer Hub — Access request flow: "Request API Access" modal with Tier selection (silver - 500 per daily) and Use Case field. Owner: Maximiliano Pizarro, Lifecycle: production.</span>

[![RHDH API Keys]({{ '/images/rhdh-kuadrant-api-keys.png' | relative_url }})]({{ '/images/rhdh-kuadrant-api-keys.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Red Hat Developer Hub — Provisioned API Keys: Silver tier approved (2/3/2026), generated API Key with usage examples in cURL, Node.js, Python and Go.</span>

## 10.2 Using the API Key from Developer Hub

Once access is approved in RHDH, the portal generates an **API Key** linked to the requested Tier. This key is stored as a Kubernetes **Secret** with the label `api: <namespace>` (e.g. `api: nfl-wallet-prod`), which is the mechanism **Authorino** (Kuadrant) uses to discover and validate credentials.

### Complete flow: from portal to request

1. **RHDH generates the Secret** with the API Key and assigns the label `api: nfl-wallet-prod`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: consumer-api-key-silver-<hash>
  namespace: nfl-wallet-prod
  labels:
    api: nfl-wallet-prod
    authorino.kuadrant.io/managed-by: authorino
    tier: silver
type: Opaque
data:
  api_key: <base64-encoded-key>
```

2. **AuthPolicy references the label** `api: nfl-wallet-prod` as the credential selector. When a request arrives, Authorino searches all Secrets with that label and validates that the `X-Api-Key` header matches one of them.

3. **The consumer uses the key** obtained from the RHDH portal in their requests:

```bash
# cURL example (as shown in the RHDH portal)
curl -X GET https://nfl-wallet-prod.apps.cluster.example.com/api-customers/Customers \
  -H "X-Api-Key: <your-api-key>"
```

```python
# Python example (as shown in the RHDH portal)
import requests

headers = {"X-Api-Key": "<your-api-key>"}
response = requests.get(
    "https://nfl-wallet-prod.apps.cluster.example.com/api-customers/Customers",
    headers=headers
)
```

4. **The Gateway validates** the request: if the key matches a Secret with the correct label and the Tier has not exceeded its quota (e.g., 500 req/day for silver), the request reaches the backend. Otherwise, it returns **403 Forbidden** or **429 Too Many Requests**.

### Relationship: Label → AuthPolicy → Secret

<div class="diagram-box">
  RHDH Portal                    Kubernetes Cluster
  ┌──────────────┐               ┌──────────────────────────────────┐
  │ Request API  │               │  Namespace: nfl-wallet-prod      │
  │ Access       │───Approve───▶ │                                  │
  │ Tier: silver │               │  Secret (label: api=nfl-wallet-  │
  │              │               │  prod, tier=silver)              │
  └──────────────┘               │       │                          │
                                 │       ▼                          │
  Consumer                       │  AuthPolicy                     │
  ┌──────────────┐               │  (selector: api=nfl-wallet-prod)│
  │ curl -H      │               │       │                          │
  │ "X-Api-Key:  │───Request───▶ │       ▼                          │
  │  <key>"      │               │  Authorino validates key         │
  └──────────────┘               │  against labeled Secrets         │
                                 │       │                          │
                                 │       ▼                          │
                                 │  ✓ 200 OK → Backend              │
                                 │  ✗ 403 Forbidden                 │
                                 │  ✗ 429 Too Many Requests         │
                                 └──────────────────────────────────┘
</div>

> **Important:** API Key Secrets must exist in the same namespace as the AuthPolicy. For production, use **Sealed Secrets** or **External Secrets Operator** instead of committing keys directly to Git.

---

# 11. Observability {#observability}

## 11.1 Observability Stack

| Component | Function |
|-----------|----------|
| **Prometheus + promxy** | Fan-out proxy for metrics from East and West |
| **Grafana** | Dashboards: request rate, response codes, duration, error rate |
| **Kiali** | Real-time Service Mesh federated topology |
| **TempoStack** | Distributed tracing backend (Jaeger-compatible) |
| **OpenTelemetry** | Instrumentation with OTLP/HTTP — L7 spans from Waypoint proxies |

## 11.2 Enable Observability with Helm

```bash
helm upgrade nfl-wallet ./helm/nfl-wallet -n nfl-wallet --install \
  --set gateway.enabled=true \
  --set observability.rhobs.enabled=true \
  --set observability.rhobs.thanosQuerier.enabled=true \
  --set observability.rhobs.podMonitorGateway.enabled=true \
  --set observability.rhobs.uiPlugin.enabled=true
```

## 11.3 Grafana Dashboard

The "NFL Wallet – All environments" dashboard includes:
- **Environment (namespace)** variable to filter by nfl-wallet-dev, test, prod
- Panels: request rate, response codes, duration, error rate, rate by service

## 11.4 Prometheus Queries (Reference)

| Metric | Example Query |
|--------|---------------|
| **Total Requests** (rate) | `sum(rate(istio_requests_total[5m]))` |
| **Successful Requests** (2xx) | `sum(rate(istio_requests_total{response_code=~"2.."}[5m]))` |
| **Error Rate** | `sum(rate(istio_requests_total{response_code=~"5.."}[5m])) / sum(rate(istio_requests_total[5m]))` |

## 11.5 Traffic Test Script

```bash
export CLUSTER_DOMAIN="cluster-thmg4.thmg4.sandbox4076.opentlc.com"
export API_KEY_TEST="nfl-wallet-customers-key"
export API_KEY_PROD="nfl-wallet-customers-key"
./observability/run-tests.sh all
```

| Command | Description |
|---------|-------------|
| `./observability/run-tests.sh all` | Run dev, test and prod |
| `./observability/run-tests.sh dev` | Dev only (no API key) |
| `./observability/run-tests.sh test` | Test only (with API_KEY_TEST) |
| `./observability/run-tests.sh prod` | Prod only (with API_KEY_PROD) |
| `./observability/run-tests.sh loop` | Continuous loop: dev + test + prod |

[![Grafana Dashboard]({{ '/images/grafana-dashboard.png' | relative_url }})]({{ '/images/grafana-dashboard.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Grafana "NFL Wallet – All environments" dashboard with metrics: request rate, response codes, duration, error rate.</span>

[![Kiali Topology]({{ '/images/service-mesh-kiali-topology.png' | relative_url }})]({{ '/images/service-mesh-kiali-topology.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Kiali — Federated Service Mesh topology showing traffic flow across namespaces (dev/test/prod).</span>

[![Kiali Service Graph]({{ '/images/service-mesh-kiali.png' | relative_url }})]({{ '/images/service-mesh-kiali.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Kiali — Multi-cluster service graph with East/West traffic, gateways and waypoints.</span>

---

# 12. Screenshots {#screenshots}

## 12.1 Wallet Application

[![Wallet Landing]({{ '/images/walletlanding.png' | relative_url }})]({{ '/images/walletlanding.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Wallet Landing Page — Entry point of the NFL Stadium Wallet web application.</span>

[![Customer List]({{ '/images/wallet.png' | relative_url }})]({{ '/images/wallet.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Customer List — Select a customer to view their team wallets.</span>

[![Wallet Balances]({{ '/images/wallet2.png' | relative_url }})]({{ '/images/wallet2.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Wallet Balances — Buffalo Bills and Las Vegas Raiders: balances and transactions.</span>

[![QR Payment]({{ '/images/wallet3.png' | relative_url }})]({{ '/images/wallet3.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">QR Payment Flow — Payment from a team wallet.</span>

[![Load Balance]({{ '/images/wallet4.png' | relative_url }})]({{ '/images/wallet4.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Load Balance — Add funds to a team wallet.</span>

## 12.2 Platform & Observability

[![Grafana Dashboard]({{ '/images/grafana-dashboard.png' | relative_url }})]({{ '/images/grafana-dashboard.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Grafana — "NFL Wallet – All environments" dashboard: request rate, response codes, duration, error rate by environment.</span>

[![Service Mesh Grafana]({{ '/images/service-mesh-grafana.png' | relative_url }})]({{ '/images/service-mesh-grafana.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Kiali — Service graph with multi-namespace traffic (dev/test/prod) and HTTP metrics.</span>

[![Kiali Topology]({{ '/images/service-mesh-kiali-topology.png' | relative_url }})]({{ '/images/service-mesh-kiali-topology.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Kiali — Detailed Service Mesh topology with node legend, workloads and services.</span>

[![Kiali Multi-Cluster]({{ '/images/service-mesh-kiali.png' | relative_url }})]({{ '/images/service-mesh-kiali.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Kiali — Multi-cluster Service Graph showing traffic between East and West with Istio gateways.</span>

[![Service Mesh Overview]({{ '/images/service-mesh.png' | relative_url }})]({{ '/images/service-mesh.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">OpenShift Console — Service Mesh view: control planes, gateways, waypoints and components.</span>

[![API Customers]({{ '/images/api-customers.png' | relative_url }})]({{ '/images/api-customers.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">API Customers — Swagger UI for the customers service.</span>

[![API Bills]({{ '/images/api-bills.png' | relative_url }})]({{ '/images/api-bills.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">API Bills — Swagger UI for the Buffalo Bills wallet service.</span>

## 12.5 Red Hat Developer Hub — Kuadrant Plugin {#rhdh-screenshots}

[![RHDH Policies]({{ '/images/rhdh-kuadrant-policies.png' | relative_url }})]({{ '/images/rhdh-kuadrant-policies.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">RHDH Kuadrant Plugin — Policies tab: PlanPolicy and AuthPolicy discovered for nfl-wallet-api-customers. Effective tiers: gold (1000/day), silver (500/day), bronze (100/day).</span>

[![RHDH API Definition]({{ '/images/rhdh-kuadrant-api-definition.png' | relative_url }})]({{ '/images/rhdh-kuadrant-api-definition.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">RHDH Kuadrant Plugin — Definition tab: NFL Wallet - Customers API v1 (OAS 3.0) with documented endpoints and per-environment server selector.</span>

[![RHDH Request Access]({{ '/images/rhdh-kuadrant-request-access.png' | relative_url }})]({{ '/images/rhdh-kuadrant-request-access.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">RHDH Kuadrant Plugin — Access request modal: Silver tier selection (500 per daily), Use Case field and Submit Request button.</span>

[![RHDH API Keys]({{ '/images/rhdh-kuadrant-api-keys.png' | relative_url }})]({{ '/images/rhdh-kuadrant-api-keys.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">RHDH Kuadrant Plugin — Provisioned API Keys with approved Silver tier, generated key and code examples in cURL, Node.js, Python and Go.</span>

[![ACM Observability]({{ '/images/acm-observability-east-west.png' | relative_url }})]({{ '/images/acm-observability-east-west.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">ACM — ApplicationSet observability-east-west: topology with Configmap, Grafana, GrafanaDashboard, GrafanaDataSource, Namespace and Route for centralized observability.</span>

[![Grafana Multi-Cluster]({{ '/images/grafana-multi-cluster.png' | relative_url }})]({{ '/images/grafana-multi-cluster.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Grafana Multi-Cluster — "NFL Wallet - All environments" dashboard with cluster filter (East/West): request rate, response codes, request duration (p50/p99), total requests, error rate and request rate by service.</span>

[![GitOps ArgoCD]({{ '/images/gitops.png' | relative_url }})]({{ '/images/gitops.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">OpenShift GitOps (ArgoCD) — Applications and sync status.</span>

[![ACM Topology]({{ '/images/ACM3.png' | relative_url }})]({{ '/images/ACM3.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">ACM — Topology with hub and managed clusters (East, West).</span>

[![ACM Applications]({{ '/images/ACM4.png' | relative_url }})]({{ '/images/ACM4.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">ACM — ApplicationSet and the 6 generated Applications.</span>

[![ACM Overview]({{ '/images/ACM.png' | relative_url }})]({{ '/images/ACM.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">ACM — Advanced Cluster Management overview.</span>

[![ACM Detail]({{ '/images/ACM2.png' | relative_url }})]({{ '/images/ACM2.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">ACM — Managed clusters detail and status.</span>

[![Observability]({{ '/images/observability.png' | relative_url }})]({{ '/images/observability.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Observability — OpenShift console with monitoring stack metrics.</span>

[![Observability Metrics]({{ '/images/observability2.png' | relative_url }})]({{ '/images/observability2.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Gateway metrics (request rate, success and error rates) available after PodMonitor/ServiceMonitor configuration.</span>

[![Observability Detail]({{ '/images/observability3.png' | relative_url }})]({{ '/images/observability3.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Detailed observability view with Istio/Envoy metrics for the NFL-Wallet gateway.</span>

[![Traffic Analysis]({{ '/images/traffic-analysis.png' | relative_url }})]({{ '/images/traffic-analysis.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Traffic Analysis — Request flow, latency and response codes.</span>

[![Jaeger Traces]({{ '/images/jaeger-traces.png' | relative_url }})]({{ '/images/jaeger-traces.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Jaeger — Distributed traces for NFL Wallet services.</span>

[![Architecture Workflow]({{ '/images/architecture-workflow.png' | relative_url }})]({{ '/images/architecture-workflow.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Complete GitOps architecture workflow diagram.</span>

[![High Level Architecture]({{ '/images/high-level-architecture.png' | relative_url }})]({{ '/images/high-level-architecture.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">High-level architecture of the NFL Wallet ecosystem.</span>

---

# 12.3 Canary / Blue-Green Deployments {#canary}

The production overlay includes an additional **canary Route** (`nfl-wallet-canary.apps.<cluster-domain>`) that points to the same gateway Service (`nfl-wallet-gateway-istio`), enabling blue/green traffic when the chart creates the corresponding HTTPRoute.

### Canary Metrics by Environment

The following Grafana screenshots show traffic behavior during a canary deployment, showing the request distribution between dev, test and prod environments:

[![Canary Blue-Green - Total Requests]({{ '/images/canary-blue-green.png' | relative_url }})]({{ '/images/canary-blue-green.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Total requests (last hour) by environment during a canary deployment — nfl-wallet-dev (green), nfl-wallet-prod (yellow), nfl-wallet-test (blue). The gradual traffic increase to production is visible.</span>

[![Canary Blue-Green - Request Rate]({{ '/images/canary-blue-green-2.png' | relative_url }})]({{ '/images/canary-blue-green-2.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Request rate by environment and service during canary — Shows how api-customers (dev), gateway-istio (prod/test) and webapp distribute traffic between versions.</span>

### Canary Definition with HTTPRoutes

Canary deployments are implemented using **two HTTPRoutes** pointing to the same Gateway but with different hostnames. This allows splitting traffic between the stable version (production) and the canary version:

```yaml
# Main HTTPRoute (stable production)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nfl-wallet-webapp
  namespace: nfl-wallet-prod
spec:
  parentRefs:
  - name: nfl-wallet-gateway
  hostnames:
  - "nfl-wallet-prod.apps.cluster-east.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: webapp
      port: 5173
      weight: 100      # 100% stable traffic
```

```yaml
# Canary HTTPRoute (new version)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nfl-wallet-webapp-canary
  namespace: nfl-wallet-prod
spec:
  parentRefs:
  - name: nfl-wallet-gateway
  hostnames:
  - "nfl-wallet-canary.apps.cluster-east.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: webapp-canary        # Canary version Service
      port: 5173
      weight: 100
```

For a **weighted canary** (percentage-based traffic on the same hostname), use a single HTTPRoute with multiple `backendRefs`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nfl-wallet-webapp-weighted
  namespace: nfl-wallet-prod
spec:
  parentRefs:
  - name: nfl-wallet-gateway
  hostnames:
  - "nfl-wallet-prod.apps.cluster-east.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: webapp               # Stable version
      port: 5173
      weight: 90                 # 90% of traffic
    - name: webapp-canary        # Canary version
      port: 5173
      weight: 10                 # 10% of traffic
```

### Kustomize Configuration

The canary route is defined in the Kustomize overlays for production:

- **Path:** `nfl-wallet/overlays/prod/kustomization.yaml` (and `prod-east`, `prod-west`)
- **Host:** `nfl-wallet-canary.apps.<cluster-domain>`
- **Target:** Same gateway Service `nfl-wallet-gateway-istio`

The production overlay includes the OpenShift Route for the canary host:

```yaml
# nfl-wallet/overlays/prod/canary-route.yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: nfl-wallet-canary
  namespace: nfl-wallet-prod
spec:
  host: nfl-wallet-canary.apps.cluster-east.example.com
  to:
    kind: Service
    name: nfl-wallet-gateway-istio
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

To change the domain, edit the patch in each corresponding overlay.

---

# 12.4 Service Mesh — Topology & Traffic {#service-mesh-topology}

The following screenshots demonstrate the Service Mesh topology and actual traffic flow through the NFL Wallet ecosystem components.

### Kiali — Multi-Namespace Topology

[![Kiali Multi-Namespace Topology]({{ '/images/service-mesh-kiali-topology.png' | relative_url }})]({{ '/images/service-mesh-kiali-topology.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Kiali Traffic Graph — Topology view with 3 namespaces (nfl-wallet-prod, nfl-wallet-dev, nfl-wallet-test). Shows 5 apps, 10 services and 15 edges. The HTTP status bar shows response code distribution (OK, 3xx, 4xx, 5xx).</span>

This view shows:
- **webapp**, **api-customers**, **api-bills**, **api-raiders** as workloads in each namespace
- **nfl-wallet-gateway-istio** as the traffic ingress point
- Connections between services showing L7 flow through Waypoint proxies
- Real-time HTTP response code distribution

### Kiali — Multi-Cluster Service Graph (East/West)

[![Kiali Service Graph]({{ '/images/service-mesh-kiali.png' | relative_url }})]({{ '/images/service-mesh-kiali.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Kiali Service Graph — Multi-cluster view showing federated services between East and West clusters. 31 services, 5 workloads, 7 edges. Istio gateways in each namespace manage ingress traffic.</span>

Cross-cluster federation is visible:
- **East** cluster (left): dev, test, prod namespaces with their gateways and services
- **West** cluster (right): replicated dev, test, prod namespaces
- Cross-cluster communication via **HBONE** for federated services

### Kiali — Service Graph with Metrics

[![Service Mesh Grafana View]({{ '/images/service-mesh-grafana.png' | relative_url }})]({{ '/images/service-mesh-grafana.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">Kiali — Versioned app graph with real-time HTTP metrics: 5 apps, 9 services, 14 edges. Shows webapp, api-raiders, api-bills, api-customers nodes with per-namespace gateways.</span>

### OpenShift Console — Mesh View

[![Service Mesh Console]({{ '/images/service-mesh.png' | relative_url }})]({{ '/images/service-mesh.png' | relative_url }}){: .doc-img-link}
<span class="img-caption">OpenShift Console — Service Mesh overview showing: 1 Cluster (Kubernetes v1.33.6), 1 ControlPlane (Istio ambient v1.27.3, 4 dataplane namespaces), 4 Gateways (neuralbank-gateway + 3 nfl-wallet-gateway), 3 Waypoints (nfl-wallet-waypoint in dev/test/prod), 1 Kiali instance. Data plane components: Jaeger, Prometheus, Grafana, ztunnel, istio-system.</span>

This view demonstrates the complete Service Mesh 3 integration in Ambient Mode:
- **Control Plane:** Istio in ambient mode with 4 dataplane namespaces
- **Gateways:** 4 gateways (one per namespace + shared)
- **Waypoints:** 3 waypoint proxies (one per environment: dev, test, prod)
- **Integrated observability:** Jaeger, Prometheus and Grafana as mesh components

---

# 13. Test Plan & Validation (QA) {#testing}

Once ArgoCD synchronization is complete, the QA or Operations team must execute the following test plan to certify the deployment.

## 13.1 Test Matrix

| ID | Component | Test Description | Success Criteria | Status |
|----|-----------|-----------------|-----------------|--------|
| QA-01 | GitOps Sync | Verify in ArgoCD UI that `nfl-wallet` application is **Healthy** and **Synced** | All resources green; pods in `Running` state | <span class="rh-tag rh-tag--gold">Pending</span> |
| QA-02 | Ambient Mesh | Run `oc get pods -n nfl-wallet`. Confirm pods have only 1 container (no sidecar) | Pods show `1/1 READY` | <span class="rh-tag rh-tag--gold">Pending</span> |
| QA-03 | Egress (ESPN) | Access frontend pod or invoke `/api/bills/scoreboard` | Valid JSON with NFL scores from ESPN | <span class="rh-tag rh-tag--gold">Pending</span> |
| QA-04 | RHDH Portal | Navigate to Developer Hub, search `nfl-wallet-api-customers` and view OpenAPI docs | Swagger/OpenAPI spec renders correctly | <span class="rh-tag rh-tag--gold">Pending</span> |
| QA-05 | Rate Limiting | Generate a temp API Key (Silver Tier). Loop 505 HTTP GET requests to `/api/customers` | Request 501 must return **HTTP 429 Too Many Requests** | <span class="rh-tag rh-tag--gold">Pending</span> |
| QA-06 | AuthPolicy | Send request without `X-API-Key` to `/api-bills` endpoint (test/prod) | Response **HTTP 403** with JSON `{"error":"Forbidden",...}` | <span class="rh-tag rh-tag--gold">Pending</span> |
| QA-07 | Cross-Cluster | From webapp (East), query a customer's balance aggregating `api-bills` (East) and `api-raiders` (West) | UI displays balances from both teams correctly | <span class="rh-tag rh-tag--gold">Pending</span> |
| QA-08 | Observability | Verify in Grafana that `istio_requests_total` metrics are received from both clusters | Dashboard shows data from East and West | <span class="rh-tag rh-tag--gold">Pending</span> |
| QA-09 | Swagger UI | Navigate to `/api/swagger` for each API (api-customers, api-bills, api-raiders) | Swashbuckle UI renders correctly with documented endpoints | <span class="rh-tag rh-tag--gold">Pending</span> |
| QA-10 | Load Test | Run `./generate-traffic-realistic.sh --workers 20 --interval 1` | RateLimitPolicies enforce 100 req/min quota; excess traffic gets 429 | <span class="rh-tag rh-tag--gold">Pending</span> |

## 13.2 Rate Limiting Test Script (QA-05)

```bash
# Validate Silver limit (500/day)
for i in {1..505}; do
  STATUS_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
    -H "Authorization: Bearer $API_KEY" \
    https://api.nfl-wallet.mydomain.com/api/customers)
  echo "Request $i: HTTP Code $STATUS_CODE"
done
# Last 5 requests should show "HTTP Code 429"
```

## 13.3 API & Traffic Verification

```bash
# Health check for all three APIs
scripts/test-apis.sh

# Interactive documentation
# Navigate to: https://<api-customers-route>/api/swagger

# Load simulation (20 concurrent workers)
./generate-traffic-realistic.sh --workers 20 --interval 1
```

## 13.4 UI Verification (Cross-Cluster)

1. **Customer List:** Verify the frontend successfully receives the customer list from `api-customers`
2. **Cross-Cluster Balance:** Select a profile and verify the UI aggregates balances from `api-bills` (East) and `api-raiders` (West) — confirms the cross-cluster federation backbone

---

# 14. API Reference {#api-reference}

| Service | Service Port | Pod Port | API Path | Documentation |
|---------|-------------|----------|----------|---------------|
| api-customers | 8080 | 8080 | `/api` | `/api/swagger` |
| api-bills | 8081 | 8080 | `/api` | `/api/swagger` |
| api-raiders | 8082 | 8080 | `/api` | `/api/swagger` |
| webapp | 5173 | 8080 | `/` | N/A |
| Kiali Dashboard | 443 | N/A | `/` | Centralized Hub |
| Grafana | 443 | N/A | `/` | Centralized Hub |

### URLs by Environment

| Environment | Host Pattern | Example |
|-------------|-------------|---------|
| Dev | `nfl-wallet-dev.apps.<clusterDomain>` | `nfl-wallet-dev.apps.cluster-thmg4...opentlc.com` |
| Test | `nfl-wallet-test.apps.<clusterDomain>` | `nfl-wallet-test.apps.cluster-thmg4...opentlc.com` |
| Prod | `nfl-wallet-prod.apps.<clusterDomain>` | `nfl-wallet-prod.apps.cluster-thmg4...opentlc.com` |

### API Keys by Environment

| Environment | Key (customers) | Header |
|-------------|-----------------|--------|
| Dev | Not required | — |
| Test | `nfl-wallet-customers-key` | `X-Api-Key` |
| Prod | `nfl-wallet-customers-key` | `X-Api-Key` |

---

# 15. Troubleshooting {#troubleshooting}

## Pods Cannot Communicate (Error 503)

**Cause:** Ambient mode dataplane components are unstable.

```bash
# Restart CNI pods
oc -n istio-cni delete pod -l k8s-app=istio-cni-node

# Restart ztunnel
oc -n ztunnel delete pod -l app=ztunnel
```

## ArgoCD Shows "Out of Sync"

**Cause:** Someone modified a resource directly on the cluster.

**Solution:** Force sync in ArgoCD → Sync → Replace.

## HTTP 403 Forbidden

**Cause:** AuthPolicy active but API Key not sent, or access pending approval in RHDH.

**Solution:** Verify `X-API-Key` header in requests. Check approval status in Developer Hub.

## HTTP 500 on /api-bills with AuthPolicy

**Cause:** AuthConfig in `istio-system` not correctly linked to gateway host.

```bash
# Verify AuthConfig
kubectl get authconfig -n istio-system

# Patch host if needed
kubectl patch authconfig <HASH> -n istio-system \
  --type=json -p='[{"op":"replace","path":"/spec/hosts","value":["<gateway-host>"]}]'
```

## SNO CSR Approval Failure

```bash
oc get csr | grep Pending | awk '{print $1}' | xargs oc adm certificate approve
```

## CORS Failure (Frontend/Backend)

**Solution:** Ensure `CORS__AllowedOrigins` in API deployments matches the webapp's public URL.

- **Development:** `*` (wildcard)
- **Production:** Specific frontend HTTPS URL

## HTTP 503 "Application is not available"

- With **ACM:** Use the managed cluster domain (east/west), not the hub
- Verify Route: `oc get route -n nfl-wallet-prod`
- Verify pods: `oc get pods -n nfl-wallet-prod`

## No Data in Grafana

1. Generate traffic: `./observability/run-tests.sh loop`
2. Verify Prometheus targets (Status → Targets)
3. Verify Service labels: `kubectl get svc -n nfl-wallet-prod -l gateway.networking.k8s.io/gateway-name`
4. In Grafana Explore, run `istio_requests_total`

---

# 16. Publish to Artifact Hub {#artifact-hub}

```bash
# 1. Package the chart
helm package helm/nfl-wallet --destination docs/

# 2. Update the Helm repo index
cd docs
helm repo index . --url https://maximilianopizarro.github.io/NFL-Wallet --merge index.yaml
cd ..

# 3. Commit and push
```

Users can install:

```bash
helm repo add nfl-wallet https://maximilianopizarro.github.io/NFL-Wallet
helm repo update
helm install nfl-wallet nfl-wallet/nfl-wallet -n nfl-wallet
```

---

<div style="text-align:center; margin-top:3rem; padding:2rem; background:var(--rh-gray-100); border-radius:4px;">
  <p style="font-size:0.9rem; color:var(--rh-gray-500);">
    <strong>NFL Stadium Wallet v2.0</strong> — Documentation generated for GitHub Pages<br>
    Stack: OpenShift 4.20+ · GitOps (ArgoCD) · OSSM 3.2 (Ambient Mode) · Kuadrant · Gateway API · RHDH · Vue.js · .NET 8<br>
    Owner: <a href="https://www.linkedin.com/in/maximiliano-gregorio-pizarro-consultor-it">Maximiliano Pizarro</a>
  </p>
</div>
