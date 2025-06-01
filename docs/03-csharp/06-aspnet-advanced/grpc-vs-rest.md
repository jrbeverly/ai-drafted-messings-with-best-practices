# gRPC vs REST

Choose the right protocol: REST for public/browser APIs, gRPC for internal service-to-service communication.

## Why It Matters

- Wrong choice costs performance (gRPC is ~3x faster) or developer ergonomics (REST is universally toolable)
- Many architectures benefit from using both simultaneously

## Quick Comparison

| Aspect | REST | gRPC |
|--------|------|------|
| Format | JSON (text) | Protobuf (binary) |
| Browser support | Native | Requires gRPC-Web |
| Streaming | Limited (SSE) | Server, client, bidirectional |
| Tooling | Excellent (curl, Postman) | Limited (grpcurl) |
| Contract | OpenAPI (optional) | Proto files (required) |
| Payload size | Larger | ~65% smaller |
| Latency | 15-30ms | 5-10ms (same datacenter) |

## Decision Matrix

**Choose REST when:**
- Public-facing API or browser/mobile clients
- Third-party integrations and webhooks
- Human-readable responses needed
- Simple CRUD with standard HTTP verbs

**Choose gRPC when:**
- Service-to-service (microservice mesh)
- High throughput or low-latency requirements
- Streaming data between services
- Polyglot environments needing shared contracts

## Recommended: Hybrid Architecture

```
Browser/Mobile ──REST──> API Gateway ──gRPC──> Internal Microservices
```

```csharp
builder.Services.AddControllers(); // REST for public
builder.Services.AddGrpc();        // gRPC for internal
app.MapControllers();
app.MapGrpcService<OrderServiceImpl>();
```

## Migration Strategy

1. Keep REST, add gRPC for new internal services
2. Add gRPC endpoints alongside existing REST for internal consumers
3. Internal callers migrate to gRPC
4. Deprecate REST for internal-only services; keep REST for public APIs

## Pitfalls to Avoid

- Forcing gRPC on browser clients without gRPC-Web
- Using REST for high-frequency internal service calls where gRPC would halve latency
- Assuming one protocol fits all use cases
