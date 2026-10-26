# Helm Package Management on Amazon EKS

**Kubernetes Package Management · Helm Release Lifecycle · AWS EKS · NGINX · Prometheus Metrics · Kubernetes Networking**

A hands-on cloud engineering project demonstrating Kubernetes application packaging, Helm chart customization, release lifecycle management, custom chart development, observability integration, and network troubleshooting on Amazon Elastic Kubernetes Service (EKS).

## 1. Project Overview

This repository documents the deployment and operation of NGINX applications using Helm on Amazon EKS.

The project demonstrates how Helm provides a declarative, reusable, and version-controlled approach to managing Kubernetes application releases, while integrating with AWS infrastructure and Kubernetes-native observability components.

Two separate Helm releases were deployed:

| Helm Release | Chart | Purpose |
|---|---|---|
| `tag-app` | Custom Helm chart | Deploy three NGINX replicas and expose the application through an AWS LoadBalancer |
| `tag-nginx` | Bitnami NGINX v25.2.12 | Deploy NGINX with a Prometheus-compatible metrics exporter and ServiceMonitor |

The project additionally demonstrates Helm chart validation, rendered manifest inspection, release upgrades, rollback operations, application testing, and Kubernetes network diagnostics.

### Technical Objectives

- Manage Kubernetes application releases using Helm.
- Understand the relationship between charts, templates, values, releases, and Kubernetes resources.
- Customize a third-party Helm chart.
- Create and deploy a custom Helm chart.
- Configure external access through a Kubernetes LoadBalancer Service.
- Demonstrate Helm upgrade, rollback, history, and uninstall operations.
- Integrate NGINX application metrics with Prometheus-compatible monitoring.
- Diagnose internal Kubernetes Service and Pod connectivity failures.

## 2. Technology Stack

| Layer | Technology | Responsibility |
|---|---|---|
| Cloud Infrastructure | AWS | Infrastructure hosting |
| Container Orchestration | Amazon EKS | Managed Kubernetes environment |
| Application Packaging | Helm v4.2.1 | Chart templating and release management |
| Container Runtime Workload | NGINX | HTTP web server |
| Application Configuration | YAML / Helm Values | Declarative workload configuration |
| External Networking | AWS LoadBalancer | Public HTTP application access |
| Internal Networking | Kubernetes Services | Service discovery and Pod traffic routing |
| Monitoring | NGINX Prometheus Exporter | Application metrics exposure |
| Monitoring Discovery | ServiceMonitor CRD | Prometheus Operator target configuration |
| Development | Ubuntu / WSL2 | Command-line environment |
| Version Control | Git / GitHub | Source and configuration management |

### Deployment Environment

| Property | Value |
|---|---|
| AWS Region | `us-east-1` |
| EKS Cluster | `shared-eks-cluster` |
| Kubernetes Namespace | `tag-eks-activity` |
| Custom Release | `tag-app` |
| Monitoring Release | `tag-nginx` |
| Custom Application Replicas | 3 |
| Application Port | 80 |
| Metrics Port | 9113 |

All application-specific resources were deployed in the dedicated `tag-eks-activity` namespace to reduce interference with other workloads sharing the EKS cluster.

Namespace separation does not independently guarantee complete network or security isolation.

## 3. Repository Structure

```text
tag-helm-eks/
├── README.md
├── .gitignore
│
├── charts/
│   ├── nginx/
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   ├── charts/
│   │   │   └── common/
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       ├── svc.yaml
│   │       ├── metrics-svc.yaml
│   │       ├── servicemonitor.yaml
│   │       └── ...
│   │
│   └── tag-app/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── _helpers.tpl
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── serviceaccount.yaml
│           ├── hpa.yaml
│           ├── ingress.yaml
│           ├── httproute.yaml
│           └── tests/
│               └── test-connection.yaml
│
├── values/
│   ├── nginx-default-values.yaml
│   ├── nginx-custom-values.yaml
│   └── nginx-metrics-values.yaml
│
└── docs/
    ├── nginx-rendered.yaml
    └── troubleshooting.md
```

**Security note:** `docs/nginx-rendered.yaml` contains a Kubernetes Secret resource and is excluded from Git using `.gitignore`. It is retained locally for inspection and should not be published without reviewing and sanitizing its contents.

## 4. Helm Architecture and Resource Relationships

### Helm Deployment Workflow

```text
Developer / Git Repository
          |
          v
     Helm CLI
          |
          +-- Chart.yaml
          +-- values.yaml
          +-- templates/
          |
          v
 Helm Template Rendering
          |
          v
 Kubernetes API Server
          |
          v
    Amazon EKS
          |
          +-- Deployment
          |      |
          |      +-- ReplicaSet
          |             |
          |             +-- NGINX Pod 1
          |             +-- NGINX Pod 2
          |             +-- NGINX Pod 3
          |
          +-- Service (LoadBalancer)
                 |
                 v
          AWS LoadBalancer
                 |
                 v
          External HTTP Client
```

Helm renders Kubernetes resource definitions from chart templates and configuration values, then submits the resulting resources to the Kubernetes API.

Kubernetes controllers reconcile those resource definitions into running workloads.

### Helm Resource Model

| Component | Responsibility |
|---|---|
| Chart | Reusable package containing Kubernetes templates and metadata |
| `Chart.yaml` | Chart name, type, version, and metadata |
| `values.yaml` | Default configuration parameters |
| `templates/` | Kubernetes resource templates |
| Release | Installed instance of a Helm chart |
| Revision | Recorded version of a Helm release |
| Namespace | Kubernetes resource scope |
| Deployment | Desired Pod replica management |
| Service | Stable network abstraction for application Pods |
| LoadBalancer | External network exposure through cloud infrastructure |

A Helm release is not the same as a Kubernetes Deployment. One Helm release may manage multiple Kubernetes resources.

## 5. Helm Installation and Repository Management

### Verify Helm

```bash
helm version
```

Environment used:

```text
Helm v4.2.1
```

### Configure Helm Repositories

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami

helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm repo update
helm repo list
```

### Search for Charts

```bash
helm search repo nginx
helm search repo bitnami/nginx
```

### Download the NGINX Chart

```bash
helm pull bitnami/nginx \
  --version 25.2.12 \
  --untar \
  --untardir charts/
```

The downloaded chart contains application templates and the Bitnami Common Library Chart dependency.

### Inspect Chart Metadata and Values

```bash
cat charts/nginx/Chart.yaml

helm show values bitnami/nginx \
  > values/nginx-default-values.yaml

helm show chart bitnami/nginx
```

The default values file provides a reference for available configuration options.

## 6. NGINX Deployment Using Helm

The initial NGINX deployment used the release name `my-nginx`.

### Install the Chart

```bash
helm install my-nginx charts/nginx \
  -n tag-eks-activity
```

### Inspect the Release

```bash
helm list -n tag-eks-activity

helm status my-nginx \
  -n tag-eks-activity

kubectl get deployments,pods,services \
  -n tag-eks-activity
```

### Customize the NGINX Service

The chart was subsequently configured to expose NGINX through a Kubernetes LoadBalancer Service.

Example configuration:

```yaml
service:
  type: LoadBalancer
```

Upgrade command:

```bash
helm upgrade my-nginx charts/nginx \
  -n tag-eks-activity \
  -f values/nginx-custom-values.yaml
```

The Kubernetes Service requested external load balancing, and AWS provisioned an associated LoadBalancer.

### Verify External HTTP Access

```bash
kubectl get svc \
  -n tag-eks-activity
```

HTTP connectivity was tested against the provisioned external endpoint.

**Observed result: HTTP 200 OK.**

## 7. Helm Release Lifecycle Management

Helm maintains release revisions, allowing configuration changes to be tracked and previous revisions to be restored.

### Upgrade

```bash
helm upgrade my-nginx charts/nginx \
  -n tag-eks-activity \
  -f values/nginx-custom-values.yaml
```

### Inspect Revision History

```bash
helm history my-nginx \
  -n tag-eks-activity
```

### Rollback

```bash
helm rollback my-nginx 1 \
  -n tag-eks-activity
```

Rollback restores the desired state represented by an earlier release revision and records a new revision.

### Uninstall

```bash
helm uninstall my-nginx \
  -n tag-eks-activity
```

These operations were demonstrated during the exercise. The initial `my-nginx` release was subsequently replaced by the metrics-enabled `tag-nginx` release.

The historical commands above describe completed lifecycle operations; they are not intended to be rerun against the current release.

## 8. Custom Helm Chart Development

A separate Helm chart was created for a three-replica NGINX application.

### Chart Creation

```bash
helm create tag-app
```

The chart was maintained under:

```text
charts/tag-app/
```

### Chart Metadata

`charts/tag-app/Chart.yaml`:

```yaml
apiVersion: v2
name: tag-app
description: Custom NGINX deployment chart for AWS EKS Helm lifecycle and connectivity testing
type: application
version: 0.1.0
appVersion: "1.16.0"
```

The `appVersion` field originated from Helm scaffolding and does not establish the actual running NGINX version.

### Application Values

Key configuration:

```yaml
replicaCount: 3

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: LoadBalancer
  port: 80
```

The mutable `latest` image tag is suitable for this exercise but should be replaced with an explicitly versioned image or immutable digest for reproducible deployments.

### Validate the Chart

```bash
helm lint charts/tag-app
```

Observed result:

```text
1 chart(s) linted, 0 chart(s) failed
```

### Render Kubernetes Manifests

```bash
helm template tag-app charts/tag-app \
  -n tag-eks-activity
```

Rendering allows inspection of the generated Kubernetes resources without installing them.

### Deploy the Custom Chart

```bash
helm install tag-app charts/tag-app \
  -n tag-eks-activity
```

### Verify the Deployment

```bash
kubectl get deployment,pods,svc \
  -n tag-eks-activity
```

**Verified application state:**

| Component | Observed Result |
|---|---|
| Helm Release | `tag-app` |
| Deployment | Running |
| Desired Replicas | 3 |
| Ready Replicas | 3 |
| NGINX Containers | Running |
| Service Type | LoadBalancer |
| External HTTP Response | 200 OK |

The three application Pods were distributed across three EC2 worker nodes during verification.

The Service registered three NGINX application endpoints.

## 9. Prometheus-Compatible NGINX Monitoring

The Bitnami NGINX chart was configured to expose application metrics using the NGINX Prometheus Exporter.

### Monitoring Architecture

```text
NGINX Application Container
             |
             | NGINX status information
             v
NGINX Prometheus Exporter
             |
             | HTTP /metrics
             | Port 9113
             v
Kubernetes Metrics Service
      tag-nginx-metrics
             |
             v
Kubernetes ServiceMonitor
          tag-nginx
             |
             | Target discovery configuration
             v
Prometheus Operator / Prometheus
             |
             v
Time-Series Metrics Storage
             |
             v
Grafana (optional visualization)
```

**Verification boundary:** This architecture illustrates the intended monitoring integration. The exporter endpoint and ServiceMonitor were verified, but Prometheus scraping and Grafana visualization were not demonstrated in this exercise.

### Enable Metrics

`values/nginx-metrics-values.yaml`:

```yaml
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
```

### Verify ServiceMonitor CRD Availability

```bash
kubectl get crd \
  servicemonitors.monitoring.coreos.com
```

The ServiceMonitor CRD was already present in the cluster.

### Validate the Chart

```bash
helm lint charts/nginx \
  -f values/nginx-metrics-values.yaml
```

Observed result:

```text
1 chart(s) linted, 0 chart(s) failed
```

### Render the Metrics Configuration

```bash
helm template tag-nginx charts/nginx \
  -n tag-eks-activity \
  -f values/nginx-metrics-values.yaml
```

### Deploy Metrics-Enabled NGINX

```bash
helm install tag-nginx charts/nginx \
  -n tag-eks-activity \
  -f values/nginx-metrics-values.yaml
```

### Verify Kubernetes Resources

```bash
kubectl get pods,svc,servicemonitor \
  -n tag-eks-activity
```

Verified resources:

| Resource | Name | Status |
|---|---|---|
| NGINX Pod | `tag-nginx` | 2/2 containers Running |
| Application Service | `tag-nginx` | LoadBalancer |
| Metrics Service | `tag-nginx-metrics` | ClusterIP, port 9113 |
| ServiceMonitor | `tag-nginx` | Created |

The NGINX Pod contained both the application container and the metrics exporter container.

## 10. Metrics Endpoint Verification

### Port-Forward the Metrics Service

```bash
kubectl port-forward \
  -n tag-eks-activity \
  svc/tag-nginx-metrics 9113:9113
```

In a separate terminal:

```bash
curl -sS --max-time 10 \
  http://localhost:9113/metrics
```

The endpoint returned Prometheus-compatible metrics.

### Filter NGINX Application Metrics

```bash
curl -sS --max-time 10 \
  http://localhost:9113/metrics \
  | grep '^nginx_'
```

### Observed Metrics

```text
nginx_connections_accepted 99
nginx_connections_active 1
nginx_connections_handled 99
nginx_connections_reading 0
nginx_connections_waiting 0
nginx_connections_writing 1
nginx_http_requests_total 48
nginx_up 1
```

### Metric Interpretation

| Metric | Meaning |
|---|---|
| `nginx_up` | Whether the exporter successfully collected NGINX status |
| `nginx_http_requests_total` | Cumulative HTTP requests |
| `nginx_connections_accepted` | Cumulative accepted connections |
| `nginx_connections_handled` | Cumulative handled connections |
| `nginx_connections_active` | Currently active connections |
| `nginx_connections_reading` | Connections reading request headers |
| `nginx_connections_writing` | Connections writing responses |
| `nginx_connections_waiting` | Idle keep-alive connections |

The observed value `nginx_up 1` indicates successful exporter-to-NGINX status collection at the time of the request.

The cumulative counters are not throughput measurements unless evaluated over time.

**Result:** NGINX application metrics were successfully exposed and retrieved.

Prometheus target discovery, successful server-side scraping, alerting, and Grafana visualization remain outside the verified scope.

## 11. Helm Test and Kubernetes Network Troubleshooting

The custom chart included a Helm test hook intended to verify connectivity to the application Service.

### Execute Helm Test

```bash
helm test tag-app \
  -n tag-eks-activity \
  --timeout 60s \
  --logs
```

### Observed Failure

The test container resolved the Service hostname to:

```text
tag-app:80 -> 10.100.160.218:80
```

However, the HTTP connection timed out.

### Diagnostic Investigation

The following checks were performed:

- Pod scheduling and container state.
- Pod logs and termination status.
- Kubernetes Service and EndpointSlice inspection.
- NetworkPolicy inspection.
- Temporary BusyBox diagnostic Pod.
- Direct connectivity to the application ClusterIP.
- Direct connectivity to an NGINX Pod IP.

### Results

| Test | Result |
|---|---|
| Application Pod readiness | Passed |
| External LoadBalancer HTTP | Passed |
| Service DNS resolution | Passed |
| Application EndpointSlice registration | Passed |
| Helm test Pod connectivity | Failed |
| Diagnostic Pod to ClusterIP | Failed |
| Diagnostic Pod to Pod IP | Failed |
| Namespace NetworkPolicy inspection | No policies found |

A separate EndpointSlice also contained the failed Helm test Pod as a non-ready endpoint because the Service selector matched the test Pod's labels.

This identified a chart selector issue, but did not establish the cause of the direct Pod-IP connectivity timeout.

### Technical Interpretation

The timeout against both the Service ClusterIP and direct Pod IP indicated that the problem was not demonstrated to be limited to Kubernetes Service virtual-IP translation.

Possible areas requiring further investigation include Pod networking, node-level routing, and AWS networking controls.

No definitive root cause was established.

The temporary diagnostic Pod and failed test Pod were deleted after investigation.

**See:** `docs/troubleshooting.md` for detailed diagnostic commands and results.

## 12. Post-Activity Discussion

### 12.1 Helm Versus Manual Kubernetes YAML

Manual Kubernetes deployments commonly use `kubectl apply` with explicitly maintained YAML manifests.

Helm adds packaging, configuration parameterization, dependency management, and release revision tracking.

| Capability | kubectl with YAML | Helm |
|---|---|---|
| Declarative resource management | Yes | Yes |
| Reusable templates | Requires additional tooling | Built in |
| Configuration overrides | Manual or external tooling | Values files |
| Release history | Not a native release abstraction | Built in |
| Application rollback | Resource-dependent | Helm release rollback |
| Chart dependencies | Not built in | Supported |
| Packaging | Manifest files | Versioned charts |

Helm is particularly useful when the same application must be deployed consistently across multiple environments.

It does not replace Kubernetes configuration management, observability, or networking diagnostics.

### 12.2 Helm in Multi-Tenant Kubernetes Environments

In a shared Kubernetes cluster, teams can maintain independently managed Helm releases in separate namespaces.

A robust multi-tenant design should consider:

- Namespace-based resource organization.
- Kubernetes RBAC and least-privilege access.
- ResourceQuota and LimitRange policies.
- NetworkPolicies where supported and appropriate.
- Consistent Helm release naming.
- Ownership of cluster-scoped resources.
- Shared infrastructure costs and operational boundaries.

This project used the dedicated `tag-eks-activity` namespace to scope its application resources.

The approach reduces accidental interference but does not independently guarantee tenant isolation.

### 12.3 Helm Integration with CI/CD and Argo CD

Helm charts can be validated automatically before deployment.

A typical pipeline may include:

```text
Git Push / Pull Request
          |
          v
    CI Workflow
          |
          +-- YAML Validation
          +-- Helm Lint
          +-- Helm Template
          +-- Security Scanning
          |
          v
   Reviewed Git Commit
          |
          v
       Argo CD
          |
          v
   Kubernetes API
          |
          v
   Amazon EKS Workloads
```

Helm and Argo CD serve complementary functions:

| Tool | Primary Responsibility |
|---|---|
| Helm | Chart packaging, templating, and configuration |
| CI Pipeline | Automated validation and quality checks |
| Argo CD | GitOps deployment and desired-state reconciliation |
| Kubernetes | Runtime orchestration and resource reconciliation |

In an Argo CD-managed Helm deployment, Argo CD typically uses Helm to render the manifests and manages reconciliation itself rather than relying on Helm CLI release operations.

This project discusses CI/CD and Argo CD integration conceptually; an Argo CD deployment was not implemented.

## 13. Results and Technical Outcomes

### Successfully Demonstrated

- Helm CLI and repository management.
- Third-party Helm chart retrieval and inspection.
- Configuration customization using values files.
- Helm installation, upgrade, rollback, and uninstall.
- Custom Helm chart development.
- Three-replica NGINX deployment.
- AWS LoadBalancer provisioning.
- External HTTP 200 verification.
- Helm chart linting and template rendering.
- NGINX metrics exporter deployment.
- ServiceMonitor creation.
- Application-level Prometheus-format metrics verification.
- Kubernetes Pod and Service network diagnostics.

### Known Limitations

| Area | Limitation |
|---|---|
| Helm Test | Internal application connectivity timed out |
| Root Cause | Not conclusively identified |
| Prometheus | Successful scraping not independently verified |
| Grafana | No dashboard verification performed |
| Custom Image | Uses mutable `nginx:latest` |
| Chart Metadata | Scaffold `appVersion` not aligned to a pinned application image |

The distinction between successful deployment, successful exporter operation, and successful end-to-end monitoring is intentionally maintained.

## 14. References

- [Helm Documentation](https://helm.sh/docs/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Bitnami NGINX Helm Chart](https://github.com/bitnami/charts/tree/main/bitnami/nginx)
- [Prometheus Operator Documentation](https://prometheus-operator.dev/)
- [NGINX Prometheus Exporter](https://github.com/nginx/nginx-prometheus-exporter)

---

**Project:** Helm Package Management on Amazon EKS

**Repository:** `tagitables/tag-helm-eks`

**Focus:** Cloud Infrastructure · Kubernetes · Helm · Release Management · Observability · Troubleshooting
