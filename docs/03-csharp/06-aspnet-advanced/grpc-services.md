# gRPC Services

High-performance, contract-first RPC framework using Protocol Buffers over HTTP/2.

## Why It Matters

- Binary serialization is 5-10x faster than JSON, payloads ~65% smaller
- Strongly-typed contracts generated from `.proto` files prevent runtime errors
- Native support for server, client, and bidirectional streaming

## Key Recommendations

- **Define contracts in `.proto` files** -- use `csharp_namespace` option and never change field numbers
- **Server setup**:
  ```csharp
  builder.Services.AddGrpc();
  app.MapGrpcService<UserServiceImpl>();
  ```
- **Implement services** by inheriting from the generated `ServiceBase` class
- **Use `RpcException`** with appropriate `StatusCode` (NotFound, PermissionDenied, Internal) for errors
- **Streaming patterns**: server streaming (`returns (stream T)`), client streaming (`stream T returns`), bidirectional (`stream T returns (stream T)`)
- **Add interceptors** for cross-cutting concerns (logging, auth, metrics):
  ```csharp
  builder.Services.AddGrpc(o => o.Interceptors.Add<LoggingInterceptor>());
  ```
- **Client setup**: `Grpc.Net.Client` + `GrpcChannel.ForAddress()` -- dispose the channel when done
- **Project file**: add `<Protobuf Include="Protos\*.proto" GrpcServices="Server" />`
- **Always use TLS in production**

## Packages

- Server: `Grpc.AspNetCore`
- Client: `Grpc.Net.Client`, `Google.Protobuf`, `Grpc.Tools`

## Pitfalls to Avoid

- Changing or reusing proto field numbers (breaks backward compatibility)
- Forgetting to mark deprecated fields with `reserved`
- Not setting deadlines on client calls (hangs indefinitely)
- Exposing gRPC directly to browsers without gRPC-Web proxy
