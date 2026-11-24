# Kubernetes Fundamentals – Simple DevOps Guide

A beginner-friendly guide to understanding important Kubernetes components, when we use them, and why we use them in real-time AWS EKS projects.

## 1. ClusterIP Service

**What is it?**

ClusterIP is the default Kubernetes Service. It provides a stable internal IP address to access application Pods inside the Kubernetes cluster.

**Why do we use it?**

To allow different applications to communicate internally and distribute traffic across multiple Pods.

**Real-time example:**

Suppose we have three Node.js backend Pods. Instead of connecting to each Pod individually, we create a ClusterIP Service that distributes requests across the available Pods.

**When to use it?**
- Frontend-to-backend communication.
- Backend microservices communication.
- Internal application APIs.

**Remember:** ClusterIP = Internal communication + Load balancing.

---

## 2. Headless Service

**What is it?**

A Headless Service is a Kubernetes Service without a ClusterIP.

We configure it using:

`clusterIP: None`

**Why do we use it?**

It allows applications to discover individual Pod IP addresses through DNS instead of using a single virtual IP for load balancing.

**Real-time example:**

Suppose we deploy a MongoDB replica set with three Pods:
- mongodb-0
- mongodb-1
- mongodb-2

Each Pod needs a stable DNS identity to communicate with other database members.

A Headless Service helps provide this DNS discovery.

**When to use it?**
- MongoDB clusters.
- Kafka brokers.
- Cassandra clusters.
- StatefulSet applications.

**Remember:** Headless Service = No ClusterIP + Individual Pod discovery.

---

## 3. StatefulSet

**What is it?**

A StatefulSet manages applications that need stable identities and persistent storage.

Unlike ordinary Deployments, StatefulSets provide predictable Pod names and ordered deployment/scaling behavior.

**Why do we use it?**

Some applications need to maintain their identity even when Pods restart.

**Real-time example:**

A database cluster has three Pods:

- mysql-0
- mysql-1
- mysql-2

If mysql-1 restarts, Kubernetes recreates it with the same identity and can reconnect it to its associated persistent storage.

**When to use it?**
- MongoDB.
- MySQL.
- Kafka.
- Cassandra.
- Other stateful distributed applications.

**Remember:** StatefulSet = Stable identity + Persistent storage association.

---

## 4. Deployment

**What is it?**

A Deployment manages stateless application Pods in Kubernetes.

It helps create, update, scale and maintain the desired number of replicas.

**Why do we use it?**

To ensure our applications are running and to perform rolling updates without manually managing individual Pods.

**Real-time example:**

Suppose we deploy a Node.js application with three replicas.

Kubernetes maintains three Pods. If one Pod crashes, Kubernetes creates a replacement.

**When to use it?**
- React applications.
- Node.js applications.
- Python APIs.
- Java microservices.
- Stateless applications.

**Remember:** Deployment = Manage stateless applications + Scaling + Rolling updates.

---

## 5. Ingress

**What is it?**

Ingress manages external HTTP and HTTPS access to applications inside a Kubernetes cluster.

It uses routing rules to direct incoming requests to different Services.

**Why do we use it?**

Instead of exposing every application separately, we can use one entry point to route traffic to multiple applications.

**Real-time example:**

Suppose we have:

- `app.example.com` → Frontend Service
- `api.example.com` → Backend Service

Ingress routes incoming requests to the appropriate Kubernetes Service.

**When to use it?**
- Hosting multiple applications.
- Domain-based routing.
- Path-based routing.
- HTTPS/TLS termination.

**Remember:** Ingress = External HTTP/HTTPS routing to Services.

---

## 6. PersistentVolume (PV)

**What is it?**

A PersistentVolume is storage made available to Kubernetes workloads independently of an individual Pod's lifecycle.

**Why do we use it?**

When a Pod is deleted or recreated, we don't want to lose important application data.

**Real-time example:**

Suppose we run a MySQL database inside Kubernetes.

If the MySQL Pod crashes, the database files should remain available on persistent storage.

We can use AWS EBS volumes as persistent storage for EKS workloads.

**When to use it?**
- Database storage.
- Application files.
- Persistent user data.
- Stateful applications.

**Remember:** PV = Persistent storage beyond Pod lifecycle.

---

## 7. PersistentVolumeClaim (PVC)

**What is it?**

A PersistentVolumeClaim is a request for storage made by a Kubernetes application.

**Why do we use it?**

Applications request storage through PVCs instead of directly managing the underlying storage resource.

**Real-time example:**

A MySQL application requests 20Gi of storage using a PVC.

Kubernetes binds the claim to a suitable PersistentVolume or dynamically provisions storage through a StorageClass.

**Remember:** PVC = Application's request for persistent storage.

---

# 8. Quick Comparison

| Kubernetes Component | Main Purpose |
|---|---|
| ClusterIP | Internal communication |
| Headless Service | Individual Pod discovery |
| StatefulSet | Manage stateful applications |
| Deployment | Manage stateless applications |
| Ingress | External HTTP/HTTPS routing |
| PersistentVolume | Persistent storage |
| PersistentVolumeClaim | Request storage |

# 9. Real-Time AWS EKS Project Example

Imagine deploying an e-commerce application on AWS EKS.

| Application | Kubernetes Component |
|---|---|
| React Frontend | Deployment |
| Node.js Backend | Deployment |
| Internal Backend API | ClusterIP |
| MongoDB Database | StatefulSet |
| MongoDB Pod Discovery | Headless Service |
| External Website Access | Ingress |
| Database Storage | PV + PVC |

## 10. Important Interview Questions

1. What is the difference between ClusterIP and Headless Service?
2. What is the difference between Deployment and StatefulSet?
3. Why do we use Ingress?
4. What happens to application data when a Pod restarts?
5. What is the difference between PV and PVC?
6. Can we use a Headless Service with a StatefulSet?
7. How do you expose an application running inside EKS to the internet?

---

**My Learning Notes**

Kubernetes is used to deploy, manage, scale and maintain containerized applications.

Understanding these seven components is a good foundation for working with real-time Kubernetes and AWS EKS production environments.
