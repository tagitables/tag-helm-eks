# Helm Test and Kubernetes Network Troubleshooting

**Amazon EKS · Kubernetes Networking · Helm Test Hooks · Service Discovery · EndpointSlices · Pod Connectivity**

## 1. Overview

This document records the investigation of an internal Kubernetes connectivity failure encountered during the deployment and testing of a custom Helm chart (`tag-app`) on Amazon Elastic Kubernetes Service (EKS).

Although the application deployed successfully with three healthy NGINX replicas and returned HTTP `200 OK` through its external AWS LoadBalancer, the Helm test hook failed to establish an HTTP connection to the application's internal Kubernetes Service.

The investigation examined Kubernetes DNS resolution, Service routing, Pod endpoints, NetworkPolicies, and direct Pod-to-Pod connectivity.

**Final outcome:** The application deployment and external access were successful. Internal connectivity tests failed, and the definitive root cause was not established.

## 2. Environment

| Property | Configuration |
|---|---|
| Cloud Provider | AWS |
| Kubernetes Platform | Amazon EKS |
| AWS Region | `us-east-1` |
| Cluster | `shared-eks-cluster` |
| Namespace | `tag-eks-activity` |
| Helm Release | `tag-app` |
| Helm Chart | `charts/tag-app` |
| Chart Version | `0.1.0` |
| Application | NGINX |
| Desired Replicas | 3 |
| Service Type | LoadBalancer |
| Service Port | 80 |
| Service ClusterIP | `10.100.160.218` |
| Test Container | BusyBox |
| Diagnostic Image | `busybox:1.36` |

All troubleshooting resources were created within the dedicated `tag-eks-activity` namespace. No shared EKS networking components or other teams' workloads were intentionally modified.

## 3. Expected Application Architecture

```text
                     External HTTP Client
                              |
                              v
                     AWS LoadBalancer
                              |
                              v
                  Kubernetes Service
                        tag-app
                  10.100.160.218:80
                              |
                +-------------+-------------+
                |             |             |
                v             v             v
           NGINX Pod 1   NGINX Pod 2   NGINX Pod 3
          172.31.1.163  172.31.2.238 172.31.2.153
                |             |             |
                +-------------+-------------+
                              |
                       Amazon EKS
```

The Kubernetes Deployment maintained three NGINX replicas.

The Service exposed the application through an AWS LoadBalancer and selected the corresponding application Pods as backends.

The Helm test was intended to verify that a temporary test Pod could reach the application using its internal Kubernetes Service name.

### Expected Internal Request Flow

```text
Helm Test Pod
      |
      | HTTP request to http://tag-app:80
      v
Kubernetes DNS
      |
      | Resolves tag-app
      v
Service ClusterIP
10.100.160.218:80
      |
      v
Kubernetes Service Routing
      |
      v
Ready NGINX Pod Endpoint
      |
      v
HTTP 200 OK
```

The expected HTTP response was not obtained from the test Pod.

## 4. Initial Application Verification

Before executing the Helm test, the custom application was deployed successfully.

### Verify Helm Release

```bash
helm list -n tag-eks-activity
```

### Verify Application Resources

```bash
kubectl get deployments,pods,svc \
  -n tag-eks-activity
```

### Observed Application Pods

| Pod | Pod IP | Status |
|---|---|---|
| `tag-app-cc97894-nvb4z` | `172.31.1.163` | Running |
| `tag-app-cc97894-ptwl2` | `172.31.2.238` | Running |
| `tag-app-cc97894-rffk4` | `172.31.2.153` | Running |

All three application Pods were ready and running at the time of verification.

### External Connectivity

The Kubernetes LoadBalancer Service was assigned an AWS load balancer endpoint.

An external HTTP request returned:

```text
HTTP/1.1 200 OK
Server: nginx/1.31.8
```

This established that the application was externally reachable at the time of testing.

It did not establish that internal Pod-to-Pod connectivity was functioning.

## 5. Helm Test Execution

The custom Helm chart included a test hook under:

```text
charts/tag-app/templates/tests/test-connection.yaml
```

The test was executed using:

```bash
helm test tag-app \
  -n tag-eks-activity \
  --timeout 60s \
  --logs
```

### Observed Failure

The Helm test failed after the temporary test Pod attempted to connect to the application's internal Service.

The container resolved the Service hostname to:

```text
tag-app:80
10.100.160.218:80
```

However, the HTTP request timed out.

The test Pod logs included:

```text
wget: can't connect to remote host (10.100.160.218): Connection timed out
```

### Interpretation

Successful hostname resolution indicated that the test container could resolve the Kubernetes Service name.

The subsequent timeout indicated that the HTTP connection was not completed within the configured test period.

DNS resolution alone does not establish successful TCP connectivity.

## 6. Test Pod Inspection

The failed test Pod was inspected using Kubernetes diagnostic commands.

```bash
kubectl describe pod tag-app-test-connection \
  -n tag-eks-activity
```

```bash
kubectl logs tag-app-test-connection \
  -n tag-eks-activity
```

### Observed Conditions

- The test Pod was scheduled.
- The BusyBox image was successfully pulled.
- The container started.
- The Service hostname resolved.
- The connection attempt timed out.
- The test terminated unsuccessfully.

These observations indicated that the failure occurred during the internal connectivity test rather than during initial container scheduling or image retrieval.

## 7. Kubernetes Service and EndpointSlice Investigation

The next investigation examined whether the Service had valid application endpoints.

### Inspect the Service

```bash
kubectl get svc tag-app \
  -n tag-eks-activity
```

### Inspect EndpointSlices

```bash
kubectl get endpointslices \
  -n tag-eks-activity
```

For detailed endpoint information:

```bash
kubectl get endpointslices \
  -n tag-eks-activity \
  -l kubernetes.io/service-name=tag-app \
  -o yaml
```

### Application Endpoints

The Service had three registered NGINX Pod addresses:

```text
172.31.1.163
172.31.2.238
172.31.2.153
```

These addresses corresponded to the running application Pods.

### Additional EndpointSlice

An additional EndpointSlice associated with the same Service contained the failed Helm test Pod:

```text
Endpoint IP: 172.31.1.12
Ready: false
Ports: null
```

This revealed that the Service's label selector also matched the test Pod.

### Selector Interpretation

Kubernetes Services select endpoints using Pod labels.

If a Helm test Pod shares labels matching the application's Service selector, it may be included in endpoint discovery even though it is not an application server.

In this case, the test Pod was not a ready backend and did not expose the expected application port.

This represented a chart selector-design issue that should be corrected.

However, the additional non-ready endpoint was not established as the cause of the observed TCP timeout, especially because a subsequent direct Pod-IP test also failed.

### Recommended Improvement

For a future chart revision, use a Service selector that matches only application Pods, and ensure the test Pod does not carry the complete application selector label set.

Inspect the current selectors and labels before modifying them:

```bash
kubectl get svc tag-app \
  -n tag-eks-activity \
  -o jsonpath='{.spec.selector}'
```

```bash
kubectl get pods \
  -n tag-eks-activity \
  --show-labels
```

No selector modification was verified as part of this troubleshooting exercise.

## 8. NetworkPolicy Inspection

Kubernetes NetworkPolicies can restrict traffic between Pods when supported and enforced by the cluster's networking implementation.

The namespace was inspected using:

```bash
kubectl get networkpolicy \
  -n tag-eks-activity
```

### Observed Result

```text
No resources found in tag-eks-activity namespace.
```

### Interpretation

No namespace-local Kubernetes NetworkPolicy resources were present at the time of inspection.

This did not rule out other networking restrictions, including AWS security groups, node-level configuration, cluster networking components, or additional infrastructure-level controls.

## 9. Temporary BusyBox Diagnostic Pod

To investigate connectivity independently of the Helm test hook, a temporary BusyBox Pod was created.

### Create Diagnostic Pod

```bash
kubectl run tag-network-debug \
  -n tag-eks-activity \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sleep 600
```

### Verify Pod State

```bash
kubectl get pod tag-network-debug \
  -n tag-eks-activity
```

The diagnostic Pod reached Running state.

Two connectivity tests were then performed.

## 10. Test A — Connectivity to the Service ClusterIP

The first test attempted to reach the application through its Kubernetes Service ClusterIP.

### Command

```bash
kubectl exec \
  -n tag-eks-activity \
  tag-network-debug -- \
  wget -T 5 -t 1 -q -O /dev/null \
  http://10.100.160.218:80/
```

### Observed Result

```text
wget: download timed out
command terminated with exit code 1
```

### Interpretation

The diagnostic Pod was unable to complete the HTTP request to the Service ClusterIP.

This reproduced the connectivity failure outside the Helm test hook.

The result indicated that the problem was not exclusive to Helm's test execution mechanism.

## 11. Test B — Direct Connectivity to an NGINX Pod IP

The second test bypassed the Kubernetes Service ClusterIP and targeted an application Pod directly.

### Command

```bash
kubectl exec \
  -n tag-eks-activity \
  tag-network-debug -- \
  wget -T 5 -t 1 -q -O /dev/null \
  http://172.31.1.163:80/
```

### Observed Result

```text
wget: download timed out
command terminated with exit code 1
```

### Interpretation

The direct Pod-IP request also timed out.

This was an important diagnostic observation because the request did not rely on Kubernetes Service virtual-IP translation.

The failure could therefore not be explained solely by an issue in Service ClusterIP routing.

However, the test did not identify the precise failing network component.

## 12. Comparative Connectivity Results

| Connectivity Path | Expected | Observed |
|---|---|---|
| External client → AWS LoadBalancer → NGINX | HTTP 200 | Passed |
| Helm test Pod → Service DNS | Name resolution | Passed |
| Helm test Pod → Service ClusterIP | HTTP response | Failed |
| Debug Pod → Service ClusterIP | HTTP response | Failed |
| Debug Pod → NGINX Pod IP | HTTP response | Failed |
| Service → registered application endpoints | Endpoints present | Passed |
| Application Pods → readiness checks | Ready | Passed |

### Diagnostic Significance

The results demonstrated a distinction between external application availability and internal workload connectivity.

A successful external LoadBalancer response did not guarantee that another Pod could reach the application through its internal Service or Pod IP.

The observed failures justified further investigation of the cluster's Pod networking and node-level traffic paths.

## 13. Potential Causes Requiring Further Investigation

The following were potential investigation areas, not confirmed causes.

### 13.1 Pod Networking

Inspect the Kubernetes CNI implementation and associated Pod network interfaces, routes, and IP allocation.

For an AWS EKS cluster, the Amazon VPC CNI configuration may be relevant.

### 13.2 AWS Networking Controls

Review applicable EC2 security groups, subnet routes, network ACLs, and other controls governing traffic between workload nodes and Pod addresses.

No specific AWS networking rule was proven responsible.

### 13.3 Node-to-Node Traffic

The application replicas were scheduled on different EC2 worker nodes.

Depending on the diagnostic Pod's placement, traffic may have traversed node boundaries.

Node placement and routing would require additional verification.

### 13.4 Application Listener

Confirm that NGINX listens on the expected container port and network interface.

Useful commands include:

```bash
kubectl describe pod tag-app-cc97894-nvb4z \
  -n tag-eks-activity
```

```bash
kubectl logs tag-app-cc97894-nvb4z \
  -n tag-eks-activity
```

The external HTTP 200 response supported that the application was serving requests through at least one working external path, but did not independently prove every internal path.

### 13.5 Service Selector Configuration

Correct the overly broad label matching that caused the test Pod to appear in an EndpointSlice.

This is a chart-quality improvement, even though it was not proven to be the underlying network failure.

## 14. Cleanup

The temporary diagnostic Pod was deleted after testing.

```bash
kubectl delete pod tag-network-debug \
  -n tag-eks-activity
```

The failed Helm test Pod was also deleted.

```bash
kubectl delete pod tag-app-test-connection \
  -n tag-eks-activity
```

The cleanup removed the temporary test resources without deleting the running application release.

No shared EKS networking resources were intentionally changed.

## 15. Final Incident Assessment

| Component | Final Verified Outcome |
|---|---|
| Helm installation of `tag-app` | Successful |
| NGINX Deployment | Successful |
| Three application replicas | Running |
| AWS LoadBalancer | Provisioned |
| External HTTP request | HTTP 200 |
| Service DNS resolution | Successful |
| EndpointSlice registration | Three application endpoints present |
| Helm test | Failed |
| Diagnostic Pod → ClusterIP | Timed out |
| Diagnostic Pod → Pod IP | Timed out |
| Root cause | Not established |
| Temporary test resources | Deleted |

### Conclusion

The custom Helm chart successfully deployed a three-replica NGINX application on Amazon EKS and exposed it through an AWS LoadBalancer.

However, internal HTTP requests originating from Kubernetes Pods failed when targeting both the application's ClusterIP and a direct application Pod IP.

The investigation confirmed that DNS resolution and application endpoint registration were functioning, while internal HTTP connectivity remained unsuccessful.

The root cause was not conclusively determined.

This exercise illustrates the importance of evaluating Kubernetes application health across multiple layers: Helm release state, Deployment readiness, Service configuration, endpoint discovery, internal network connectivity, and external application access.

**Status: Deployment successful; internal connectivity issue unresolved.**
