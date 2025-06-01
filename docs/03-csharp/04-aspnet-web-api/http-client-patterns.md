# HTTP Client Patterns

Proper HttpClient usage. Avoid socket exhaustion. Typed clients, named clients, resilience patterns.

## Principle

Never create HttpClient directly. Use IHttpClientFactory. Configure once, reuse. Handle transient failures.

## Socket Exhaustion Problem

```csharp
// ❌ BAD - Creates new HttpClient every call
public async Task<string> CallApiSlowAsync()
{
    using var client = new HttpClient();  // Socket exhaustion!
    return await client.GetStringAsync("https://api.example.com/data");
}

// Problem: Each HttpClient holds a socket connection
// Sockets aren't released immediately after disposal
// High-traffic apps run out of available sockets
```

## Named HttpClient

```csharp
// Configure
builder.Services.AddHttpClient("ExternalApi", client =>
{
    client.BaseAddress = new Uri("https://api.example.com");
    client.Timeout = TimeSpan.FromSeconds(30);
    client.DefaultRequestHeaders.Add("User-Agent", "MyApp/1.0");
    client.DefaultRequestHeaders.Add("Accept", "application/json");
});

// Usage
public class ExternalApiService
{
    private readonly IHttpClientFactory _httpClientFactory;

    public ExternalApiService(IHttpClientFactory httpClientFactory)
    {
        _httpClientFactory = httpClientFactory;
    }

    public async Task<string> GetDataAsync()
    {
        var client = _httpClientFactory.CreateClient("ExternalApi");

        var response = await client.GetAsync("/data");
        response.EnsureSuccessStatusCode();

        return await response.Content.ReadAsStringAsync();
    }
}
```

## Typed HttpClient

```csharp
// Define typed client
public class GitHubApiClient
{
    private readonly HttpClient _httpClient;

    public GitHubApiClient(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<GitHubUser?> GetUserAsync(string username)
    {
        var response = await _httpClient.GetAsync($"/users/{username}");

        if (!response.IsSuccessStatusCode)
        {
            return null;
        }

        return await response.Content.ReadFromJsonAsync<GitHubUser>();
    }

    public async Task<GitHubRepo[]> GetUserReposAsync(string username)
    {
        var response = await _httpClient.GetAsync($"/users/{username}/repos");
        response.EnsureSuccessStatusCode();

        return await response.Content.ReadFromJsonAsync<GitHubRepo[]>()
            ?? Array.Empty<GitHubRepo>();
    }
}

// Register
builder.Services.AddHttpClient<GitHubApiClient>(client =>
{
    client.BaseAddress = new Uri("https://api.github.com");
    client.DefaultRequestHeaders.Add("User-Agent", "MyApp");
    client.DefaultRequestHeaders.Add("Accept", "application/vnd.github.v3+json");
});

// Inject and use
public class UserService
{
    private readonly GitHubApiClient _githubClient;

    public UserService(GitHubApiClient githubClient)
    {
        _githubClient = githubClient;
    }

    public async Task<GitHubUser?> GetGitHubUserAsync(string username)
    {
        return await _githubClient.GetUserAsync(username);
    }
}
```

## Connection Pooling

```csharp
builder.Services.AddHttpClient("ExternalApi")
    .ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
    {
        PooledConnectionLifetime = TimeSpan.FromMinutes(5),
        PooledConnectionIdleTimeout = TimeSpan.FromMinutes(2),
        MaxConnectionsPerServer = 20
    });
```

## Retry Policy (Polly)

```csharp
// Install: dotnet add package Microsoft.Extensions.Http.Polly

builder.Services.AddHttpClient("ExternalApi")
    .AddTransientHttpErrorPolicy(policy =>
        policy.WaitAndRetryAsync(
            retryCount: 3,
            sleepDurationProvider: retryAttempt =>
                TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)), // Exponential backoff
            onRetry: (outcome, timespan, retryCount, context) =>
            {
                Console.WriteLine($"Retry {retryCount} after {timespan}");
            }));
```

## Circuit Breaker

```csharp
builder.Services.AddHttpClient("ExternalApi")
    .AddTransientHttpErrorPolicy(policy =>
        policy.CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(30)));

// After 5 consecutive failures, circuit opens
// Requests fail fast for 30 seconds
// Then half-open state to test if service recovered
```

## Combined Policies

```csharp
var retryPolicy = Policy
    .HandleResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
    .WaitAndRetryAsync(3, retryAttempt =>
        TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));

var circuitBreakerPolicy = Policy
    .HandleResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
    .CircuitBreakerAsync(5, TimeSpan.FromSeconds(30));

var timeoutPolicy = Policy.TimeoutAsync<HttpResponseMessage>(
    TimeSpan.FromSeconds(10));

builder.Services.AddHttpClient("ExternalApi")
    .AddPolicyHandler(retryPolicy)
    .AddPolicyHandler(circuitBreakerPolicy)
    .AddPolicyHandler(timeoutPolicy);
```

## Request/Response Logging

```csharp
public class LoggingHttpMessageHandler : DelegatingHandler
{
    private readonly ILogger<LoggingHttpMessageHandler> _logger;

    public LoggingHttpMessageHandler(ILogger<LoggingHttpMessageHandler> logger)
    {
        _logger = logger;
    }

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        _logger.LogInformation(
            "Request: {Method} {Url}",
            request.Method,
            request.RequestUri);

        var stopwatch = Stopwatch.StartNew();

        var response = await base.SendAsync(request, cancellationToken);

        stopwatch.Stop();

        _logger.LogInformation(
            "Response: {Method} {Url} - {StatusCode} - {Duration}ms",
            request.Method,
            request.RequestUri,
            (int)response.StatusCode,
            stopwatch.ElapsedMilliseconds);

        return response;
    }
}

// Register
builder.Services.AddTransient<LoggingHttpMessageHandler>();

builder.Services.AddHttpClient("ExternalApi")
    .AddHttpMessageHandler<LoggingHttpMessageHandler>();
```

## Authorization Header

```csharp
public class AuthorizationHttpMessageHandler : DelegatingHandler
{
    private readonly ITokenService _tokenService;

    public AuthorizationHttpMessageHandler(ITokenService tokenService)
    {
        _tokenService = tokenService;
    }

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        // Add auth token to every request
        var token = await _tokenService.GetAccessTokenAsync();

        request.Headers.Authorization =
            new AuthenticationHeaderValue("Bearer", token);

        return await base.SendAsync(request, cancellationToken);
    }
}

builder.Services.AddHttpClient("SecureApi")
    .AddHttpMessageHandler<AuthorizationHttpMessageHandler>();
```

## Typed Request/Response

```csharp
public record GetUserRequest(string Username);

public record GetUserResponse(string Id, string Name, string Email);

public class TypedApiClient
{
    private readonly HttpClient _httpClient;

    public TypedApiClient(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<GetUserResponse?> GetUserAsync(GetUserRequest request)
    {
        var response = await _httpClient.GetAsync($"/users/{request.Username}");

        if (!response.IsSuccessStatusCode)
        {
            return null;
        }

        return await response.Content.ReadFromJsonAsync<GetUserResponse>();
    }

    public async Task<GetUserResponse> CreateUserAsync(CreateUserRequest request)
    {
        var response = await _httpClient.PostAsJsonAsync("/users", request);
        response.EnsureSuccessStatusCode();

        return await response.Content.ReadFromJsonAsync<GetUserResponse>()
            ?? throw new InvalidOperationException("Failed to deserialize response");
    }
}
```

## Refit (Alternative)

```csharp
// Install: dotnet add package Refit

// Define API interface
public interface IGitHubApi
{
    [Get("/users/{username}")]
    Task<GitHubUser> GetUserAsync(string username);

    [Get("/users/{username}/repos")]
    Task<GitHubRepo[]> GetUserReposAsync(string username);

    [Post("/users")]
    Task<GitHubUser> CreateUserAsync([Body] CreateUserRequest request);
}

// Register
builder.Services
    .AddRefitClient<IGitHubApi>()
    .ConfigureHttpClient(client =>
    {
        client.BaseAddress = new Uri("https://api.github.com");
        client.DefaultRequestHeaders.Add("User-Agent", "MyApp");
    })
    .AddTransientHttpErrorPolicy(policy =>
        policy.WaitAndRetryAsync(3, _ => TimeSpan.FromSeconds(2)));

// Use
public class UserService
{
    private readonly IGitHubApi _githubApi;

    public UserService(IGitHubApi githubApi)
    {
        _githubApi = githubApi;
    }

    public async Task<GitHubUser> GetUserAsync(string username)
    {
        return await _githubApi.GetUserAsync(username);
    }
}
```

## Testing HttpClient

```csharp
public class ExternalApiClientTests
{
    [Fact]
    public async Task GetDataAsync_ReturnsData()
    {
        // Arrange
        var handler = new MockHttpMessageHandler();
        handler.When("/data")
               .Respond("application/json", "{\"value\":\"test\"}");

        var httpClient = handler.ToHttpClient();
        httpClient.BaseAddress = new Uri("https://api.example.com");

        var client = new ExternalApiClient(httpClient);

        // Act
        var result = await client.GetDataAsync();

        // Assert
        Assert.Equal("test", result.Value);
    }
}

// Or with IHttpClientFactory mock
[Fact]
public async Task Service_CallsApi()
{
    // Arrange
    var factory = Substitute.For<IHttpClientFactory>();
    var httpClient = new HttpClient(new MockHttpMessageHandler());

    factory.CreateClient("ExternalApi").Returns(httpClient);

    var service = new ExternalApiService(factory);

    // Act
    await service.GetDataAsync();

    // Assert
    factory.Received(1).CreateClient("ExternalApi");
}
```

## Error Handling

```csharp
public class ExternalApiClient
{
    public async Task<ApiResponse<User>> GetUserAsync(string id)
    {
        try
        {
            var response = await _httpClient.GetAsync($"/users/{id}");

            if (response.StatusCode == HttpStatusCode.NotFound)
            {
                return ApiResponse<User>.Error("User not found");
            }

            response.EnsureSuccessStatusCode();

            var user = await response.Content.ReadFromJsonAsync<User>();

            return ApiResponse<User>.Success(user!);
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "HTTP request failed");
            return ApiResponse<User>.Error("Failed to connect to API");
        }
        catch (TaskCanceledException ex)
        {
            _logger.LogError(ex, "Request timeout");
            return ApiResponse<User>.Error("Request timeout");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unexpected error");
            return ApiResponse<User>.Error("Unexpected error occurred");
        }
    }
}
```

## Guidelines

**HttpClient Creation:**
- ✅ Use IHttpClientFactory
- ✅ Named or typed clients
- ❌ Never create HttpClient directly in services

**Configuration:**
- Set base address once
- Configure timeout appropriately
- Add default headers (User-Agent, Accept)
- Configure connection pooling

**Resilience:**
- Implement retry with exponential backoff
- Use circuit breaker for unstable services
- Set appropriate timeouts
- Handle transient failures gracefully

**Logging:**
- Log request/response
- Include duration
- Log errors and retries
- Sanitize sensitive data

**Testing:**
- Mock IHttpClientFactory
- Use test message handlers
- Test error scenarios
- Verify retry behavior

## Benefits

Efficient. Connection pooling, no socket exhaustion.

Resilient. Retry, circuit breaker, timeout.

Testable. Easy to mock and test.

Flexible. Named, typed, delegating handlers.

## Related

- [dependency-injection.md](../03-dotnet/dependency-injection.md) - Registering clients
- [minimal-api-basics.md](./minimal-api-basics.md) - Calling from endpoints
- [logging-structured-serilog.md](../03-dotnet/logging-structured-serilog.md) - Request logging
- [unit-testing-best-practices.md](../02-testing/unit-testing-best-practices.md) - Testing clients
