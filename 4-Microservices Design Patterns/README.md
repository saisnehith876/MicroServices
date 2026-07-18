# Microservices Design Patterns — In Depth

Splitting a monolith into microservices solves organizational and scaling problems, but introduces a new class of distributed-systems challenges: how do services find each other? How do they survive partial failures? How is a transaction managed across separate databases? This README covers the proven patterns that address these problems.

---

## 1. Service Discovery & Registration Pattern

### 1.1 The problem it solves

In a monolith, one module calls another via a simple in-process method call — no networking involved. In microservices, Service A needs to call Service B **over the network**, which means it needs to know B's IP address and port.

This sounds trivial until you consider a real deployment:

- Services run as multiple **instances** for scaling (e.g., 5 replicas of Order Service)
- Instances are **ephemeral** — containers get killed and rescheduled by Kubernetes, IPs change constantly
- Instances **scale up/down** dynamically based on load (autoscaling)
- Hardcoding IPs anywhere is not just fragile — it's actively impossible to maintain at scale

Manually tracking and updating IP addresses for every service, every instance, every deployment is unworkable. **Service Discovery** solves this by letting services find each other dynamically at runtime.

### 1.2 How it works — two core pieces

**Service Registry**
A centralized (or distributed) database that keeps track of all currently running service instances and their network locations (IP + port). Every instance registers itself here on startup and deregisters (or is removed via health check failure) on shutdown.

**Service Discovery**
The mechanism a client uses to *look up* a healthy instance from the registry when it needs to call another service.

### 1.3 Registration patterns

**Self-Registration**
The service instance itself registers with the registry on startup and sends periodic heartbeats to prove it's alive. If heartbeats stop, the registry removes it.

```
Order Service instance starts
        │
        ▼
Registers itself with Registry: "OrderService @ 10.0.1.5:8080"
        │
        ▼
Sends heartbeat every N seconds
```

- Simple to implement
- Couples the service to the registry's client library

**Third-Party Registration**
An external agent (often a sidecar or orchestration platform) handles registration on the service's behalf — the service itself has no knowledge of the registry.

- Example: Kubernetes does this automatically — when a Pod becomes "Ready" (passes its readiness probe), it's automatically added to the Service's endpoint list. The application code doesn't need any registry-specific logic at all.
- Cleaner separation of concerns — the service just runs; the platform manages discovery

### 1.4 Discovery patterns — how a caller finds an instance

**Client-Side Discovery**
The calling service queries the registry directly, gets a list of healthy instances, and picks one itself (often using a load-balancing strategy like round robin).

```
Order Service ──▶ queries Registry ──▶ gets [10.0.1.5, 10.0.1.6, 10.0.1.7]
Order Service ──▶ picks one (e.g., round robin) ──▶ calls 10.0.1.6 directly
```

- Tools: **Netflix Eureka** (with Ribbon for client-side load balancing), **Consul**
- Pro: no extra network hop
- Con: discovery logic is baked into every client (in every language your services use)

**Server-Side Discovery**
The calling service simply sends the request to a well-known load balancer/router, which handles the registry lookup and routing internally. The caller doesn't need to know about the registry at all.

```
Order Service ──▶ calls http://payment-service/  ──▶  Load Balancer / Kube-proxy
                                                              │
                                                              ▼
                                                  looks up registry, routes to
                                                     a healthy Payment instance
```

- Tools: **Kubernetes Services** (kube-proxy + internal DNS), **AWS ELB/ALB**, **Consul + Envoy**
- Pro: client code stays simple — just call a stable, friendly hostname
- Con: an extra network hop through the load balancer

### 1.5 Popular tools

| Tool | Notes |
|---|---|
| **Netflix Eureka** | Classic Java/Spring Cloud service registry, client-side discovery |
| **Consul (HashiCorp)** | Service registry + health checking + key-value store + supports both discovery styles |
| **Kubernetes (built-in)** | Services + kube-proxy + internal DNS provide server-side discovery automatically — most common in modern cloud-native stacks |
| **Apache Zookeeper** | Older, general-purpose coordination service also used for service discovery |
| **etcd** | Distributed key-value store, used internally by Kubernetes itself for cluster state |

### 1.6 Health checks — the piece that makes this actually work

A registry is only useful if it doesn't route traffic to dead instances. This requires continuous health checking:

- **Liveness check** — "Is the process still running?" If it fails repeatedly, the instance is restarted.
- **Readiness check** — "Is the instance ready to accept traffic?" (e.g., has it finished loading a local cache, connected to its database?) An instance that's alive but not ready is removed from the routing pool temporarily.

Without health checks, service discovery would just be a static list of IPs — the "dynamic" part comes entirely from continuously verifying which instances are actually usable.

### 1.7 In modern practice

If you're running on **Kubernetes**, you largely get service discovery "for free" — every Service gets a stable internal DNS name (`payment-service.namespace.svc.cluster.local`), and kube-proxy handles routing to healthy Pod IPs behind it. You don't need to run Eureka or Consul separately unless you have multi-cluster or non-Kubernetes environments to bridge.

---

## 2. Resilience Patterns

Network calls between services **will fail** — a downstream service might be slow, overloaded, or completely down. Resilience patterns exist to stop one failure from cascading into a full system outage.

### 2.1 Circuit Breaker Pattern

**Problem:** If Service B is down, Service A keeps calling it anyway — every failed call wastes time (waiting for timeouts) and resources, and if A has many callers, the failure ripples upward.

**How it works:** A Circuit Breaker wraps outgoing calls and tracks failures. It has three states:

```
   CLOSED ──(failure threshold exceeded)──▶ OPEN
     ▲                                        │
     │                                (after timeout period)
     │                                        ▼
     └──(test call succeeds)────────── HALF-OPEN
                                     (test call fails ──▶ back to OPEN)
```

- **Closed** — normal operation, requests pass through. Failures are counted.
- **Open** — too many failures detected; the breaker "trips" and **immediately rejects** further calls without even attempting them, for a cooldown period. This gives the failing service time to recover and stops wasting caller resources.
- **Half-Open** — after the cooldown, a limited number of test requests are allowed through. If they succeed, the breaker closes (resumes normal traffic). If they fail, it reopens.

**Tools:** Resilience4j (Java), Hystrix (legacy, Netflix), Polly (.NET)

### 2.2 Retry Pattern

**Problem:** Some failures are transient — a brief network blip, a momentary spike in load. Failing immediately on the first error is often overly aggressive.

**How it works:** Automatically re-attempt a failed request, usually with:
- **Exponential backoff** — wait longer between each retry (e.g., 1s, 2s, 4s, 8s) to avoid hammering an already-struggling service
- **Jitter** — add randomness to backoff timing so many clients don't retry in synchronized bursts
- **Max retry limit** — stop after N attempts and surface the failure

**Important caveat:** Retry pattern is dangerous *without* Circuit Breaker — retrying against a service that's completely down just multiplies load and delays failure detection. In practice, Retry and Circuit Breaker are almost always used **together**.

### 2.3 Bulkhead Pattern

**Problem:** If one downstream dependency is slow, it can exhaust a shared resource pool (e.g., a thread pool or connection pool), starving *unrelated* calls to healthy services. One bad dependency takes down everything.

**How it works:** Named after ship design — a ship's hull is divided into isolated watertight compartments (bulkheads), so a leak in one compartment doesn't sink the whole ship.

Applied to microservices: **allocate separate, isolated resource pools** (thread pools, connection pools) per downstream dependency.

```
Without Bulkhead:                      With Bulkhead:
┌─────────────────────┐               ┌───────┐ ┌───────┐ ┌───────┐
│  Shared thread pool   │              │ Pool A │ │ Pool B │ │ Pool C │
│  (20 threads)         │              │ (7)    │ │ (7)    │ │ (6)    │
│                        │              └───────┘ └───────┘ └───────┘
│ Slow Service C call     │                  │         │         │
│ exhausts all 20 threads │              Service A  Service B  Service C
│ → blocks calls to A, B  │              (isolated)  (isolated) (isolated,
└─────────────────────┘                                          exhausted,
                                                                doesn't affect A/B)
```

### 2.4 Rate Limiter Pattern

**Problem:** A service can be overwhelmed by too many requests — whether from a traffic spike, a misbehaving client, or an intentional abuse/DDoS attempt.

**How it works:** Cap the number of requests a service (or a specific client/API key) is allowed to make in a given time window. Requests beyond the limit are rejected (typically with HTTP `429 Too Many Requests`) or queued.

Common algorithms:
- **Token Bucket** — a bucket holds tokens, refilled at a fixed rate; each request consumes a token, request is rejected if the bucket is empty
- **Leaky Bucket** — requests are processed at a fixed steady rate, excess requests overflow/queue
- **Fixed/Sliding Window Counter** — count requests within a rolling time window

**Where it's applied:** Often enforced at the **API Gateway** level (see below) rather than in every individual service, so the rule is centralized and consistent.

---

## 3. Gateway Pattern

### 3.1 The problem it solves

Without a gateway, every client (web app, mobile app, third-party integrator) would need to know the address of every individual microservice, and every microservice would need to independently implement authentication, logging, and tracing. That's duplicated logic across dozens of services, and it exposes internal architecture directly to external clients.

### 3.2 How it works

An **API Gateway** sits between clients and the microservices, acting as a single entry point:

```
                          ┌─────────────────┐
   Client (Web/Mobile) ──▶│   API Gateway     │
                          │  - Auth/Security   │
                          │  - Rate Limiting   │
                          │  - Logging/Tracing │
                          │  - Request Routing │
                          └─────────────────┘
                             │      │      │
                             ▼      ▼      ▼
                        Order   Payment  Inventory
                        Service Service   Service
```

**Cross-cutting concerns centralized in the gateway:**
- **Authentication/Authorization** — validate JWT tokens/API keys once, at the edge, instead of in every service
- **Logging & Tracing** — attach a correlation/trace ID to every incoming request so it can be followed across all downstream services (critical for observability — this is where tools like Dynatrace's PurePath tracing hooks in)
- **Rate Limiting** — enforce request quotas centrally
- **Request Routing** — route `/orders/*` to Order Service, `/payments/*` to Payment Service, etc.
- **Request Aggregation** — combine multiple backend calls into a single response for the client (avoids the client making 5 separate calls)
- **Protocol Translation** — e.g., expose REST/JSON externally while internal services communicate via gRPC

### 3.3 Related pattern — Backend for Frontend (BFF)

A variation where each type of client (web, mobile, third-party) gets its **own** tailored gateway, since a mobile app typically needs a leaner, more optimized response shape than a web dashboard.

### 3.4 Popular tools

| Tool | Notes |
|---|---|
| **Kong** | Popular open-source API Gateway, plugin-based |
| **NGINX** | Can act as a lightweight gateway/reverse proxy |
| **AWS API Gateway** | Fully managed, integrates natively with AWS services (Lambda, etc.) |
| **Spring Cloud Gateway** | Java/Spring ecosystem native option |
| **Istio (Service Mesh)** | Handles gateway-like concerns (and more) at the infrastructure/network layer rather than application layer |

---

## 4. Saga Pattern

### 4.1 The problem it solves

In a monolith, a multi-step business operation (e.g., "place an order") can be wrapped in a single **ACID database transaction** — if any step fails, the whole thing rolls back cleanly.

In microservices, Order, Payment, and Inventory each have **separate databases**. There is no way to wrap a transaction across all three — distributed ACID transactions (like two-phase commit) don't scale well and go against the whole point of service independence.

The **Saga pattern** solves this by breaking the operation into a sequence of **local transactions**, each committed independently within its own service, with **compensating actions** defined to undo previous steps if a later step fails.

### 4.2 Example — "Place an Order" saga

```
1. Order Service:    create order (status: PENDING)         ✅
2. Payment Service:  charge customer                          ✅
3. Inventory Service: reserve stock                            ❌ FAILS (out of stock)

Compensating actions run in reverse:
3'. Payment Service:  refund customer               (compensates step 2)
2'. Order Service:    mark order as CANCELLED        (compensates step 1)
```

Each step is a real, committed local transaction — there's no global rollback. Instead, failure triggers a **chain of compensating transactions** that undo the effects of prior successful steps.

### 4.3 Two implementation styles

**Choreography-based Saga**
Each service publishes events, and other services react to them independently — no central coordinator.

```
Order Service ──publishes──▶ OrderCreated
                                   │
Payment Service (listens) ──▶ charges customer ──publishes──▶ PaymentCompleted
                                   │
Inventory Service (listens) ──▶ reserves stock ──publishes──▶ StockReserved
```

- Pro: fully decoupled, no single point of failure
- Con: as the number of steps grows, the event chain becomes hard to trace and reason about ("who reacts to what" is scattered across services)

**Orchestration-based Saga**
A central **Saga Orchestrator** explicitly tells each service what to do next, and handles compensation logic if a step fails.

```
                    ┌────────────────────┐
                    │  Saga Orchestrator   │
                    └────────────────────┘
                       │        │        │
                       ▼        ▼        ▼
                    Order    Payment   Inventory
                   Service   Service    Service
```

- Pro: the entire workflow logic lives in one place — easier to understand, monitor, and modify
- Con: the orchestrator becomes a critical component and a potential bottleneck if not designed carefully

### 4.4 When to use which
- **Choreography** — simple sagas with only 2-3 steps, where you want maximum decoupling
- **Orchestration** — complex, multi-step sagas where visibility and centralized error handling matter more than pure decoupling (most real-world e-commerce checkout flows use this)

---

## 5. Other Supporting Patterns

### 5.1 Load Balancing
Distributes incoming requests across multiple instances of a service so no single instance is overwhelmed. Works hand-in-hand with Service Discovery — once you know the healthy instances, a load balancing algorithm (round robin, least connections, weighted, etc.) decides which one gets the next request. Can happen client-side (with the discovery registry) or server-side (via a dedicated load balancer/gateway).

### 5.2 Domain-Driven Design (DDD)
Provides the methodology for correctly defining service boundaries in the first place — Bounded Contexts, Ubiquitous Language, and Context Mapping (Shared Kernel, Anti-Corruption Layer, etc.). Without well-defined boundaries, none of the other patterns matter much — you'll just have a poorly-split distributed monolith no matter how much resilience tooling you bolt on. *(See the dedicated DDD README for the full breakdown.)*

### 5.3 CQRS (Command Query Responsibility Segregation)
Separates the **write model** (commands — create, update, delete) from the **read model** (queries), often using entirely different data models or even different databases for each.

```
Write side:                          Read side:
Order Service ──writes──▶ Orders DB     Orders DB ──replicated/projected──▶ Read-optimized
(normalized, transactional)                                                  view (denormalized,
                                                                               fast queries)
```

- Useful when read and write workloads have very different scaling needs (e.g., a product catalog is read millions of times per write)
- Often paired with **Event Sourcing**, where the write side stores a log of events (not just current state), and the read side is a materialized view built by replaying those events

### 5.4 Event-Driven Architecture (EDA)
Services communicate by publishing and reacting to **events** rather than calling each other directly. This underlies both the Choreography-based Saga and Kafka-based communication covered in the communication README — it's the broader architectural philosophy that those specific patterns are built on. The core benefit: producers and consumers don't need to know about each other at all, only about the shape of the event.

---

## 6. How These Patterns Fit Together

None of these patterns are used in isolation — a real e-commerce checkout flow typically touches nearly all of them:

```
1. Client calls API Gateway (auth, rate limiting, tracing applied here)
        │
2. Gateway routes to Order Service, discovered via Service Registry
        │
3. Order Service calls Inventory Service — wrapped in a Circuit Breaker
   with Retry, isolated in its own Bulkhead thread pool
        │
4. Order Service kicks off a Saga (Orchestration-based) to coordinate
   Order → Payment → Inventory across their separate databases
        │
5. Each service instance is load balanced across its replicas
        │
6. If a step fails, compensating transactions run automatically
```

---

## 7. Quick Reference

| Pattern | Solves | Typical Tooling |
|---|---|---|
| Service Discovery & Registration | Finding healthy service instances dynamically | Kubernetes DNS, Consul, Eureka |
| Circuit Breaker | Preventing cascading failures from a downed dependency | Resilience4j, Hystrix |
| Retry | Recovering from transient/temporary failures | Resilience4j, Polly |
| Bulkhead | Isolating failures so they don't exhaust shared resources | Resilience4j (thread pool isolation) |
| Rate Limiter | Preventing overload from excessive requests | API Gateway plugins, Resilience4j |
| Gateway | Centralizing auth, logging, tracing, routing | Kong, NGINX, AWS API Gateway, Spring Cloud Gateway |
| Saga | Managing transactions across separate databases | Kafka/RabbitMQ (events) + custom orchestrator, or frameworks like Camunda |
| Load Balancing | Distributing traffic across service instances | Kubernetes Services, NGINX, cloud LBs |
| DDD | Defining clean, meaningful service boundaries | Modeling technique, not a tool |
| CQRS | Scaling reads and writes independently | Custom implementation, often with event sourcing |
| Event-Driven Architecture | Decoupled service communication | Kafka, RabbitMQ |
