# GKE — The Interview-Friendly Guide

> **Google Kubernetes Engine (GKE), explained from zero to production**
>
> A practical, Medium-style guide for a Java Backend Engineer preparing for cloud interviews — especially useful if you already have the **Google Cloud Associate Cloud Engineer** certification.

---

# 1. First: What is GKE?

**GKE = Google Kubernetes Engine.**

In one sentence:

> **GKE is Google's managed service for running and managing Kubernetes clusters on Google Cloud.**

Let's break that down.

### What is Kubernetes?

Kubernetes is a system that helps you run **containers** reliably.

Suppose you have a Spring Boot application:

```text
Spring Boot Application
        ↓
     Docker
        ↓
    Container
```

Now imagine your application becomes popular.

You might need:

```text
        Users
          |
          v
     Load Balancer
       /   |   \
      /    |    \
   App    App    App
  Pod     Pod    Pod
```

And tomorrow:

* one application crashes
* traffic increases
* you need 20 instances instead of 3
* one machine fails
* you deploy a new version
* you need to roll back
* you need different configurations
* you need monitoring

Doing all of that manually becomes painful.

**Kubernetes automates much of this.**

GKE is Google's managed Kubernetes offering.

---

# 2. The 10-Year-Old Explanation

Imagine you own a restaurant.

You have:

* 10 chefs
* 5 kitchens
* hundreds of customers

You don't want to manually tell every chef:

> "Go cook this order."

Instead, you have a **manager**.

The manager:

* assigns work
* replaces unavailable chefs
* opens another kitchen when necessary
* makes sure enough chefs are available
* moves work around when something fails

Kubernetes is like that manager.

Google manages much of the Kubernetes infrastructure for you through **GKE**.

---

# 3. Why Does GKE Exist?

Without Kubernetes, you might have:

```text
VM 1 → Application
VM 2 → Application
VM 3 → Application
VM 4 → Application
```

You would need to manage:

* deployment
* scaling
* networking
* service discovery
* failures
* configuration
* container scheduling
* rolling updates

With Kubernetes:

```text
              Kubernetes
                  |
       -----------------------
       |          |          |
      Pod        Pod        Pod
       |          |          |
      App        App        App
```

Kubernetes continuously tries to make the actual system match your desired system.

That concept is **extremely important for interviews**.

---

# 4. Desired State

Suppose you tell Kubernetes:

> "I want 5 instances of my application."

Kubernetes remembers:

```text
Desired state = 5 Pods
```

But currently:

```text
Actual state = 4 Pods
```

Kubernetes notices the difference:

```text
Desired = 5
Actual  = 4
```

So it creates another Pod.

Now:

```text
Desired = 5
Actual  = 5
```

This is one of the fundamental ideas behind Kubernetes.

---

# 5. What is a Kubernetes Cluster?

A **cluster** is basically a collection of machines running Kubernetes.

Conceptually:

```text
             GKE Cluster
                  |
       -----------------------
       |                     |
 Control Plane            Nodes
                             |
                   ----------------
                   |      |       |
                  Pod    Pod     Pod
```

A cluster has two major sides:

1. **Control plane**
2. **Worker nodes**

---

# 6. Control Plane

The control plane is basically the **brain of Kubernetes**.

It decides:

* What should run?
* Where should it run?
* How many Pods should exist?
* What happens when a Pod dies?
* How should deployments happen?

Important components include:

```text
API Server
Scheduler
Controller Manager
etcd
```

### API Server

The API Server is the main communication point.

For example:

```bash
kubectl get pods
```

Your request goes roughly:

```text
kubectl
   |
   v
API Server
   |
   v
Kubernetes
```

---

# 7. etcd

`etcd` is Kubernetes' distributed key-value store.

Think:

> **etcd = Kubernetes' database containing cluster state.**

It stores things such as:

```text
Pods
Deployments
Services
Secrets
Configuration
Cluster state
```

You normally don't directly interact with etcd in GKE.

---

# 8. Scheduler

Suppose you create:

```text
Pod
```

But you haven't said which machine should run it.

The scheduler decides.

```text
Pod
 |
 v
Scheduler
 |
 +----> Node 1
 |
 +----> Node 2
 |
 +----> Node 3
```

It considers things such as:

* available resources
* CPU
* memory
* constraints
* affinity
* taints/tolerations

---

# 9. Controller Manager

Controllers continuously compare:

```text
Desired state
      vs
Actual state
```

Example:

```text
Desired Pods = 3
Running Pods = 2
```

Controller:

> "We need one more."

Creates:

```text
Pod 3
```

---

# 10. Worker Node

A node is a machine where your workloads actually run.

For example:

```text
GKE Cluster

Node 1
 ├── Pod A
 ├── Pod B
 └── Pod C

Node 2
 ├── Pod D
 └── Pod E
```

A node is typically a Compute Engine VM in GKE.

---

# 11. What is a Pod?

This is one of the **most commonly asked Kubernetes questions**.

A Pod is the **smallest deployable unit in Kubernetes**.

Usually:

```text
Pod
 |
 └── Container
       |
       └── Spring Boot Application
```

But a Pod can contain multiple tightly coupled containers:

```text
Pod
 ├── Application Container
 └── Sidecar Container
```

Containers inside the same Pod share:

* network namespace
* IP address
* localhost
* certain storage volumes

---

# 12. Pod ≠ Container

Very important.

A container is:

> A packaged application.

A Pod is:

> Kubernetes' unit for running one or more containers.

Usually:

```text
1 Pod → 1 main application container
```

---

# 13. Pod Lifecycle

A simplified lifecycle:

```text
Pending
   ↓
Running
   ↓
Succeeded
```

or:

```text
Pending
   ↓
Running
   ↓
Failed
```

Pods are considered relatively disposable.

You generally should **not think of a Pod as a permanent server**.

---

# 14. Why Don't We Create Pods Directly?

You can.

But in production, you normally use:

```text
Deployment
   ↓
ReplicaSet
   ↓
Pods
```

Why?

Because the Deployment manages the Pods for you.

If a Pod dies:

```text
Pod dies
   ↓
ReplicaSet notices
   ↓
New Pod created
```

---

# 15. Deployment

A Deployment describes how your application should run.

Example:

```yaml
replicas: 3
```

Meaning:

> Keep 3 instances of this application running.

Conceptually:

```text
Deployment
     |
     v
ReplicaSet
     |
 ----------------
 |      |       |
Pod    Pod     Pod
```

---

# 16. ReplicaSet

ReplicaSet's job is simple:

> Maintain the required number of Pods.

If:

```text
replicas = 3
```

then:

```text
Pod 1
Pod 2
Pod 3
```

If Pod 2 crashes:

```text
Pod 1
Pod 2 ❌
Pod 3
```

ReplicaSet creates:

```text
Pod 4
```

---

# 17. Deployment vs ReplicaSet

Interview answer:

> **Deployment manages the application rollout and ReplicaSets, while a ReplicaSet ensures the desired number of Pod replicas are running.**

---

# 18. What is a Service?

Pods are temporary.

Their IP addresses can change.

Imagine:

```text
Pod A
IP = 10.0.0.10
```

Pod crashes.

New Pod:

```text
Pod B
IP = 10.0.0.25
```

If another application directly depends on the Pod IP, everything breaks.

That's where **Service** comes in.

---

# 19. Service = Stable Network Endpoint

Think of a Service as a permanent phone number.

```text
             Service
          10.20.30.40
                |
       -------------------
       |        |        |
      Pod      Pod      Pod
```

Pods may come and go.

The Service remains stable.

---

# 20. How Does a Service Find Pods?

Using **labels and selectors**.

Example:

Pod:

```yaml
labels:
  app: payment
```

Service:

```yaml
selector:
  app: payment
```

Then:

```text
Service
   |
   | selector: app=payment
   |
   +----> Pod
   +----> Pod
   +----> Pod
```

---

# 21. Types of Kubernetes Services

The common ones:

```text
ClusterIP
NodePort
LoadBalancer
ExternalName
```

---

# 22. ClusterIP

Default service type.

Used for internal communication.

Example:

```text
Order Service
      |
      v
Payment Service
```

Payment Service could use:

```text
ClusterIP
```

It is reachable inside the cluster.

Typical production architecture:

```text
Frontend
   |
   v
Order Service
   |
   v
Payment Service
   |
   v
Database
```

---

# 23. NodePort

Exposes the Service through a port on every node.

Conceptually:

```text
Client
  |
Node IP:30080
  |
Service
  |
Pods
```

Useful for learning and certain scenarios, but usually you don't manually expose production applications this way when using GKE's managed load-balancing features.

---

# 24. LoadBalancer

Creates/exposes a cloud load balancer.

Conceptually:

```text
Internet
   |
   v
Google Cloud Load Balancer
   |
   v
Kubernetes Service
   |
   v
Pods
```

---

# 25. Ingress

This is another **very important interview topic**.

Ingress manages external HTTP/HTTPS traffic routing.

Imagine:

```text
             Internet
                |
                v
            Ingress
           /       \
          /         \
     /orders      /payments
        |             |
        v             v
 Order Service   Payment Service
```

So:

```text
/api/orders
      ↓
Order Service

/api/payments
      ↓
Payment Service
```

---

# 26. Service vs Ingress

A common interview question.

### Service

Provides network access to Pods.

### Ingress

Provides HTTP/HTTPS routing into the cluster.

Simple mental model:

```text
Ingress
   ↓
Service
   ↓
Pods
```

---

# 27. GKE Architecture

A simplified production architecture:

```text
                   Internet
                      |
                      v
             Google Cloud LB
                      |
                      v
                   Ingress
                      |
          -----------------------
          |                     |
     Order Service        Payment Service
          |                     |
      -----------           ----------
      |    |    |           |   |   |
     Pod  Pod  Pod         Pod Pod Pod
```

---

# 28. GKE Modes

This is an important GKE-specific topic.

GKE primarily provides:

### GKE Standard

You have more control over:

* nodes
* node pools
* machine types
* scaling configuration
* networking
* infrastructure configuration

You are responsible for more infrastructure decisions.

---

### GKE Autopilot

Google manages much more of the underlying infrastructure.

You mainly focus on:

```text
Application
   ↓
Pod specification
```

rather than manually managing nodes.

---

# 29. Standard vs Autopilot

Think:

```text
STANDARD

You
 |
 +---- Node configuration
 +---- Node pools
 +---- Machine types
 +---- Infrastructure choices
```

vs

```text
AUTOPILOT

You
 |
 +---- Application
 +---- Pod requirements

Google
 |
 +---- Infrastructure management
 +---- Node management
 +---- More operational automation
```

### Interview answer

> **GKE Standard provides more control over cluster infrastructure, while Autopilot abstracts more of the node and infrastructure management so teams can focus more on workloads.**

---

# 30. When Would You Choose Standard?

Potential reasons include:

* need detailed node-level control
* specialized workloads
* custom node configurations
* specific infrastructure requirements
* workloads requiring more control over scheduling/infrastructure

---

# 31. When Would You Choose Autopilot?

Useful when:

* you want less infrastructure management
* you want Kubernetes without managing nodes extensively
* workload-driven resource management fits your application
* your team wants to reduce operational overhead

---

# 32. What is a Node Pool?

A node pool is a group of nodes with the same configuration.

Example:

```text
GKE Cluster
 |
 +── General Node Pool
 |      ├── Node
 |      ├── Node
 |      └── Node
 |
 +── Memory Node Pool
        ├── Node
        └── Node
```

---

# 33. Why Use Multiple Node Pools?

Suppose:

```text
Normal APIs
Batch jobs
Machine learning
```

have different requirements.

You could have:

```text
Node Pool A
General-purpose VMs

Node Pool B
High-memory VMs

Node Pool C
GPU VMs
```

Then schedule workloads appropriately.

---

# 34. What is a Namespace?

A namespace logically separates resources inside a Kubernetes cluster.

Example:

```text
GKE Cluster
 |
 +── development
 |
 +── staging
 |
 +── production
```

Or:

```text
production
 |
 +── payments
 +── orders
 +── users
```

Namespaces are useful for:

* organization
* access control
* resource quotas
* isolation boundaries

Important:

> A namespace is logical isolation, not the same thing as a completely separate cluster.

---

# 35. ConfigMap

ConfigMap stores non-sensitive configuration.

Example:

```text
ENVIRONMENT=production
LOG_LEVEL=INFO
PAYMENT_URL=https://payment-service
```

Your application can consume it as environment variables or mounted files.

---

# 36. Secret

Secrets are intended for sensitive values.

Examples:

```text
DB_PASSWORD
API_KEY
TOKEN
CERTIFICATE
```

But remember:

> Kubernetes Secret does not automatically mean "magically encrypted everywhere."

You should understand how secrets are protected, access-controlled, and integrated with cloud secret-management systems.

For GCP production systems, **Secret Manager** is also an important service to know.

---

# 37. ConfigMap vs Secret

| ConfigMap                   | Secret                |
| --------------------------- | --------------------- |
| Non-sensitive configuration | Sensitive information |
| URLs                        | Passwords             |
| Feature flags               | Tokens                |
| Environment settings        | API credentials       |

---

# 38. Resource Requests and Limits

This is **very important in production interviews**.

Suppose a Pod needs:

```text
CPU: 500m
Memory: 512Mi
```

You can specify:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"

  limits:
    cpu: "1"
    memory: "1Gi"
```

---

# 39. Request vs Limit

### Request

The amount of resources Kubernetes should consider when scheduling the Pod.

Think:

> "I need at least this much."

### Limit

The maximum resource amount allowed for the container.

Think:

> "Don't let me go beyond this."

---

# 40. Why Are Requests Important?

Imagine a node has:

```text
4 CPU
```

Pod A requests:

```text
2 CPU
```

Pod B requests:

```text
2 CPU
```

Kubernetes understands:

```text
2 + 2 = 4 CPU
```

and schedules accordingly.

Without sensible requests, resource scheduling becomes unpredictable.

---

# 41. What Happens When a Pod Uses Too Much Memory?

Memory is different from CPU.

If a container exceeds its memory limit, it can be terminated by the kernel.

You may see:

```text
OOMKilled
```

Meaning:

> Out Of Memory.

This is a common production troubleshooting scenario.

---

# 42. CPU Throttling

CPU limits can cause throttling when the container tries to consume more CPU than allowed.

So blindly setting very low CPU limits can cause performance problems.

---

# 43. Horizontal Pod Autoscaler — HPA

HPA automatically changes the number of Pods based on metrics.

Example:

```text
Normal traffic

3 Pods
```

Traffic increases:

```text
CPU = 80%
```

HPA:

```text
3 → 5 Pods
```

Traffic increases further:

```text
5 → 10 Pods
```

---

# 44. HPA Mental Model

```text
Metrics
   |
   v
HPA
   |
   v
Deployment
   |
   v
More/Fewer Pods
```

---

# 45. Vertical Pod Autoscaler — VPA

VPA changes the resource requirements of Pods.

For example:

```text
Pod currently:
500Mi memory

VPA recommendation:
1Gi memory
```

Conceptually:

```text
HPA → changes number of Pods

VPA → changes resources per Pod
```

They solve different problems.

---

# 46. Cluster Autoscaler

HPA scales:

```text
Pods
```

Cluster Autoscaler scales:

```text
Nodes
```

Example:

```text
HPA:
3 Pods → 20 Pods
```

But there isn't enough node capacity.

Cluster Autoscaler can add nodes.

```text
Nodes:
3 → 6
```

---

# 47. The Scaling Picture

This is worth memorizing:

```text
Traffic increases
       |
       v
      HPA
       |
       v
More Pods
       |
       v
Not enough node capacity?
       |
       v
Cluster Autoscaler
       |
       v
More Nodes
```

---

# 48. Liveness Probe

Liveness asks:

> **"Is this application alive?"**

If it repeatedly fails, Kubernetes can restart the container.

Example:

```text
GET /actuator/health/liveness
```

---

# 49. Readiness Probe

Readiness asks:

> **"Can this application receive traffic?"**

This is different from liveness.

Example:

Application starts:

```text
Application running
Database connection initializing
Cache warming
```

The process may be alive.

But it isn't ready to serve traffic.

Therefore:

```text
Liveness = OK
Readiness = NOT READY
```

Traffic should not be sent to it.

---

# 50. Startup Probe

Useful for applications that take a long time to start.

Example:

```text
Spring Boot application
     ↓
Starts in 60 seconds
```

Without proper configuration, Kubernetes might incorrectly think:

> "This application is dead."

Startup probe gives it time to initialize.

---

# 51. The Three Probes

Remember:

```text
Startup
   ↓
"Did you successfully start?"

Liveness
   ↓
"Are you still alive?"

Readiness
   ↓
"Can you receive traffic?"
```

---

# 52. Rolling Deployment

Suppose:

```text
Version 1
5 Pods
```

You deploy:

```text
Version 2
```

Instead of killing all 5:

```text
V1 V1 V1 V1 V1
```

and then starting:

```text
V2 V2 V2 V2 V2
```

Kubernetes can gradually replace them.

```text
V1 V1 V1 V1 V1
      ↓
V2 V1 V1 V1 V1
      ↓
V2 V2 V1 V1 V1
      ↓
V2 V2 V2 V1 V1
      ↓
V2 V2 V2 V2 V2
```

This is a rolling update.

---

# 53. Why Rolling Deployment?

Because you want:

```text
Minimal downtime
+
Controlled rollout
+
Easy rollback
```

---

# 54. Canary Deployment

Canary means:

> Send a small amount of traffic to the new version first.

Example:

```text
Version 1 → 95%
Version 2 → 5%
```

Monitor:

* errors
* latency
* CPU
* business metrics

Then gradually increase Version 2.

```text
5%
 ↓
10%
 ↓
25%
 ↓
50%
 ↓
100%
```

---

# 55. Blue-Green Deployment

Two environments:

```text
Blue = Current
Green = New
```

Initially:

```text
Traffic
  |
  v
Blue
```

Deploy and test Green.

Then switch:

```text
Traffic
  |
  v
Green
```

Rollback can be:

```text
Traffic
  |
  v
Blue
```

---

# 56. Rolling vs Canary vs Blue-Green

| Strategy   | Idea                                                  |
| ---------- | ----------------------------------------------------- |
| Rolling    | Gradually replace old Pods                            |
| Canary     | Send small traffic to new version                     |
| Blue-Green | Maintain two versions/environments and switch traffic |

---

# 57. What is a Container Image?

A container image contains what your application needs to run.

For example:

```text
Java Runtime
+
Spring Boot JAR
+
Dependencies
+
Configuration defaults
```

You might build:

```text
payment-service:1.0
```

and store it in:

**Artifact Registry**.

---

# 58. Typical GKE Deployment Flow

A very common production pipeline:

```text
Developer
   |
   v
Git
   |
   v
CI/CD
   |
   v
Build Docker Image
   |
   v
Artifact Registry
   |
   v
GKE
   |
   v
Deployment
   |
   v
Pods
```

---

# 59. Artifact Registry

Artifact Registry stores artifacts such as:

* container images
* language packages
* other build artifacts

For GKE:

```text
Docker Image
      ↓
Artifact Registry
      ↓
GKE pulls image
```

---

# 60. kubectl

`kubectl` is the command-line tool used to interact with Kubernetes.

Common commands:

```bash
kubectl get pods
```

```bash
kubectl get nodes
```

```bash
kubectl get services
```

```bash
kubectl get deployments
```

```bash
kubectl describe pod <pod-name>
```

```bash
kubectl logs <pod-name>
```

```bash
kubectl exec -it <pod-name> -- /bin/sh
```

---

# 61. The Commands You Should Know for Interviews

### List Pods

```bash
kubectl get pods
```

### More information

```bash
kubectl get pods -o wide
```

### Describe

```bash
kubectl describe pod <name>
```

### Logs

```bash
kubectl logs <pod>
```

### Previous container logs

Useful after crashes:

```bash
kubectl logs <pod> --previous
```

### Deployments

```bash
kubectl get deployments
```

### Services

```bash
kubectl get svc
```

### Apply YAML

```bash
kubectl apply -f deployment.yaml
```

### Delete

```bash
kubectl delete -f deployment.yaml
```

---

# 62. How Does GKE Connect to Google Cloud?

GKE integrates with many Google Cloud services.

Common architecture:

```text
GKE
 |
 +── Cloud Load Balancing
 |
 +── Artifact Registry
 |
 +── Cloud Logging
 |
 +── Cloud Monitoring
 |
 +── Secret Manager
 |
 +── IAM
 |
 +── VPC
 |
 +── Cloud SQL
 |
 +── Memorystore
 |
 +── Pub/Sub
```

Knowing these integrations is particularly useful for a Google Cloud interview.

---

# 63. GKE Networking

This can become deep, but understand the basics first.

GKE workloads need networking between:

```text
Pod → Pod
Pod → Service
Pod → Internet
Pod → Google Cloud services
Pod → external systems
```

---

# 64. VPC

GKE runs within Google Cloud networking.

Think:

```text
Google Cloud
   |
   v
VPC
   |
   v
GKE Cluster
   |
   +── Nodes
   +── Pods
```

---

# 65. Pod IP

Each Pod generally gets an IP address.

Example:

```text
Pod A → 10.10.0.5
Pod B → 10.10.0.6
```

But Pods are ephemeral.

Therefore applications shouldn't normally depend on individual Pod IPs.

Use:

```text
Service
```

for stable application-to-application communication.

---

# 66. Internal vs External Traffic

### Internal

```text
Service A
   |
   v
Service B
```

Typically use internal Kubernetes networking / Services.

### External

```text
Internet
   |
   v
Google Load Balancer
   |
   v
GKE
```

---

# 67. Network Policies

NetworkPolicy allows you to control which Pods can communicate.

Without policy:

```text
Pod A → Pod B
Pod C → Pod B
Pod D → Pod B
```

With policy:

```text
Pod A ─────> Pod B
Pod C ─X───> Pod B
Pod D ─X───> Pod B
```

This helps implement network-level restrictions.

---

# 68. Kubernetes RBAC

RBAC means:

> **Role-Based Access Control**

It controls:

```text
Who
can do
what
on which resource
```

Example:

```text
Developer
   |
   +── Can read Pods
   +── Can read logs
   +── Cannot delete production cluster
```

---

# 69. IAM vs Kubernetes RBAC

This is a common GCP interview question.

### IAM

Controls access to Google Cloud resources.

Examples:

```text
GKE
GCS
BigQuery
Pub/Sub
Compute Engine
```

### Kubernetes RBAC

Controls access to Kubernetes resources.

Examples:

```text
Pods
Deployments
Services
ConfigMaps
Secrets
```

Think:

```text
Google Cloud permissions → IAM

Kubernetes API permissions → RBAC
```

---

# 70. Workload Identity Federation for GKE

This is a **very important production concept**.

Suppose your Spring Boot application in GKE needs to access:

```text
Cloud Storage
```

Bad approach:

```text
Put service account key JSON
inside container
```

Don't do that.

Better:

```text
Kubernetes Service Account
          |
          v
Workload Identity
          |
          v
Google Cloud IAM
          |
          v
Cloud Storage
```

This allows workloads to authenticate without embedding long-lived service-account keys.

---

# 71. Example

Your application:

```text
payment-service
```

needs:

```text
gs://payment-data
```

You give its workload identity only the required permissions.

This follows:

> **Least privilege**

---

# 72. GKE Security Layers

Think about security at multiple levels:

```text
Internet
   ↓
Cloud Load Balancer
   ↓
Ingress
   ↓
Network Policy
   ↓
Service
   ↓
Pod
   ↓
Application
   ↓
IAM / Workload Identity
   ↓
Google Cloud Services
```

Security is not one setting.

---

# 73. GKE + Cloud SQL

A common production architecture:

```text
Internet
   |
   v
Load Balancer
   |
   v
GKE
   |
   v
Spring Boot
   |
   v
Cloud SQL
```

The application should generally connect using secure/private networking where appropriate rather than exposing the database publicly without need.

---

# 74. GKE + Redis

For caching:

```text
GKE
 |
 v
Spring Boot
 |
 +----> Redis
 |
 +----> Database
```

On GCP, **Memorystore** is commonly used for managed Redis-compatible caching.

---

# 75. GKE + Pub/Sub

For asynchronous architecture:

```text
Order Service
     |
     v
Pub/Sub
     |
     v
Notification Service
```

This is particularly useful when the producer shouldn't wait for the consumer.

For example:

```text
User creates order
       |
       v
Order Service
       |
       v
Pub/Sub
       |
       +------> Email Service
       |
       +------> Notification Service
       |
       +------> Analytics
```

---

# 76. Why Kubernetes Instead of Just Compute Engine?

This is a classic interview question.

You can absolutely run Spring Boot on Compute Engine.

But with Kubernetes you get standardized orchestration capabilities such as:

* desired-state management
* automated scheduling
* service discovery
* rolling deployments
* self-healing
* autoscaling
* declarative configuration

The trade-off is:

> Kubernetes adds significant complexity.

---

# 77. GKE vs Compute Engine

| Compute Engine                   | GKE                                          |
| -------------------------------- | -------------------------------------------- |
| VM-focused                       | Container/Kubernetes-focused                 |
| You manage application processes | Kubernetes manages workloads                 |
| More manual orchestration        | Automated orchestration                      |
| Simpler for simple workloads     | Better suited to complex container platforms |
| VM-level control                 | Kubernetes abstraction                       |

---

# 78. When NOT to Use GKE

Very important.

Don't use Kubernetes simply because:

> "It's modern."

GKE may be unnecessary if:

* application is very simple
* only one small service exists
* workload doesn't need Kubernetes features
* team lacks Kubernetes operational expertise
* operational complexity outweighs benefits

Other GCP services may be more appropriate depending on the workload.

For example:

```text
Simple HTTP container
        ↓
Cloud Run
```

may be simpler than:

```text
GKE Cluster
 + Nodes
 + Pods
 + Services
 + Ingress
 + Autoscaling
 + Networking
 + RBAC
```

---

# 79. GKE vs Cloud Run

Very common GCP interview question.

### Cloud Run

Think:

> "I have a container. Just run it."

### GKE

Think:

> "I need Kubernetes and control over a container platform."

Conceptually:

```text
Simple container workload
        ↓
     Cloud Run
```

versus:

```text
Complex container platform
        ↓
       GKE
```

---

# 80. Self-Healing

Suppose:

```text
3 Pods
```

One crashes:

```text
Pod 1
Pod 2 ❌
Pod 3
```

Kubernetes detects:

```text
Desired = 3
Actual = 2
```

and creates another.

```text
Pod 1
Pod 3
Pod 4
```

This is self-healing.

---

# 81. What If a Node Dies?

Suppose:

```text
Node 1
 ├── Pod A
 ├── Pod B

Node 2
 ├── Pod C
 └── Pod D
```

Node 1 dies.

Kubernetes can reschedule managed workloads onto available/new nodes, subject to resource and scheduling constraints.

```text
Node 2
 ├── Pod C
 ├── Pod D
 ├── Pod A
 └── Pod B
```

The exact behavior depends on the workload, capacity, constraints, and cluster configuration.

---

# 82. Production Scenario

Imagine you're building an e-commerce backend.

You have:

```text
Order Service
Payment Service
Inventory Service
Notification Service
```

Architecture:

```text
                   Internet
                       |
                       v
                Load Balancer
                       |
                       v
                    Ingress
                       |
       --------------------------------
       |              |               |
     Order          Payment       Inventory
    Service         Service         Service
       |              |               |
      Pods           Pods            Pods
       |              |               |
       ----------- Pub/Sub -----------
                       |
                Notification
                   Service
                       |
                      Pods
```

Database:

```text
Services
   |
   v
Cloud SQL
```

Cache:

```text
Services
   |
   v
Memorystore
```

Images:

```text
CI/CD
  |
  v
Artifact Registry
  |
  v
GKE
```

Monitoring:

```text
GKE
 |
 +── Cloud Logging
 |
 +── Cloud Monitoring
```

---

# 83. Production Deployment Flow

A typical flow:

```text
Developer
   |
   v
Git
   |
   v
CI/CD
   |
   +---- Build
   |
   +---- Test
   |
   +---- Security checks
   |
   v
Container Image
   |
   v
Artifact Registry
   |
   v
GKE Deployment
   |
   v
Rolling Update
   |
   v
Pods
```

---

# 84. What Happens When a New Deployment Happens?

Suppose:

```text
v1 → 5 Pods
```

You deploy v2.

Deployment creates/manages a new ReplicaSet.

```text
Deployment
    |
    +---- ReplicaSet v1
    |       ├── Pod
    |       ├── Pod
    |       └── Pod
    |
    +---- ReplicaSet v2
            ├── Pod
            └── Pod
```

As rollout progresses:

```text
v1 Pods ↓
v2 Pods ↑
```

Eventually:

```text
v1 = 0
v2 = 5
```

---

# 85. Rollback

Suppose v2 has a serious bug.

You can roll back the Deployment.

Common command:

```bash
kubectl rollout undo deployment/payment-service
```

Check rollout:

```bash
kubectl rollout status deployment/payment-service
```

History:

```bash
kubectl rollout history deployment/payment-service
```

---

# 86. Important Deployment Strategies

For interviews, know:

```text
Rolling
Canary
Blue-Green
Recreate
```

And be able to explain:

* how traffic moves
* downtime
* rollback
* resource requirements
* risk

---

# 87. Scheduling

Kubernetes needs to decide:

> Which node should run this Pod?

It considers:

* CPU
* memory
* requests
* node labels
* affinity
* taints
* tolerations
* topology constraints

---

# 88. Taints and Tolerations

Imagine a node is reserved for:

```text
GPU workloads
```

You don't want normal workloads accidentally running there.

You can put a **taint** on the node.

Only Pods with the matching **toleration** can be scheduled there.

Mental model:

```text
Node:
"Keep out!"

Pod:
"I have permission."

→ Allowed
```

---

# 89. Node Affinity

Affinity says:

> "I prefer or require this Pod to run on certain nodes."

Example:

```text
Pod
 ↓
Node with label:
workload=memory-intensive
```

---

# 90. Anti-Affinity

Anti-affinity says:

> "Don't place these Pods together."

This can help distribute replicas across failure domains.

For example:

```text
Node 1 → Order Pod A
Node 2 → Order Pod B
Node 3 → Order Pod C
```

rather than:

```text
Node 1 → Order Pod A
Node 1 → Order Pod B
Node 1 → Order Pod C
```

---

# 91. Availability Zones

For production workloads, you should think about failure domains.

Instead of:

```text
Zone A
 |
All Pods
```

you can distribute workloads across zones.

Conceptually:

```text
Region
 |
 +── Zone A → Pods
 |
 +── Zone B → Pods
 |
 +── Zone C → Pods
```

If one zone experiences an outage, your application can potentially continue serving from other zones.

---

# 92. Regional vs Zonal GKE

A **zonal cluster** is associated with a single zone.

A **regional cluster** spans multiple zones within a region and provides higher availability for the control plane and workloads when appropriately configured.

For production, understand the availability trade-offs and workload distribution rather than blindly choosing one.

---

# 93. High Availability

High availability means:

> The application continues operating despite certain failures.

For GKE:

```text
Multiple Pods
+
Multiple Nodes
+
Multiple Zones
+
Load Balancing
```

can provide resilience.

But simply running Kubernetes does **not automatically make your application highly available**.

---

# 94. The "3 Replicas" Trap

Suppose you say:

> "I have 3 replicas, so I'm highly available."

Not necessarily.

If all three Pods are on the same node:

```text
Node 1
 ├── Pod A
 ├── Pod B
 └── Pod C
```

Node 1 dies:

```text
Everything gone
```

Better:

```text
Node 1 → Pod A
Node 2 → Pod B
Node 3 → Pod C
```

And ideally across appropriate zones.

---

# 95. PodDisruptionBudget

A PodDisruptionBudget helps maintain availability during **voluntary disruptions**.

For example:

```text
3 replicas
```

You might specify:

```text
minAvailable = 2
```

Meaning:

> Don't voluntarily disrupt too many replicas simultaneously.

It doesn't protect against every type of failure.

---

# 96. Graceful Shutdown

This is important for Spring Boot applications.

Imagine:

```text
Pod receives SIGTERM
```

You don't want Kubernetes to immediately kill the application while requests are running.

You want:

```text
SIGTERM
  ↓
Stop accepting new work
  ↓
Finish existing requests
  ↓
Shutdown
```

This is especially important for APIs and message consumers.

---

# 97. Common GKE Failure Scenario #1

### Problem

```text
Pod = CrashLoopBackOff
```

What do you do?

Don't randomly restart it.

Investigate.

```bash
kubectl get pods
```

Then:

```bash
kubectl describe pod <pod>
```

Then:

```bash
kubectl logs <pod>
```

If it restarted:

```bash
kubectl logs <pod> --previous
```

Look for:

* application exception
* configuration issue
* missing Secret
* bad environment variable
* failed health probe
* OOMKilled
* image problem

---

# 98. CrashLoopBackOff

This doesn't necessarily mean:

> "Kubernetes is broken."

It usually means the container is repeatedly starting and crashing, and Kubernetes is backing off before restarting it again.

Typical flow:

```text
Start
 ↓
Crash
 ↓
Restart
 ↓
Crash
 ↓
Restart
 ↓
Increasing backoff
```

---

# 99. ImagePullBackOff

This means Kubernetes is having trouble pulling the container image.

Potential reasons:

```text
Wrong image name
Wrong tag
Image doesn't exist
Authentication problem
Registry access issue
Network issue
```

Investigate:

```bash
kubectl describe pod <pod>
```

---

# 100. Pod Pending

If a Pod stays:

```text
Pending
```

possible causes include:

* insufficient CPU
* insufficient memory
* node selector mismatch
* affinity constraints
* taint/toleration issue
* resource quota
* scheduling constraints

First investigation:

```bash
kubectl describe pod <pod>
```

---

# 101. Service Not Working

Suppose:

```text
Pod = Running
Service = Running
```

but requests fail.

Check:

```text
Service
 ↓
Selector
 ↓
Endpoints
 ↓
Pod labels
```

A common mistake:

```yaml
Service selector:
app: payment
```

but Pods have:

```yaml
app: payments
```

No matching Pods.

---

# 102. Debugging Flow

Memorize this:

```text
User
 ↓
Load Balancer
 ↓
Ingress
 ↓
Service
 ↓
Endpoints
 ↓
Pod
 ↓
Container
 ↓
Application
 ↓
Database / External Service
```

When something breaks, walk down this chain.

---

# 103. Monitoring

You need to know:

```text
Is it running?
Is it healthy?
Is it fast?
Is it overloaded?
Is it failing?
```

Google Cloud provides:

**Cloud Monitoring**

and

**Cloud Logging**

for observability.

---

# 104. Logs

Application logs might contain:

```text
ERROR Payment failed
```

You can investigate through Cloud Logging or Kubernetes tooling.

For a Java application:

```text
Spring Boot
   ↓
stdout/stderr
   ↓
Container
   ↓
GKE
   ↓
Cloud Logging
```

---

# 105. Metrics

Important metrics include:

### Infrastructure

* CPU
* memory
* disk
* network

### Kubernetes

* Pod count
* restart count
* pending Pods
* node utilization

### Application

* request latency
* error rate
* throughput
* active requests

---

# 106. The Three Golden Signals

A useful interview concept:

```text
Latency
Traffic
Errors
```

Often extended in practice with saturation:

```text
Latency
Traffic
Errors
Saturation
```

---

# 107. GKE Cost Optimization

Production interviewers may ask:

> "How would you reduce GKE cost?"

Possible areas:

### Right-size resources

Don't request:

```text
4 CPU
8 GB
```

if application only needs:

```text
500m CPU
512Mi
```

### Autoscaling

Use:

```text
HPA
Cluster Autoscaler
```

where appropriate.

### Node pools

Choose suitable machine types.

### Scheduling

Avoid keeping expensive nodes idle.

### Workload placement

Separate specialized workloads when justified.

---

# 108. Overprovisioning Problem

Suppose:

```text
10 Pods

Each requests:
2 CPU
4 GB
```

But actual usage is:

```text
100m CPU
500MB
```

You're potentially wasting resources.

This is why monitoring and right-sizing matter.

---

# 109. Security Best Practices

For production:

```text
Least privilege
+
Workload Identity
+
Network Policies
+
RBAC
+
Secret management
+
Private networking where appropriate
+
Image scanning
+
Regular patching
```

Avoid:

```text
Service account JSON
inside container
```

when workload identity mechanisms can be used.

---

# 110. GKE Upgrade

Kubernetes and GKE versions need upgrades.

Why?

* security fixes
* bug fixes
* supported versions
* new features

A production organization should have a controlled upgrade strategy.

Think:

```text
Development
    ↓
Staging
    ↓
Production
```

rather than upgrading production blindly.

---

# 111. What is a GKE Release Channel?

GKE provides release channels to manage the pace of Kubernetes version updates.

Conceptually:

```text
Faster updates
     ↑
     |
Regular
     |
Stable
     ↓
More conservative
```

The exact available channels and behavior can evolve, so for production decisions check current GKE documentation.

For interviews, understand the concept:

> Release channels help organizations balance access to newer Kubernetes versions against operational stability.

---

# 112. GKE Autopilot Security Model

Autopilot applies more opinionated configuration and manages more infrastructure for you.

This reduces the amount of node administration you perform.

The trade-off is:

```text
Less infrastructure control
       vs
Less infrastructure management
```

---

# 113. GKE Standard Security Model

Standard provides greater control over:

* node pools
* machine types
* node configuration
* scheduling
* infrastructure

But that also means:

> More responsibility.

---

# 114. Declarative Configuration

Kubernetes is largely declarative.

Instead of saying:

```text
Start Pod
Then start another Pod
Then restart it if it fails
```

you say:

```yaml
replicas: 3
```

You're describing:

> "This is what I want."

Kubernetes figures out how to get there.

---

# 115. YAML

Typical Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: payment-service

spec:
  replicas: 3

  selector:
    matchLabels:
      app: payment

  template:
    metadata:
      labels:
        app: payment

    spec:
      containers:
        - name: payment
          image: payment-service:1.0
          ports:
            - containerPort: 8080
```

Understand the hierarchy:

```text
Deployment
    |
    └── Pod Template
            |
            └── Container
```

---

# 116. Service YAML

Conceptually:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: payment-service

spec:
  selector:
    app: payment

  ports:
    - port: 80
      targetPort: 8080
```

Meaning:

```text
Service port 80
      ↓
Pod port 8080
```

---

# 117. Port vs TargetPort

Another common interview question.

```text
Service
 port: 80
    |
    v
targetPort: 8080
    |
    v
Container
```

### `port`

Port exposed by the Service.

### `targetPort`

Port where the application is listening inside the Pod.

---

# 118. `containerPort`

`containerPort` documents/exposes the port expected by the container specification.

It doesn't by itself create external network access.

This distinction is frequently misunderstood.

---

# 119. Labels

Labels are key-value metadata.

Example:

```yaml
app: payment
environment: production
team: payments
```

Used by Kubernetes to identify resources.

---

# 120. Annotations

Annotations store additional metadata/configuration.

For example, certain GKE integrations can use annotations to influence cloud load-balancing or other behavior.

Simple distinction:

```text
Labels → identify/select objects

Annotations → attach additional metadata/configuration
```

---

# 121. StatefulSet

Deployment is generally used for stateless applications.

What if you have:

```text
Database
Kafka
ZooKeeper-like stateful systems
```

where identity and persistent storage matter?

You may need:

**StatefulSet**

StatefulSet provides stable identity and storage semantics for stateful workloads.

---

# 122. Deployment vs StatefulSet

| Deployment                  | StatefulSet                  |
| --------------------------- | ---------------------------- |
| Usually stateless apps      | Stateful workloads           |
| Pods interchangeable        | Pods have stable identity    |
| Random-looking Pod identity | Stable Pod identity          |
| Typical REST API            | Databases / stateful systems |

---

# 123. DaemonSet

DaemonSet means:

> Run a Pod on every eligible node.

Example:

```text
Node 1 → Logging Agent
Node 2 → Logging Agent
Node 3 → Logging Agent
```

Common use cases:

* logging agents
* monitoring agents
* node-level agents

---

# 124. Job

Job is for work that should complete.

Example:

```text
Process 1 million records
```

Run:

```text
Job
 ↓
Pod
 ↓
Complete
```

---

# 125. CronJob

CronJob runs Jobs on a schedule.

Example:

```text
Every night at 2 AM
        ↓
     CronJob
        ↓
       Job
        ↓
      Backup
```

---

# 126. Kubernetes Workload Types

Memorize this table:

| Workload    | Use                       |
| ----------- | ------------------------- |
| Deployment  | Stateless application     |
| StatefulSet | Stateful application      |
| DaemonSet   | One Pod per eligible node |
| Job         | One-time task             |
| CronJob     | Scheduled task            |

---

# 127. PersistentVolume

Containers are ephemeral.

If a container writes:

```text
/data/file.txt
```

and then disappears, that data may disappear too unless stored on persistent storage.

Kubernetes provides:

```text
PersistentVolume
PersistentVolumeClaim
```

---

# 128. PVC

A PersistentVolumeClaim means:

> "My application needs persistent storage."

Conceptually:

```text
Application
    |
    v
PVC
    |
    v
Persistent Volume
    |
    v
Storage
```

---

# 129. EmptyDir

`emptyDir` is temporary storage associated with a Pod.

Useful for:

* temporary files
* intermediate processing
* sharing temporary data between containers in the same Pod

When the Pod is removed, the `emptyDir` contents are lost.

---

# 130. GKE and Storage

GKE can integrate with Google Cloud storage technologies through Kubernetes storage mechanisms.

Common concepts to know:

```text
PersistentVolume
PersistentVolumeClaim
StorageClass
CSI drivers
```

---

# 131. Sidecar Pattern

A Pod can contain multiple containers.

Example:

```text
Pod
 |
 +── Spring Boot
 |
 +── Logging Agent
```

The second container is a **sidecar**.

It supports the main application.

---

# 132. Init Container

An init container runs **before the main application container**.

Example:

```text
Init Container
      ↓
Prepare configuration
      ↓
Main Container
      ↓
Spring Boot
```

Useful for initialization tasks.

---

# 133. Service Discovery

Suppose:

```text
order-service
```

needs:

```text
payment-service
```

Instead of finding a Pod IP manually:

```text
10.0.0.17
```

use Kubernetes DNS/service discovery.

Conceptually:

```text
payment-service
```

resolves to the Service.

So:

```text
Order Service
      |
      v
http://payment-service
      |
      v
Service
      |
      v
Payment Pods
```

---

# 134. Why DNS Is Important

Pods are ephemeral.

Their IPs can change.

Service DNS remains stable.

Therefore:

> Applications should generally communicate using Service names rather than Pod IPs.

---

# 135. Ingress vs Gateway API

Modern Kubernetes networking also includes the **Gateway API**, which provides more expressive and extensible traffic-management concepts than traditional Ingress.

For interviews:

```text
Ingress → older/common HTTP routing API

Gateway API → newer, more expressive Kubernetes networking model
```

GKE supports cloud-integrated networking capabilities around these concepts.

---

# 136. GKE Load Balancing

A common external request path:

```text
User
 |
 v
Google Cloud Load Balancer
 |
 v
GKE
 |
 v
Ingress / Gateway
 |
 v
Service
 |
 v
Pod
```

Understanding this flow is more valuable than memorizing isolated terms.

---

# 137. What Happens When a Pod Becomes Unhealthy?

Suppose:

```text
Pod A
Readiness = FAIL
```

The Pod may remain running, but it is removed from the set of endpoints receiving traffic.

So:

```text
Service
 |
 +---- Pod B
 +---- Pod C
```

instead of:

```text
Service
 |
 +---- Pod A ❌
 +---- Pod B
 +---- Pod C
```

This is why readiness probes matter.

---

# 138. Liveness Failure vs Readiness Failure

### Readiness fails

```text
Stop sending traffic
```

### Liveness fails

```text
Restart container
```

That distinction is extremely important.

---

# 139. A Real Spring Boot Example

Imagine:

```text
Spring Boot
Port 8080
```

You might expose:

```text
/actuator/health/liveness
/actuator/health/readiness
```

Kubernetes:

```text
Readiness → /actuator/health/readiness
Liveness  → /actuator/health/liveness
```

Then:

```text
Application healthy
       ↓
Receive traffic
```

---

# 140. Production Interview Scenario

### Interviewer:

> Your application is running on GKE. CPU suddenly goes to 90%. What do you do?

Don't immediately say:

> "Increase CPU."

Think systematically.

```text
CPU increased
      |
      +── Traffic increased?
      |
      +── Bad deployment?
      |
      +── Infinite loop?
      |
      +── DB latency?
      |
      +── Dependency issue?
      |
      +── Garbage collection?
      |
      +── Thread contention?
```

Check:

```text
Metrics
Logs
Recent deployments
Traffic
Pod behavior
Application profiling
```

Then decide whether scaling is appropriate.

---

# 141. Scenario: Pod Keeps Restarting

Your approach:

```text
kubectl get pods
        ↓
kubectl describe pod
        ↓
kubectl logs
        ↓
kubectl logs --previous
        ↓
Check events
        ↓
Check probes
        ↓
Check resources
        ↓
Check configuration
```

Potential causes:

```text
OOMKilled
Bad configuration
Application crash
Failed liveness probe
Missing Secret
Dependency failure
```

---

# 142. Scenario: Deployment Is Stuck

Possible reasons:

```text
Insufficient resources
Image can't be pulled
Readiness probe failing
Scheduling constraints
Quota
Configuration error
```

Check:

```bash
kubectl rollout status deployment/my-app
```

Then:

```bash
kubectl describe deployment my-app
```

and inspect Pods/events.

---

# 143. Scenario: Users Get 503

Think from outside → inside:

```text
Internet
 ↓
Load Balancer
 ↓
Ingress
 ↓
Service
 ↓
Endpoints
 ↓
Pods
```

Potential issue:

```text
No ready Pods
```

because:

```text
Readiness probe failing
```

or:

```text
Service selector incorrect
```

or:

```text
Ingress/backend configuration
```

---

# 144. Scenario: Application Cannot Access GCS

Don't immediately modify the code.

Check:

```text
Pod
 ↓
Kubernetes Service Account
 ↓
Workload Identity
 ↓
Google IAM
 ↓
GCS permissions
```

Likely problems:

* wrong identity
* missing IAM role
* wrong bucket
* network/access issue
* application configuration

---

# 145. Scenario: Pods Cannot Communicate

Check:

```text
Service
 ↓
Selector
 ↓
Endpoints
 ↓
NetworkPolicy
 ↓
DNS
 ↓
Port
 ↓
Application
```

Also verify:

```text
Pod → Pod
Pod → Service
```

---

# 146. Scenario: One Node Is Overloaded

Check:

```text
Node utilization
Pod resource requests
Scheduling rules
Affinity
Taints
Pod distribution
```

Potential fixes:

```text
Right-size resources
Add nodes
Adjust scheduling
Use autoscaling
Distribute replicas
```

---

# 147. Scenario: Database Is Slow

Don't assume GKE is the problem.

Trace:

```text
User
 ↓
Load Balancer
 ↓
Pod
 ↓
Spring Boot
 ↓
DB connection pool
 ↓
Cloud SQL
 ↓
Query
```

Could be:

```text
DB query
Connection pool
Network
Locking
Indexes
DB CPU
Application behavior
```

This is where your backend experience becomes extremely valuable.

---

# 148. GKE Observability Mental Model

Think in four layers:

```text
Infrastructure
      ↓
Kubernetes
      ↓
Application
      ↓
Business
```

Example:

```text
Node CPU
 ↓
Pod CPU
 ↓
API latency
 ↓
Order failure rate
```

Good production engineers connect all four.

---

# 149. GKE Interview Question: Why Kubernetes?

Good answer:

> Kubernetes provides a declarative platform for orchestrating containers. It handles scheduling, service discovery, self-healing, rolling deployments, and autoscaling, which becomes difficult to manage manually as the number of services and instances grows.

---

# 150. Why GKE Instead of Self-Managed Kubernetes?

Good answer:

> GKE is Google's managed Kubernetes service. Google manages significant portions of the Kubernetes control-plane and integrates Kubernetes with Google Cloud services such as networking, IAM, logging, monitoring, and load balancing. This reduces operational overhead compared with building and maintaining Kubernetes yourself.

---

# 151. Why GKE Instead of Docker?

Docker primarily provides:

```text
Containerization
```

Kubernetes provides:

```text
Orchestration
```

Think:

```text
Docker
"What is my application package?"

Kubernetes
"Where should it run?
How many should run?
What happens if one dies?"
```

---

# 152. Docker vs Kubernetes vs GKE

```text
Docker
   ↓
Build/run containers

Kubernetes
   ↓
Orchestrate containers

GKE
   ↓
Google-managed Kubernetes platform
```

---

# 153. Important GKE Terms Cheat Sheet

```text
Cluster
→ Collection of Kubernetes resources/nodes

Control Plane
→ Kubernetes brain

Node
→ Machine that runs workloads

Pod
→ Smallest deployable Kubernetes unit

Container
→ Application runtime package

Deployment
→ Manages stateless application rollout

ReplicaSet
→ Maintains desired Pod count

Service
→ Stable network endpoint

Ingress
→ HTTP/HTTPS routing

Namespace
→ Logical grouping

ConfigMap
→ Non-sensitive configuration

Secret
→ Sensitive configuration

HPA
→ Scales Pods

VPA
→ Adjusts Pod resources

Cluster Autoscaler
→ Scales nodes

StatefulSet
→ Stateful workloads

DaemonSet
→ Pod on each eligible node

Job
→ One-time workload

CronJob
→ Scheduled workload

PVC
→ Request for persistent storage

RBAC
→ Kubernetes authorization

IAM
→ Google Cloud authorization

Workload Identity
→ Secure workload-to-GCP authentication
```

---

# 154. The Most Important Relationships

If you remember only one diagram, remember this:

```text
                  GKE CLUSTER
                       |
        -------------------------------
        |                             |
   Control Plane                   Nodes
                                      |
                         -------------------------
                         |           |           |
                        Pod         Pod         Pod
                         |           |           |
                    Container   Container   Container
```

And:

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

Networking:

```text
Ingress
   ↓
Service
   ↓
Pods
```

Scaling:

```text
HPA
 ↓
Pods

Cluster Autoscaler
 ↓
Nodes
```

Security:

```text
IAM
 ↓
Google Cloud

RBAC
 ↓
Kubernetes

Workload Identity
 ↓
Pod → Google Cloud
```

---

# 155. GKE Production Checklist

Before calling a GKE deployment "production ready", think about:

### Application

* [ ] Container image
* [ ] Health endpoints
* [ ] Graceful shutdown
* [ ] Stateless where possible

### Kubernetes

* [ ] Deployment
* [ ] Service
* [ ] Resource requests
* [ ] Resource limits
* [ ] Readiness probe
* [ ] Liveness probe
* [ ] Startup probe where required
* [ ] Appropriate replicas

### Scaling

* [ ] HPA where required
* [ ] Cluster autoscaling where required
* [ ] Correct resource sizing

### Networking

* [ ] Ingress/Gateway
* [ ] Service discovery
* [ ] Network policies where required
* [ ] Appropriate load balancing

### Security

* [ ] IAM
* [ ] RBAC
* [ ] Workload Identity
* [ ] Secret management
* [ ] Least privilege
* [ ] Image security

### Reliability

* [ ] Multi-zone strategy where required
* [ ] Pod distribution
* [ ] PodDisruptionBudget where appropriate
* [ ] Graceful shutdown
* [ ] Backup/recovery strategy for stateful dependencies

### Observability

* [ ] Logs
* [ ] Metrics
* [ ] Alerts
* [ ] Traces where useful
* [ ] Application dashboards

### Operations

* [ ] CI/CD
* [ ] Rollback strategy
* [ ] Upgrade strategy
* [ ] Cost monitoring

---

# 156. The 20 Questions You Should Absolutely Be Able to Answer

Before your interview, make sure you can explain these **without memorizing definitions**:

### Fundamentals

1. What is GKE?
2. Why use GKE?
3. Kubernetes vs Docker?
4. GKE vs Compute Engine?
5. GKE vs Cloud Run?
6. Standard vs Autopilot?
7. What is a cluster?
8. What is a node?
9. What is a Pod?
10. What is a Deployment?

### Networking

11. What is a Service?
12. ClusterIP vs NodePort vs LoadBalancer?
13. What is Ingress?
14. Service vs Ingress?
15. How does service discovery work?
16. How do Pods communicate?

### Scaling & Reliability

17. HPA vs VPA vs Cluster Autoscaler?
18. Liveness vs Readiness vs Startup?
19. What happens when a Pod dies?
20. What happens when a node dies?

---

# 157. Senior-Level Questions You Should Prepare

For a ~5 YOE backend engineer, don't stop at definitions.

Be prepared for:

### Scenario 1

> Your Spring Boot Pod keeps restarting. How do you troubleshoot?

### Scenario 2

> Your application suddenly gets 10x traffic. How would GKE handle it?

### Scenario 3

> HPA increased Pods but performance didn't improve. Why?

### Scenario 4

> Pods are running but users receive 503.

### Scenario 5

> A Pod cannot access GCS.

### Scenario 6

> A deployment is stuck.

### Scenario 7

> One node goes down.

### Scenario 8

> Your application takes 90 seconds to start.

### Scenario 9

> You need zero/minimal downtime deployment.

### Scenario 10

> You need to deploy a new version to only 5% of users.

### Scenario 11

> Three replicas are running, but your application is still unavailable when a node fails.

### Scenario 12

> Your GKE bill suddenly increases.

### Scenario 13

> A Pod is Pending.

### Scenario 14

> ImagePullBackOff occurs after deployment.

### Scenario 15

> Your service has no endpoints.

---

# 158. The Senior Engineer Way of Answering GKE Questions

Don't answer:

> "I'll increase the number of Pods."

Instead:

> "First I'd identify whether the bottleneck is application CPU, memory, downstream latency, database capacity, or another dependency. If the workload is horizontally scalable and resource utilization supports it, I'd use HPA and ensure sufficient node capacity through cluster autoscaling. I'd also verify that readiness probes and resource requests are configured correctly."

That sounds much more like an engineer who has operated systems rather than someone who memorized Kubernetes commands.

---

# 159. The Ultimate Mental Model

If you forget everything during the interview, remember this:

```text
                         GKE
                          |
                 "Run my containers"
                          |
              -----------------------
              |                     |
          Control Plane           Nodes
              |                     |
        "What should happen?"   "Run workloads"
                                    |
                                  Pods
                                    |
                                Containers
```

Then:

```text
                    Application
                        |
                        v
                    Deployment
                        |
                        v
                   ReplicaSet
                        |
              -------------------
              |        |        |
             Pod      Pod      Pod
              \        |       /
               \       |      /
                    Service
                       |
                       v
                    Ingress
                       |
                       v
                  Load Balancer
                       |
                       v
                    Internet
```

And:

```text
Traffic increases
       |
       v
      HPA
       |
       v
 More Pods
       |
       v
Need more capacity?
       |
       v
Cluster Autoscaler
       |
       v
 More Nodes
```

And finally:

```text
Pod
 |
 +── Workload Identity → GCP APIs
 |
 +── Service → Other Pods
 |
 +── ConfigMap → Configuration
 |
 +── Secret → Sensitive configuration
 |
 +── PVC → Persistent storage
 |
 +── Logs/Metrics → Observability
```

---

# 160. One-Minute GKE Interview Answer

If the interviewer says:

> **"Tell me what you know about GKE."**

You can structure your answer like this:

> **GKE is Google's managed Kubernetes service for deploying and operating containerized applications. A GKE cluster consists of a Kubernetes control plane and worker nodes, with applications running inside Pods.**
>
> **For stateless applications, I would typically use Deployments, which manage ReplicaSets and Pods. Services provide stable networking to Pods, while Ingress or Gateway-based solutions can expose and route HTTP traffic from outside the cluster.**
>
> **For production workloads, I'd consider resource requests and limits, readiness/liveness probes, HPA for Pod scaling, and cluster autoscaling for node capacity. For high availability, I'd distribute replicas across nodes and appropriate failure domains.**
>
> **From a security perspective, I'd use IAM for Google Cloud access, Kubernetes RBAC for Kubernetes resources, and Workload Identity for workloads accessing Google Cloud services rather than embedding service-account keys in containers.**
>
> **GKE also integrates closely with Google Cloud services such as Artifact Registry, Cloud Logging, Cloud Monitoring, Cloud Load Balancing, Secret Manager, Cloud SQL, Pub/Sub and VPC networking.**
>
> **I'd choose GKE when I need Kubernetes orchestration capabilities and more control over containerized workloads; for simpler container workloads, I would also evaluate whether Cloud Run provides a simpler operational model.**

---

# 161. Final Cheat Sheet — GKE in One Page

```text
GKE
│
├── Kubernetes Cluster
│   │
│   ├── Control Plane
│   │   ├── API Server
│   │   ├── Scheduler
│   │   ├── Controllers
│   │   └── etcd
│   │
│   └── Nodes
│       │
│       └── Pods
│           │
│           └── Containers
│
├── Workloads
│   ├── Deployment
│   ├── StatefulSet
│   ├── DaemonSet
│   ├── Job
│   └── CronJob
│
├── Networking
│   ├── Service
│   │   ├── ClusterIP
│   │   ├── NodePort
│   │   └── LoadBalancer
│   │
│   ├── Ingress
│   ├── Gateway API
│   ├── DNS
│   └── NetworkPolicy
│
├── Scaling
│   ├── HPA → Pods
│   ├── VPA → Pod resources
│   └── Cluster Autoscaler → Nodes
│
├── Configuration
│   ├── ConfigMap
│   └── Secret
│
├── Storage
│   ├── PV
│   ├── PVC
│   └── StorageClass
│
├── Security
│   ├── IAM
│   ├── RBAC
│   ├── Workload Identity
│   ├── NetworkPolicy
│   └── Secret management
│
├── Reliability
│   ├── Multiple replicas
│   ├── Multi-zone distribution
│   ├── Probes
│   ├── PDB
│   └── Graceful shutdown
│
├── Deployment
│   ├── Rolling
│   ├── Canary
│   └── Blue-Green
│
└── Google Cloud Integration
    ├── Artifact Registry
    ├── Cloud Load Balancing
    ├── Cloud Logging
    ├── Cloud Monitoring
    ├── Secret Manager
    ├── Cloud SQL
    ├── Memorystore
    ├── Pub/Sub
    └── VPC
```

## The 10 things I'd memorize first

If you're revising this shortly before an interview, prioritize these:

**1.** GKE = managed Kubernetes on Google Cloud.

**2.** `Cluster → Nodes → Pods → Containers`.

**3.** `Deployment → ReplicaSet → Pods`.

**4.** `Ingress → Service → Pods`.

**5.** `HPA → Pods`, `Cluster Autoscaler → Nodes`.

**6.** `Readiness = receive traffic`, `Liveness = restart`, `Startup = give startup time`.

**7.** `IAM = GCP access`, `RBAC = Kubernetes access`, `Workload Identity = secure Pod → GCP authentication`.

**8.** `Standard = more control`, `Autopilot = more managed`.

**9.** `Deployment = stateless`, `StatefulSet = stateful`, `DaemonSet = one per node`, `Job = one-time`, `CronJob = scheduled`.

**10.** When troubleshooting, follow the request path:

```text
User
 ↓
Load Balancer
 ↓
Ingress
 ↓
Service
 ↓
Endpoint
 ↓
Pod
 ↓
Container
 ↓
Application
 ↓
Dependency
```

That mental model will take you surprisingly far in a GKE interview.
