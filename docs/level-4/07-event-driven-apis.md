# 07 · Event-Driven APIs & Async Patterns

Request/response REST assumes the client wants an answer right now. A
lot of real systems are better modeled as events flowing between
services asynchronously — this module covers when and how.

## Events vs. requests

A request says "do this, and tell me what happened." An event says
"this happened" — with no expectation of who, if anyone, reacts.

```json
{
  "event": "order.confirmed",
  "id": "evt_501",
  "data": { "order_id": 42, "total": 89.99 },
  "occurred_at": "2026-08-31T14:22:01Z"
}
```

`orders-service` publishes this once. `inventory-service`,
`notifications-service`, and `analytics-service` each independently
consume it — `orders-service` has no idea any of them exist, unlike the
direct-call composition pattern in module 1.

## Message brokers: Kafka, SQS, RabbitMQ

```bash
# publish
curl -X POST http://kafka-proxy.internal/topics/orders \
  -d '{"event": "order.confirmed", "data": {"order_id": 42}}'
```

A broker sits between publishers and subscribers, durably storing
events so a consumer that's temporarily down doesn't miss them — it
processes the backlog once it's back, rather than the event being lost
forever the way a direct HTTP call would be if the receiver was down.

## At-least-once delivery and idempotency

Most brokers guarantee **at-least-once** delivery, not exactly-once —
a consumer might see the same event twice (e.g. after a broker
rebalance). Consumers must be idempotent, exactly like the
`Idempotency-Key` pattern from Level 2:

```python
def handle_order_confirmed(event):
    if db.processed_events.exists(event["id"]):
        return  # already handled this exact event, skip
    db.processed_events.insert(event["id"])
    inventory.decrement(event["data"]["order_id"])
```

## The outbox pattern: don't lose events on a crash

A naive approach — write to the database, then publish the event — has
a gap: if the process crashes between the two steps, the database
change happened but the event never fired, and downstream systems
silently drift out of sync.

```sql
BEGIN;
INSERT INTO orders (id, status) VALUES (42, 'confirmed');
INSERT INTO outbox_events (event, payload) VALUES ('order.confirmed', '{"order_id": 42}');
COMMIT;
```

Both writes happen in the **same database transaction** — either both
commit or neither does. A separate relay process reads `outbox_events`
and actually publishes to the broker, retrying safely (each row
published exactly once, marked complete) — this guarantees the event
is *eventually* published if the order write succeeded, without a
distributed transaction across the database and the broker.

## Webhooks vs. message brokers

Level 3's webhooks are event-driven too, but push events across
organizational boundaries over plain HTTP — appropriate for
notifying external partners. A message broker (Kafka, SQS) is for
event flow *within* your own systems, where you control both ends and
want durability, ordering, and replay guarantees a webhook's simple
HTTP POST doesn't offer.

## Worked example: keeping a search index in sync

A `products-service` owns product data; a separate `search-service`
needs a near-real-time copy to power search.

1. `products-service` writes a product update and an outbox event in
   one transaction.
2. The outbox relay publishes `product.updated` to a Kafka topic.
3. `search-service` consumes the topic, idempotently upserts into its
   search index (checking the event ID against what it's already
   processed).
4. If `search-service` is down for an hour, Kafka retains the topic's
   events; on restart it resumes from its last committed offset and
   catches up — no manual reconciliation needed, and `products-service`
   never had to know `search-service` existed or was down.

## Exercise

1. Why does at-least-once delivery mean consumers must be written
   idempotently, and what breaks if they aren't?
2. Walk through why writing to the database and publishing an event as
   two separate steps (without an outbox) can silently lose events, and
   how the outbox pattern's single transaction fixes it.
3. Compare event-driven fan-out (this module) with the synchronous API
   composition pattern (module 1) for keeping `search-service` in sync
   with `products-service` — what does each approach cost in latency,
   coupling, and failure isolation?
4. A consumer processes `order.confirmed` twice due to an at-least-once
   redelivery, and it isn't idempotent — it decrements inventory twice.
   Design the idempotency check that would have prevented this.
