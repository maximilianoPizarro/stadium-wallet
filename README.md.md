Guía Oficial de Instalación, Pruebas y Arquitectura: Ecosistema NFL Wallet
Versión: 2.0 (Extendida)
Estado: Producción
Owner: Maximiliano Pizarro
Stack Tecnológico: OpenShift, OpenShift GitOps (ArgoCD), OpenShift Service Mesh 3 (Ambient Mode), Kuadrant, Gateway API, Red Hat Developer Hub (RHDH), Vue.js, .NET 8.

1. Resumen Ejecutivo
Este documento proporciona la guía definitiva para el despliegue, configuración y validación del ecosistema NFL Wallet. La plataforma adopta un enfoque moderno basado en GitOps, seguridad "Zero-Trust" sin sidecars mediante OSSM3 (Ambient Mode), y una gestión integral del ciclo de vida de las APIs a través de Kuadrant y Red Hat Developer Hub. El sistema se compone de un frontend interactivo y tres microservicios core (api-customers, api-bills, api-raiders), los cuales interactúan con fuentes de datos externas (API de ESPN) de forma segura y auditable.

2. Prerrequisitos de Infraestructura
Antes de iniciar la instalación, el clúster de OpenShift debe contar con los siguientes operadores instalados y configurados por el administrador (Cluster Admin):
OpenShift GitOps: Para la sincronización declarativa del repositorio.
OpenShift Service Mesh 3 (Sail Operator): Proveedor del plano de control de Istio Ambient Mode.
Gateway API Operator: Para el enrutamiento y exposición de servicios modernos.
Kuadrant Operator: Para la gestión de Rate Limiting y Auth Policies.
Red Hat Developer Hub (RHDH): Instancia desplegada con el plugin de Kuadrant habilitado.

3. Arquitectura y Flujos de Datos
La arquitectura se divide en tres capas principales: Interfaz de Usuario (RHDH/Frontend), Plano de Control/Enrutamiento (Gateway API + Kuadrant + OSSM3), y Plano de Datos (Microservicios + APIs Externas).
3.1 Diagrama de Arquitectura de Red y Service Mesh
Code snippet
graph TD
    subgraph Plano de Gestión
        DevHub[Red Hat Developer Hub\nAPI Portal]
        Argo[OpenShift GitOps\nSincronización Continua]
    end

    subgraph OpenShift Cluster (Namespace: nfl-wallet)
        GW[Gateway API / Kuadrant Ingress]
        
        subgraph OSSM3 Ambient Mesh (Zero-Trust)
            Z[ztunnel - L4 Secure Overlay / mTLS]
            WP[Waypoint Proxy - L7 Auth/Routing]
            
            UI[webapp - Vue.js port 5173]
            CAPI[api-customers - .NET port 8080]
            BAPI[api-bills - .NET port 8080]
            RAPI[api-raiders - .NET port 8080]
        end
    end
    
    subgraph Servicios Externos
        ESPN[API Pública de ESPN\nScoreboards & Stats]
    end

    User((Usuario Final)) --> GW
    Dev((Desarrollador)) --> DevHub
    Argo -- "Aplica Manifiestos" --> OpenShift Cluster
    
    GW --> Z
    Z <--> UI
    UI -- "Llamadas API" --> Z
    Z <--> WP
    WP --> CAPI
    WP --> BAPI
    WP --> RAPI
    
    RAPI -- "Egress Traffic" --> ESPN
    BAPI -- "Egress Traffic" --> ESPN
3.2 El Caso de Uso: Integración API ESPN
Los microservicios de facturación (api-bills) y equipos (api-raiders) requieren datos deportivos en tiempo real.
Endpoint Objetivo: https://site.api.espn.com/apis/site/v2/sports/football/nfl/scoreboard
Gestión de Tráfico de Salida (Egress): Gracias a OSSM3 Ambient Mode, las peticiones HTTP salientes desde los pods .NET son interceptadas por el ztunnel de su nodo. Esto permite aplicar políticas de salida, monitorizar latencias hacia ESPN y garantizar que los servicios internos solo se comunican con endpoints externos autorizados, sin la penalización de rendimiento de inyectar un contenedor sidecar en cada pod.

4. Guía de Instalación Paso a Paso (GitOps)
La instalación no se realiza mediante comandos imperativos, sino declarando el estado deseado en OpenShift GitOps (ArgoCD).
Paso 1: Configurar el Repositorio de Destino
Asegúrese de que el repositorio https://github.com/maximilianopizarro/nfl-wallet-gitops.git sea accesible desde el clúster.
Paso 2: Aplicar la Application Root de ArgoCD
Ejecute el siguiente manifiesto para que ArgoCD comience a gestionar el ciclo de vida del namespace nfl-wallet:
YAML
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
Nota del Administrador: Una vez aplicado, ArgoCD desplegará los Deployments, Services, HTTPRoutes, y las políticas de Kuadrant de forma ordenada.

5. Configuración de Connectivity Link y Service Mesh
El repositorio ossm3-ambient-mode define cómo la aplicación se integra con la malla de servicios de nueva generación.
5.1 Enrolamiento en Ambient Mode
El namespace se inscribe en la malla mediante una simple etiqueta. ArgoCD aplica esto automáticamente:
YAML
apiVersion: v1
kind: Namespace
metadata:
  name: nfl-wallet
  labels:
    istio.io/dataplane-mode: ambient
Validación: Verifique que los pods de la aplicación no tengan el contenedor istio-proxy, pero que el tráfico se encripte mediante mTLS (gestionado por el DaemonSet ztunnel a nivel de nodo).
5.2 Despliegue de Waypoint Proxy y L7 Policies
Para habilitar capacidades de Capa 7 (HTTP), desplegamos un Waypoint Proxy.
YAML
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

6. Gestión de APIs: Kuadrant y Red Hat Developer Hub
La gobernanza de las APIs se centraliza mediante Kuadrant en el backend y RHDH en el frontend, brindando una experiencia de autoservicio para los desarrolladores.
6.1 Definición de APIProduct en el Clúster
Kuadrant utiliza el recurso APIProduct y RateLimitPolicy para proteger los endpoints.
YAML
apiVersion: kuadrant.io/v1beta1
kind: RateLimitPolicy
metadata:
  name: api-customers-rlp
  namespace: nfl-wallet
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: api-customers-route
  limits:
    "silver-tier":
      rates:
      - limit: 500
        duration: 1d
6.2 Flujo de Autoservicio en RHDH (Basado en la Interfaz Visual)
El desarrollador que desee integrar la API de clientes de la NFL Wallet debe seguir este flujo en el portal:
Descubrimiento: En el catálogo de RHDH, localizar la entidad nfl-wallet-api-customers (Tipo: API - OpenAPI, Lifecycle: production).
Solicitud de Acceso: Hacer clic en el botón superior derecho + Request API Access.
Configuración del Tier: En la ventana modal de Kuadrant / DevHub:
Select Tier: Seleccionar la opción predefinida en el clúster: silver (500 per daily).
Use Case (opcional): Proveer una justificación técnica o de negocio para la revisión del administrador (API Owner).
Aprobación y Aprovisionamiento: Al presionar Submit Request, Kuadrant orquesta la creación de una credencial (API Key o Token OIDC) específica para ese consumidor.
Enforcement: A partir de ese momento, el Gateway API intercepta las llamadas a /api/customers, valida la credencial y aplica el límite estricto de 500 peticiones al día.

7. Plan de Pruebas y Validación (QA)
Una vez finalizada la sincronización de ArgoCD, el equipo de QA u Operaciones debe ejecutar el siguiente plan de pruebas para certificar el despliegue.
ID Prueba
Componente
Descripción de la Prueba
Criterio de Éxito
Estado
QA-01
GitOps Sync
Verificar en la UI de ArgoCD que la aplicación nfl-wallet esté en estado Healthy y Synced.
Todos los recursos en verde; pods en estado Running.
[ ]
QA-02
Ambient Mesh
Ejecutar oc get pods -n nfl-wallet. Confirmar que los pods tienen solo 1 contenedor (sin sidecar).
Pods muestran 1/1 READY.
[ ]
QA-03
Egress (ESPN)
Acceder al pod del frontend o invocar /api/bills/scoreboard.
La respuesta contiene un JSON válido con los scores de la NFL proveídos por ESPN.
[ ]
QA-04
RHDH Portal
Navegar a Developer Hub, buscar nfl-wallet-api-customers y visualizar la documentación OpenAPI.
La especificación Swagger/OpenAPI renderiza correctamente.
[ ]
QA-05
Rate Limiting
Generar una API Key temporal (Tier Silver). Realizar un bucle de 505 peticiones HTTP GET a /api/customers usando curl.
La petición 501 debe devolver un código HTTP 429 Too Many Requests.
[ ]

Script de Prueba de Rate Limiting (QA-05)
Bash
# Validar el límite Silver (500/día)
for i in {1..505}; do
  STATUS_CODE=$(curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $API_KEY" https://api.nfl-wallet.midominio.com/api/customers)
  echo "Petición $i: Código HTTP $STATUS_CODE"
done
# Las últimas 5 peticiones deben mostrar "Código HTTP 429"

8. Troubleshooting (Solución de Problemas Frecuentes)
Los Pods no se comunican entre sí (Error 503): * Solución: Revise el estado del ztunnel en el nodo correspondiente ejecutando oc get pods -n istio-system -l app=ztunnel. Reinicie el pod si es necesario.
ArgoCD indica "Out of Sync":
Solución: Alguien pudo haber modificado un recurso directamente en el clúster (oc edit). Fuerce una sincronización en ArgoCD seleccionando Sync -> Replace.
Las peticiones a la API devuelven HTTP 403 Forbidden:
Solución: La AuthPolicy de Kuadrant está activa pero no se está enviando la API Key en las cabeceras, o el acceso solicitado a través de Red Hat Developer Hub aún está en estado "Pendiente de aprobación" por el Owner.








NFL Stadium Wallet: Comprehensive Red Hat GitOps Deployment & Testing Guide (OpenShift 4.20+)

This guide provides a deep dive into the architecture, deployment mechanics, and validation procedures for the NFL Stadium Wallet solution, leveraging cutting-edge Red Hat technologies on OpenShift Container Platform 4.20 and beyond.1. Architectural Overview & Design Principles

The NFL Stadium Wallet is a mission-critical digital payments and identity platform, specifically engineered to handle the massive concurrent transaction volumes typical of large-scale stadium environments. Its architecture is founded on principles of high availability, data isolation, and geographic resilience through a federated multi-cluster design.Three-Tier Application Architecture Refined

The solution is structured into a classic, yet modern, three-tier model:
Tier
Component
Technology Stack
Core Functionality
Scalability & Resilience
Frontend Tier
Single Page Application (SPA) - webapp
Vue 3, Vite, served by Apache HTTP Server (UBI8 httpd-24)
Provides the unified user interface for fan login, real-time balance inquiries, and generation of QR codes for payments.
Designed for stateless operation; easily scaled via standard OpenShift Horizontal Pod Autoscaler (HPA).
Backend API Tier
Independent Microservices (3)
.NET 8.0 ASP.NET Core
ApiCustomers: Centralized, read-heavy identity management and customer profile services. ApiWalletBuffaloBills: Transactional logic and state for the Buffalo Bills venue. ApiWalletLasVegasRaiders: Transactional logic and state for the Las Vegas Raiders venue.
Microservices are independently deployable and scalable, adhering to the "shared-nothing" principle across transactional boundaries.
Data Layer
Persistent Storage
Independent SQLite Databases (customers.db, buffalobills.db, lasvegasraiders.db)
Local persistence for each respective API.
Strict Data Isolation: Ensures that a failure or breach in one team's wallet data does not compromise the others or the central customer registry. Note: For full production deployment, these would be migrated to high-availability database solutions like PostgreSQL on OpenShift, potentially using the Crunchy Data operator.

Multi-Cluster Topology & Federation with Service Mesh 3.2

To achieve resilience and optimal regional performance, the system utilizes a Hub-and-Spoke model, governed by Red Hat's platform and management tools.
Hub Cluster (Control Plane): This cluster acts as the centralized management and observability nexus.
Red Hat Advanced Cluster Management (ACM): Serves as the "Single Source of Truth" (SSoT) for configuration, managing the deployment lifecycle across all spokes.
OpenShift GitOps (Argo CD): The primary engine for declarative configuration management, driven by ACM's ApplicationSets.
Centralized Observability Hub: Hosts monitoring tools (Prometheus, Grafana, Kiali) aggregated from the spokes.
Data Clusters (East/West - Spoke Clusters): These clusters host the application workloads and are the endpoints for fan traffic.
OpenShift Service Mesh 3 (OSSM3) in Ambient Mode: Utilized for networking and security. Ambient mode adopts a zero-sidecar architecture, drastically reducing resource overhead and simplifying application lifecycle management by eliminating the need for sidecar injection.
Federation Backbone: Establishes secure, multi-primary connectivity between the East and West clusters. This is critical for the webapp (e.g., residing in the East) to securely and transparently access the ApiWalletLasVegasRaiders (residing in the West) as if it were a local service. This cross-cluster communication is facilitated using HBONE (HTTP/2-Based ENcryption) for secure L4 transport.
2. Environment Prerequisites & Local Preparation

A robust deployment requires a fully configured OpenShift environment and necessary local tooling.Cluster Requirements & Sizing
Requirement
Details
Rationale
OpenShift Container Platform
Version 4.20 or newer, with cluster-admin privileges.
Ensures compatibility with OSSM 3.2 and the latest Kuadrant policies.
Topology
Minimum of three distinct clusters: Hub (ACM/GitOps), East (Workloads), West (Workloads).
Essential for validating the multi-cluster federation and resilience model.
Single Node OpenShift (SNO) Note
When deploying on SNO for proof-of-concept, the maxPods setting must be increased (recommended minimum: 500).
Accommodates the resource demands of the Service Mesh's Connectivity Link and the Kuadrant control planes without node capacity constraints.

Tooling & CLI Access

The following must be installed and configured on the local workstation:
oc CLI: Must be logged into all three cluster contexts (oc login <hub-context>, oc login <east-context>, oc login <west-context>).
.NET 8.0 SDK & Node.js 20: Required for local development and pre-deployment validation of the backend and frontend logic.
Podman: Used for building, managing, and local testing of the UBI8-based container images before pushing to the registry.
Ansible: The primary engine for executing the automated multi-cluster initialization playbooks (e.g., setting up the initial Service Mesh trust boundary).
Registry Access

A valid login is required for the image registry. The solution pulls baseline images from quay.io/maximilianopizarro. Any customized builds or forks of the application must be pushed to a user-controlled namespace (e.g., quay.io/<your-namespace>) using the podman push command.3. Service Mesh 3.2 (Ambient Mode) Initialization

The deployment utilizes the Sail Operator to deploy OSSM 3.2 in Ambient mode. This architecture is central to the project's operational efficiency and security model.Trust Establishment and Federation Backbone

The integrity of the federation relies on a robust identity mechanism:
Shared Root CA: A single Root Certificate Authority (CA) is securely distributed and trusted by all Data Clusters (East/West). This mutual trust is the foundation for secure service-to-service communication across regional boundaries.
meshNetworks: Configuration objects that explicitly define the cross-cluster network reachability, enabling service discovery across the federated topology.
East-West HBONE Gateways: Dedicated gateway components that facilitate multi-primary service discovery and provide the secure L4 transport layer (HBONE) necessary for the seamless, encrypted communication between services in different regions.
Zero-Sidecar Security Model Deep Dive

Ambient mode separates L4 and L7 security functions into specialized, resource-efficient components:
L4 Security (ztunnel):
Functions at the node level, handling foundational security and telemetry.
Responsibilities include: Mutual TLS (mTLS) for all East-West traffic, L4 telemetry collection, and ensuring all cross-cluster traffic is wrapped in transport encryption.
L7 Policy (Waypoints):
These are dedicated, per-service Envoy proxies, only deployed where complex L7 logic is required.
In this architecture, Waypoints are strategically deployed specifically for the api-customers and the team-specific wallet APIs (api-bills, api-raiders).
Their role is to handle fine-grained functions like: Advanced Layer 7 telemetry collection and complex L7 policy enforcement (e.g., custom HTTP routing, advanced access control) without injecting any sidecars into the actual application pods.
4. GitOps Deployment Strategy with ACM and Argo CD

The entire application lifecycle is managed declaratively, ensuring consistency and auditability across environments.Hub Configuration and Orchestration

Deployment begins by applying the root configuration manifest:
app-nfl-wallet-acm.yaml: This file is applied to the ACM Hub cluster.
ApplicationSets: This ACM feature automatically generates and manages the lifecycle of multiple Argo CD Applications. It defines parameters to deploy the application across different clusters (East, West) and environments (dev, test, prod).
Placements: ACM's powerful scheduling mechanism is used to dictate where specific components land. Example: The configuration explicitly enforces a separation of duties and locality: placing the ApiWalletBuffaloBills (Bills API) in the east cluster and the ApiWalletLasVegasRaiders (Raiders API) in the west cluster, optimizing for regional data access and resilience.
Namespace and Helm Strategy
Namespace Strategy: All workloads are isolated into environment-specific namespaces: nfl-wallet-dev, nfl-wallet-test, and nfl-wallet-prod. This strict separation simplifies policy application and access control.
Helm Integration: The deployment relies on a standardized, version-controlled Helm chart:
Repo URL: [https://maximilianopizarro.github.io/NFL-Wallet](https://maximilianopizarro.github.io/NFL-Wallet)
Chart Name: nfl-wallet
5. Connectivity Link & Gateway Policy Configuration

Traffic ingress and robust security controls are implemented using the native Kubernetes Gateway API, augmented by the Kuadrant project (Rate Limiting and Authentication).Ingress with Gateway API (HTTPRoute)

The public-facing frontend is exposed using a standard HTTPRoute definition:
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nfl-wallet-frontend
spec:
  parentRefs:
  - name: nfl-gateway # Assumes a deployed Kuadrant/Gateway API managed Gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: webapp # Target the frontend service
      port: 5173 # The service port for the Vue/Vite frontend
Rate Limiting with Kuadrant

To maintain stability and prevent service exhaustion, the high-demand api-customers endpoint is protected by a RateLimitPolicy:
apiVersion: kuadrant.io/v1beta2
kind: RateLimitPolicy
metadata:
  name: api-customers-limit
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: api-customers-route # Targets the specific route for customer API
  limits:
    "customer-api-standard":
      rates:
      - limit: 100 # Maximum 100 requests
        duration: 1 # per 1
        unit: minute # minute
This policy enforces a standard quota of 100 requests per minute against the customer identity API.Security Policy with Authorino (AuthPolicy)

Access to sensitive customer and wallet data is secured at the gateway level using an AuthPolicy powered by Authorino, enforcing an X-API-Key mechanism:
apiVersion: kuadrant.io/v1beta2
kind: AuthPolicy
metadata:
  name: api-customers-auth
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: api-customers-route
  rules:
    authentication:
      "api-key-auth":
        apiKey:
          selector:
            matchLabels:
              api-key: "true" # Selects secrets/configs labeled as API keys
        credentials:
          authorizationHeader:
            prefix: "X-API-Key" # Expects the key in the custom header
This configuration ensures that any request reaching the api-customers-route must contain a valid X-API-Key in the request header, which is then validated against secrets managed within OpenShift.6. Observability Stack Configuration

A unified and centralized observability stack, hosted on the ACM Hub cluster, is essential for monitoring the federated data clusters.
Metric & Trace Aggregation:
promxy: Utilized as a fan-out proxy for Prometheus, allowing a single query endpoint to fetch metrics simultaneously from the Prometheus instances running in both the East and West data clusters.
TempoStack: A Jaeger-compatible distributed tracing backend used for ingesting and analyzing trace data.
Red Hat build of OpenTelemetry: This is the primary instrumentation method. It is configured to use OTLP/HTTP for trace ingestion. Crucially, the configuration specifies collecting L7 spans directly from the Waypoint proxies in the data clusters, providing deep visibility into the L7 traffic flow and policy enforcement.
Multi-Cluster Kiali: A single Kiali instance on the Hub cluster is configured to visualize the entire federated mesh. It draws data from the aggregated metrics and traces to provide a comprehensive, real-time topology map of service dependencies and traffic flow across the East and West regions.
7. Deployment Validation & Testing Procedures

Successful deployment must be validated through rigorous testing procedures covering API functionality, security, performance, and UI integration.API & Traffic Verification
Scripted Validation: Execute scripts/test-apis.sh. This script performs basic health checks (/health) and basic CRUD operations against all three backend APIs (api-customers, api-bills, api-raiders) to ensure they are accessible and functioning.
Interactive Documentation: Navigate to the /api/swagger endpoint for any deployed API (e.g., https://<api-customers-route>/api/swagger). This verifies the ASP.NET Core Swashbuckle UI is correctly exposed and allows for interactive testing of all documented API endpoints.
Traffic Generation & Policy Enforcement: Execute the load simulation tool: ./generate-traffic-realistic.sh --workers 20 --interval 1. This simulates a high-load stadium scenario with 20 concurrent workers hitting the endpoints every second. The primary objective is to verify that the Kuadrant RateLimitPolicies are actively enforcing the configured 100 requests/minute quota and gracefully throttling excess traffic (returning 429 Too Many Requests).
UI Verification (Cross-Cluster Functionality)

Access the exposed webapp route and perform the following crucial checks:
Customer List Population: Verify that the frontend successfully communicates with and receives the customer list from the api-customers service.
Cross-Cluster Balance Aggregation: Select a customer profile. The UI must successfully aggregate and display the customer's balance by simultaneously querying the api-bills (potentially in East) and api-raiders (potentially in West) APIs. This confirms the successful operation of the cross-cluster federation backbone and seamless L4/L7 transport.
8. API Reference & Endpoint Mapping

The following table provides the service ports as defined in the nfl-wallet Helm chart for reference.
Service
Service Port (Cluster IP)
Pod/Container Port
API Path
Documentation
api-customers
 8080
 8080
 /api
 /api/swagger
api-bills
 8081
 8080
 /api
 /api/swagger
api-raiders
 8082
 8080
 /api
 /api/swagger
webapp
 5173
 8080
 /
 N/A
Kiali Dashboard
 443
 N/A
 /
 Centralized Hub Access
Grafana
 443
 N/A
 /
 Centralized Hub Access

9. Troubleshooting & RecoveryDataplane Connectivity Issues (503 Errors)

If services begin returning 503 "Service Unavailable" errors, the Ambient mode dataplane components may be unstable or require re-initialization.

Action: Restart the underlying Service Mesh components:
# Restart the CNI (Container Network Interface) pods
oc -n istio-cni delete pod -l k8s-app=istio-cni-node
# Restart the L4 security/transport tunnel pods
oc -n ztunnel delete pod -l app=ztunnel
SNO CSR Approval Failure

When running on Single Node OpenShift (SNO), major configuration changes (like increasing maxPods) often trigger a node reboot. If services fail to connect after the node returns, check for pending Kubernetes Certificate Signing Requests (CSRs).

Action: Approve all pending kubelet-serving CSRs:
oc get csr | grep Pending | awk '{print $1}' | xargs oc adm certificate approve
CORS Failure (Frontend/Backend Mismatch)

The most frequent failure when integrating the frontend with the backend is a Cross-Origin Resource Sharing (CORS) mismatch.

Cause: The browser blocks the frontend's request to the backend APIs because the API does not explicitly allow the frontend's origin URL.

Action: Ensure the CORS__AllowedOrigins environment variable in your API deployments (checked via appsettings.json or OpenShift Deployment environment variables) precisely matches the public URL of the webapp.
Development Environments: May be temporarily set to * (wildcard) for ease of testing.
Production Environments: Must be set to the specific, secured HTTPS route of the frontend (e.g., [https://wallet.nfl.com](https://wallet.nfl.com)).
