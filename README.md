# Microservices — Design Patterns, Tools & Development Lifecycle

> Popular interview question this README addresses:
> **"Tell me about the tools and design patterns that you have used in microservices."**

A developer is involved right from **creating** a microservice to **deploying** it on the cloud — understanding each phase of microservices development is important. This README walks through those phases in order: defining boundaries → communication → deployment → design patterns.

---

## 1. Defining Microservice Boundaries (Bounded Context)

The first step when creating a microservice is defining its **boundary** — i.e., what it is and isn't responsible for.

**Example:** An `sms-service` should only handle SMS-sending logic. It should **not** deal with payment processing inside the same service — that belongs to a separate `payment-service`.

Getting this boundary right (often guided by **Domain-Driven Design / DDD**) is critical for long-term **maintainability** — mixed responsibilities inside one service quietly turn microservices back into a monolith.

---

## 2. Microservices Communication

Microservices need to talk to each other. Common ways to do this:

- **RESTful web services** — synchronous, request/response over HTTP
- **Messaging services** — asynchronous, decoupled communication
- **Event streaming** — publish/subscribe to a continuous stream of events

**Popular message brokers:**
- Apache Artemis
- ActiveMQ
- RabbitMQ
- Kafka

---

## 3. Microservices Deployment

**Virtual Machines** are widely used in the industry, but they aren't ideal for microservices:

- Creating a VM is a **manual, tedious process**:
  1. Allocating memory/resources
  2. OS installation
  3. Library/dependency installation
- Since a VM runs a full OS, it's **slow to bootstrap and shut down**

Microservices, on the other hand, need **fast, on-demand deployment** — especially when load increases and services need to **autoscale**.

This is where **Docker** and **Kubernetes** come in:
- Containers **don't carry a full OS**, making them lightweight and fast compared to VMs
- Kubernetes handles orchestration, scaling, and self-healing of containerized services

---

## 4. Microservices Design Patterns

Splitting a monolith into microservices makes development simpler in some ways, but introduces new challenges. Here are the proven design patterns used to solve them:

### 4.1 Service Discovery & Registration
With many microservices, tracking the IPs/locations of all running service instances manually isn't feasible. **Service Discovery & Registration** solves this by letting services register themselves and discover each other dynamically.

### 4.2 Resilience Patterns
Microservices must be **fault-tolerant** since network calls between services can fail. Key patterns:
- **Circuit Breaker Pattern** — stops calling a failing service temporarily to prevent cascading failures
- **Retry Pattern** — automatically retries a failed request
- **Bulkhead Pattern** — isolates resources so one failing component doesn't exhaust resources for others
- **Rate Limiter Pattern** — controls the number of requests a service accepts to avoid overload

### 4.3 Gateway Pattern
Used to address **cross-cutting concerns** — Security (auth), Logging, and Tracing — in one centralized place instead of duplicating this logic across every service. Typically implemented via an **API Gateway**.

### 4.4 Saga Pattern
Used to manage **distributed transactions** across multiple services (since a single ACID transaction across separate databases isn't possible). A Saga is a sequence of local transactions with **compensating actions** if a step fails.

### 4.5 Other Supporting Patterns
- **Load Balancing** — distributes incoming requests across multiple instances of a service
- **DDD (Domain-Driven Design)** — helps define clean service boundaries
- **CQRS (Command Query Responsibility Segregation)** — separates read and write operations for better performance/scalability
- **Event-Driven Architecture** — services communicate via events instead of direct calls, improving decoupling

---

## 5. Quick Reference — Tools & Patterns Summary

| Phase | Tools / Patterns |
|---|---|
| **Boundary definition** | Domain-Driven Design (DDD), Bounded Context |
| **Communication** | REST, Kafka, RabbitMQ, ActiveMQ, Apache Artemis |
| **Deployment** | Docker, Kubernetes |
| **Service discovery** | Service Discovery & Registration pattern |
| **Resilience** | Circuit Breaker, Retry, Bulkhead, Rate Limiter |
| **Cross-cutting concerns** | API Gateway pattern |
| **Distributed transactions** | Saga pattern |
| **Performance/scale** | Load Balancing, CQRS, Event-Driven Architecture |

---

## 6. Interview Takeaway

When asked *"What tools and design patterns have you used in microservices?"* — structure your answer around the **lifecycle**, not just a list:

1. How you defined service boundaries (DDD)
2. How services communicate (REST/Kafka/RabbitMQ)
3. How you deploy and scale (Docker/Kubernetes)
4. How you handle failures and cross-cutting concerns (Circuit Breaker, API Gateway)
5. How you manage distributed transactions (Saga)

This shows an understanding of the **full lifecycle**, not just isolated buzzwords.
