# 08 · gRPC & Alternative Protocols

REST over HTTP/JSON isn't the only way to build an API. This module
covers gRPC and other protocols you'll meet in the wild, and when they
beat REST.

## gRPC: HTTP/2 + Protocol Buffers

gRPC defines a service in a `.proto` file, generates strongly-typed
client and server code in almost any language, and communicates using
compact binary Protocol Buffers over HTTP/2.

```protobuf
syntax = "proto3";

service OrderService {
  rpc GetOrder (GetOrderRequest) returns (Order);
  rpc StreamOrderUpdates (GetOrderRequest) returns (stream Order);
}

message GetOrderRequest { int32 id = 1; }

message Order {
  int32 id = 1;
  double total = 2;
  string status = 3;
}
```

A generated client calls this like a local function, not an HTTP
request:

```python
order = order_service_stub.GetOrder(GetOrderRequest(id=42))
print(order.total)
```

Under the hood this is still a request over the network — but the
developer writes no URL, no JSON parsing, no manual serialization.

## Why gRPC over REST/JSON

- **Performance** — binary Protobuf is smaller and faster to
  (de)serialize than JSON; HTTP/2 multiplexes many calls over one
  connection.
- **Strong contracts** — the `.proto` file is the single source of truth
  for types, generated on both client and server, so mismatches are
  caught at compile time, not at runtime like a malformed JSON body.
- **Native streaming** — `stream Order` above is a first-class
  bidirectional or server-streaming call; REST has no equivalent without
  bolting on SSE or WebSockets.

## Why REST still wins for public/browser-facing APIs

- Browsers can't natively speak gRPC (it needs HTTP/2 trailers browsers
  don't fully expose) — gRPC-Web exists but adds a proxy layer.
- REST/JSON is debuggable with `curl` and readable by a human; a
  Protobuf wire message is not.
- Public third-party integrators expect REST; gRPC assumes both sides
  control their codegen pipeline, which is realistic inside one company,
  less so across organizations.

gRPC is the common choice for **internal service-to-service** calls in a
microservices architecture (module 1, Level 4); REST/JSON stays the
default for **public and browser-facing** APIs.

## Other alternatives worth knowing

**WebSockets** — a persistent, full-duplex connection for real-time,
bidirectional communication (chat, live dashboards):

```javascript
const ws = new WebSocket("wss://api.example.com/v1/orders/42/stream");
ws.onmessage = (event) => console.log(JSON.parse(event.data));
```

**Server-Sent Events (SSE)** — one-way server → client streaming over
plain HTTP, simpler than WebSockets when the client never needs to send
messages back over the same connection:

```bash
curl -N https://api.example.com/v1/orders/42/events \
  -H "Accept: text/event-stream"
```

```
event: status_changed
data: {"status": "shipped"}

event: status_changed
data: {"status": "delivered"}
```

**MQTT** — a lightweight pub/sub protocol built for constrained IoT
devices over unreliable networks; overkill for a typical web API, the
right choice for a fleet of sensors.

## Worked example: picking a protocol

A company has a public REST API for customers, and internally, a fleet
of a dozen microservices calling each other thousands of times a
second, plus a live "order status" feed customers want to watch update
in real time.

- Customer-facing CRUD → **REST/JSON**, cacheable, debuggable, matches
  customer expectations.
- Service-to-service calls → **gRPC**, for the performance and strong
  typing at high call volume.
- Live order-status feed → **SSE**, because it's one-directional
  (server informs client) and simpler to operate than WebSockets for
  that shape of problem.

## Exercise

1. Why can't a browser call a gRPC service directly the way it calls a
   REST endpoint with `fetch`?
2. Give one scenario where SSE is the better choice over WebSockets, and
   one where it's the wrong choice.
3. A `.proto` file changes a field's type from `int32` to `string`. Is
   this breaking in the same way a JSON REST field-type change is
   (module 6)? Why does Protobuf field *numbering* matter for
   compatibility even when types don't change?
4. Explain why gRPC is more common for internal microservice traffic
   than for public developer-facing APIs.
