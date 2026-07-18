# Microservices Communication — In Depth

Microservices are only useful if they can talk to each other reliably. This README covers the three major communication styles used in production systems — **synchronous APIs**, **Message Queues (MQ)**, and **Kafka (event streaming)** — how they differ, when to use each, and how they typically combine in a real system.

---

## 1. Two Fundamental Communication Styles

Before picking a tool, understand the underlying choice:

| | Synchronous | Asynchronous |
|---|---|---|
| **How it works** | Caller sends a request and **waits** for a response | Caller sends a message and **moves on** — response (if any) comes later |
| **Coupling** | Tight — both services must be up at the same time | Loose — receiver can be down/slow, message waits |
| **Examples** | REST, gRPC | Message Queues (RabbitMQ, ActiveMQ), Kafka |
| **Failure behavior** | Caller blocks or fails immediately if callee is down | Message persists until consumer is ready |
| **Best for** | Requests needing an immediate answer (e.g., "is this item in stock?") | Background work, notifications, decoupled workflows |

Most real systems use **both** — synchronous APIs for immediate reads, asynchronous messaging for anything that can happen "eventually."

---

## 2. Synchronous Communication — APIs (REST / gRPC)

### 2.1 How it works
Service A calls Service B directly over HTTP (REST) or a binary protocol (gRPC), and **waits** for the response before continuing.

```
Order Service  ──── GET /price/{productId} ────▶  Pricing Service
Order Service  ◀─────── 200 OK { price }  ────────  Pricing Service
```

### 2.2 REST
- Uses standard HTTP verbs (GET, POST, PUT, DELETE) and JSON payloads
- Human-readable, widely supported, easy to debug (curl/Postman)
- Stateless — each request carries all the context needed

### 2.3 gRPC
- Uses Protocol Buffers (binary format) over HTTP/2 — much faster and smaller payloads than JSON/REST
- Strongly typed contracts (`.proto` files) shared between services — fewer integration bugs
- Supports streaming (client-streaming, server-streaming, bidirectional) — not just request/response
- Common inside internal microservice-to-microservice calls where performance matters more than human readability

### 2.4 When to use synchronous APIs
- The caller **needs an immediate answer** to proceed (e.g., checking real-time inventory before confirming an order)
- Simple CRUD operations between two services
- Internal, low-latency calls where gRPC's speed advantage matters

### 2.5 The downside
- **Tight coupling** — if Pricing Service is down, Order Service's request fails or hangs
- **Cascading failures** — a slow downstream service can back up the entire call chain
- This is exactly why resilience patterns exist: **Circuit Breaker**, **Retry**, **Timeout**, **Bulkhead** (see the design patterns README for details)

---

## 3. Asynchronous Communication — Message Queues (MQ)

### 3.1 How it works
Instead of calling another service directly, a service **publishes a message to a queue**. A separate consumer picks it up and processes it **whenever it's ready** — the producer doesn't wait.

```
Order Service ──▶ [ Queue: order.created ] ──▶ Notification Service (consumes when ready)
```

### 3.2 Point-to-point vs Pub/Sub
- **Point-to-point (queue)** — one message is consumed by exactly **one** consumer, even if multiple consumers are listening (used for load-balancing work across workers)
- **Publish/Subscribe (topic)** — one message is delivered to **every** subscriber independently (used when multiple services each need to react to the same event)

### 3.3 Popular MQ brokers

| Broker | Notes |
|---|---|
| **RabbitMQ** | Most popular general-purpose broker; supports complex routing (exchanges: direct, topic, fanout, headers) |
| **ActiveMQ** | Java/JMS-based, mature, widely used in enterprise Java shops |
| **Apache Artemis** | ActiveMQ's next-gen successor — better performance, still JMS-compliant |
| **Amazon SQS** | Fully managed cloud queue service, no infrastructure to run |

### 3.4 Why use MQ instead of a direct API call?

- **Decoupling** — Order Service doesn't need to know Notification Service exists, or whether it's currently up
- **Buffering / backpressure handling** — if Notification Service is overwhelmed, messages simply queue up instead of failing requests
- **Retry built-in** — most brokers automatically redeliver a message if the consumer fails to acknowledge it
- **Dead Letter Queue (DLQ)** — messages that repeatedly fail processing get moved to a separate queue for manual inspection, instead of being lost or blocking the pipeline

### 3.5 When to use MQ
- Fire-and-forget tasks: sending an email/SMS, generating an invoice PDF, updating a search index
- Work that must survive a temporary consumer outage
- Smoothing out traffic spikes (queue absorbs a burst, workers drain it steadily)

---

## 4. Event Streaming — Kafka

Kafka looks like a message broker on the surface, but it's architecturally different — and solves different problems.

### 4.1 Key differences from traditional MQ

| | Traditional MQ (RabbitMQ/ActiveMQ) | Kafka |
|---|---|---|
| **Message lifecycle** | Message is typically **deleted** once consumed/acknowledged | Message is **retained** on disk for a configured period (hours to forever), regardless of consumption |
| **Multiple consumers** | Usually one consumer per message (unless using pub/sub) | Many independent consumer groups can each read the **same** stream at their own pace |
| **Replay** | Not possible once a message is consumed and removed | Consumers can **rewind and replay** old messages by resetting their offset |
| **Ordering** | Depends on broker/queue config | Strong ordering guaranteed **within a partition** |
| **Throughput** | Good | Extremely high — built for massive, continuous event streams |
| **Primary use case** | Task queues, decoupled RPC-like async work | Event sourcing, real-time analytics, activity streams, audit logs |

### 4.2 Core Kafka concepts

- **Topic** — a named stream of events (e.g., `order-events`, `payment-events`)
- **Partition** — a topic is split into partitions for parallelism; each partition preserves strict message order
- **Producer** — publishes events to a topic
- **Consumer Group** — a set of consumers sharing the work of reading a topic; each partition is read by only one consumer within a group (this is how Kafka scales horizontally)
- **Offset** — a pointer tracking how far a consumer has read; because Kafka retains messages, a consumer can reset its offset and **replay history**
- **Broker** — a Kafka server; a **cluster** is made of multiple brokers for fault tolerance

### 4.3 Why Kafka fits event-driven microservices so well

In an e-commerce system, many services care about the same event, but for completely different reasons:

```
                         ┌─────────────────┐
                         │ Kafka topic:      │
   Order Service ──────▶ │ order-created      │
                         └─────────────────┘
                              │       │       │
                 ┌────────────┘       │       └────────────┐
                 ▼                    ▼                    ▼
        Inventory Service     Payment Service      Notification Service
        (reserve stock)       (charge customer)     (send confirmation)
```

Each service is an **independent consumer group** reading the same `order-created` topic — none of them block each other, and none of them need to know the others exist. This is fundamentally different from an MQ point-to-point queue, where the message would typically go to just one consumer.

### 4.4 When to use Kafka
- Multiple, independent services need to react to the **same** event
- You need **event replay** — e.g., rebuilding a service's state from scratch, or feeding a new service that joins later and needs historical events
- High-throughput streams: clickstream data, IoT telemetry, audit/event logs
- **Event Sourcing** — storing state as a sequence of events rather than just the current row in a table (the event log becomes the source of truth)
- Backing infrastructure for the **Saga pattern** in distributed transactions (coordinating Order → Payment → Inventory via a chain of events)

---

## 5. Putting It All Together — E-commerce Order Flow Example

A realistic order flow typically combines all three styles:

```
1. Customer clicks "Buy Now"
        │
        ▼
2. Order Service (REST/API call from frontend) — synchronous, needs immediate response
        │
        ▼
3. Order Service checks stock ── gRPC call ──▶ Inventory Service — synchronous, needs answer now
        │
        ▼
4. Order Service publishes "OrderCreated" event ──▶ Kafka topic: order-events
        │
        ├──▶ Payment Service (consumer group)     — charges the customer asynchronously
        ├──▶ Inventory Service (consumer group)   — reserves/decrements stock asynchronously
        └──▶ Notification Service (consumer group)— sends confirmation email/SMS

5. Notification Service, after sending the email, pushes a task to an MQ (RabbitMQ)
   for a background worker to also generate and store a PDF invoice — fire-and-forget
```

- **Step 2–3**: synchronous API/gRPC — the customer is waiting, needs a fast, immediate answer
- **Step 4**: Kafka — multiple independent services need to react to the same order event, and this event is valuable to retain (audit trail, analytics, replay)
- **Step 5**: MQ — a simple one-off background task, no need for replay or multiple consumers, just reliable fire-and-forget delivery

---

## 6. Choosing the Right Tool — Quick Reference

| Scenario | Use |
|---|---|
| Need an answer right now to continue processing | REST or gRPC (synchronous API) |
| Internal service-to-service call, performance-critical | gRPC |
| One-off background task (email, invoice, image resize) | Message Queue (RabbitMQ/ActiveMQ/SQS) |
| Multiple services must independently react to the same event | Kafka |
| Need to replay historical events or rebuild service state | Kafka |
| Coordinating a distributed transaction (Saga pattern) | Kafka (or MQ, depending on scale) |
| Smoothing out a traffic spike before processing | Message Queue |
| Real-time analytics / audit log / event sourcing | Kafka |

---

## 7. Summary

- **APIs (REST/gRPC)** — synchronous, tightly coupled, best when the caller needs an immediate answer.
- **Message Queues (RabbitMQ, ActiveMQ, Artemis, SQS)** — asynchronous, decoupled, best for one-off background work with reliable delivery and retry.
- **Kafka** — asynchronous event streaming, retains history, supports multiple independent consumers and replay — best for event-driven architectures where many services care about the same event.

A well-designed microservices system rarely uses just one of these — it combines synchronous APIs for real-time needs with MQ/Kafka for decoupled, resilient, event-driven workflows.
