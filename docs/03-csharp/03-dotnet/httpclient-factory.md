# HttpClient Factory

Manage `HttpClient` instances with `IHttpClientFactory` for proper connection pooling and DNS handling.

## Why It Matters

- `new HttpClient()` causes socket exhaustion and ignores DNS changes
- Factory manages connection pooling, handler lifecycle, and disposal automatically
- Built-in support for retry policies, circuit breakers, and delegating handlers

## Never Do This

```csharp
using var client = new HttpClient();  // Socket exhaustion risk
```

## Three Registration Styles

```csharp
// 1. Basic -- inject IHttpClientFactory, call CreateClient()
builder.Services.AddHttpClient();

// 2. Named -- configure per API
builder.Services.AddHttpClient("GitHub", client =>
{
    client.BaseAddress = new Uri("https://api.github.com/");
    client.Timeout = TimeSpan.FromSeconds(30);
});

// 3. Typed (preferred) -- inject HttpClient directly into service
builder.Services.AddHttpClient<IGitHubService, GitHubService>(client =>
{
    client.BaseAddress = new Uri("https://api.github.com/");
});
```

## Typed Client Usage

```csharp
public class GitHubService(HttpClient httpClient) : IGitHubService
{
    public async Task<Repository> GetRepoAsync(string owner, string repo)
    {
        var response = await httpClient.GetAsync($"repos/{owner}/{repo}");
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<Repository>()
            ?? throw new InvalidOperationException("Invalid response");
    }
}
```

## Resilience with Polly

```csharp
// Package: Microsoft.Extensions.Http.Polly
builder.Services.AddHttpClient<IGitHubService, GitHubService>()
    .AddTransientHttpErrorPolicy(p =>
        p.WaitAndRetryAsync(3, attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt))))
    .AddTransientHttpErrorPolicy(p =>
        p.CircuitBreakerAsync(5, TimeSpan.FromSeconds(30)));
```

## Delegating Handlers (Middleware for HTTP)

```csharp
builder.Services.AddTransient<AuthorizationHandler>();
builder.Services.AddHttpClient<IApiService, ApiService>()
    .AddHttpMessageHandler<AuthorizationHandler>();
```

Handlers execute outer-to-inner: Auth -> Logging -> Retry -> HTTP call.

## Pitfalls to Avoid

- Creating `HttpClient` with `new` instead of the factory
- Forgetting to set `BaseAddress` (relative URIs fail silently)
- Not setting timeouts (default is 100 seconds)
- Disposing typed `HttpClient` instances (the factory manages their lifecycle)

## Related

- [dependency-injection.md](./dependency-injection.md) -- Service registration
- [configuration-options-pattern.md](./configuration-options-pattern.md) -- API configuration
