# Microservices Deployment — VMs, Containers, Load Balancers, Docker & Kubernetes

Deployment is the phase where a lot of microservices' theoretical benefits either get realized or quietly get lost. This README covers the foundational infrastructure concepts — Virtual Machines, Containers, Load Balancers — and how Docker and Kubernetes come together to make microservices deployment actually practical.

---

## 1. What is a Virtual Machine (VM)?

### 1.1 The core idea
A **Virtual Machine** is a complete, isolated emulation of a physical computer — including its own full **Operating System** — running on top of a physical host machine via a layer called a **Hypervisor**.

```
┌───────────────────────────────────────────┐
│              Physical Host Machine            │
│  ┌──────────────────────────────────────┐  │
│  │            Hypervisor                    │  │
│  │  ┌───────────┐  ┌───────────┐  ┌───────┐│  │
│  │  │   VM 1      │  │   VM 2      │  │  VM 3 ││  │
│  │  │ Guest OS    │  │ Guest OS    │  │Guest  ││  │
│  │  │ + App        │  │ + App        │  │OS+App ││  │
│  │  └───────────┘  └───────────┘  └───────┘│  │
│  └──────────────────────────────────────┘  │
│              Host Operating System              │
│              Physical Hardware (CPU/RAM)        │
└───────────────────────────────────────────┘
```

### 1.2 Hypervisor types
- **Type 1 (bare-metal)** — runs directly on the physical hardware, no host OS underneath (e.g., VMware ESXi, Microsoft Hyper-V, KVM). Used in production data centers/cloud providers.
- **Type 2 (hosted)** — runs as an application on top of an existing host OS (e.g., VirtualBox, VMware Workstation). Common for local development/testing.

### 1.3 Why creating a VM is a tedious, manual process
Setting up a VM traditionally involves several manual steps:

1. **Resource allocation** — deciding and reserving CPU, memory (RAM), and disk space upfront
2. **OS installation** — installing a full guest operating system (Windows, Ubuntu, CentOS, etc.) from scratch
3. **Library/dependency installation** — manually installing runtime dependencies, patches, and application libraries on top of that fresh OS

Each of these steps takes real time — provisioning a new VM can take minutes to tens of minutes even when partially automated, and much longer when done manually.

### 1.4 Why VMs are slow to bootstrap and shut down
Since every VM carries a **complete OS** (its own kernel, its own drivers, its own init system), starting a VM means booting an entire operating system — the same process as turning on a physical computer. This typically takes **30 seconds to a few minutes**. Shutdown is similarly slow, since the guest OS has to gracefully terminate all its own processes and services.

### 1.5 Why this is a problem for microservices
Microservices environments need to react to load changes **quickly** — spinning up new instances when traffic spikes, and shutting them down when it drops (autoscaling). A VM's slow boot/shutdown cycle makes it a poor fit for this kind of rapid, elastic scaling. By the time a new VM instance finishes booting, the traffic spike may already be over — or worse, users have already experienced degraded performance.

---

## 2. What are Containers?

### 2.1 The core idea
A **Container** packages an application and everything it needs to run (code, runtime, system libraries, configuration) — but **without a full guest OS**. Instead, containers share the **host machine's OS kernel** and are isolated from each other using OS-level features (namespaces and cgroups on Linux).

```
┌────────────────────────────────────────────┐
│               Physical / Virtual Host             │
│  ┌───────────────────────────────────────┐  │
│  │           Container Runtime (Docker)         │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  │  │
│  │  │Container 1│  │Container 2│  │Container 3│  │  │
│  │  │  App + libs │  │  App + libs │  │  App + libs │  │  │
│  │  │  (no OS)    │  │  (no OS)    │  │  (no OS)    │  │  │
│  │  └─────────┘  └─────────┘  └─────────┘  │  │
│  └───────────────────────────────────────┘  │
│              Shared Host Operating System           │
│              Physical Hardware (CPU/RAM)            │
└────────────────────────────────────────────┘
```

### 2.2 How isolation works without a full OS
Containers rely on Linux kernel features to achieve isolation while sharing one kernel:

- **Namespaces** — isolate what a container can *see*: its own process IDs (PID namespace), network interfaces (network namespace), filesystem mount points (mount namespace), hostname (UTS namespace), etc. From inside a container, it looks like it has its own machine, even though it's sharing the host kernel.
- **cgroups (control groups)** — limit and account for what a container can *use*: CPU quota, memory limit, disk I/O, network bandwidth. This prevents one container from starving others of resources.

*(Given your Linux internals background — process states, systemd, cgroups/namespaces — this is the exact same mechanism you've likely already studied, just applied specifically to container isolation.)*

### 2.3 Why containers are fast
Since a container doesn't boot a kernel — it just starts a process (or a small set of processes) within the isolation boundaries provided by namespaces/cgroups — starting a container is nearly as fast as starting a regular process. Typical container startup time is **milliseconds to a couple of seconds**, compared to a VM's 30+ seconds.

### 2.4 Container images
A container runs from an **image** — a lightweight, standalone, executable package that includes the application code, runtime, libraries, and settings. Images are built in **layers** (each instruction in a Dockerfile creates a new layer), and layers are cached and reused across builds/images, making image builds and transfers efficient.

---

## 3. Virtual Machine vs Container

| | Virtual Machine | Container |
|---|---|---|
| **Isolation unit** | Full OS (kernel + userland) | Process, isolated via namespaces/cgroups |
| **Kernel** | Each VM has its own kernel | Shares the host's kernel |
| **Boot time** | 30 seconds to minutes | Milliseconds to a few seconds |
| **Size** | Gigabytes (full OS image) | Megabytes (just app + dependencies) |
| **Resource overhead** | High — every VM runs a full OS's worth of background processes | Low — no duplicate OS overhead |
| **Density** | Fewer VMs per physical host | Many more containers per host |
| **Isolation strength** | Strong — full hardware-level virtualization boundary | Weaker than a VM — shares kernel, so a kernel-level exploit *could* theoretically cross container boundaries |
| **Portability** | Less portable — tied to hypervisor/OS specifics | Highly portable — "build once, run anywhere" via image + runtime |
| **Best for** | Running different OSes on one host, strong security/compliance isolation | Fast, elastic, high-density microservices deployment |

```
VM:                                Container:
┌─────────────────┐             ┌─────────────────┐
│  App                │             │  App                │
│  Libraries          │             │  Libraries          │
│  Guest OS (full)     │             │  (shares host kernel) │
│  Hypervisor          │             │  Container Runtime    │
│  Host OS             │             │  Host OS               │
│  Hardware            │             │  Hardware              │
└─────────────────┘             └─────────────────┘
   Gigabytes, slow boot            Megabytes, fast boot
```

### 3.1 Why this matters for microservices
Microservices need to scale **quickly and elastically** — spinning up new instances the moment load increases, and shutting them down the moment it drops. A VM's slow bootstrap makes this impractical. Containers solve exactly this problem: fast startup/shutdown, small footprint, and high density (many containers can run on the same hardware that would host far fewer VMs) — which is why **Docker and Kubernetes have become the default microservices deployment stack**, replacing VM-per-service deployment models.

---

## 4. Load Balancer

### 4.1 The problem it solves
Once a microservice is deployed as **multiple container instances** (for scaling and fault tolerance), something needs to decide which instance handles each incoming request. A **Load Balancer** sits in front of these instances and distributes incoming traffic across them.

```
                       ┌─────────────┐
   Client Requests ───▶│ Load Balancer │
                       └─────────────┘
                        │      │      │
                        ▼      ▼      ▼
                  Instance 1  Instance 2  Instance 3
                  (Order Svc)  (Order Svc)  (Order Svc)
```

### 4.2 Why microservices specifically need this
- Multiple replicas of the same service run simultaneously — traffic must be spread across them so no single instance is overwhelmed
- Instances come and go dynamically (scaling up/down, crashes, rolling deployments) — the load balancer needs to route only to **healthy** instances
- Without load balancing, you'd need clients to somehow know about every instance and manually pick one — completely unworkable at scale

### 4.3 Load balancing algorithms
| Algorithm | How it works |
|---|---|
| **Round Robin** | Requests distributed in rotation across instances (1, 2, 3, 1, 2, 3...) |
| **Least Connections** | Routes to whichever instance currently has the fewest active connections |
| **Weighted Round Robin** | Instances with more capacity get proportionally more traffic |
| **IP Hash / Consistent Hashing** | Same client consistently routed to the same instance — useful for session stickiness |
| **Least Response Time** | Routes to the instance with the lowest recent latency |

### 4.4 Health checks
A load balancer continuously checks each instance's health (via a `/healthz` endpoint, TCP connection check, etc.) and removes unhealthy instances from rotation — ensuring traffic is never routed to a dead or degraded instance. This ties directly into the **readiness/liveness probes** used in Kubernetes (covered below).

### 4.5 Layer 4 vs Layer 7 load balancing
- **Layer 4 (Transport layer)** — routes based on IP address and port only, doesn't inspect the request content. Faster, protocol-agnostic. Example: a plain TCP load balancer, AWS NLB.
- **Layer 7 (Application layer)** — inspects HTTP content (headers, path, hostname) to make routing decisions, e.g., routing `/orders/*` to Order Service and `/payments/*` to Payment Service on the same load balancer. Example: NGINX, AWS ALB, Kubernetes Ingress controllers.

### 4.6 Where load balancing happens in a microservices stack
- **External load balancer** — sits at the edge, distributing traffic from the internet across your cluster (e.g., AWS ALB/NLB, or a Kubernetes `LoadBalancer` type Service)
- **Internal load balancing** — happens automatically between microservices too. In Kubernetes, a `Service` object load-balances traffic across all healthy Pods backing it, using `kube-proxy`

---

## 5. Docker

### 5.1 What Docker is
Docker is the most widely used **container runtime and tooling platform** — it provides the tools to build container images, run containers, and manage their lifecycle on a single host.

### 5.2 Key Docker concepts

| Concept | What it is |
|---|---|
| **Dockerfile** | A text file with instructions to build a container image (base image, copy files, install dependencies, define startup command) |
| **Image** | The built, immutable artifact produced from a Dockerfile — a snapshot of the application + its environment |
| **Container** | A running instance of an image |
| **Docker Hub / Registry** | A place to store and share images (public or private) |
| **Docker Compose** | A tool for defining and running multi-container applications locally (e.g., an app + its database + a message broker, all started together) |

### 5.3 A simple example (conceptual)
```
Dockerfile:
  FROM node:20
  COPY . /app
  RUN npm install
  CMD ["npm", "start"]

Build:  docker build -t order-service:1.0 .
Run:    docker run -p 8080:8080 order-service:1.0
```

This produces a portable image that runs identically on a developer's laptop, a CI pipeline, or a production server — solving the classic "works on my machine" problem, since the entire runtime environment is packaged inside the image.

### 5.4 Why Docker alone isn't enough for production microservices
Docker is excellent for building and running **individual** containers, but production microservices need more:
- Running containers across **multiple machines**, not just one
- Automatically **restarting** failed containers
- **Scaling** the number of instances up/down based on load
- **Service discovery** and internal load balancing between services
- **Rolling updates** with zero downtime, and rollback if something breaks

This is exactly the gap **Kubernetes** fills.

---

## 6. Kubernetes

### 6.1 What Kubernetes is
Kubernetes (often abbreviated **K8s**) is a **container orchestration platform** — it manages containers (typically built with Docker) across a cluster of machines, automating deployment, scaling, healing, and networking.

### 6.2 Core Kubernetes concepts

| Concept | What it is |
|---|---|
| **Pod** | The smallest deployable unit in Kubernetes — wraps one or more tightly-coupled containers that share networking/storage |
| **Deployment** | Describes the desired state for a set of Pods (e.g., "run 5 replicas of order-service:1.0") — Kubernetes continuously works to match actual state to this desired state |
| **Service** | A stable network endpoint (with its own DNS name) that load-balances traffic across a set of Pods, even as individual Pods are replaced |
| **Ingress** | Manages external HTTP(S) access into the cluster, with host/path-based routing rules — essentially a Layer 7 load balancer/gateway for the cluster |
| **Node** | A physical or virtual machine that's part of the Kubernetes cluster and runs Pods |
| **Namespace** | A way to logically partition a cluster (e.g., separate `dev`, `staging`, `prod` environments) |
| **ConfigMap / Secret** | Externalized configuration and sensitive data, injected into Pods without baking them into the image |

### 6.3 Self-healing and autoscaling
- **Self-healing** — if a Pod crashes or fails its health check, Kubernetes automatically restarts or replaces it to match the desired replica count defined in the Deployment
- **Horizontal Pod Autoscaler (HPA)** — automatically increases or decreases the number of Pod replicas based on observed metrics (CPU usage, memory, or custom metrics like request rate)
- **Cluster Autoscaler** — can even add/remove entire Nodes (machines) from the cluster based on overall resource demand

### 6.4 Health checks in Kubernetes
- **Liveness probe** — "Is this container still alive?" If it fails repeatedly, Kubernetes restarts the container
- **Readiness probe** — "Is this container ready to receive traffic?" If not ready, the Pod is temporarily removed from the Service's load-balancing pool without being restarted

*(This is the same liveness/readiness concept from the Service Discovery README — Kubernetes is one of the most common ways this pattern is implemented in practice.)*

### 6.5 How Docker and Kubernetes work together
```
1. Developer writes a Dockerfile, builds an image        → Docker
2. Image is pushed to a registry (Docker Hub / ECR / GCR)   → Docker
3. Kubernetes Deployment references that image, specifies
   desired replica count                                    → Kubernetes
4. Kubernetes schedules Pods (running that image) across
   available Nodes in the cluster                            → Kubernetes
5. A Kubernetes Service load-balances traffic across
   those Pods, and re-routes automatically as Pods
   come and go                                                → Kubernetes
6. HPA watches load and scales replica count up/down          → Kubernetes
```

Docker handles **building and running a single container**. Kubernetes handles **orchestrating many containers across many machines** — scheduling, scaling, healing, networking, and load balancing, all automated.

---

## 7. Putting It All Together — Deployment Flow for a Microservice

```
1. Code is containerized using Docker (Dockerfile → Image)
        │
        ▼
2. Image pushed to a container registry
        │
        ▼
3. Kubernetes Deployment pulls the image, creates Pods
   across available Nodes in the cluster
        │
        ▼
4. Kubernetes Service load-balances traffic across
   all healthy Pods (readiness probes determine "healthy")
        │
        ▼
5. Ingress / external Load Balancer routes external
   traffic into the cluster, to the right Service
        │
        ▼
6. Horizontal Pod Autoscaler watches load and adjusts
   Pod replica count automatically
        │
        ▼
7. If a Pod crashes, Kubernetes automatically restarts
   or replaces it — self-healing, no manual intervention
```

---

## 8. Summary

- **Virtual Machines** run a full guest OS per instance — strong isolation, but slow to boot and heavy on resources. Not ideal for fast-scaling microservices.
- **Containers** share the host's kernel and use namespaces/cgroups for isolation — lightweight, fast to start, high density. The natural fit for microservices.
- **Load Balancers** distribute traffic across multiple service instances, using algorithms like round robin or least connections, and continuously check instance health.
- **Docker** builds and runs individual containers from portable, layered images.
- **Kubernetes** orchestrates containers across a cluster — handling scheduling, scaling (HPA), self-healing, service discovery, and load balancing (via Services and Ingress) automatically.
- Together, **Docker + Kubernetes** replace the traditional VM-per-service deployment model, enabling the fast, elastic, resilient deployment that microservices architectures actually need.
