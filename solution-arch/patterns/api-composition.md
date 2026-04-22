# API Composition Pattern

**Topic:** [[solution-arch/topics/microservices]], [[solution-arch/topics/integration-patterns]]
**Related concepts:** [[solution-arch/concepts/api-gateway]], [[solution-arch/patterns/bff]], [[solution-arch/patterns/database-per-service]], [[solution-arch/concepts/cqrs]]

## What it solves
When a client needs data owned by multiple services and no shared database exists, the API Composition pattern aggregates responses from multiple service calls into a single result. It is the simplest approach to cross-service queries in microservices.

## How it works

```
Client: "Show order #123 with customer name and product details"

                        API Composer
                     (API Gateway / BFF)
                            │
          ┌─────────────────┼─────────────────────┐
          ▼                 ▼                     ▼
   Order Service      User Service          Product Service
   GET /orders/123    GET /users/u1          GET /products/p42
          │                 │                     │
          └─────────────────┼─────────────────────┘
                            ▼
                        Merge & return:
                        {
                          orderId:  "123",
                          status:   "shipped",
                          customer: { name: "Alice", email: "..." },
                          items: [{ product: "Widget", price: 9.99 }]
                        }
```

## Implementation Approaches

### Sequential Composition
Call services one after another. Simple but slow — latencies add up.

```
order = GET /orders/123          # 50ms
user  = GET /users/{order.userId} # 40ms
items = GET /products?ids=...    # 60ms

Total: 150ms (serialised)
```

### Parallel Composition
Call independent services simultaneously. Faster — latency = slowest single call.

```python
# Concurrent fetch
order, user, products = await asyncio.gather(
    get_order(order_id),
    get_user(user_id),
    get_products(product_ids)
)
# Total: max(50ms, 40ms, 60ms) = 60ms
```

Always parallelise when calls are independent (no data dependency between them).

### Scatter-Gather
Broadcast a query to multiple services; aggregate all responses.

```
Search request ──▶ Product Service (fast index)
               └──▶ User Content Service (reviews)
               └──▶ Inventory Service (stock status)

Wait for all to respond (or timeout), merge results
```

---

## Trade-offs

| Benefit | Cost |
|---------|------|
| Simple implementation — just HTTP calls | Latency increases with each service call |
| No data duplication | Calling service fails if any dependency is down |
| Data always fresh (no caching layer) | Coupled to downstream availability |
| Works without event infrastructure | N+1 problem if items loop (call per item) |

---

## N+1 Problem in API Composition

```
Bad (N+1):
  order = GET /orders/123       # returns 10 items with product IDs
  for item in order.items:
    product = GET /products/{item.productId}  # 10 separate calls!
  
  Total: 1 + 10 = 11 calls → high latency

Good (batch):
  order = GET /orders/123
  ids = [item.productId for item in order.items]
  products = GET /products?ids=p1,p2,...,p10   # 1 batch call
  
  Total: 2 calls
```

Always design downstream APIs to accept batch/bulk queries when composition is expected.

---

## When API Composition Falls Short → Use CQRS

```
API Composition works for:
  ✅ Simple aggregations (one order + one user)
  ✅ Small number of services to call (≤ 3–4)
  ✅ Low volume (latency acceptable)

Use CQRS read model instead when:
  ❌ Complex queries with sorting/pagination across service boundaries
     (e.g., "all orders with customer name, sorted by date, page 2")
  ❌ High-volume reads (synchronous fan-out is too slow)
  ❌ Downstream services are frequently unavailable
  ❌ Query requires aggregation/computation (sum of all order values per user)
```

For complex cross-service queries, pre-compute and materialise the view using events.

---

## Signal phrases
- "How do you query data owned by multiple services?"
- "How does the Order Service get the customer name without accessing the Users DB?"
- "What is API composition and when would you use CQRS instead?"
- "How do you avoid the N+1 problem in microservices?"

## Complexity
Low — just HTTP calls. The challenge is keeping latency acceptable (parallelise) and handling partial failures gracefully (return partial data rather than failing entirely when one service is unavailable).

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
