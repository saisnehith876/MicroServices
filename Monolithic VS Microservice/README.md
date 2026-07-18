
# Microservices Architecture — Overview

## 1. What is a Monolith?

A **monolithic architecture** is when an entire application — UI, business logic, data access, everything — is built and deployed as a **single unit**.

Example: An e-commerce app where Order, Payment, Inventory, User, and Notification logic all live in **one codebase**, connect to **one database**, and are deployed together as **one artifact** (one `.jar`, one `.war`, one process).

```
┌─────────────────────────────┐
│         Monolith App         │
│  ┌─────┐ ┌─────┐ ┌────────┐  │
│  │Order│ │ Pay │ │Inventory│  │
│  └─────┘ └─────┘ └────────┘  │
│         Single Codebase       │
│         Single Deploy         │
└──────────────┬────────────────┘
               │
        ┌──────▼──────┐
        │  One Shared  │
        │   Database   │
        └──────────────┘
```

---

## 2. Why did Microservices come into play?

As apps grew (think Amazon, Netflix, Uber scale), monoliths started breaking down — not technically at first, but **organizationally and operationally**. A few forces pushed the industry toward microservices:

- **Team scaling** — Hundreds of engineers working in one codebase = constant merge conflicts, blocked deployments, and coordination overhead.
- **Release velocity** — Companies wanted to ship features multiple times a day, not once every few weeks.
- **Cloud-native infrastructure** — Containers (Docker) and orchestration (Kubernetes) made it *practical* to run many small independently-deployed services.
- **Selective scaling needs** — Some parts of an app (e.g., checkout during a sale) need to scale far more than others (e.g., user profile settings).

Netflix, Amazon, and Uber are the textbook case studies — all migrated from monoliths to microservices as they scaled globally.

---

## 3. Problems with Monolithic Architecture

| Problem | Explanation |
|---|---|
| **Tight coupling** | Every module depends on shared code/database — a small change in one place can break something unrelated. |
| **Slow deployments** | Even a one-line fix in Payment requires redeploying the *entire* application. |
| **Scaling is all-or-nothing** | Can't scale just the Order module — you have to scale the whole app, wasting resources. |
| **Single point of failure** | A bug or crash in one module (e.g., Notification service leaking memory) can take down the entire app. |
| **Technology lock-in** | The whole app is stuck on one language/framework/database — hard to adopt better tools for specific problems. |
| **Slow onboarding** | New engineers need to understand a massive, tangled codebase before they can contribute safely. |
| **Difficult CI/CD** | Large build/test cycles slow release velocity — every change re-triggers a full regression cycle. |

---

## 4. How Microservices Solve These Issues

**Microservices architecture** breaks the application into small, independent services — each owning a specific business capability (Order, Payment, Inventory, User, etc.), each **independently deployable**, and each often with **its own database**.

```
┌────────┐   ┌────────┐   ┌───────────┐   ┌────────┐
│ Order  │   │Payment │   │ Inventory │   │  User  │
│Service │   │Service │   │  Service  │   │Service │
└───┬────┘   └───┬────┘   └─────┬─────┘   └───┬────┘
    │            │              │             │
 ┌──▼──┐      ┌──▼──┐        ┌──▼──┐       ┌──▼──┐
 │ DB  │      │ DB  │        │ DB  │       │ DB  │
 └─────┘      └─────┘        └─────┘       └─────┘
```

| Monolith Problem | Microservices Fix |
|---|---|
| Tight coupling | Services communicate via well-defined APIs/events — internal changes don't ripple across the system |
| Slow deployments | Each service is deployed independently — fix Payment without touching Order |
| All-or-nothing scaling | Scale only the services under load (e.g., scale Order + Inventory during a flash sale, leave others as-is) |
| Single point of failure | A crash in Notification service doesn't take down Payment or Order |
| Tech lock-in | Each team picks the best stack for their service (Node.js for one, Go for another, Postgres vs MongoDB, etc.) |
| Slow onboarding | New engineers only need to understand one small service, not the entire system |
| Slow CI/CD | Small, independent build/test/deploy pipelines per service — much faster releases |

---

## 5. The Trade-off

Microservices aren't free — they introduce new complexity:

- **Distributed systems problems** — network latency, partial failures, eventual consistency
- **Data consistency across services** — no more simple SQL joins across services (solved via API composition, event-driven sync, or the Saga pattern)
- **Operational overhead** — need service discovery, centralized logging, distributed tracing (e.g., Dynatrace, Jaeger), API gateways, and container orchestration (Kubernetes)
- **Testing complexity** — integration testing across services is harder than testing one codebase

> **Rule of thumb:** Microservices solve *organizational* and *scaling* problems, but introduce *distributed systems* problems. They're worth it at scale — not always necessary for small apps/teams.

---

## 6. Summary

| | Monolith | Microservices |
|---|---|---|
| **Deployment** | Single unit | Independent per service |
| **Database** | Shared | Per-service |
| **Scaling** | Whole app | Per service |
| **Team structure** | Centralized | Decentralized, per-service ownership |
| **Failure impact** | System-wide | Isolated (mostly) |
| **Complexity** | Simple to start, hard to scale | Harder to start, easier to scale |
| **Best for** | Small apps, small teams, early-stage products | Large, complex, high-scale systems with multiple teams |
