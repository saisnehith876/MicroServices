# Database per Service Pattern

## 1. Why Not Just Share One Big Database?

If Payment, Order, Inventory, and User services all shared a single database, you'd basically have a **distributed monolith** — you'd get microservices in name, but a monolith in practice.

Here's why sharing breaks things:

### 1.1 Tight Coupling via Shared Schema
If Order service and Payment service both read/write the same `orders` table, then a schema change in one service (say, Order team renames a column) can silently break Payment service. Teams can no longer deploy independently — which defeats the whole point of microservices.

### 1.2 No Clear Ownership
Who's allowed to write to the `payments` table? If multiple services can touch it, you lose the guarantee that only Payment service's business logic governs that data's integrity.

### 1.3 Scaling Mismatch
Order and Payment have very different load patterns — Order might get hit 10x more during a flash sale. If they share one DB, you can't scale/tune that DB independently for each service's access pattern (e.g., Payment might need strong consistency + encryption, Order might need high write throughput).

### 1.4 Blast Radius
One giant shared DB going down takes *every* service down with it. Separate DBs mean Payment DB issues don't necessarily take down Inventory.

### 1.5 Tech Flexibility
Different services often benefit from different database types:

| Service | Suitable DB type |
|---|---|
| **Order service** | Relational (Postgres/MySQL) — for transactions |
| **Inventory** | Redis — for fast stock counts |
| **Product catalog / search** | Elasticsearch or MongoDB — flexible schema, fast search |
| **Payment** | Strongly consistent relational DB, often isolated for compliance (PCI-DSS) |

You couldn't do this if everything is jammed into one shared database.

---

## 2. So Yes — More Microservices = More Databases

This is a well-known tradeoff. It's not free. You're explicitly trading:

- ❌ Simplicity of one DB, one source of truth, easy joins
- ✅ For independence, scalability, and fault isolation per service

---

## 3. The New Problem: No More Simple SQL Joins

Say you want to show **"Order + Payment status + User name"** on one screen. Previously that was one SQL join. Now, with separate databases, you need a different strategy.

### Option A — API Composition
The frontend/API gateway calls Order service, then Payment service, then User service, and stitches the response together.

```
Client ──▶ API Gateway
              │
              ├──▶ Order Service    (get order details)
              ├──▶ Payment Service  (get payment status)
              └──▶ User Service     (get user name)
              │
              ▼
      Gateway combines all 3 responses into one
```

- Simple to implement, no data duplication
- Downside: multiple network calls per request, slower, and fails if any one service is down

### Option B — Event-Driven / Data Replication (CQRS-ish)
When Order service creates an order, it publishes an `OrderCreated` event (via Kafka/RabbitMQ). Payment and Shipping services subscribe and store **their own copy** of just the fields they need. No live cross-service calls needed at read time — each service already has a local, denormalized copy.

```
Order Service ──publishes──▶ OrderCreated event
                                    │
                        ┌───────────┴───────────┐
                        ▼                        ▼
              Payment Service              Shipping Service
              (stores local copy:          (stores local copy:
               order_id, amount,            order_id, address,
               user_id)                     items)
```

- Fast reads — no live cross-service calls needed
- Downside: eventual consistency (local copies can briefly lag behind the source of truth)

### Option C — Saga Pattern for Transactions
Since you can't do a single ACID transaction across Order DB + Payment DB + Inventory DB, you use a **Saga**: a sequence of local transactions coordinated via events, with **compensating actions** if something fails.

```
1. Order Service:     create order                  ✅
2. Payment Service:   charge customer                 ❌ FAILS

Compensating action:
2'. publish "PaymentFailed"
1'. Order Service:    mark order as CANCELLED (compensates step 1)
```

---

## 4. Worked Example — "Place an Order" in an E-commerce App

Let's walk through a concrete e-commerce checkout flow, showing how Order, Payment, and Inventory — each with their own separate database — coordinate a single business operation using events, including what happens when a step fails.

### Services and their databases

| Service | Owns | Database |
|---|---|---|
| **Order Service** | Order records, order status | Postgres |
| **Payment Service** | Payment transactions, refunds | Postgres (PCI-DSS isolated) |
| **Inventory Service** | Stock counts per product | Redis |

None of these services can see or query each other's database directly. All coordination happens through events.

### 4.1 Happy path — everything succeeds

```
1. Customer clicks "Place Order"
        │
        ▼
2. Order Service:
   - Creates order in its own DB, status = PENDING
   - Publishes event: "OrderCreated" { orderId, userId, items, amount }
        │
        ▼
3. Payment Service (subscribed to OrderCreated):
   - Charges the customer, writes transaction to its own DB
   - Publishes event: "PaymentCompleted" { orderId, transactionId }
        │
        ▼
4. Inventory Service (subscribed to PaymentCompleted):
   - Decrements stock count in its own DB (Redis)
   - Publishes event: "StockReserved" { orderId }
        │
        ▼
5. Order Service (subscribed to StockReserved):
   - Updates its own order record, status = CONFIRMED
        │
        ▼
6. Notification Service (subscribed to OrderConfirmed):
   - Sends confirmation email/SMS to customer
```

At every step, **each service only ever writes to its own database.** Order Service never touches Payment's database, Payment Service never touches Inventory's database — they only communicate by publishing and reacting to events (this is the Saga pattern from the design patterns README, in choreography style).

### 4.2 Failure path — Payment fails (e.g., card declined)

```
1. Order Service:  create order, status = PENDING            ✅
        │
        ▼
2. Payment Service: attempt to charge customer                ❌ FAILS
   - Publishes event: "PaymentFailed" { orderId, reason }
        │
        ▼
3. Order Service (subscribed to PaymentFailed):
   - Compensating action: updates its own order record,
     status = CANCELLED
   - No refund needed since payment never succeeded
        │
        ▼
4. Notification Service (subscribed to OrderCancelled):
   - Notifies customer that the order could not be completed
```

### 4.3 Failure path — Inventory fails (out of stock, after payment succeeded)

This is the more interesting case — a **later** step fails after an **earlier** step already committed real-world side effects (money was already charged).

```
1. Order Service:   create order, status = PENDING            ✅
2. Payment Service: charge customer                            ✅
3. Inventory Service: attempt to reserve stock                 ❌ FAILS (out of stock)
   - Publishes event: "StockReservationFailed" { orderId }
        │
        ▼
Compensating actions run, in reverse order of what succeeded:
        │
4. Payment Service (subscribed to StockReservationFailed):
   - Issues a refund to the customer
   - Publishes event: "PaymentRefunded" { orderId }
        │
        ▼
5. Order Service (subscribed to PaymentRefunded):
   - Updates its own order record, status = CANCELLED
        │
        ▼
6. Notification Service:
   - Notifies customer: "Sorry, item went out of stock — you've been refunded"
```

This is exactly why the Saga pattern needs **compensating transactions** rather than a simple rollback — Payment Service can't "undo" a charge the way a database transaction rolls back; it has to explicitly issue a *new* transaction (a refund) to reverse the effect.

### 4.4 Why this matters for the "Order + Payment + User" read problem

Now tie this back to Section 3 (API Composition vs Event-Driven Replication). Once this event flow is in place, showing "Order + Payment status + User name" on one screen becomes cheap:

- Order Service already has `orderId`, `status`, `userId` in its own DB
- Payment Service can maintain a small local read-replica of `orderId → paymentStatus`, kept in sync purely by consuming `PaymentCompleted` / `PaymentFailed` events — no live call to Payment Service needed at read time
- The same events that power the Saga (transaction coordination) are *also* what power fast, decoupled reads (Option B from Section 3)

This is the payoff of the whole architecture: the event stream isn't just for handling failures — it's simultaneously solving the "no more SQL joins" problem from Section 3.

---

## 5. Summary

| Challenge | Solution |
|---|---|
| Cross-service reads (e.g., combining Order + Payment + User data) | API Composition, or Event-Driven Data Replication (local read copies) |
| Cross-service transactions (e.g., Order → Payment → Inventory) | Saga pattern with compensating actions |
| Keeping local copies in sync | Event-driven updates via Kafka/RabbitMQ (`OrderCreated`, `PaymentFailed`, etc.) |

**Core tradeoff:** Database per Service buys independence, scalability, and fault isolation — at the cost of giving up simple joins and ACID transactions across services. The patterns above (API Composition, Event-Driven Replication, Saga) exist specifically to fill that gap.
