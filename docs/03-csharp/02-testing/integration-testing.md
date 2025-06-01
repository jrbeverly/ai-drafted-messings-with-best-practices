# Integration Testing

Verify that multiple components work together correctly using real implementations, not mocks.

## Why It Matters

- Catches issues that unit tests miss (wiring, serialization, middleware)
- Validates API contracts (request/response formats, status codes)
- Tests real database access, service-layer orchestration, and auth flows

## Key Tools

- **Package:** `Microsoft.AspNetCore.Mvc.Testing`
- **`WebApplicationFactory<Program>`** -- spins up an in-memory test server
- **`IClassFixture<T>`** -- shares factory across tests in a class
- **`IAsyncLifetime`** -- async setup/teardown for test data

## WebApplicationFactory Pattern

```csharp
// Expose Program for testing: add `public partial class Program { }` to Program.cs

public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Swap real services for test doubles
            services.RemoveAll<IAmazonDynamoDB>();
            services.AddSingleton<IAmazonDynamoDB, InMemoryDynamoDb>();
        });
    }
}
```

## Writing an Integration Test

```csharp
public class UserApiTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client;
    public UserApiTests(CustomWebApplicationFactory factory) => _client = factory.CreateClient();

    [Fact]
    public async Task CreateUser_ValidRequest_ReturnsCreated()
    {
        var response = await _client.PostAsJsonAsync("/api/v1/users",
            new { Email = "john@example.com", Name = "John Doe" });
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
    }
}
```

## What to Integration Test

- API endpoints (full request/response cycle)
- Repository methods against a test database
- Service layer with real repositories, mocked externals only
- Authentication and authorization flows

## Pitfalls to Avoid

- Testing every permutation here instead of unit tests
- Sharing mutable test data between tests (create fresh per test, clean up after)
- Hitting real external APIs (mock or use contract tests)
- Skipping `public partial class Program { }` -- factory cannot discover the app without it

## Related

- [xunit-basics.md](./xunit-basics.md) -- Test fundamentals
- [unit-testing-best-practices.md](./unit-testing-best-practices.md) -- Unit test patterns
