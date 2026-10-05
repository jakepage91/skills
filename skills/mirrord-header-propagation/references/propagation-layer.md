# Minimal propagation layer

Use these when a service has no stack that can propagate baggage, or its stack misses a hop. Default to the **OpenTelemetry API with the W3C baggage propagator and no exporter**. That needs no collector or tracing setup, and it stays compatible if the team adds tracing later. Adapt the snippets to the frameworks and clients actually in the repo, and put each one at a single boundary (middleware, interceptor, producer/consumer wrapper), not at call sites.

The contract every snippet implements:

1. **Ingress:** read `baggage` from the request/message and put it in the context of this request/message.
2. **Egress:** write the context's baggage onto every outgoing request/message.
3. **Merge, don't replace:** the propagator serialises all members; never write a `baggage` value containing only `mirrord-session`.

## Go

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
    "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
)

func init() {
    // No TracerProvider is needed for baggage-only propagation.
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{}, propagation.Baggage{}))
}

// HTTP: wrap the server handler and the client transport.
handler = otelhttp.NewHandler(mux, "server")
client := &http.Client{Transport: otelhttp.NewTransport(http.DefaultTransport)}
// Outgoing requests must be built with the request's ctx: http.NewRequestWithContext(ctx, ...)

// gRPC
grpc.NewServer(grpc.StatsHandler(otelgrpc.NewServerHandler()))
grpc.NewClient(addr, grpc.WithStatsHandler(otelgrpc.NewClientHandler()))
```

Kafka (any client) — a carrier over the client's header slice. Example for `segmentio/kafka-go`:

```go
type kafkaHeaders struct{ h *[]kafka.Header }

func (c kafkaHeaders) Get(k string) string {
    for _, h := range *c.h { if h.Key == k { return string(h.Value) } }
    return ""
}
func (c kafkaHeaders) Set(k, v string) {
    for i, h := range *c.h { if h.Key == k { (*c.h)[i].Value = []byte(v); return } }
    *c.h = append(*c.h, kafka.Header{Key: k, Value: []byte(v)})
}
func (c kafkaHeaders) Keys() []string {
    ks := make([]string, 0, len(*c.h)); for _, h := range *c.h { ks = append(ks, h.Key) }; return ks
}

// produce
otel.GetTextMapPropagator().Inject(ctx, kafkaHeaders{&msg.Headers})
// consume — per message
ctx := otel.GetTextMapPropagator().Extract(context.Background(), kafkaHeaders{&m.Headers})
handle(ctx, m)
```

SQS (`aws-sdk-go-v2`):

```go
// produce
carrier := propagation.MapCarrier{}
otel.GetTextMapPropagator().Inject(ctx, carrier)
if input.MessageAttributes == nil { input.MessageAttributes = map[string]types.MessageAttributeValue{} }
for k, v := range carrier {
    input.MessageAttributes[k] = types.MessageAttributeValue{DataType: aws.String("String"), StringValue: aws.String(v)}
}
// consume — request the attributes, then extract per message
recv.MessageAttributeNames = []string{"All"}
carrier := propagation.MapCarrier{}
for k, v := range m.MessageAttributes { if v.StringValue != nil { carrier[k] = *v.StringValue } }
ctx := otel.GetTextMapPropagator().Extract(context.Background(), carrier)
```

Pub/Sub (`cloud.google.com/go/pubsub`): the same `MapCarrier` pattern against `msg.Attributes` (publish) and `m.Attributes` inside the `Receive` callback (consume). RabbitMQ (`amqp091-go`): the same against `amqp.Table` in `Publishing.Headers` / `Delivery.Headers` (values are `interface{}` — store strings).

Goroutines: pass `ctx` into anything started from a handler. `go work()` without `ctx` drops baggage.

## Python

```python
from opentelemetry import baggage, context
from opentelemetry.propagate import inject, extract, set_global_textmap
from opentelemetry.propagators.composite import CompositePropagator
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator
from opentelemetry.baggage.propagation import W3CBaggagePropagator

set_global_textmap(CompositePropagator([TraceContextTextMapPropagator(), W3CBaggagePropagator()]))
```

HTTP/gRPC: prefer the instrumentation packages (`opentelemetry-instrumentation-flask|django|fastapi|requests|httpx|grpc`) — they work without an exporter. Manual fallback:

```python
# server middleware
token = context.attach(extract(request.headers))
try: ...handle...
finally: context.detach(token)

# client
headers = {}; inject(headers); requests.get(url, headers=headers)
```

Kafka (`confluent_kafka`):

```python
# produce — headers as a list of (str, bytes)
carrier = {}; inject(carrier)
producer.produce(topic, value, headers=[(k, v.encode()) for k, v in carrier.items()])

# consume — per message
msg = consumer.poll(1.0)
carrier = {k: v.decode() for k, v in (msg.headers() or [])}
token = context.attach(extract(carrier))
try: handle(msg)
finally: context.detach(token)
```

SQS (`boto3`):

```python
carrier = {}; inject(carrier)
attrs = {k: {"DataType": "String", "StringValue": v} for k, v in carrier.items()}
sqs.send_message(QueueUrl=url, MessageBody=body, MessageAttributes={**existing, **attrs})

resp = sqs.receive_message(QueueUrl=url, MessageAttributeNames=["All"])
for m in resp.get("Messages", []):
    carrier = {k: v["StringValue"] for k, v in m.get("MessageAttributes", {}).items() if "StringValue" in v}
    token = context.attach(extract(carrier))
    try: handle(m)
    finally: context.detach(token)
```

Pub/Sub: `publisher.publish(topic, data, **carrier)` (attributes are kwargs); in the subscriber callback, `extract(dict(message.attributes))`. RabbitMQ (`pika`): `pika.BasicProperties(headers=carrier)` / `extract(properties.headers or {})`.

Celery / thread pools: `contextvars` don't cross processes. For Celery, `opentelemetry-instrumentation-celery` carries context in task headers. For `ThreadPoolExecutor`, submit with `contextvars.copy_context().run`.

## Node.js / TypeScript

```ts
import { propagation, context } from '@opentelemetry/api';
import { W3CBaggagePropagator, W3CTraceContextPropagator, CompositePropagator } from '@opentelemetry/core';
import { AsyncLocalStorageContextManager } from '@opentelemetry/context-async-hooks';

context.setGlobalContextManager(new AsyncLocalStorageContextManager().enable());
propagation.setGlobalPropagator(new CompositePropagator({
  propagators: [new W3CTraceContextPropagator(), new W3CBaggagePropagator()],
}));
```

HTTP: `@opentelemetry/instrumentation-http` (+ `-express` / `-fastify` / `-undici`) registered with `registerInstrumentations` — no exporter needed. Manual fallback:

```ts
// server (express)
app.use((req, _res, next) => context.with(propagation.extract(context.active(), req.headers), next));
// client
const headers: Record<string, string> = {};
propagation.inject(context.active(), headers);
await fetch(url, { headers });
```

Kafka (`kafkajs`):

```ts
// produce
const headers: Record<string, string> = {};
propagation.inject(context.active(), headers);
await producer.send({ topic, messages: [{ value, headers }] });

// consume — per message (eachBatch: do this inside the loop over batch.messages)
await consumer.run({
  eachMessage: async ({ message }) => {
    const carrier = Object.fromEntries(
      Object.entries(message.headers ?? {}).map(([k, v]) => [k, v?.toString() ?? '']));
    await context.with(propagation.extract(context.active(), carrier), () => handle(message));
  },
});
```

SQS (`@aws-sdk/client-sqs`): inject into a record, map to `MessageAttributes: { [k]: { DataType: 'String', StringValue: v } }`; receive with `MessageAttributeNames: ['All']` and extract per message. Pub/Sub: `topic.publishMessage({ data, attributes: carrier })` / `propagation.extract(context.active(), message.attributes)`. RabbitMQ (`amqplib`): `channel.publish(ex, key, buf, { headers: carrier })` / `msg.properties.headers`.

BullMQ / Redis Pub/Sub: mirrord filters these on **top-level payload fields**, so put the value there:

```ts
const carrier: Record<string, string> = {};
propagation.inject(context.active(), carrier);
await queue.add(name, { ...data, baggage: carrier.baggage });
// worker
new Worker(name, async (job) =>
  context.with(propagation.extract(context.active(), { baggage: job.data.baggage ?? '' }), () => handle(job)));
```

## Java / Kotlin

Prefer the OTel Java agent (no code) or Spring Boot Micrometer Tracing — see libraries.md. Without either, add `opentelemetry-api` + `opentelemetry-context` and set the propagator once:

```java
TextMapPropagator prop = TextMapPropagator.composite(
    W3CTraceContextPropagator.getInstance(), W3CBaggagePropagator.getInstance());
ContextPropagators propagators = ContextPropagators.create(prop);
```

Kafka with a `ProducerInterceptor` (produce side) and per-record extraction (consume side):

```java
TextMapSetter<Headers> SETTER = (h, k, v) -> { h.remove(k); h.add(k, v.getBytes(UTF_8)); };
TextMapGetter<Headers> GETTER = new TextMapGetter<>() {
  public Iterable<String> keys(Headers h) {
    return StreamSupport.stream(h.spliterator(), false).map(Header::key).toList(); }
  public String get(Headers h, String k) {
    Header x = h.lastHeader(k); return x == null ? null : new String(x.value(), UTF_8); }
};

// ProducerInterceptor.onSend
prop.inject(Context.current(), record.headers(), SETTER);

// consumer loop — per record
for (ConsumerRecord<K, V> r : records) {
  try (Scope s = prop.extract(Context.root(), r.headers(), GETTER).makeCurrent()) { handle(r); }
}
```

Executors: wrap with `Context.taskWrapping(executor)` so tasks inherit the context.

## No dependency allowed

If the team won't take the OTel API, keep the raw header value in the framework's request-scoped context (Go `context.WithValue`, Python `contextvars.ContextVar`, Node `AsyncLocalStorage`, Java `ThreadLocal`/Reactor context) and copy it verbatim onto every outgoing request and message. Don't parse or rewrite it. Note in the report that this layer only carries the raw header and won't interoperate with tracing if tracing is added later.
