<img width="1219" height="816" alt="Screenshot 2026-11-24 at 1 10 37 PM" src="https://github.com/user-attachments/assets/194e262b-33f8-4c60-8586-7d454f9182fb" />

# Kubernetes Concepts – Simple DevOps Guide

A quick reference to understand Kubernetes components, their purpose, and real-world usage in AWS EKS projects.

## 1. Kubernetes Architecture

| Concept | What is it? | Why / Where do we use it? |
|---|---|---|
| Cluster | Collection of Control Plane and Worker Nodes | Runs containerized applications. |
| Control Plane | Manages the Kubernetes cluster | Controls and maintains cluster operations. |
| API Server | Entry point for Kubernetes requests | Receives requests from kubectl and other components. |
| etcd | Kubernetes database | Stores cluster configuration and state. |
| Scheduler | Assigns Pods to Worker Nodes | Decides where new Pods should run. |
| Controller Manager | Runs Kubernetes controllers | Maintains the desired cluster state. |
| Worker Node | Machine that runs application workloads | Hosts Pods and containers. |
| kubelet | Agent running on each node | Ensures assigned Pods are running. |
| kube-proxy | Service networking component, when deployed | Implements Service networking rules. |
| Container Runtime | Runs containers, such as containerd | Executes application containers. |

## 2. Workloads

| Concept | What is it? | Why / Where do we use it? |
|---|---|---|
| Pod | Smallest deployable Kubernetes unit | Runs application containers. |
| Deployment | Manages stateless application Pods | Backend and frontend applications. |
| ReplicaSet | Maintains the required Pod count | Ensures replicas are available. |
| StatefulSet | Manages stateful applications | MongoDB, Kafka and databases. |
| DaemonSet | Runs a Pod on every eligible node | Monitoring and logging agents. |
| Job | Runs a task until completion | Database migration and batch processing. |
| CronJob | Runs Jobs on a schedule | Daily backups and scheduled cleanup. |

## 3. Networking

| Concept | What is it? | Why / Where do we use it? |
|---|---|---|
| Pod IP | Unique IP assigned to a Pod | Identifies and connects to individual Pods. |
| ClusterIP | Internal Service IP | Frontend-to-backend communication. |
| NodePort | Exposes a Service on a worker node port | Basic external access and testing. |
| LoadBalancer | Exposes a Service through a cloud load balancer | Public application access. |
| Ingress | HTTP/HTTPS routing rules | Routes domains and paths to Services. |
| Headless Service | Service without a ClusterIP | Individual Pod discovery for StatefulSets. |
| EndpointSlice | Stores backend endpoint information | Tracks eligible Pod IPs and ports. |
| DNS | Service and Pod name resolution | Connects applications using DNS names. |
| NetworkPolicy | Controls Pod network traffic | Restricts unwanted communication. |

### Kubernetes traffic flow

```text
Internet
   |
   v
AWS Load Balancer
   |
   v
Ingress
   |
   v
ClusterIP Service
   |
   v
EndpointSlice
   |
   +---- Pod 1 :3000
   |
   +---- Pod 2 :3000
   |
   +---- Pod 3 :3000
```

## 4. Storage

| Concept | What is it? | Why / Where do we use it? |
|---|---|---|
| PV | PersistentVolume | Provides persistent storage. |
| PVC | PersistentVolumeClaim | Requests storage for an application. |
| StorageClass | Storage provisioning configuration | Dynamically creates storage. |
| CSI Driver | Storage integration interface | Connects Kubernetes to AWS EBS or EFS. |

**Example:** MongoDB Pod → PVC → PV → AWS EBS.

## 5. Configuration Management

| Concept | What is it? | Why / Where do we use it? |
|---|---|---|
| ConfigMap | Stores non-sensitive configuration | Environment variables and application settings. |
| Secret | Stores sensitive configuration | Database passwords and API credentials. |

## 6. Security

| Concept | What is it? | Why / Where do we use it? |
|---|---|---|
| ServiceAccount | Identity for a Pod or application process | Access Kubernetes APIs and authorized resources. |
| RBAC | Role-Based Access Control | Manages user and application permissions. |
| Role | Defines namespace-level permissions | Controls access within a namespace. |
| RoleBinding | Assigns a Role to an identity | Grants namespace permissions. |
| ClusterRole | Defines cluster-wide or reusable permissions | Manages access to cluster resources. |
| ClusterRoleBinding | Assigns a ClusterRole | Grants cluster-level permissions. |
| Pod Identity | AWS EKS workload identity integration | Gives Pods access to authorized AWS services. |

## 7. Scaling and Resources

| Concept | What is it? | Why / Where do we use it? |
|---|---|---|
| HPA | Horizontal Pod Autoscaler | Automatically increases or decreases Pod replicas. |
| VPA | Vertical Pod Autoscaler | Recommends or adjusts CPU and memory requests. |
| Requests | Minimum resources requested by a container | Helps Kubernetes schedule Pods. |
| Limits | Maximum CPU and memory usage allowed | Prevents excessive resource consumption. |

## 8. Health Checks

| Concept | What is it? | Why / Where do we use it? |
|---|---|---|
| Liveness Probe | Checks whether an application is alive | Restarts unhealthy containers. |
| Readiness Probe | Checks whether an application can receive traffic | Prevents traffic reaching unready Pods. |
| Startup Probe | Checks whether an application has started | Supports applications with slow startup. |

## 9. Deployment Strategies

| Concept | What is it? | Why / Where do we use it? |
|---|---|---|
| Rolling Update | Gradually replaces old Pods | Updates applications with minimal downtime. |
| Rollback | Returns to a previous Deployment revision | Recovers from failed releases. |
| Blue-Green | Maintains old and new application environments | Switches traffic after validation. |
| Canary | Releases a new version to a small traffic percentage | Tests new releases before full deployment. |

## 10. Monitoring and Logging

| Tool / Concept | What is it? | Why / Where do we use it? |
|---|---|---|
| Prometheus | Metrics monitoring system | Collects application and cluster metrics. |
| Grafana | Visualization platform | Creates monitoring dashboards. |
| Fluent Bit | Log collection agent | Collects and forwards logs. |
| OpenTelemetry | Observability framework | Collects traces, metrics and logs. |

---

## 11. Real-Time AWS EKS Architecture

Example: E-commerce application.

```text
                 INTERNET
                    |
                    v
              AWS ALB
                    |
                    v
                 INGRESS
                    |
           +--------+--------+
           |                 |
           v                 v
     Frontend Service   Backend Service
       ClusterIP          ClusterIP
           |                 |
           v                 v
      Frontend Pods     Backend Pods
       Deployment        Deployment
                             |
                             v
                       MongoDB Service
                             |
                             v
                       MongoDB StatefulSet
                             |
                             v
                        PVC + AWS EBS

Monitoring: Prometheus + Grafana
Logging: Fluent Bit
Security: RBAC + Secrets + NetworkPolicy
Scaling: HPA
```

## 12. Quick Revision – Which concept should I use?

| Requirement | Kubernetes Concept |
|---|---|
| Run an application | Pod |
| Manage application replicas | Deployment |
| Run a database cluster | StatefulSet |
| Run agents on every node | DaemonSet |
| Internal application communication | ClusterIP |
| External node-level access | NodePort |
| Public cloud load balancer | LoadBalancer |
| Route multiple websites | Ingress |
| Discover individual Pods | Headless Service |
| Store persistent data | PV + PVC |
| Store application settings | ConfigMap |
| Store credentials | Secret |
| Restrict access | RBAC |
| Automatically scale Pods | HPA |
| Check application health | Probes |
| Monitor application metrics | Prometheus |
| Create dashboards | Grafana |

---

## Learning Goal

Understand the purpose of each Kubernetes component, where it is used, and how the components work together in a real-world AWS EKS project.

**Learn → Deploy → Monitor → Troubleshoot → Scale**
