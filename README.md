# cell-based-architecture
-------------------------------

What is cell?

Cell is a self contained unit which is having its own compute resources, networking and database.

Why cell?

  1. To achieve fault isolation.
  2. To reduce blast radius

Mental model of traffic flow
-----------------------------

[ Client / Internet ]
         │
         ▼
    1. Route 53           (DNS & Global Routing)
         │
         ▼
2. Istio Ingress Gateway  (Edge TLS / Infrastructure Gateway)
         │
         ▼
      3. Tyk              (API Management & Policy Enforcement)
         │
         ▼
  4. EKS Cluster / App    (Internal Workloads / Services)

  Mental architectural mapping of Cell based architetcure.
  --------------------------------------------------------
  ```text
┌─────────────────────────────────────────────────────────────────────────────┐
│ EKS Cluster (Domain Cell)                                                   │
│                                                                             │
│  ┌────────────────────────┐  ┌────────────────────────┐                    │
│  │ Capability Cell 1      │  │ Capability Cell 2      │  ... (More Apps)   │
│  │ (ns: app-1)            │  │ (ns: app-2)            │                    │
│  │ App Pods + Istio Envoy │  │ App Pods + Istio Envoy │                    │
│  └───────────▲────────────┘  └───────────▲────────────┘                    │
│              │                           │                                  │
│  ┌───────────┴───────────────────────────┴───────────────────────────────┐  │
│  │ Service Mesh & Platform Layer                                         │  │
│  ├──────────────────────┬──────────────────────┬────────────────────────┤  │
│  │ ns: cell-fabric      │ ns: tyk              │ ns: harness-delegate   │  │
│  │ - Cell Operator      │ - Tyk Data Plane     │ - Harness Delegate     │  │
│  │ - Istio Control Plane│   (tyk-dp)           │   (Connects to         │  │
│  │ - OTEL Collector     │ - Tyk Pump           │    Harness SaaS)       │  │
│  │                      │ - Tyk Gateway        │                        │  │
│  └──────────────────────┴──────────────────────┴────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘


Component Analysis & Roles :
------------------------------------
1. Domain Cell (EKS Cluster):
   Role: The outer infrastructure boundary. If an entire EKS cluster fails, that whole domain cell goes down.

2. Capability Cells (Application Namespaces):
   Role: Soft isolation at the workload layer. Enforcing isolation here requires Kubernetes Resource Quotas and NetworkPolicies so one application namespace doesn't starve or impact another.

3. cell-fabric Namespace (The Platform Brain):
   Cell Operator: Manages the custom lifecycle and CRDs for your cells.

4. Istio (Control Plane / istiod): Configures mTLS, service routing, and traffic controls between capability cells.
5. OTEL (OpenTelemetry): Collects traces and metrics from Envoy proxies and application pods to give full cross-cell observability.

6. tyk Namespace (API Gateway Layer):
   Tyk Gateway / tyk-dp (Data Plane): Handles incoming API calls, enforces auth, rate-limiting, and routes traffic into the respective capability cell namespaces.

7. Tyk Pump: Asynchronously moves API metrics and analytics from Tyk Redis to your long-term storage or dashboard.

8. harness-delegate Namespace (CI/CD Deployment Control):
   Harness Delegate: Outbound worker polling Harness SaaS for deployment jobs, ensuring secure, pull-based GitOps/deployments without exposing cluster inbound ports.

Questions open for discussion : 
1. Data Isolation: Do these capability cells (application namespaces) share a single central database cluster, or does each capability cell bring its own dedicated database instance? (If the DB is shared, a database crash will still impact all namespaces).

2. Blast Radius Level: Is this EKS cluster a single cell among many EKS clusters across AWS regions/accounts, or is this single EKS cluster the primary boundary?

