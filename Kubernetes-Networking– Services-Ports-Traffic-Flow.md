# Kubernetes Networking – Services, Ports & Traffic Flow
<img width="1219" height="782" alt="Screenshot 2026-11-24 at 1 03 32 PM" src="https://github.com/user-attachments/assets/60c87aca-b6f0-48ba-a42a-cf337b187b61" />


A simple guide to understanding how external and internal requests reach Pods in Kubernetes, how Services work, and when to use ClusterIP, NodePort, LoadBalancer, Ingress and Headless Services.

## 1. Why do we need Kubernetes Services?

In Kubernetes, Pods are created and deleted dynamically.

For example, imagine we deploy a Node.js backend application with three replicas:

- Backend Pod 1
- Backend Pod 2
- Backend Pod 3

Each Pod gets its own IP address.

If a Pod restarts, it can receive a different IP address.

So, how can other applications communicate with these Pods without knowing their individual IP addresses?

**Answer: Kubernetes Services.**

A Service provides a stable way to access a group of Pods.

It uses labels and selectors to identify matching Pods and routes traffic to eligible endpoints.

---

## 2. Understand Kubernetes IP Addresses

Kubernetes has three important types of IP addresses.

| IP Address | Belongs To | Purpose |
|---|---|---|
| Pod IP | Individual Pod | Identifies a Pod |
| ClusterIP | Kubernetes Service | Provides a stable internal Service IP |
| Node IP | Worker Node / EC2 | Identifies the Kubernetes worker machine |

Example:

```text
Worker Node IP: 10.0.1.20

Backend Pod 1:  10.0.2.11
Backend Pod 2:  10.0.2.12
Backend Pod 3:  10.0.2.13

Service IP:     10.100.5.10
```

**Remember:**

- Every Pod has its own Pod IP.
- Every ClusterIP Service has its own virtual IP.
- Every worker node has its own Node IP.

In AWS EKS with the standard Amazon VPC CNI, Pod IPs are generally allocated from your VPC subnet address space.

---

# 3. Types of Kubernetes Services

## A. ClusterIP Service

### What is ClusterIP?

ClusterIP is the default Kubernetes Service type.

It provides an internal IP address to communicate with applications inside the Kubernetes cluster.

### Why do we use it?

To allow internal communication between applications and distribute traffic across matching Pods.

### Real-time example

Suppose we have three Node.js backend Pods.

Instead of connecting to individual Pod IPs, we create a ClusterIP Service.

The Service forwards requests to eligible backend Pods.

### Architecture

```text
Frontend Application
        |
        v
Backend ClusterIP Service
        |
        +------ Backend Pod 1
        |
        +------ Backend Pod 2
        |
        +------ Backend Pod 3
```

### YAML Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: node-backend
  ports:
    - port: 80
      targetPort: 3000
      protocol: TCP
```

### When do we use it?

- Frontend-to-backend communication.
- Backend microservices communication.
- Internal APIs.
- Database access through a normal Service.

**Remember: ClusterIP = Internal communication + Stable virtual IP.**

---

## B. NodePort Service

### What is NodePort?

NodePort exposes a Kubernetes Service on a specific port of every worker node.

The default NodePort range is 30000–32767.

### Why do we use it?

To allow external clients to access an application through a worker node IP and NodePort.

### Real-time example

Suppose your worker node IP is:

```text
10.0.1.20
```

Your NodePort is:

```text
30080
```

You can access the application through:

```text
http://10.0.1.20:30080
```

### Architecture

```text
External User
      |
      v
Worker Node IP:30080
      |
      v
NodePort Service
      |
      +------ Backend Pod 1
      |
      +------ Backend Pod 2
      |
      +------ Backend Pod 3
```

### YAML Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-nodeport
spec:
  type: NodePort
  selector:
    app: node-backend
  ports:
    - port: 80
      targetPort: 3000
      nodePort: 30080
```

### When do we use it?

- Basic external access.
- Testing Kubernetes applications.
- Learning Kubernetes networking.
- As a traffic path behind certain load balancer configurations.

**Remember: NodePort = Node IP + NodePort number.**

---

## C. LoadBalancer Service

### What is LoadBalancer?

LoadBalancer exposes a Kubernetes Service externally using a cloud provider's load balancer.

In AWS EKS, this can provision an AWS Network Load Balancer (NLB) for a Service of type LoadBalancer.

### Why do we use it?

To expose an application externally without manually accessing individual worker node IPs.

### Real-time example

Suppose your application is running in AWS EKS.

You create a LoadBalancer Service.

AWS provisions a load balancer with an external address.

Users can access the application through that address.

### Architecture

```text
External User
      |
      v
AWS Network Load Balancer
      |
      v
LoadBalancer Service
      |
      +------ Backend Pod 1
      |
      +------ Backend Pod 2
      |
      +------ Backend Pod 3
```

### YAML Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-lb
spec:
  type: LoadBalancer
  selector:
    app: node-backend
  ports:
    - port: 80
      targetPort: 3000
```

### When do we use it?

- Exposing applications to external users.
- Public application access.
- Applications requiring a cloud-managed network load balancer.
- TCP/UDP services that need external access.

**Remember: LoadBalancer = External cloud load balancer.**

---

## D. Ingress

### What is Ingress?

Ingress is a Kubernetes API resource that defines rules for routing HTTP and HTTPS traffic to different Services.

An Ingress Controller implements those rules.

In AWS EKS, AWS Load Balancer Controller can create an Application Load Balancer (ALB) for Ingress resources.

### Why do we use it?

Suppose you have multiple applications:

- Frontend application.
- Backend API.
- Admin application.

Instead of creating a separate public load balancer for every application, you can use routing rules through an Ingress and share an ALB.

### Real-time example

```text
https://app.example.com
        |
        v
Frontend Service

https://api.example.com
        |
        v
Backend Service

https://admin.example.com
        |
        v
Admin Service
```

### Architecture

```text
          Internet User
                |
                v
         AWS Application
          Load Balancer
                |
                v
          Ingress Rules
                |
       +--------+--------+
       |        |        |
       v        v        v
   Frontend  Backend   Admin
   Service   Service   Service
       |        |        |
       v        v        v
     Pods     Pods     Pods
```

### When do we use it?

- Multiple applications.
- Domain-based routing.
- Path-based routing.
- HTTPS/TLS termination.
- Centralized HTTP/HTTPS traffic management.

**Remember: Ingress = HTTP/HTTPS routing to multiple Services.**

Note: Ingress is not a Service type. It is a routing resource that requires an Ingress Controller.

---

## E. Headless Service

### What is a Headless Service?

A Headless Service is a Kubernetes Service without a ClusterIP.

We configure it using:

```yaml
clusterIP: None
```

### Why do we use it?

It allows applications to discover individual Pod IP addresses through DNS instead of using one virtual Service IP for load balancing.

### Real-time example

Suppose we deploy MongoDB with three StatefulSet Pods:

```text
mongodb-0
mongodb-1
mongodb-2
```

Each Pod needs a stable network identity to communicate with other database members.

A Headless Service provides individual Pod DNS discovery.

### Architecture

```text
       MongoDB Headless Service
                 |
       +---------+---------+
       |         |         |
       v         v         v
   MongoDB-0 MongoDB-1 MongoDB-2
       |         |         |
       v         v         v
     Pod IP    Pod IP    Pod IP
```

Example DNS names:

```text
mongodb-0.mongodb-headless.default.svc.cluster.local

mongodb-1.mongodb-headless.default.svc.cluster.local

mongodb-2.mongodb-headless.default.svc.cluster.local
```

### YAML Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-headless
spec:
  clusterIP: None
  selector:
    app: mongodb
  ports:
    - port: 27017
      targetPort: 27017
```

### When do we use it?

- MongoDB replica sets.
- Kafka brokers.
- Cassandra clusters.
- StatefulSet applications.
- Distributed applications requiring individual Pod discovery.

**Remember: Headless Service = No ClusterIP + Individual Pod discovery.**

---

# 4. Understanding Kubernetes Ports

This is one of the most important concepts in Kubernetes networking.

Consider this example:

```text
Service Port:       80
Target Port:        3000
Container Port:     3000
NodePort:           30080
```

| Port | Meaning |
|---|---|
| port | Port exposed by the Kubernetes Service |
| targetPort | Port where traffic is forwarded inside the Pod |
| containerPort | Port declared in the container specification |
| nodePort | Port exposed on each worker node |

### Example traffic flow

```text
User Request
     |
     v
Service Port: 80
     |
     v
Target Port: 3000
     |
     v
Backend Pod
     |
     v
Node.js Application listening on 3000
```

Important: `containerPort` is a declaration in the Pod specification. It does not automatically make the application listen on that port.

The `targetPort` must match the actual port on which the application listens.

---

# 5. How Kubernetes Identifies Which Pod Receives Traffic

Imagine we have three backend Pods.

```text
backend-pod-1: 10.0.2.11:3000
backend-pod-2: 10.0.2.12:3000
backend-pod-3: 10.0.2.13:3000
```

All three Pods have the same label:

```yaml
labels:
  app: node-backend
```

Our Service has this selector:

```yaml
selector:
  app: node-backend
```

Kubernetes identifies all matching Pods using labels and selectors.

It maintains their endpoint information through EndpointSlices.

### Traffic distribution

```text
          Backend Service
           ClusterIP
               |
               v
          EndpointSlices
               |
       +-------+-------+
       |       |       |
       v       v       v
     Pod 1   Pod 2   Pod 3
     :3000   :3000   :3000
```

The Service networking implementation routes each connection to an eligible endpoint.

The actual distribution is not guaranteed to be perfectly equal between Pods.

### What happens if one Pod fails?

Suppose Pod 2 becomes Not Ready.

```text
Pod 1 = Ready
Pod 2 = Not Ready
Pod 3 = Ready
```

Kubernetes updates the EndpointSlice readiness information.

New traffic is routed only to eligible ready endpoints.

A Deployment can also create a replacement Pod to maintain the desired replica count.

---

# 6. Complete AWS EKS Production Architecture

Imagine deploying an e-commerce application.

It contains:

- React frontend.
- Node.js backend.
- MongoDB database.

### Architecture

```text
                  INTERNET
                     |
                     v
              AWS ALB / NLB
                     |
                     v
             Ingress Controller
                     |
              Ingress Rules
                     |
             +-------+-------+
             |               |
             v               v
       Frontend Service   Backend Service
          ClusterIP          ClusterIP
             |               |
             v               v
       Frontend Pods      Backend Pods
          (3 Pods)          (3 Pods)
                              |
                              v
                       MongoDB Service
                              |
                              v
                     MongoDB StatefulSet
                              |
                              v
                       Headless Service
                              |
                       MongoDB Pods
                         (3 Members)
                              |
                              v
                       Persistent Storage
                           AWS EBS
```

### Which component is used and why?

| Application / Requirement | Kubernetes Component | Reason |
|---|---|---|
| Frontend application | Deployment | Manage stateless replicas |
| Backend application | Deployment | Manage stateless replicas |
| Internal communication | ClusterIP | Stable internal access |
| Public HTTP/HTTPS access | Ingress + Controller | Route external traffic |
| External TCP service | LoadBalancer | Provision cloud load balancer |
| Basic node-level access | NodePort | Access through Node IP and port |
| MongoDB database | StatefulSet | Stable identity and storage association |
| MongoDB discovery | Headless Service | Individual Pod DNS |
| Database storage | PV + PVC | Persistent data |

---

# 7. How to Troubleshoot Kubernetes Networking

When an application is not accessible, check the following commands.

### Step 1: Check Services

```bash
kubectl get svc -A
```

### Step 2: Check Pods and Pod IPs

```bash
kubectl get pods -o wide
```

### Step 3: Check Service configuration

```bash
kubectl describe svc backend-service
```

### Step 4: Check EndpointSlices

```bash
kubectl get endpointslices \
  -l kubernetes.io/service-name=backend-service
```

### Step 5: Check Ingress

```bash
kubectl get ingress
```

### Step 6: Check application logs

```bash
kubectl logs <pod-name>
```

### Step 7: Check application listening port

```bash
kubectl exec -it <pod-name> -- sh
```

Inside the container:

```bash
ss -lntp
```

### Common problems

| Problem | What to check |
|---|---|
| Application not accessible | Service configuration |
| Wrong port | targetPort |
| No endpoints | Labels, selectors and Pod readiness |
| Pod connection refused | Application listening port |
| Ingress not working | Ingress Controller and routing rules |
| Load balancer unhealthy | Target health, probes and networking |
| Traffic blocked | NetworkPolicy and AWS security rules |

---

# 8. Quick Comparison

| Component | Internal / External | Main Purpose |
|---|---|---|
| ClusterIP | Internal | Stable Service access |
| NodePort | External via node | Node IP + NodePort |
| LoadBalancer | External | Cloud-managed load balancer |
| Ingress | External HTTP/HTTPS routing | Multiple applications |
| Headless | Internal discovery | Individual Pod DNS |

---

# 9. Important Kubernetes Interview Questions

1. What is the difference between ClusterIP and NodePort?
2. What is the difference between NodePort and LoadBalancer?
3. What is the difference between Ingress and LoadBalancer?
4. How does a Service identify matching Pods?
5. What is an EndpointSlice?
6. How does Kubernetes distribute traffic across multiple Pods?
7. What happens when a Pod becomes Not Ready?
8. What is the difference between Pod IP and ClusterIP?
9. What is the difference between port and targetPort?
10. Why do we use Headless Services with StatefulSets?
11. How does AWS Load Balancer Controller work with EKS?
12. How would you troubleshoot a Service that has no endpoints?

---

# 10. Final Things to Remember

- **ClusterIP:** Internal communication.
- **NodePort:** Access through worker node IP and port.
- **LoadBalancer:** External access using a cloud load balancer.
- **Ingress:** Route HTTP/HTTPS traffic to multiple Services.
- **Headless Service:** Discover individual Pods.
- **Pod IP:** Individual Pod address.
- **EndpointSlice:** Maintains Service backend endpoint information.
- **Labels and Selectors:** Identify matching Pods.

## Learning Goal

Understand how a request travels from the internet to an AWS EKS application, how Kubernetes identifies the correct backend Pods, and how to troubleshoot networking issues in a production environment.

**User → AWS Load Balancer → Ingress → Service → EndpointSlice → Pod → Application**<img width="1219" height="782" alt="Screenshot 2026-11-24 at 1 03 32 PM" src="https://github.com/user-attachments/assets/e5961b62-b190-4638-a589-2b749756b632" />
