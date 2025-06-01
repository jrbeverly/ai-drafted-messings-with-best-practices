# Kestrel Configuration

Configure ASP.NET Core's default web server: endpoints, HTTPS, connection limits, and protocols.

## Why It Matters

- Kestrel settings control ports, TLS, request size limits, and protocol support
- Production defaults are too permissive -- tune limits to match your workload
- Reverse proxy deployments require forwarded headers to preserve client IP and scheme

## Key Recommendations

- **Basic endpoint setup**:
  ```csharp
  builder.WebHost.ConfigureKestrel(options =>
  {
      options.ListenAnyIP(5000);
      options.ListenAnyIP(5001, lo => lo.UseHttps());
  });
  ```
- **Connection limits** (tune for production):
  ```csharp
  options.Limits.MaxConcurrentConnections = 1000;
  options.Limits.MaxRequestBodySize = 10_000_000; // 10 MB
  options.Limits.KeepAliveTimeout = TimeSpan.FromMinutes(2);
  options.Limits.RequestHeadersTimeout = TimeSpan.FromSeconds(30);
  ```
- **Enable HTTP/2** for modern clients and gRPC: `listenOptions.Protocols = HttpProtocols.Http1AndHttp2`
- **HTTP/3 (QUIC)** for mobile/high-latency: `HttpProtocols.Http1AndHttp2AndHttp3` (add `Alt-Svc` header)
- **HTTPS in production**: load certificate from file or config, use SNI for multi-domain
- **Development**: `dotnet dev-certs https --trust`
- **Behind reverse proxy**: configure `ForwardedHeaders` middleware to trust your proxy IP:
  ```csharp
  builder.Services.Configure<ForwardedHeadersOptions>(o =>
      o.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto);
  app.UseForwardedHeaders();
  ```
- **Disable min data rate** for large file upload endpoints via `IHttpMinRequestBodyDataRateFeature`
- **Supports Unix sockets** (Linux) and named pipes (Windows) for inter-process communication

## Config via appsettings.json

```json
{ "Kestrel": { "Endpoints": { "Http": { "Url": "http://localhost:5000" } },
  "Limits": { "MaxRequestBodySize": 30000000 } } }
```

## Pitfalls to Avoid

- Exposing Kestrel directly to the internet without a reverse proxy
- Not configuring `ForwardedHeaders` (client IP shows as proxy IP)
- Setting `MaxRequestBodySize` too high (enables abuse) or too low (breaks file uploads)
- Forgetting to configure HTTP/2 limits when running gRPC services
