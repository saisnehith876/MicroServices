# Domain-Driven Design & Bounded Contexts — E-commerce Example

## 1. The Core Idea of DDD

Domain-Driven Design starts from a simple but easy-to-miss insight: **the same real-world word can mean different things in different parts of the business**, and forcing it into one shared model is where most badly-designed microservices go wrong.

The word **"Product"** in an e-commerce app is a perfect example. It's not one entity — it's four different models depending on context:

| Bounded Context | What "Product" means there |
|---|---|
| **Catalog context** | `name`, `description`, `photos`, `category` — what a customer browses |
| **Inventory context** | `SKU`, `stock count`, `warehouse location` — nobody here cares about marketing copy |
| **Pricing context** | `base price`, `active discounts`, `tax rules` — changes constantly, independently of the catalog |
| **Order context** | A **frozen snapshot** of name/price *as they were at the moment of purchase* — so a future price change doesn't retroactively alter a past order |

```
┌─────────────────────┐    ┌─────────────────────┐
│   Catalog context    │    │  Inventory context   │
│  name, description,  │    │  SKU, stock count,   │
│       photos          │    │      warehouse        │
└─────────────────────┘    └─────────────────────┘

┌─────────────────────┐    ┌─────────────────────┐
│   Pricing context    │    │    Order context      │
│  price, discounts,   │    │  frozen snapshot at   │
│      tax rules        │    │       purchase         │
└─────────────────────┘    └─────────────────────┘
```

This is the heart of DDD's **Bounded Context** concept: each of these is a self-contained model with its own **Ubiquitous Language** — a shared vocabulary that's precise *within that context*, but doesn't need to match the vocabulary in a neighboring context.

Trying to build one giant "Product" table that satisfies all four contexts is exactly how monoliths end up with a 40-column `products` table that nobody fully understands — and exactly how "microservices" end up secretly coupled even though they're technically separate services.

---

## 2. How You Actually Identify Boundaries in Practice

You don't draw these lines arbitrarily. A few practical techniques:

### 2.1 Business Capability Mapping
Ask: what are the distinct *capabilities* the business performs? Not technical layers (not "database layer" or "API layer") — actual business functions: managing the catalog, taking payments, managing stock, fulfilling orders, handling reviews. Each capability is a candidate service.

### 2.2 Event Storming
A workshop technique where you map out every significant business event in sticky-note form:
`OrderPlaced` → `PaymentAuthorized` → `StockReserved` → `OrderShipped` → `PaymentFailed`

Events that cluster together and share the same vocabulary tend to belong to the same bounded context. Events that trigger a *reaction* in a completely different vocabulary (`PaymentAuthorized` → triggers `StockReserved`) are usually a signal of a context boundary — with an event flowing across it.

### 2.3 Data Ownership Test
Ask: *who is the single authoritative owner of this piece of data?* Only `Inventory Service` should be allowed to write to stock counts. If two services both need to write to the same table, that's a sign the boundary is drawn wrong.

### 2.4 Conway's Law / Team Structure
In practice, boundaries often mirror team structure — if you have a dedicated Payments team, the Payment service boundary should map to what that team owns end-to-end. Misaligned boundaries (one team owning parts of three different services) cause constant cross-team coordination overhead — which defeats the purpose of microservices.

---

## 3. What Happens When Order Needs Pricing Data?

This is the part that trips people up: if Order context doesn't own `price`, how does it know what to charge?

Two common approaches:

- **Synchronous call** — Order service calls `GET /price/{productId}` on Pricing service at checkout time. Simple, but creates a runtime dependency — if Pricing service is down, checkout breaks.
- **Event-driven local copy** — Order service subscribes to a `PriceChanged` event from Pricing service and keeps a **local read replica** of just the price fields it needs. At checkout, it reads its own local copy — no live cross-service call needed. More resilient, but means eventual consistency (Order's copy might lag by a few seconds).

Either way — **Order never becomes the owner of price**. It either asks Pricing directly, or keeps a subordinate, denormalized copy. This distinction (owner vs. consumer of data) is the actual mechanism that keeps bounded contexts from silently merging back into a shared database.

---

## 4. Context Mapping — How Bounded Contexts Relate to Each Other

Once you've drawn boundaries, DDD gives you named patterns for *how* two contexts should relate:

| Pattern | Meaning | Example |
|---|---|---|
| **Shared Kernel** | Two contexts intentionally share a small common model | Order and Shipping might share a `Money` value object |
| **Customer–Supplier** | One context depends on another; the "supplier" is expected to accommodate the "customer's" needs | Order (customer) depends on Pricing (supplier) for price data |
| **Conformist** | A downstream context just accepts the upstream model as-is, no negotiation | A small internal Reporting service conforms entirely to Order's data shape |
| **Anti-Corruption Layer (ACL)** | A translation layer that prevents an external/legacy system's messy model from leaking into your clean domain model | Wrapping a legacy third-party payment gateway's clunky API so Payment service's internal model stays clean |
| **Open Host Service** | A context exposes a well-defined, stable public API/protocol for others to consume | Catalog service exposing a versioned public REST API that Search, Recommendation, and Order services all consume |

---

## 5. The Anti-Pattern to Watch For

The single most common mistake: defining services around **technical layers** ("UI service", "database service", "validation service") instead of **business capabilities**. That's not DDD — that's just a distributed monolith with extra network hops.

> A good bounded context always maps to a coherent piece of **business meaning**, not a technical concern.

---

## 6. Worked Example — Applying This to a Different Domain

The same splitting logic applies to any domain. For example, in a movie-streaming app, "Movie" would split the same way "Product" did above:

| Context | What "Movie" means there |
|---|---|
| **Catalog context** | `title`, `genre`, `poster`, `synopsis` |
| **Watchlist context** | `user_id`, `movie_id`, `added_at` |
| **Ratings context** | `aggregate_score`, `review_text`, `review_count` |

Same pattern, different domain: one word, multiple bounded models — each owned by a different service, connected via APIs or events rather than a shared table.

---

## 7. Summary

- A **Bounded Context** is an explicit boundary within which a domain model and its vocabulary (**Ubiquitous Language**) are consistent.
- The same real-world entity (e.g., "Product") can — and should — have **different models** in different contexts.
- Boundaries are identified via **business capabilities**, **event storming**, **data ownership**, and **team structure** — not arbitrary technical layers.
- Cross-context data needs are handled via **synchronous APIs** or **event-driven local copies** — never by sharing a database table.
- **Context Mapping patterns** (Shared Kernel, Customer–Supplier, Conformist, Anti-Corruption Layer, Open Host Service) describe how bounded contexts should relate to one another.
- The core anti-pattern to avoid: splitting services by **technical layer** instead of **business capability**.
