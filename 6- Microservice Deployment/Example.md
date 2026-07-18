# Microservices Deployment — E-commerce App Example

This README walks through a concrete example — deploying an e-commerce app's microservices — tying together Virtual Machines, Containers, Load Balancers, Docker, and Kubernetes from the main deployment README into one real scenario.

---

## The App

A typical e-commerce app split into microservices:

- **Order Service**
- **Payment Service**
- **Inventory Service**
- **User Service**
- **Notification Service**

Each service needs to be deployed, scaled independently, and stay available even as traffic fluctuates (e.g., a flash sale).

---

## Step 1 — Why not deploy each service on its own VM?

A naive approach: spin up one VM per service, install its dependencies, and run it there.

```
┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
│  VM: Order  │  │ VM: Payment │  │VM: Inventory│  │  VM: User    │
│  Full OS      │  │  Full OS      │  │  Full OS      │  │  Full OS      │
│  + App        │  │  + App        │  │  + App        │  │  + App        │
└───────────┘  └───────────┘  └───────────┘  └───────────┘
```

**Problem:** During a flash sale, Order Service traffic might spike 10x while Payment and Inventory stay roughly steady. To handle the spike, you'd need to provision *new* VMs for Order Service — but each new VM takes minutes to boot (full OS installation + dependency setup). By the time the new VM is ready, the sale's peak traffic may already have passed. Slow, resource-heavy, and a poor match for the kind of quick, elastic scaling this system needs.

---

## Step 2 — Containerizing each service with Docker

Instead, each service is packaged as a lightweight Docker image:

```
Dockerfile (Order Service):
  FROM node:20
  COPY . /app
  RUN npm install
  CMD ["npm", "start"]

docker build -t order-service:1.0 .
docker build -t payment-service:1.0 .
docker build -t inventory-service:1.0 .
docker build -t user-service:1.0 .
docker build -t notification-service:1.0 .
```

Each image is pushed to a container registry (Docker Hub, AWS ECR, etc.). Now, starting a new instance of Order Service takes **milliseconds to a couple of seconds** — no OS boot involved, since containers share the host's kernel.

---

## Step 3 — Orchestrating with Kubernetes

Docker alone runs containers on a single machine — it doesn't handle scaling across multiple machines, restarting failed containers, or routing traffic. Kubernetes takes over from here.

```
Kubernetes Deployment: order-service
  replicas: 3
  image: order-service:1.0

Kubernetes Deployment: payment-service
  replicas: 2
  image: payment-service:1.0

Kubernetes Deployment: inventory-service
  replicas: 2
  image: inventory-service:1.0
```

Kubernetes schedules these Pods across the cluster's Nodes, and continuously ensures the actual running Pod count matches the desired replica count — restarting any Pod that crashes.

---

## Step 4 — Load balancing traffic across instances

Each service has multiple replicas, so a **Kubernetes Service** is created for each — automatically load balancing requests across all healthy Pods:

```
                          ┌─────────────────┐
   Requests for /orders ──▶│ Service: order-svc │
                          └─────────────────┘
                             │       │       │
                             ▼       ▼       ▼
                        Order Pod 1  Order Pod 2  Order Pod 3
```

Readiness probes ensure that if Order Pod 2 is temporarily unhealthy (e.g., still starting up, or its dependency isn't ready), the Service automatically stops routing traffic to it — no manual intervention needed.

An **Ingress** (Layer 7 load balancer) sits at the edge of the cluster, routing external traffic based on the URL path:

```
                       ┌──────────┐
   Client Request ────▶│  Ingress   │
                       └──────────┘
                       /orders   → order-svc
                       /payments → payment-svc
                       /inventory → inventory-svc
                       /users     → user-svc
```

---

## Step 5 — Autoscaling during the flash sale

A **Horizontal Pod Autoscaler (HPA)** is configured for Order Service to watch CPU/request-rate metrics:

```
HPA: order-service
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilization: 70%
```

When the flash sale hits and Order Service's CPU usage crosses 70%, Kubernetes automatically spins up additional Order Service Pods — within seconds, not minutes, because containers start almost instantly. As traffic drops after the sale, Kubernetes scales back down to save resources.

Meanwhile, Payment Service and Inventory Service — which didn't see the same spike — stay at their normal replica count. This is the **scaling mismatch problem** solved directly: each service scales independently based on its own load, something a shared VM-per-app-tier model couldn't do nearly as efficiently.

---

## End-to-End Flow During the Flash Sale

```
1. Thousands of customers hit "Buy Now" simultaneously
        │
        ▼
2. Ingress routes /orders traffic to the order-svc Service
        │
        ▼
3. order-svc load balances across Order Service Pods
        │
        ▼
4. HPA detects rising CPU usage, scales Order Service
   from 3 Pods → 15 Pods within seconds
        │
        ▼
5. Order Service calls Payment Service and Inventory Service
   (steady replica count — no spike on their end)
        │
        ▼
6. If any Order Pod crashes under load, Kubernetes
   automatically restarts it — self-healing, no downtime
        │
        ▼
7. After the sale ends, HPA scales Order Service back
   down to 3 Pods, freeing up cluster resources
```

---

## Why This Matters

This example shows exactly why Docker + Kubernetes replaced VM-per-service deployment for microservices:

| Requirement during the flash sale | How it's met |
|---|---|
| Scale Order Service fast without touching other services | Independent Kubernetes Deployments + HPA per service |
| New instances ready in seconds, not minutes | Containers (no OS boot) instead of VMs |
| Traffic spread evenly across all instances | Kubernetes Service (internal load balancing) |
| External traffic routed to the right service | Ingress (Layer 7 load balancer) |
| Failed instances recovered automatically | Kubernetes self-healing via liveness/readiness probes |
| Resources freed up once the sale ends | HPA scaling back down automatically |

A VM-based deployment could technically achieve some of this too — but far slower, at much higher resource cost, and with far more manual operational overhead.
