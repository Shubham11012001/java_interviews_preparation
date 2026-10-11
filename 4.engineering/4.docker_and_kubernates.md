# Docker & Kubernetes: The Architect's Interview Cheatsheet

*Explained simply enough for a 10-year-old, deep enough for a Technical Architect interview.*

---

## Part 0: The Big Picture (Read This First)

**The lunchbox analogy 🍱**

Imagine you cook a perfect meal at home. You send it to a friend, but when they heat it up in their kitchen, it tastes different: different stove, different pans, missing spices.

**Docker** is a *sealed lunchbox*. You pack the food, the spices, even a mini-stove inside. Wherever it's opened, it works **exactly the same**.

Now imagine you have **10,000 lunchboxes** to deliver across a city, some get dropped, some need replacing, some need to be doubled during rush hour. You need a **delivery manager**.

**Kubernetes** is that manager. It decides where each lunchbox goes, replaces broken ones, and adds more when demand spikes.

| Tool | Job | Analogy |
|---|---|---|
| **Docker** | Packages and runs ONE app consistently | Sealed lunchbox |
| **Kubernetes** | Manages MANY containers across MANY machines | Delivery manager / city traffic control |

**One-liner for interviews:**
> "Docker packages applications into portable containers. Kubernetes orchestrates those containers at scale, handling scheduling, self-healing, scaling, and networking."

---

## Part 1: Docker

### 1.1 Why does Docker exist? (The Problem)

> *"It works on my machine!"* 😅

Before Docker:
- Dev has Java 11, server has Java 8 → app crashes
- Different OS libraries, different config → "works locally, fails in prod"
- Setting up a new developer's laptop took **days**
- Running multiple apps on one server meant dependency conflicts

**Docker's solution:** bundle app + dependencies + config into one **image** that runs identically everywhere.

### 1.2 Containers vs Virtual Machines (Top Interview Question)

**Analogy:** 
- **VM** = a whole separate *house* (own plumbing, electricity, walls) for each family. Heavy.
- **Container** = a separate *apartment* in one building. Shares plumbing and electricity (the OS kernel), but has its own locked door. Light.

| Feature | Virtual Machine | Container |
|---|---|---|
| Includes | Full guest OS + app | App + libraries only |
| Size | GBs | MBs |
| Startup | Minutes | Seconds (or milliseconds) |
| Isolation | Strong (hardware-level via hypervisor) | Process-level (shares host kernel) |
| Resource use | Heavy | Lightweight |
| Portability | Moderate | Excellent |
| Best for | Strong isolation, different OSes | Microservices, fast scaling, CI/CD |

```
 VM Architecture              Container Architecture
┌─────┐ ┌─────┐ ┌─────┐      ┌─────┐ ┌─────┐ ┌─────┐
│App A│ │App B│ │App C│      │App A│ │App B│ │App C│
│Libs │ │Libs │ │Libs │      │Libs │ │Libs │ │Libs │
│GuestOS│GuestOS│GuestOS│     └──┬──┴──┬──┴──┬──┘
└──┬──┘ └──┬──┘ └──┬──┘         Docker Engine
   Hypervisor                    Host OS
   Host OS                       Hardware
   Hardware
```

**How containers achieve isolation (architect-level answer):**
- **Namespaces** → isolate *what a process can see* (PID, network, filesystem, users)
- **cgroups (control groups)** → limit *what a process can use* (CPU, memory, I/O)
- **Union filesystem (OverlayFS)** → layered, efficient storage

### 1.3 Core Docker Concepts

| Concept | Simple Meaning | Analogy |
|---|---|---|
| **Dockerfile** | Recipe/instructions to build an image | Recipe card |
| **Image** | Read-only template (built from Dockerfile) | Frozen meal / cookie cutter |
| **Container** | Running instance of an image | The actual cookie |
| **Registry** | Storage for images (Docker Hub, ECR, GCR, ACR) | App store for images |
| **Volume** | Persistent storage outside container lifecycle | External hard drive |
| **Network** | How containers talk to each other | Intercom system |
| **Docker Engine/Daemon** | Background service that builds and runs containers | The kitchen |

**Image vs Container (very common question):**
> Image = class. Container = object. One image can create many containers.

### 1.4 Docker Architecture

```
Docker CLI  ──REST API──▶  Docker Daemon (dockerd)  ──▶  containerd  ──▶  runc
(you type)                 (builds, manages)              (lifecycle)      (actually creates container)
                                   │
                                   ▼
                              Registry (pull/push)
```

### 1.5 Dockerfile: Essentials

```dockerfile
# Example: Java Spring Boot app
FROM eclipse-temurin:17-jre-alpine     # Base image
WORKDIR /app                            # Working directory
COPY target/app.jar app.jar             # Copy file from host to image
EXPOSE 8080                             # Documents the port (doesn't publish it)
ENV SPRING_PROFILES_ACTIVE=prod         # Environment variable
ENTRYPOINT ["java", "-jar", "app.jar"]  # Command that runs on start
```

**Key instructions:**

| Instruction | Purpose |
|---|---|
| `FROM` | Base image |
| `RUN` | Execute command **at build time** (creates a layer) |
| `CMD` | Default command **at runtime** (can be overridden) |
| `ENTRYPOINT` | Fixed command at runtime (args appended) |
| `COPY` | Copy files from host |
| `ADD` | Like COPY + can extract tar and fetch URLs (prefer `COPY`) |
| `WORKDIR` | Set working directory |
| `ENV` | Set environment variable (persists in image) |
| `ARG` | Build-time variable only |
| `EXPOSE` | Documents port |
| `VOLUME` | Declares mount point |
| `USER` | Run as non-root user |
| `HEALTHCHECK` | Defines how to check container health |

**CMD vs ENTRYPOINT (classic trap):**
- `ENTRYPOINT` = "this container *is* this program"
- `CMD` = "default arguments, easily replaced"
- Together: `ENTRYPOINT ["java","-jar","app.jar"]` + `CMD ["--spring.profiles.active=dev"]`

**COPY vs ADD:** Use `COPY`. `ADD` has hidden magic (auto-extract, URL fetch) that causes surprises.

**ENV vs ARG:** `ARG` exists only during build. `ENV` persists in the running container. ⚠️ Never put secrets in either: both are visible in image history.

### 1.6 Image Layers & Caching (Interview Favorite)

Every instruction creates a **layer**. Layers are cached. If a layer changes, **all layers after it are rebuilt**.

**❌ Bad (cache busts every code change):**
```dockerfile
COPY . .
RUN mvn dependency:resolve
RUN mvn package
```

**✅ Good (dependencies cached unless pom.xml changes):**
```dockerfile
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package
```

**Rule:** Put things that change **least** at the top, **most** at the bottom.

### 1.7 Multi-Stage Builds (Must-Know Optimization)

Build tools (Maven, JDK, compilers) are needed to *build* but not to *run*. Don't ship them.

```dockerfile
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# Stage 2: Run (tiny final image)
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /build/target/app.jar app.jar
USER 1001
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Result:** Image drops from ~700MB → ~150MB. Smaller = faster pulls, smaller attack surface.

### 1.8 Essential Docker Commands

```bash
# Images
docker build -t myapp:1.0 .            # Build image
docker images                           # List images
docker pull nginx:latest                # Download image
docker push myrepo/myapp:1.0            # Upload image
docker rmi <image>                      # Remove image
docker image prune                      # Remove unused images

# Containers
docker run -d -p 8080:80 --name web nginx   # Run detached, map port
docker ps                                    # Running containers
docker ps -a                                 # All containers
docker stop <container>                      # Graceful stop (SIGTERM)
docker kill <container>                      # Force stop (SIGKILL)
docker rm <container>                        # Remove container
docker logs -f <container>                   # Follow logs
docker exec -it <container> /bin/sh          # Shell into container
docker inspect <container>                   # Detailed JSON info
docker stats                                 # Live resource usage

# Cleanup
docker system prune -a                       # Remove everything unused (careful!)

# Volumes & Networks
docker volume create mydata
docker network create mynet
docker run -v mydata:/var/lib/data myapp
```

**Port mapping `-p 8080:80`** → `hostPort:containerPort`. Traffic hitting host's 8080 goes to container's 80.

### 1.9 Docker Networking

| Driver | What it does | Use case |
|---|---|---|
| **bridge** (default) | Private network on a single host; containers talk by IP/name (custom bridge) | Most local setups |
| **host** | Container shares host's network stack, no isolation | Max performance, no port mapping |
| **none** | No networking | Fully isolated batch jobs |
| **overlay** | Multi-host networking (Swarm) | Cross-host communication |
| **macvlan** | Container gets its own MAC/IP on physical network | Legacy apps needing direct LAN presence |

**Tip:** On a *user-defined* bridge network, containers can reach each other **by name** (built-in DNS). The default bridge doesn't offer this.

### 1.10 Docker Storage

Containers are **ephemeral**: when removed, their data is gone.

| Type | Description | Use case |
|---|---|---|
| **Volume** | Managed by Docker, stored in Docker's area | Databases, production data ✅ |
| **Bind mount** | Maps a host folder into container | Local development (live code reload) |
| **tmpfs** | In-memory only, never written to disk | Sensitive temp data, caches |

### 1.11 Docker Compose

Runs **multi-container apps** on one machine using a single YAML file.

```yaml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=db
    depends_on:
      - db
  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=secret
volumes:
  pgdata:
```

```bash
docker compose up -d      # Start everything
docker compose down       # Stop and remove
docker compose logs -f
```

**When to use:** Local dev, testing, small single-host setups. **Not** for production at scale, that's Kubernetes.

⚠️ `depends_on` only waits for container *start*, not for the app to be *ready*. Use healthchecks.

### 1.12 Docker Security Best Practices

1. ✅ Run as **non-root** (`USER`)
2. ✅ Use **minimal base images** (alpine, distroless, slim)
3. ✅ **Pin versions** (`node:20.11-alpine`, never `latest`)
4. ✅ **Scan images** (Trivy, Snyk, Grype, Docker Scout)
5. ✅ Never bake **secrets** into images; inject at runtime
6. ✅ Use **read-only root filesystem** where possible
7. ✅ **Drop Linux capabilities** you don't need
8. ✅ Use **.dockerignore** (exclude `.git`, `node_modules`, secrets)
9. ✅ Sign images (Cosign / Docker Content Trust)
10. ✅ Keep host and Docker updated

### 1.13 Docker: When, Where, Why

| ✅ Use Docker When | ❌ Avoid / Be Careful When |
|---|---|
| Microservices architecture | Heavy GUI desktop applications |
| Need environment consistency (dev = prod) | Need a different OS kernel (Windows app on Linux host) |
| CI/CD pipelines | Extremely high-security multi-tenant isolation (consider VMs/gVisor/Kata) |
| Fast onboarding for developers | Stateful DBs in production without a solid storage strategy |
| Scaling horizontally | Apps requiring direct hardware access |
| Packaging legacy apps for portability | |

---

## Part 2: Kubernetes (K8s)

*(K8s = "K" + 8 letters + "s". Cute abbreviation. Greek for "helmsman" or "pilot".)*

### 2.1 Why Kubernetes? (The Problem Docker Alone Can't Solve)

Docker runs containers on **one machine**. In production, you ask:

- What if the container **crashes** at 3 AM?
- What if the **whole server** dies?
- How do I **scale** from 2 to 50 instances during a sale?
- How do I deploy a new version with **zero downtime**?
- How do containers on **different machines** find each other?
- How do I manage **secrets and config** for hundreds of services?
- How do I **load balance** traffic?

**Kubernetes answers all of these.** It's a **container orchestration platform** (originally by Google, based on their internal system *Borg*; now maintained by CNCF).

**Kubernetes gives you:**
- 🔁 **Self-healing**: restarts failed containers, replaces dead nodes
- 📈 **Auto-scaling**: horizontal (pods) and vertical (resources)
- 🚀 **Rolling updates & rollbacks**: zero-downtime deployments
- ⚖️ **Service discovery & load balancing**
- 🔐 **Secret & config management**
- 💾 **Storage orchestration**
- 🧩 **Declarative configuration**: you say *what* you want, K8s figures out *how*

### 2.2 The Core Idea: Declarative Model & Reconciliation Loop

You don't say *"start 3 containers."* You say *"I **want** 3 running."*

Kubernetes constantly compares **desired state** vs **actual state** and fixes the difference.

> **Analogy:** A thermostat. You set 24°C. It doesn't care why the room got cold. It just keeps correcting.

```
Desired state (YAML): 3 replicas
        ↓
Controller watches actual state
        ↓
Actual = 2 (one crashed)?  → Create 1 more
Actual = 4?                → Delete 1
```

This **control loop** is the heart of Kubernetes. Memorize it.

### 2.3 Kubernetes Architecture

```
┌──────────────────── CONTROL PLANE (the brain) ────────────────────┐
│  API Server   │   etcd    │  Scheduler  │  Controller Manager     │
│  (front door) │ (database)│ (placement) │  (reconciliation loops) │
│               │           │             │  + Cloud Controller Mgr │
└───────────────────────────────┬───────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
┌─── Worker Node 1 ──┐  ┌─── Worker Node 2 ──┐  ┌─── Worker Node 3 ──┐
│ kubelet            │  │ kubelet            │  │ kubelet            │
│ kube-proxy         │  │ kube-proxy         │  │ kube-proxy         │
│ Container runtime  │  │ Container runtime  │  │ Container runtime  │
│ [Pod][Pod][Pod]    │  │ [Pod][Pod]         │  │ [Pod][Pod][Pod]    │
└────────────────────┘  └────────────────────┘  └────────────────────┘
```

**Control Plane Components**

| Component | Role | Analogy |
|---|---|---|
| **kube-apiserver** | The single entry point. All commands (kubectl, internal components) go through it. Validates & processes REST requests | Reception desk |
| **etcd** | Distributed key-value store holding **all cluster state**. If lost without backup, cluster state is lost | The master record book |
| **kube-scheduler** | Decides *which node* a new pod runs on (based on resources, affinity, taints, etc.) | Seating host at a restaurant |
| **kube-controller-manager** | Runs controllers (Deployment, ReplicaSet, Node, Job controllers...) that reconcile state | Supervisors |
| **cloud-controller-manager** | Talks to cloud provider APIs (load balancers, volumes, nodes) | Cloud liaison |

**Worker Node Components**

| Component | Role |
|---|---|
| **kubelet** | Agent on each node. Ensures containers described in PodSpecs are running & healthy. Reports to API server |
| **kube-proxy** | Maintains network rules (iptables/IPVS) so Services route traffic to Pods |
| **Container runtime** | Actually runs containers (containerd, CRI-O). *Docker Engine is no longer directly used since K8s 1.24 (dockershim removed), but Docker-built images still work perfectly* |

**What happens when you run `kubectl apply -f deployment.yaml`? (Gold interview answer)**

1. `kubectl` sends request to **API Server**
2. API Server authenticates, authorizes (RBAC), runs **admission controllers**, validates
3. Object stored in **etcd**
4. **Deployment controller** notices → creates a **ReplicaSet**
5. **ReplicaSet controller** → creates **Pod** objects
6. **Scheduler** sees unscheduled Pods → assigns each to a node
7. **kubelet** on that node sees assignment → tells container runtime to pull image & start container
8. kubelet reports status back to API Server → stored in etcd

### 2.4 Core Kubernetes Objects

#### 🟦 Pod
- **Smallest deployable unit.** A wrapper around 1+ containers.
- Containers in a pod share: **network (same IP, localhost)**, **storage volumes**, lifecycle.
- Pods are **ephemeral**: they die, and a replacement gets a *new IP*.

> **Analogy:** A pod is a room; containers are people in it who share the same address and fridge.

**Multi-container patterns:**
- **Sidecar** → helper alongside main app (log shipper, Envoy proxy / service mesh)
- **Init container** → runs *before* main container, to completion (DB migration, wait-for-dependency)
- **Ambassador** → proxy to outside world
- **Adapter** → standardizes output (e.g., metrics format)

#### 🟦 ReplicaSet
Ensures N identical pods are running. You rarely create these directly. **Deployments** manage them.

#### 🟦 Deployment
Manages ReplicaSets and provides **rolling updates, rollbacks, scaling**. For **stateless** apps.

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
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Can create 1 extra pod during update
      maxUnavailable: 0    # Never drop below desired count
  template:
    metadata:
      labels:
        app: payment
    spec:
      containers:
      - name: payment
        image: myrepo/payment:1.4.2
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
```

```bash
kubectl rollout status deployment/payment-service
kubectl rollout history deployment/payment-service
kubectl rollout undo deployment/payment-service      # Rollback!
kubectl scale deployment/payment-service --replicas=10
```

#### 🟦 StatefulSet
For **stateful** apps (databases, Kafka, Elasticsearch, ZooKeeper).
Gives pods: **stable identity** (`db-0`, `db-1`), **stable storage** (each gets its own PVC), **ordered startup/shutdown**.

| Deployment | StatefulSet |
|---|---|
| Pods interchangeable | Each pod unique |
| Random names (`app-7d9f-xk2`) | Ordered names (`app-0`, `app-1`) |
| Shared/no storage | Dedicated persistent storage per pod |
| Parallel start | Ordered start |

#### 🟦 DaemonSet
Runs **one pod on every node** (or selected nodes). Use for: log collectors (Fluentd), monitoring agents (Prometheus node-exporter), network plugins (CNI), security agents.

#### 🟦 Job & CronJob
- **Job** → run a task to completion (batch processing, migration)
- **CronJob** → Job on a schedule (nightly report, cleanup)

#### 🟦 Service (Networking Stable Endpoint)
Pods come and go with changing IPs. A **Service** gives a **stable IP + DNS name** and load-balances across matching pods (selected via **labels**).

| Type | Exposure | Use case |
|---|---|---|
| **ClusterIP** (default) | Internal only | Service-to-service communication |
| **NodePort** | Opens a port (30000-32767) on every node | Quick testing, on-prem |
| **LoadBalancer** | Provisions cloud LB (AWS ELB, GCP LB) | Expose single service publicly |
| **ExternalName** | DNS alias to external service | Pointing to external DB/API |
| **Headless** (`clusterIP: None`) | No virtual IP, returns pod IPs directly | StatefulSets, client-side discovery |

**DNS format:** `<service>.<namespace>.svc.cluster.local`

#### 🟦 Ingress
Smart **HTTP/HTTPS router** at Layer 7: host/path-based routing, TLS termination, one load balancer for many services.

```
Internet → Ingress Controller (NGINX/Traefik/ALB) → 
    /api/*     → api-service
    /shop/*    → shop-service
    app.com    → frontend-service
```

⚠️ Ingress *resource* is just rules. You need an **Ingress Controller** installed to enforce them. (Note: **Gateway API** is the newer, more expressive successor to Ingress.)

**Service vs Ingress:** Service = L4, exposes pods. Ingress = L7, routes HTTP traffic to *Services*.

#### 🟦 ConfigMap & Secret
- **ConfigMap** → non-sensitive config (URLs, feature flags, properties files)
- **Secret** → sensitive data (passwords, tokens, certs)

⚠️ **Secrets are only base64-*encoded*, NOT encrypted by default.** Production fixes: enable **encryption at rest** for etcd, use **external secret managers** (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, External Secrets Operator), strict RBAC.

Consume via: environment variables or mounted volume files.

#### 🟦 Namespace
Virtual cluster inside a cluster. Isolate teams/environments (`dev`, `staging`, `prod`). Apply **ResourceQuotas**, **LimitRanges**, **NetworkPolicies**, **RBAC** per namespace. *Not* a strong security boundary on its own.

#### 🟦 Volumes: PV, PVC, StorageClass

| Object | Meaning | Analogy |
|---|---|---|
| **PersistentVolume (PV)** | Actual piece of storage in cluster | A parking space |
| **PersistentVolumeClaim (PVC)** | A pod's *request* for storage | A parking ticket request |
| **StorageClass** | Template for **dynamic provisioning** (SSD vs HDD, etc.) | Parking space type menu |

**Access modes:** `ReadWriteOnce (RWO)`, `ReadOnlyMany (ROX)`, `ReadWriteMany (RWX)`, `ReadWriteOncePod`.

#### 🟦 Labels, Selectors & Annotations
- **Labels** → key/value tags for *identifying and selecting* objects (`app=payment`, `env=prod`). This is how Services find Pods!
- **Annotations** → non-identifying metadata (build info, tool config)

### 2.5 Scheduling Controls (Architect-Level)

| Feature | Purpose |
|---|---|
| **nodeSelector** | Simple: run only on nodes with a label |
| **Node Affinity** | Advanced version of nodeSelector (required/preferred rules) |
| **Pod Affinity** | Schedule *near* certain pods (e.g., app near cache) |
| **Pod Anti-Affinity** | Schedule *away* from certain pods (spread replicas across nodes/zones for HA) ✅ |
| **Taints & Tolerations** | Node *repels* pods unless they tolerate the taint (dedicated GPU nodes, control plane) |
| **Topology Spread Constraints** | Evenly spread pods across zones/nodes |
| **PriorityClass** | Important pods can preempt less important ones |

> **Taint = "bouncer at the club door". Toleration = "VIP pass".**

### 2.6 Resource Management: Requests vs Limits (Very Common Question)

```yaml
resources:
  requests:        # Guaranteed minimum; used by SCHEDULER for placement
    cpu: "250m"
    memory: "256Mi"
  limits:          # Maximum allowed
    cpu: "500m"
    memory: "512Mi"
```

- **CPU limit exceeded** → container is **throttled** (slowed down)
- **Memory limit exceeded** → container is **OOMKilled** (terminated) 💀

**QoS Classes:**
| Class | Condition | Eviction priority |
|---|---|---|
| **Guaranteed** | requests == limits for all containers | Last to be evicted |
| **Burstable** | requests < limits | Middle |
| **BestEffort** | No requests/limits set | First to be evicted |

**ResourceQuota** → caps total resources per namespace. **LimitRange** → default/min/max per pod/container.

### 2.7 Health Probes (Critical for Production)

| Probe | Question it answers | If it fails |
|---|---|---|
| **Liveness** | "Is the app alive or stuck/deadlocked?" | Container is **restarted** |
| **Readiness** | "Is the app ready to receive traffic?" | Pod **removed from Service endpoints** (no restart) |
| **Startup** | "Has the slow-starting app finished booting?" | Disables liveness/readiness until success; kills if never succeeds |

> **Analogy:** Liveness = "Is the doctor's heart beating?" Readiness = "Is the doctor at their desk ready to see patients?" Startup = "Has the doctor finished arriving at work?"

⚠️ **Classic mistake:** Making liveness probe depend on external dependencies (DB). If DB blips, all pods restart → cascading failure. Keep liveness **internal & simple**; put dependency checks in readiness.

### 2.8 Scaling

| Scaler | What it scales | Based on |
|---|---|---|
| **HPA** (Horizontal Pod Autoscaler) | Number of **pods** | CPU, memory, custom metrics |
| **VPA** (Vertical Pod Autoscaler) | **CPU/memory** of pods | Observed usage |
| **Cluster Autoscaler** | Number of **nodes** | Pending pods that can't be scheduled |
| **KEDA** | Pods (event-driven, can scale to zero) | Kafka lag, queue depth, cron, etc. |

```bash
kubectl autoscale deployment payment --cpu-percent=70 --min=3 --max=20
```

**Flow in a traffic spike:** Load ↑ → HPA adds pods → no room on nodes → pods Pending → Cluster Autoscaler adds nodes → pods scheduled.

⚠️ HPA needs **resource requests** defined, and **Metrics Server** installed. Don't use HPA and VPA on the same metric.

### 2.9 Deployment Strategies

| Strategy | How | Pros | Cons | Use when |
|---|---|---|---|---|
| **Rolling Update** (default) | Replace pods gradually | Zero downtime, simple | Two versions run simultaneously; slower rollback | Most apps |
| **Recreate** | Kill all old, start new | Simple, clean | **Downtime** | Dev, or apps that can't run 2 versions |
| **Blue/Green** | Two full environments, switch traffic instantly | Instant rollback, easy testing | 2× resources | Critical releases |
| **Canary** | Send 5% traffic to new version, increase gradually | Lowest risk, real-user validation | More complex (needs mesh/ingress rules) | High-risk changes |
| **A/B Testing** | Route by user attributes/headers | Feature experiments | Complex | Product experiments |

Tools: **Argo Rollouts**, **Flagger**, **Istio**, NGINX Ingress canary annotations.

### 2.10 Networking Deep-Dive

**The Kubernetes network model (3 rules):**
1. Every Pod gets its **own unique IP**
2. All Pods can talk to all other Pods **without NAT**
3. Agents on a node can talk to all pods on that node

- **CNI (Container Network Interface)** plugins implement this: **Calico, Cilium, Flannel, Weave, AWS VPC CNI**
- **NetworkPolicy** → firewall rules between pods. By default, *everything can talk to everything*. ⚠️ Requires a CNI that supports it (Calico, Cilium).
- **CoreDNS** → cluster DNS

**Best practice:** Default-deny NetworkPolicy, then allow only what's needed (zero-trust).

**Service Mesh (Istio, Linkerd):** sidecar proxies handling mTLS, retries, circuit breaking, traffic shifting, observability: without changing app code. Adds complexity and latency, so use when you have many microservices and need those features.

### 2.11 Security in Kubernetes

| Area | Practice |
|---|---|
| **RBAC** | Least privilege. `Role`/`ClusterRole` + `RoleBinding`/`ClusterRoleBinding`. Avoid `cluster-admin` |
| **ServiceAccounts** | Dedicated per workload; disable auto-mount token if unused |
| **Pod Security Standards** | `Privileged` / `Baseline` / `Restricted` (enforced via Pod Security Admission; replaced deprecated PodSecurityPolicy) |
| **SecurityContext** | `runAsNonRoot`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`, drop capabilities |
| **NetworkPolicies** | Default deny, explicit allow |
| **Secrets** | Encrypt etcd, external secret stores, rotate regularly |
| **Image security** | Scan, sign, use private registries, admission policies (OPA Gatekeeper / Kyverno) |
| **API server** | Restrict access, enable audit logging |
| **Runtime security** | Falco, seccomp, AppArmor |
| **Supply chain** | SBOMs, signed images, trusted registries |

### 2.12 Essential kubectl Commands

```bash
# View
kubectl get pods -o wide -n <namespace>
kubectl get all
kubectl get nodes
kubectl describe pod <pod>            # 🔥 Best debugging command (see Events at bottom)
kubectl logs <pod> -f                 # Follow logs
kubectl logs <pod> --previous         # Logs from crashed previous container ✅
kubectl logs <pod> -c <container>     # Specific container
kubectl top pod / kubectl top node    # Resource usage

# Create / Update
kubectl apply -f file.yaml            # Declarative (preferred)
kubectl create -f file.yaml           # Imperative create
kubectl delete -f file.yaml
kubectl edit deployment <name>

# Debug
kubectl exec -it <pod> -- /bin/sh
kubectl port-forward pod/<pod> 8080:80
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl debug <pod> -it --image=busybox      # Ephemeral debug container

# Rollouts
kubectl rollout status|history|undo deployment/<name>

# Context
kubectl config get-contexts
kubectl config use-context <name>
kubectl config set-context --current --namespace=<ns>

# Quick YAML generation (saves time)
kubectl create deployment web --image=nginx --dry-run=client -o yaml
```

### 2.13 Helm & Packaging

- **Helm** = package manager for Kubernetes. A **Chart** = templated bundle of YAML. `values.yaml` = configuration per environment.
- ```bash
  helm install myapp ./chart -f values-prod.yaml
  helm upgrade myapp ./chart
  helm rollback myapp 1
  ```
- Alternative: **Kustomize** (patch-based overlays, built into kubectl: `kubectl apply -k`). Helm = templating; Kustomize = overlay/patching.

### 2.14 GitOps

**Idea:** Git is the **single source of truth** for cluster state. A tool inside the cluster continuously syncs the cluster to match Git.
- Tools: **Argo CD**, **Flux**
- Benefits: audit trail, easy rollback (`git revert`), consistency, no direct `kubectl` access to prod.

### 2.15 Observability

| Pillar | Tools |
|---|---|
| **Metrics** | Prometheus + Grafana, Metrics Server |
| **Logs** | EFK/ELK (Elasticsearch, Fluentd/Fluent Bit, Kibana), Loki |
| **Traces** | Jaeger, Tempo, OpenTelemetry |
| **Alerting** | Alertmanager, PagerDuty |

Key metrics: **RED** (Rate, Errors, Duration) for services; **USE** (Utilization, Saturation, Errors) for resources.

### 2.16 Managed Kubernetes

| Cloud | Service |
|---|---|
| AWS | **EKS** |
| Google Cloud | **GKE** (also Autopilot mode) |
| Azure | **AKS** |

**Why managed?** Cloud runs and patches the control plane for you. You manage worker nodes (or use serverless modes like GKE Autopilot / EKS Fargate).
**Self-managed** (kubeadm, kOps, Rancher, OpenShift) → full control, much more operational burden.

*(Given your GCP Associate Cloud Engineer background: GKE integrates with Cloud Load Balancing, Workload Identity, Cloud Logging/Monitoring, Artifact Registry, and offers Standard vs Autopilot modes, which is a strong point to bring up in interviews.)*

### 2.17 Local Kubernetes Options
**Minikube, kind (Kubernetes in Docker), k3s/k3d, Docker Desktop K8s, MicroK8s**

---

## Part 3: Docker vs Kubernetes Cheat Table

| | Docker | Kubernetes |
|---|---|---|
| **What** | Containerization platform | Container orchestration platform |
| **Scope** | Single host | Cluster of many hosts |
| **Job** | Build, ship, run containers | Deploy, scale, heal, network containers |
| **Scaling** | Manual (or Compose/Swarm) | Automatic (HPA, Cluster Autoscaler) |
| **Self-healing** | Basic restart policies | Full (reschedules on other nodes) |
| **Load balancing** | Limited | Built-in (Services, Ingress) |
| **Complexity** | Low | High |
| **Config** | Dockerfile, Compose | YAML manifests, Helm |

**They are NOT competitors.** Docker builds the image; Kubernetes runs and manages containers from it.

**Docker Swarm vs Kubernetes:** Swarm is simpler but has limited features and a much smaller ecosystem. Kubernetes is the industry standard.

---

## Part 4: When to Use / Not Use Kubernetes

### ✅ Use Kubernetes When
- Many microservices (rule of thumb: 10+ services)
- Need high availability, autoscaling, zero-downtime deploys
- Multi-cloud or hybrid cloud strategy (avoid vendor lock-in)
- Large teams needing standardized deployment platform
- Variable/spiky traffic
- Need rich ecosystem (service mesh, operators, GitOps)

### ❌ Don't Use Kubernetes When
- Simple monolith or 1-3 small services (**Docker Compose / PaaS / serverless is enough**)
- Small team without ops/platform expertise
- Tight budget (control plane + node overhead + learning curve)
- Pure event-driven, short-lived functions (serverless fits better)
- Latency-critical workloads needing bare-metal tuning

> **Architect's wisdom:** *"Kubernetes is a platform for building platforms. Don't adopt it because it's trendy; adopt it when the operational complexity it solves exceeds the complexity it adds."*

---

## Part 5: Production Scenarios (The Money Section 💰)

### 🔥 Scenario 1: Pod stuck in `CrashLoopBackOff`
**Meaning:** Container starts, crashes, K8s restarts it, crashes again, with increasing delay.
**Debug flow:**
1. `kubectl describe pod <pod>` → check **Events**, exit code, reason
2. `kubectl logs <pod> --previous` → see why it crashed
3. Common causes: app bug, **missing env/config/secret**, wrong command, **OOMKilled** (exit code 137), failing liveness probe, can't connect to dependency
4. Fix and redeploy

### 🔥 Scenario 2: Pod stuck in `Pending`
**Meaning:** Scheduler can't place it.
**Causes:** Insufficient CPU/memory on nodes, node taints without tolerations, unsatisfiable affinity, PVC not bound, node selector mismatch.
**Debug:** `kubectl describe pod` → Events say e.g. *"0/5 nodes available: insufficient memory"*.
**Fix:** Add nodes (Cluster Autoscaler), reduce requests, fix selectors/tolerations.

### 🔥 Scenario 3: `ImagePullBackOff` / `ErrImagePull`
**Causes:** Wrong image name/tag, private registry without `imagePullSecrets`, registry down, network issue, rate limits (Docker Hub).
**Fix:** Verify image, create/attach pull secret, use a private registry/mirror.

### 🔥 Scenario 4: `OOMKilled` (Exit code 137)
**Cause:** Memory usage exceeded limit.
**Fix:** Profile app, raise limit, fix memory leak. **For Java:** JVM must be container-aware (`-XX:MaxRAMPercentage=75`) so heap respects the cgroup limit; old JVMs read host memory and get killed. *(This is highly relevant to your Java background; mention it!)*

### 🔥 Scenario 5: App is running but users get 502/503
**Debug checklist:**
- Are pods **Ready**? (readiness probe failing?)
- Does Service **selector match pod labels**? (`kubectl get endpoints <svc>`: empty = mismatch!)
- Is `targetPort` correct?
- Ingress rules/backends correct?
- NetworkPolicy blocking?

### 🔥 Scenario 6: Zero-downtime deployment
- Rolling update with `maxUnavailable: 0`
- **Readiness probes** (so traffic only hits ready pods)
- **PodDisruptionBudget** (min available pods during voluntary disruptions)
- **Graceful shutdown**: handle SIGTERM, `terminationGracePeriodSeconds`, `preStop` hook to drain connections
- Multiple replicas spread across nodes/zones

### 🔥 Scenario 7: Node goes down
Node controller marks it `NotReady` → after timeout (~5 min default, tolerations), pods are evicted → Deployment/ReplicaSet recreates pods on healthy nodes. **StatefulSet pods** need careful handling (volume attachment, may need manual intervention for RWO volumes).

### 🔥 Scenario 8: Traffic spike (flash sale)
HPA scales pods → Cluster Autoscaler adds nodes → ensure **PodDisruptionBudgets**, **resource requests set**, **pre-scale before known events** (HPA reacts with delay), use **CDN/caching**, **rate limiting**, **queue-based load leveling**.

### 🔥 Scenario 9: Rollback a bad release
`kubectl rollout undo deployment/<name>` or Helm rollback or `git revert` (GitOps). Prevent with: canary releases, automated health analysis (Argo Rollouts), readiness gating.

### 🔥 Scenario 10: Database in Kubernetes: yes or no?
- **Generally:** use managed DBs (RDS, Cloud SQL) for production: backups, HA, patching handled.
- **If you must:** StatefulSet + PVCs + **Operators** (CloudNativePG, Strimzi for Kafka, etc.) + tested backups.
- **Why caution:** storage performance, failover complexity, data-loss risk.

### 🔥 Scenario 11: Secret leaked / rotation
Rotate immediately at source → update external secret store → rolling restart pods → audit access → investigate how it leaked. Use short-lived credentials & Workload Identity instead of static keys.

### 🔥 Scenario 12: Cluster upgrade with no downtime
Upgrade control plane first → then nodes one at a time (cordon → drain → upgrade → uncordon) or **surge/blue-green node pools**. Respect PDBs. Test in staging. Check deprecated APIs *before* upgrading (`kubectl convert`, `pluto`). Never skip minor versions (one minor at a time).

### 🔥 Scenario 13: Cost explosion
Right-size requests/limits (VPA recommendations), enable autoscaling, use **spot/preemptible nodes** for fault-tolerant workloads, clean unused PVs/LBs, namespace quotas, cost tools (Kubecost, OpenCost).

### 🔥 Scenario 14: Noisy neighbour
One pod hogs resources → set **limits**, **ResourceQuotas**, **LimitRanges**, proper **QoS**, dedicated node pools with taints.

### 🔥 Scenario 15: Disaster Recovery
Back up **etcd** (snapshots) + **PVs** (Velero) + keep manifests in Git (GitOps) → can rebuild cluster declaratively. Multi-region/multi-cluster for critical systems. Define **RPO/RTO**.

---

## Part 6: Best Practices ✅ vs Worst Practices ❌

### Docker

| ✅ Best | ❌ Worst |
|---|---|
| Multi-stage builds, small base images | Using `latest` tag in production |
| Pin versions | Running as root |
| One process per container | Putting secrets in Dockerfile/ENV |
| `.dockerignore` | Huge images with build tools included |
| Non-root user, scan images | Using SSH inside containers |
| Layer ordering for caching | Storing data inside container filesystem |
| Healthchecks | `COPY . .` first, busting cache |
| Log to stdout/stderr | Writing logs to files inside container |

### Kubernetes

| ✅ Best | ❌ Worst |
|---|---|
| Set **requests & limits** on everything | No resource limits (noisy neighbours, node crashes) |
| Readiness + liveness + startup probes | Liveness probe depending on DB |
| Multiple replicas + anti-affinity + PDBs | Single replica for critical services |
| Namespaces + RBAC + NetworkPolicies | Everything in `default` namespace with cluster-admin |
| Declarative configs in Git (GitOps) | Manual `kubectl edit` in prod ("snowflake clusters") |
| Immutable image tags (version/digest) | `latest` tag |
| External secrets manager | Plain secrets committed to Git |
| Graceful shutdown handling | Ignoring SIGTERM → dropped requests |
| Centralized logging & monitoring | "kubectl logs" as your only observability |
| Use managed K8s (EKS/GKE/AKS) | Self-managing control plane without expertise |
| Pods are cattle, not pets | Storing state on pod local disk |
| Regularly upgrade clusters | Running 3-year-old unsupported versions |

---

## Part 7: Worst-Case Disasters (What Can Go Wrong)

| Disaster | Cause | Prevention |
|---|---|---|
| **Entire cluster data lost** | etcd corrupted, no backups | Regular etcd snapshots, HA control plane (3 or 5 etcd nodes) |
| **Cascading restarts** | Bad liveness probes, dependency outage | Simple liveness; circuit breakers |
| **Node crash from memory exhaustion** | No limits | Requests/limits, QoS, eviction thresholds |
| **Data loss on StatefulSet** | Deleting PVC/wrong reclaim policy | `Retain` reclaim policy, backups (Velero) |
| **Security breach** | Over-privileged RBAC, exposed dashboard/API, vulnerable images | Least privilege, scanning, private API endpoint, audit logs |
| **Outage during upgrade** | Deprecated APIs, no PDB, big-bang upgrade | Staged upgrades, staging tests |
| **Crypto-mining in your cluster** | Exposed kubelet/API, malicious image | Network policies, admission control, runtime security |
| **Surprise cloud bill** | Autoscaling without max limits, orphaned LBs/disks | `maxReplicas`, quotas, cost monitoring |
| **DNS failure (CoreDNS overload)** | Heavy lookups, no DNS caching | Scale CoreDNS, NodeLocal DNSCache |
| **Thundering herd** | All pods restart/retry simultaneously | Jitter, backoff, gradual rollouts |
| **IP exhaustion** | Large clusters with small subnets | Plan CIDRs carefully upfront |

---

## Part 8: Advanced Concepts (Architect-Level Differentiators)

- **Operators & CRDs:** Extend Kubernetes with custom resources + controllers encoding operational knowledge (e.g., a `PostgresCluster` object that handles backups and failover). Pattern = CRD + controller.
- **Admission Controllers:** Intercept API requests before persistence. *Mutating* (modify) and *Validating* (accept/reject). Tools: OPA Gatekeeper, Kyverno.
- **Multi-tenancy:** Namespaces + quotas + RBAC + NetworkPolicy (soft), or separate clusters / vCluster (hard).
- **Multi-cluster / Multi-region:** For DR, latency, compliance. Tools: Karmada, Anthos, Fleet, service mesh multi-cluster.
- **Service Mesh:** Istio/Linkerd, mTLS, traffic management, observability.
- **PodDisruptionBudget (PDB):** Guarantees minimum available pods during *voluntary* disruptions (drains, upgrades).
- **Pod lifecycle & graceful shutdown:** SIGTERM → `preStop` hook → grace period (default 30s) → SIGKILL.
- **Container runtimes:** Docker (build) → containerd/CRI-O (run) → runc. **OCI** = open standard for images/runtimes.
- **CRI, CNI, CSI:** Container Runtime / Network / Storage Interfaces. Pluggable standards that make K8s extensible.
- **etcd & Raft:** Consensus algorithm; needs an odd number of members (3/5) for quorum.
- **Cluster API / IaC:** Provision clusters via Terraform / Cluster API.
- **Serverless on K8s:** Knative, KEDA.
- **Spot instances:** Use for stateless, fault-tolerant workloads (with PDBs and multiple instance types).

---

## Part 9: Interview Q&A Rapid Fire

**Q: What is Docker?**
A platform to package apps with dependencies into lightweight, portable containers.

**Q: Docker image vs container?**
Image is a read-only template; container is a running instance of it.

**Q: Why are containers lighter than VMs?**
They share the host OS kernel instead of running a full guest OS.

**Q: What is a Pod and why not just containers?**
Smallest K8s unit; groups tightly coupled containers sharing network and storage. K8s manages Pods so related containers (app + sidecar) schedule together.

**Q: Deployment vs StatefulSet?**
Deployment: stateless, interchangeable pods. StatefulSet: stable identity, ordered operations, per-pod persistent storage.

**Q: How do Services find Pods?**
Via **label selectors**, which populate the Service's **Endpoints/EndpointSlices**.

**Q: Liveness vs Readiness?**
Liveness failure → restart. Readiness failure → removed from load balancing, no restart.

**Q: What if etcd fails?**
Control plane can't change state (no new deployments/scaling); *existing running workloads continue* on nodes. Restore from snapshot.

**Q: What if the control plane goes down, do apps die?**
No. Running pods keep serving traffic. You lose management ability (scheduling, scaling, self-healing).

**Q: How does K8s do self-healing?**
Controllers reconcile desired vs actual state; kubelet restarts containers; probes detect failures; nodes' failures lead to rescheduling.

**Q: ClusterIP vs NodePort vs LoadBalancer?**
Internal only / exposes on each node's port / cloud load balancer public IP. Each builds on the previous.

**Q: How do you secure a cluster?**
RBAC least privilege, network policies, Pod Security Standards, secrets management, image scanning, private API server, audit logs, runtime security, regular patching.

**Q: How do you handle config across environments?**
ConfigMaps/Secrets + Helm values or Kustomize overlays + GitOps.

**Q: Taints vs Node Affinity?**
Taints *repel* pods from nodes (node-side). Affinity *attracts* pods to nodes (pod-side). Often used together for dedicated node pools.

**Q: What is a sidecar?**
A helper container in the same pod (logging, proxy, security) extending main app without code changes.

**Q: Kubernetes vs Docker Swarm vs Nomad?**
K8s: most powerful, huge ecosystem, steepest learning curve. Swarm: simple, limited. Nomad: simple, flexible (non-container workloads too).

**Q: Is Kubernetes secure by default?**
No. Defaults are permissive (all pods can talk to each other, secrets are just base64, etc.). Must be hardened.

**Q: How would you design a CI/CD pipeline for K8s?**
Code push → build & unit test → build Docker image → scan → push to registry → update manifest/Helm values in Git → Argo CD syncs to cluster → canary/rolling deploy → automated smoke tests/metrics analysis → promote or rollback.

**Q: How do you migrate a monolith to Kubernetes?**
Containerize as-is first ("lift and shift") → externalize config/state → add health endpoints, graceful shutdown → deploy to K8s → gradually strangle into microservices using the Strangler Fig pattern. Don't do big-bang rewrites.

**Q: What are the 12-Factor App principles' relevance?**
Config in env, stateless processes, logs as streams, disposability, dev/prod parity: the foundation of container-friendly apps.

---

## Part 10: Architect's Decision Framework (Say This in Interviews)

When asked *"Would you use Kubernetes for X?"*, answer with trade-offs:

1. **Scale & complexity**: how many services, how much traffic variance?
2. **Team capability**: do we have platform/SRE skills?
3. **Cost**: TCO including operations, not just infra
4. **Availability requirements**: SLAs/SLOs?
5. **Portability needs**: multi-cloud/vendor lock-in?
6. **Alternatives**: PaaS (App Engine, Cloud Run, Beanstalk), serverless, ECS
7. **Managed vs self-hosted**
8. **Security & compliance**
9. **Migration path & risk**

> **Strong closing line:** *"I'd start with the simplest thing that meets requirements, adopt Kubernetes when scale, team, and complexity justify it, use a managed offering, and invest early in observability, security, and GitOps."*

---

## Part 11: Last-Minute Memory Hooks 🧠

```
DOCKER   → Build once, run anywhere        (Dockerfile → Image → Container)
K8s      → Declare what you want           (Desired state → Reconcile)
POD      → Smallest unit, disposable       (Cattle, not pets)
DEPLOY   → Stateless, rolling updates
STATEFUL → Identity + storage
SERVICE  → Stable address for moving pods  (labels → endpoints)
INGRESS  → HTTP front door
CONFIGMAP/SECRET → Config out of images
PROBES   → Liveness restarts; Readiness gates traffic
HPA      → More pods | CA → More nodes
REQUESTS → For scheduling | LIMITS → For enforcing
ETCD     → Brain's memory (BACK IT UP!)
GITOPS   → Git is truth
```

**Debug mantra:** `get → describe → logs → events → exec`

**The 10-year-old summary:**
> Docker puts your toy in a box so it works the same everywhere. Kubernetes is the toy-store manager who makes sure there are always enough boxes on the shelves, replaces broken ones, and opens more counters when lots of kids show up.

---

Good luck with the interviews, Shubham! With your Java, GCP, and architecture background, lean into real production stories (JVM memory in containers, GKE, zero-downtime deployments) to stand out. 🚀

If you want, I can follow this up with a **mock interview round** (scenario-based questions with model answers) or a **Kubernetes YAML cheat-sheet** next.