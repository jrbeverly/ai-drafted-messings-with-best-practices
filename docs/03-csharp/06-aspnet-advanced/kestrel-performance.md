# Kestrel Performance

Optimize Kestrel for high throughput with protocol tuning, compression, caching, and pipeline ordering.

## Why It Matters

- Kestrel is already fast, but defaults leave performance on the table for specific workloads
- Response compression alone can reduce payload sizes by 50-90%
- Pipeline ordering determines how much work each request does

## Key Recommendations

- **Enable HTTP/2** (multiplexing, header compression) -- set `HttpProtocols.Http1AndHttp2`
- **Enable HTTP/3** for mobile and high-latency clients (0-RTT, survives IP changes)
- **Response compression** with Brotli (better ratio than gzip):
  ```csharp
  builder.Services.AddResponseCompression(o =>
  {
      o.EnableForHttps = true;
      o.Providers.Add<BrotliCompressionProvider>();
  });
  builder.Services.Configure<BrotliCompressionProviderOptions>(o =>
      o.Level = CompressionLevel.Fastest);
  app.UseResponseCompression();
  ```
- **Output caching** bypasses the entire pipeline for cached responses (microsecond latency):
  ```csharp
  builder.Services.AddOutputCache(o =>
      o.AddPolicy("Default", b => b.Expire(TimeSpan.FromSeconds(60))));
  app.UseOutputCache();
  ```
- **Optimize pipeline order**: static files first (short-circuit), compression early, output cache before auth, endpoints last
- **Disable buffering** for streaming/upload endpoints: `context.Features.Get<IHttpResponseBodyFeature>()?.DisableBuffering()`
- **Use `ArrayPool<byte>`** for buffer-heavy code to reduce GC pressure
- **Thread pool tuning** only when measured starvation occurs: `ThreadPool.SetMinThreads(100, 100)`

## Benchmarking

- Use **bombardier** or **k6** for load testing
- Use **BenchmarkDotNet** for microbenchmarks (serialization, hot paths)
- Use **dotnet-trace** to profile production workloads
- Prefer `System.Text.Json` over Newtonsoft (2-3x faster, lower allocations)

## Pitfalls to Avoid

- Compressing responses under 1 KB (overhead exceeds benefit)
- Tuning thread pool without measuring (more threads = more memory)
- Skipping load tests and assuming default settings are optimal
- Enabling compression without `EnableForHttps = true` (disabled by default for HTTPS)
