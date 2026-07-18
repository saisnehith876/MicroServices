# Microservices Architecture — The Complete Picture

Every previous README in this repo covered one piece of the puzzle in isolation — communication, resilience patterns, deployment, database-per-service, failure handling. This README is the missing piece: **how all of those pieces fit together into one coherent architecture**, and what actually happens, component by component, when a single request flows through the system.

---

## 1. The Full Architecture — All Layers at Once

```
                             +-------------------+
                             |      Client         |
                             |   (Web / Mobile)     |
                             +-------------------+
                                       |
                                       v
                             +-------------------+
                             | Load Balancer /      |   <- Layer 4/7 entry point
                             |     Ingress           |
                             +-------------------+
                                       |
                                       v
                             +-------------------+
                             |    API Gateway        |   <- auth, rate limit,
                             | (auth/rate/logging)   |      logging, routing
                             +-------------------+
                                       |
                     +-----------------+-----------------+
                     |                 |                 |
                     v                 v                 v
             +-------------+   +-------------+   +-------------+
             | Order Service |   | Payment Svc  |   | Inventory Svc|
             | (N replicas)  |   | (N replicas)  |   | (N replicas) |
             +-------------+   +-------------+   +-------------+
                     |                 |                 |
                     v                 v                 v
             +-------------+   +-------------+   +-------------+
             |   Order DB    |   |  Payment DB   |   | Inventory DB |
             |  (Postgres)   |   |  (Postgres,    |   |   (Redis)     |
             |               |   |  PCI-isolated) |   |               |
             +-------------+   +-------------+   +-------------+
                     |                 |                 |
                     +-----------------+-----------------+
                                       |
                                       v
                             +-------------------+
                             |  Message Broker /     |   <- Kafka / RabbitMQ
                             |     Event Bus          |      async, decoupled
                             +-------------------+
                                       |
                     +-----------------+-----------------+
                     |                 |                 |
                     v                 v                 v
             +-------------+   +-------------+   +-------------+
             |Notification   |   |  Analytics    |   |  Search /     |
             |   Service      |   |   Service      |   | Recommend Svc |
             +-------------+   +-------------+   +-------------+

  =====================================================================
                     Cross-cutting infrastructure layer
  =====================================================================
   Service Discovery Registry  |  Observability (logs / metrics / traces)
   (Kubernetes DNS / Consul)   |  Config & Secrets  |  CI/CD Pipeline
```

This is the "everything at once" view. The rest of this README walks through **each layer**, then traces **one real request** through the entire stack from top to bottom.

---

## 2. Layer-by-Layer Breakdown

### 2.1 Client Layer
Web app, mobile app, or third-party API consumer. Doesn't know or care how many microservices exist behind the scenes — it only ever talks to one entry point.

### 2.2 Load Balancer / Ingress
The first stop inside your infrastructure. Distributes incoming traffic across multiple Gateway/cluster entry points and performs health checks so traffic never hits a dead node. In Kubernetes, this is typically a cloud Load Balancer (AWS ALB/NLB) feeding into an **Ingress controller**, which does path/host-based routing.

*(Full depth: see the Deployment README, Section 4.)*

### 2.3 API Gateway
The single, unified entry point into the microservices layer. Centralizes cross-cutting concerns so they don't need to be duplicated inside every service:
- Authentication/Authorization (validating JWTs/API keys once, at the edge)
- Rate limiting
- Logging and distributed trace ID injection (correlation ID for the whole request)
- Routing requests to the correct backend service based on path (`/orders/*` → Order Service)
- Sometimes: request aggregation (combining multiple backend calls into one client-facing response)

*(Full depth: see the Design Patterns README, Section 3.)*

### 2.4 Service Discovery
Behind the Gateway, services don't call each other using hardcoded IPs — they look each other up dynamically through a **Service Registry**, since instances are ephemeral (scaling, restarts, deployments constantly change IPs). In Kubernetes, this is handled automatically via internal DNS + `kube-proxy`.

*(Full depth: see the Design Patterns README, Section 1.)*

### 2.5 Individual Microservices (the business logic layer)
Each service — Order, Payment, Inventory, User, Notification — owns a **single, well-defined business capability**, following DDD's Bounded Context principle. Each runs as **multiple replicas** for scaling and fault tolerance, fronted by internal load balancing (a Kubernetes Service).

Each service wraps its outgoing calls to other services in resilience patterns — Timeout, Retry, Circuit Breaker, Bulkhead — so that one failing dependency doesn't cascade.

*(Full depth: see the DDD README and the Handling Failure README.)*

### 2.6 Database per Service
Each service owns its own database — no service reaches into another's data store directly. Different services can use different database technologies suited to their access patterns (Postgres for Order/Payment, Redis for Inventory, Elasticsearch for Search).

*(Full depth: see the Database per Service README.)*

### 2.7 Message Broker / Event Bus (Kafka / RabbitMQ)
The backbone for **asynchronous** communication. Services publish events (`OrderCreated`, `PaymentCompleted`) instead of calling each other directly for anything that doesn't need an immediate response. This is what powers:
- The **Saga pattern** for distributed transactions across services
- **Event-driven data replication**, letting services keep local read copies instead of making live cross-service calls
- Decoupled background processing (Notification, Analytics, Search indexing)

*(Full depth: see the Communication README.)*

### 2.8 Cross-Cutting Infrastructure Layer
Runs underneath everything and isn't tied to any one service:
- **Service Discovery Registry** — as above
- **Observability** — centralized logging, metrics, and distributed tracing (this is where a single request's journey across 5 services becomes visible as one trace — directly analogous to a Dynatrace PurePath)
- **Config & Secrets Management** — externalized configuration (Kubernetes ConfigMaps/Secrets) instead of baking config into images
- **CI/CD Pipeline** — each service is built, tested, and deployed **independently** (this is the actual payoff of microservices — teams ship without waiting on each other)

---

## 3. Tracing One Real Request Through the Entire Stack

Let's follow a single "place an order" request from click to confirmation, through **every layer** described above.

```
1. CLIENT
   Customer clicks "Place Order" on the web app
        │
        ▼
2. LOAD BALANCER / INGRESS
   Request hits the cluster's external Load Balancer,
   routed to a healthy Ingress controller node
        │
        ▼
3. API GATEWAY
   - Validates the customer's auth token
   - Attaches a trace/correlation ID to this request
   - Applies rate limiting (this customer is within limits)
   - Routes POST /orders → Order Service, via Service Discovery
        │
        ▼
4. SERVICE DISCOVERY
   Gateway asks the registry: "who are the healthy
   instances of order-service right now?"
   → gets back 3 healthy Pod IPs, picks one
        │
        ▼
5. ORDER SERVICE (instance #2, one of 3 replicas)
   - Validates the request payload
   - Writes a new order to Order DB (Postgres),
     status = PENDING
   - Needs current price → calls Pricing Service
     synchronously (wrapped in Timeout + Circuit Breaker)
        │
        ▼
6. PRICING SERVICE
   Returns current price via REST/gRPC call
        │
        ▼
7. ORDER SERVICE
   - Publishes "OrderCreated" event to Kafka topic
     (order-events), does NOT wait for downstream
     services — this part is asynchronous
   - Immediately returns "Order received" response
     to the customer (fast response, not blocked on
     the rest of the flow)
        │
        ▼
8. KAFKA (order-events topic)
   Delivers the "OrderCreated" event independently
   to every subscribed consumer group:
        │
        ├──▶ PAYMENT SERVICE
        │       - Charges the customer (idempotency key
        │         attached, in case of retries)
        │       - Writes to Payment DB
        │       - Publishes "PaymentCompleted"
        │
        ├──▶ INVENTORY SERVICE
        │       - Reserves stock in Inventory DB (Redis)
        │       - Publishes "StockReserved"
        │
        └──▶ ANALYTICS SERVICE
                - Logs the event for reporting/dashboards
                  (fire-and-forget, no impact on order flow)
        │
        ▼
9. ORDER SERVICE (subscribed to PaymentCompleted +
   StockReserved — this is the Saga coordination)
   - Once both events are received, updates the order
     status = CONFIRMED in its own DB
        │
        ▼
10. NOTIFICATION SERVICE (subscribed to OrderConfirmed)
    - Sends confirmation email/SMS to the customer
        │
        ▼
11. OBSERVABILITY (running throughout, in parallel)
    - Every hop above (Gateway → Order → Pricing →
      Kafka → Payment/Inventory/Notification) was tagged
      with the same trace ID from step 3
    - A distributed trace now shows the ENTIRE journey
      as one connected timeline — which hop was slow,
      which service failed, if any
```

### What if something fails mid-flow?
If Inventory Service fails to reserve stock at step 8 (out of stock), it publishes `StockReservationFailed` instead of `StockReserved`. Order Service (still listening) triggers the **Saga's compensating transaction**: tells Payment Service to refund, and marks the order `CANCELLED` — exactly as detailed in the Database per Service and Handling Failure READMEs.

If Pricing Service is slow at step 5, the **Circuit Breaker** wrapping that call trips after repeated failures, and Order Service can either retry, or fail fast and tell the customer to try again — rather than hanging indefinitely.

---

## 4. Synchronous vs Asynchronous — Where Each Is Used in This Flow

| Step | Style | Why |
|---|---|---|
| Client → Gateway → Order Service | Synchronous (REST) | Customer is actively waiting for a response |
| Order Service → Pricing Service | Synchronous (gRPC) | Order needs the price *right now* to proceed |
| Order Service → Kafka (`OrderCreated`) | Asynchronous | Customer doesn't need to wait for Payment/Inventory/Notification to finish — order confirmation can happen in the background |
| Payment/Inventory/Notification → Kafka | Asynchronous | None of these services need to block the customer's response |

This mixed model — **synchronous where an immediate answer is required, asynchronous everywhere else** — is the practical reality of almost every production microservices system, and it's why the Communication README treats these as complementary, not competing, choices.

---

## 5. How Each Design Pattern Maps to a Layer in the Architecture

| Layer / Concern | Pattern(s) applied |
|---|---|
| Entry point | Load Balancer, API Gateway |
| Finding service instances | Service Discovery & Registration |
| Service-to-service calls | Timeout, Retry, Circuit Breaker, Bulkhead |
| Protecting from overload | Rate Limiter |
| Cross-service transactions | Saga (choreography or orchestration) |
| Cross-service reads | API Composition, Event-Driven Replication (CQRS-ish) |
| Data ownership | Database per Service |
| Decoupled communication | Kafka / RabbitMQ, Event-Driven Architecture |
| Service boundaries | Domain-Driven Design (Bounded Contexts) |
| Deployment & scaling | Docker (containers) + Kubernetes (orchestration, HPA) |
| Failure containment | Fallback/Graceful Degradation, Dead Letter Queue, Idempotency |
| Visibility into the system | Distributed Tracing, Metrics, Logging (Observability) |

---

## 6. Why This Full Picture Matters (Especially for Interviews)

Reciting design patterns in isolation ("I know Circuit Breaker, I know Saga...") is far weaker than being able to explain **where each pattern sits in the request flow and why it's there**. The strongest answer to "walk me through microservices architecture" is exactly what Section 3 does above: start from the client, go layer by layer, and narrate what happens — including what happens when something fails.

A good structure for that kind of answer:
1. Entry point (Load Balancer → Gateway) and what it centralizes
2. How services find each other (Service Discovery)
3. How a service protects itself when calling another (resilience patterns)
4. How data ownership works (Database per Service) and how cross-service reads/writes are handled (API Composition / Events / Saga)
5. Where synchronous vs asynchronous communication is used, and why
6. How it's deployed and scaled (Docker/Kubernetes)
7. How you'd actually know if something broke (Observability)

---

## 7. Summary

Microservices architecture isn't any single pattern — it's the composition of many purpose-built pieces, each solving one specific problem that arises from distributing a system across the network:

- **Load Balancer + Gateway** get a request into the system safely and efficiently
- **Service Discovery** lets services find each other despite constant IP churn
- **Resilience patterns** stop one failure from becoming everyone's failure
- **Database per Service** gives each service full ownership and independence
- **Kafka/RabbitMQ** decouple services that don't need to talk synchronously
- **Saga** manages transactions that span multiple databases
- **Docker + Kubernetes** deploy and scale all of this reliably and elastically
- **Observability** ties the whole distributed system back into one traceable, understandable picture

Every README in this repo is a magnified view of one node or one edge in the diagram at the top of this file. This one is the map that shows how they all connect.
