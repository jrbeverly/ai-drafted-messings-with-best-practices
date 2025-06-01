# TempData

Temporary data between requests. Redirect scenarios. Cookie or session provider. Short-lived data.

## Principle

TempData survives one redirect. Useful for success messages, errors. Stored in cookies or session.

## Basic TempData Usage

```csharp
// Set TempData
app.MapPost("/users", async (CreateUserRequest request, IUserService userService) =>
{
    var user = await userService.CreateUserAsync(request);

    // Store success message in TempData
    context.TempData["SuccessMessage"] = $"User {user.Name} created successfully";

    // Redirect
    return Results.RedirectToRoute("GetUser", new { id = user.Id });
});

// Read TempData after redirect
app.MapGet("/users/{id}", (string id, HttpContext context) =>
{
    var successMessage = context.TempData["SuccessMessage"] as string;

    if (successMessage is not null)
    {
        // Display message (consumed after read)
        Console.WriteLine(successMessage);
    }

    // Get user...
    return Results.Ok(new { Id = id });
}).WithName("GetUser");
```

## TempData Configuration

```csharp
var builder = WebApplication.CreateBuilder(args);

// Cookie-based TempData (default in .NET 6+)
builder.Services.AddControllersWithViews();

// Or explicit configuration
builder.Services.AddSession(); // If using session provider
builder.Services.AddSessionStateTempDataProvider();

// Cookie provider (default, no session required)
builder.Services.AddCookieTempDataProvider(options =>
{
    options.Cookie.Name = ".MyApp.TempData";
    options.Cookie.IsEssential = true;
});

var app = builder.Build();

app.UseSession(); // Only if using session provider

app.Run();
```

## TempData Lifecycle

```
Request 1: Set TempData
    ↓
TempData["Message"] = "Success"
    ↓
Redirect to Request 2
    ↓
Request 2: Read TempData
    ↓
var message = TempData["Message"]
    ↓
TempData deleted (consumed)
    ↓
Request 3: TempData no longer available
```

## Keep TempData

```csharp
app.MapGet("/page1", (HttpContext context) =>
{
    context.TempData["Message"] = "Important message";

    return Results.Ok("Set message");
});

app.MapGet("/page2", (HttpContext context) =>
{
    var message = context.TempData["Message"] as string;

    // Keep for next request (don't delete)
    context.TempData.Keep("Message");

    return Results.Ok(new { Message = message, Kept = true });
});

app.MapGet("/page3", (HttpContext context) =>
{
    var message = context.TempData["Message"] as string;

    // Message still available because we called Keep()
    return Results.Ok(new { Message = message });
});
```

## Peek (Read Without Consuming)

```csharp
app.MapGet("/peek", (HttpContext context) =>
{
    // Read without marking for deletion
    var message = context.TempData.Peek("Message") as string;

    // Message still available on next request
    return Results.Ok(new { Message = message });
});
```

## TempData vs Session vs ViewData

| Feature | TempData | Session | ViewData/ViewBag |
|---------|----------|---------|------------------|
| **Lifetime** | One redirect | Until session expires | Single request |
| **Storage** | Cookie or session | Server-side (memory/Redis) | Server memory |
| **Use Case** | Success/error messages | User-specific data | View-specific data |
| **Survives Redirect** | Yes | Yes | No |
| **Size Limit** | 4 KB (cookie) | Large | Large |

## Success/Error Messages Pattern

```csharp
public static class TempDataExtensions
{
    public static void SetSuccessMessage(this ITempDataDictionary tempData, string message)
    {
        tempData["SuccessMessage"] = message;
    }

    public static void SetErrorMessage(this ITempDataDictionary tempData, string message)
    {
        tempData["ErrorMessage"] = message;
    }

    public static void SetWarningMessage(this ITempDataDictionary tempData, string message)
    {
        tempData["WarningMessage"] = message;
    }

    public static string? GetSuccessMessage(this ITempDataDictionary tempData)
    {
        return tempData["SuccessMessage"] as string;
    }

    public static string? GetErrorMessage(this ITempDataDictionary tempData)
    {
        return tempData["ErrorMessage"] as string;
    }
}

// Usage
app.MapPost("/users", async (
    CreateUserRequest request,
    IUserService userService,
    HttpContext context) =>
{
    try
    {
        var user = await userService.CreateUserAsync(request);

        context.TempData.SetSuccessMessage($"User {user.Name} created successfully");

        return Results.RedirectToRoute("GetUser", new { id = user.Id });
    }
    catch (Exception ex)
    {
        context.TempData.SetErrorMessage($"Failed to create user: {ex.Message}");

        return Results.RedirectToRoute("CreateUserForm");
    }
});

app.MapGet("/users/create", (HttpContext context) =>
{
    var errorMessage = context.TempData.GetErrorMessage();

    return Results.Ok(new
    {
        Form = "Create User Form",
        Error = errorMessage
    });
}).WithName("CreateUserForm");
```

## Complex Object Storage

```csharp
public static class TempDataExtensions
{
    public static void SetObject<T>(this ITempDataDictionary tempData, string key, T value)
    {
        tempData[key] = JsonSerializer.Serialize(value);
    }

    public static T? GetObject<T>(this ITempDataDictionary tempData, string key)
    {
        var json = tempData[key] as string;
        return json is null ? default : JsonSerializer.Deserialize<T>(json);
    }
}

// Usage
app.MapPost("/wizard/step1", (Step1Data data, HttpContext context) =>
{
    context.TempData.SetObject("WizardData", data);

    return Results.Redirect("/wizard/step2");
});

app.MapGet("/wizard/step2", (HttpContext context) =>
{
    var step1Data = context.TempData.GetObject<Step1Data>("WizardData");

    if (step1Data is null)
    {
        return Results.Redirect("/wizard/step1"); // Start over
    }

    return Results.Ok(new { Step = 2, PreviousData = step1Data });
});
```

## Multi-Step Form Wizard

```csharp
public record WizardState
{
    public string? Step1Data { get; init; }
    public string? Step2Data { get; init; }
    public string? Step3Data { get; init; }
    public int CurrentStep { get; init; }
}

app.MapGet("/wizard/start", (HttpContext context) =>
{
    var state = new WizardState { CurrentStep = 1 };
    context.TempData.SetObject("WizardState", state);

    return Results.Ok("Step 1");
});

app.MapPost("/wizard/step1", (Step1Request request, HttpContext context) =>
{
    var state = context.TempData.GetObject<WizardState>("WizardState")
        ?? new WizardState();

    state = state with
    {
        Step1Data = request.Data,
        CurrentStep = 2
    };

    context.TempData.SetObject("WizardState", state);

    return Results.Redirect("/wizard/step2");
});

app.MapPost("/wizard/step2", (Step2Request request, HttpContext context) =>
{
    var state = context.TempData.GetObject<WizardState>("WizardState");

    if (state is null || state.CurrentStep != 2)
    {
        return Results.Redirect("/wizard/start");
    }

    state = state with
    {
        Step2Data = request.Data,
        CurrentStep = 3
    };

    context.TempData.SetObject("WizardState", state);

    return Results.Redirect("/wizard/step3");
});

app.MapPost("/wizard/complete", async (
    HttpContext context,
    IWizardService wizardService) =>
{
    var state = context.TempData.GetObject<WizardState>("WizardState");

    if (state is null || state.CurrentStep != 3)
    {
        return Results.Redirect("/wizard/start");
    }

    // Process complete wizard
    await wizardService.ProcessAsync(state);

    context.TempData.SetSuccessMessage("Wizard completed successfully");

    return Results.Redirect("/dashboard");
});
```

## TempData with API Endpoints

```csharp
// TempData in APIs (rare, but possible)
app.MapPost("/api/users", async (
    CreateUserRequest request,
    IUserService userService,
    HttpContext context) =>
{
    var user = await userService.CreateUserAsync(request);

    // Store notification for next request
    context.TempData["Notification"] = "User created";

    return Results.Created($"/api/users/{user.Id}", user);
});

app.MapGet("/api/notifications", (HttpContext context) =>
{
    var notification = context.TempData["Notification"] as string;

    return Results.Ok(new { Notification = notification });
});
```

## Testing TempData

```csharp
public class TempDataTests
{
    [Fact]
    public async Task CreateUser_SetsTempData()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient(new WebApplicationFactoryClientOptions
        {
            AllowAutoRedirect = false // Don't follow redirects
        });

        // Act
        var request = new CreateUserRequest { Name = "John Doe", Email = "john@example.com" };
        var response = await client.PostAsJsonAsync("/users", request);

        // Assert
        Assert.Equal(HttpStatusCode.Redirect, response.StatusCode);

        // Follow redirect
        var redirectResponse = await client.GetAsync(response.Headers.Location);
        redirectResponse.EnsureSuccessStatusCode();

        // TempData should be available after redirect
        var content = await redirectResponse.Content.ReadAsStringAsync();
        Assert.Contains("created successfully", content);
    }
}
```

## Cookie vs Session Provider

```csharp
// Cookie provider (default, stateless)
builder.Services.AddCookieTempDataProvider(options =>
{
    options.Cookie.Name = ".MyApp.TempData";
    options.Cookie.IsEssential = true;
    options.Cookie.HttpOnly = true;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
});

// Session provider (requires session middleware)
builder.Services.AddSession();
builder.Services.AddSessionStateTempDataProvider();

app.UseSession();

// Trade-offs:
// Cookie: No server state, 4 KB limit, fast
// Session: Larger data, server-side storage, requires session middleware
```

## Guidelines

**When to Use TempData:**
- Success/error messages after redirect
- Multi-step forms/wizards
- Data between POST-REDIRECT-GET pattern
- Temporary notifications

**When NOT to Use TempData:**
- Persistent user data (use database)
- Long-lived data (use session)
- Data within same request (use ViewData/HttpContext.Items)
- Large objects (use session or database)

**Best Practices:**
- Keep TempData small (< 1 KB)
- Use for one-time messages only
- Check for null (may have expired)
- Clear TempData after use (automatic)
- Use typed extensions for complex objects

**Security:**
- TempData is not encrypted (use HTTPS)
- Don't store sensitive data (passwords, tokens)
- Validate data from TempData (don't trust)
- Use secure cookies (HttpOnly, Secure flags)

## Benefits

Simple. Easy to pass data across redirects.

Automatic. Cleared after read.

Flexible. Cookie or session storage.

Common Pattern. Success/error messages.

## Related

- [session-state.md](./session-state.md) - Session storage
- [cookie-authentication.md](../05-auth/cookie-authentication.md) - Cookies
- [minimal-api-basics.md](../04-aspnet-web-api/minimal-api-basics.md) - Endpoints
- [error-handling.md](../04-aspnet-web-api/error-handling.md) - Error messages
