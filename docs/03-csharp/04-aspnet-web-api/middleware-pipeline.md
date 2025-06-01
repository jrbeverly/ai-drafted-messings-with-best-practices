# Middleware Pipeline

Middleware components process HTTP requests and responses. Order matters. Each middleware can pass to next or short-circuit.

## Principle

Request flows down, response flows up. Order matters. Short-circuit when appropriate.

## Pipeline Order

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// 1. Exception handling (first to catch all errors)
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler("/error");
}

// 2. HTTPS redirection
app.UseHttpsRedirection();

// 3. Static files (short-circuits for static content)
app.UseStaticFiles();

// 4. Routing (matches endpoints)
app.UseRouting();

// 5. CORS (after routing, before auth)
app.UseCors();

// 6. Authentication (before authorization)
app.UseAuthentication();

// 7. Authorization (after authentication)
app.UseAuthorization();

// 8. Custom middleware
app.UseMiddleware<RequestLoggingMiddleware>();

// 9. Endpoints (terminal middleware)
app.MapControllers();
app.MapGet("/", () => "Hello World");

app.Run();
```

## Pipeline Flow

```
Request  → Exception Handler → HTTPS Redirect → Static Files → Routing
           ↓                   ↓                 ↓              ↓
         CORS → Authentication → Authorization → Custom MW → Endpoints
           ↓         ↓              ↓              ↓            ↓
Response ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ←  ←
```

## Built-in Middleware

```csharp
// Exception handling
app.UseExceptionHandler("/error");
app.UseStatusCodePages();

// Security
app.UseHttpsRedirection();
app.UseHsts();

// Static files
app.UseStaticFiles();
app.UseDefaultFiles();

// Routing
app.UseRouting();

// CORS
app.UseCors("MyPolicy");

// Authentication & Authorization
app.UseAuthentication();
app.UseAuthorization();

// Response compression
app.UseResponseCompression();

// Response caching
app.UseResponseCaching();

// Rate limiting
app.UseRateLimiter();

// Endpoints
app.MapControllers();
```

## Inline Middleware

```csharp
// Use - runs for all requests
app.Use(async (context, next) =>
{
    Console.WriteLine($"Before: {context.Request.Path}");

    await next(context);

    Console.WriteLine($"After: {context.Response.StatusCode}");
});

// Run - terminal middleware (no next)
app.Run(async context =>
{
    await context.Response.WriteAsync("Terminal middleware");
});

// Map - branch pipeline based on path
app.Map("/admin", adminApp =>
{
    adminApp.UseAuthentication();
    adminApp.UseAuthorization();
    adminApp.Run(async context =>
    {
        await context.Response.WriteAsync("Admin section");
    });
});

// MapWhen - conditional branching
app.MapWhen(
    context => context.Request.Query.ContainsKey("preview"),
    previewApp =>
    {
        previewApp.Run(async context =>
        {
            await context.Response.WriteAsync("Preview mode");
        });
    });
```

## Request Logging Middleware

```csharp
app.Use(async (context, next) =>
{
    var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();
    var stopwatch = Stopwatch.StartNew();

    logger.LogInformation(
        "Request: {Method} {Path}",
        context.Request.Method,
        context.Request.Path);

    await next(context);

    stopwatch.Stop();

    logger.LogInformation(
        "Response: {Method} {Path} - {StatusCode} - {Duration}ms",
        context.Request.Method,
        context.Request.Path,
        context.Response.StatusCode,
        stopwatch.ElapsedMilliseconds);
});
```

## Short-Circuiting

```csharp
// Static files short-circuits if file found
app.UseStaticFiles(); // Handles GET /style.css → short-circuits

// Custom short-circuit
app.Use(async (context, next) =>
{
    if (context.Request.Path == "/health")
    {
        // Short-circuit - don't call next()
        context.Response.StatusCode = 200;
        await context.Response.WriteAsync("Healthy");
        return;
    }

    await next(context);
});

// API key validation (short-circuit if invalid)
app.Use(async (context, next) =>
{
    var apiKey = context.Request.Headers["X-API-Key"].ToString();

    if (string.IsNullOrEmpty(apiKey) || !IsValidApiKey(apiKey))
    {
        context.Response.StatusCode = 401;
        await context.Response.WriteAsJsonAsync(new { error = "Invalid API key" });
        return; // Short-circuit
    }

    await next(context);
});
```

## Conditional Middleware

```csharp
// Only in development
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
    app.UseSwagger();
    app.UseSwaggerUI();
}

// Only in production
if (app.Environment.IsProduction())
{
    app.UseHsts();
    app.UseResponseCompression();
}

// Feature flag
if (builder.Configuration.GetValue<bool>("Features:EnableCaching"))
{
    app.UseResponseCaching();
}
```

## Middleware with Options

```csharp
// Static files with options
app.UseStaticFiles(new StaticFileOptions
{
    FileProvider = new PhysicalFileProvider(
        Path.Combine(builder.Environment.ContentRootPath, "public")),
    RequestPath = "/static"
});

// Response compression with options
app.UseResponseCompression(new ResponseCompressionOptions
{
    EnableForHttps = true
});

// CORS with options
app.UseCors(policy =>
{
    policy.WithOrigins("https://example.com")
          .AllowAnyMethod()
          .AllowAnyHeader();
});
```

## Branching Pipelines

```csharp
// API branch
app.Map("/api", apiApp =>
{
    apiApp.UseCors("ApiPolicy");
    apiApp.UseAuthentication();
    apiApp.UseAuthorization();
    apiApp.UseRateLimiter();

    apiApp.MapGet("/users", GetUsers);
    apiApp.MapPost("/users", CreateUser);
});

// Admin branch
app.Map("/admin", adminApp =>
{
    adminApp.UseAuthentication();
    adminApp.UseAuthorization();

    adminApp.MapGet("/dashboard", GetDashboard);
    adminApp.MapGet("/users", GetAllUsers);
});

// Public branch (no auth)
app.Map("/public", publicApp =>
{
    publicApp.UseResponseCaching();

    publicApp.MapGet("/books", GetPublicBooks);
    publicApp.MapGet("/authors", GetPublicAuthors);
});
```

## Error Handling Order

```csharp
// GOOD - Exception handler first
app.UseExceptionHandler("/error");
app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();

// BAD - Exception handler after other middleware
app.UseHttpsRedirection();
app.UseAuthentication();
app.UseExceptionHandler("/error"); // ❌ Won't catch auth errors
```

## Performance Middleware

```csharp
// Response compression (before endpoints)
app.UseResponseCompression();

// Response caching (before endpoints)
app.UseResponseCaching();

// Request timing
app.Use(async (context, next) =>
{
    var stopwatch = Stopwatch.StartNew();

    context.Response.OnStarting(() =>
    {
        stopwatch.Stop();
        context.Response.Headers.Add(
            "X-Response-Time",
            stopwatch.ElapsedMilliseconds.ToString());
        return Task.CompletedTask;
    });

    await next(context);
});
```

## Request ID Middleware

```csharp
app.Use(async (context, next) =>
{
    // Generate or extract request ID
    var requestId = context.Request.Headers["X-Request-ID"].FirstOrDefault()
        ?? Guid.NewGuid().ToString();

    context.Items["RequestId"] = requestId;

    // Add to response headers
    context.Response.OnStarting(() =>
    {
        context.Response.Headers.Add("X-Request-ID", requestId);
        return Task.CompletedTask;
    });

    await next(context);
});
```

## Common Mistakes

```csharp
// ❌ BAD: CORS after routing
app.UseRouting();
app.UseCors();  // Too late, won't work correctly

// ✅ GOOD: CORS before authorization
app.UseRouting();
app.UseCors();
app.UseAuthorization();

// ❌ BAD: Authorization before authentication
app.UseAuthorization();
app.UseAuthentication();  // Auth needs to run first!

// ✅ GOOD: Authentication before authorization
app.UseAuthentication();
app.UseAuthorization();

// ❌ BAD: Static files after routing
app.UseRouting();
app.UseStaticFiles();  // Won't short-circuit efficiently

// ✅ GOOD: Static files before routing
app.UseStaticFiles();
app.UseRouting();
```

## Guidelines

**Ordering Principles:**
1. Exception handling first
2. HTTPS redirection early
3. Static files before routing (for performance)
4. Routing before auth
5. CORS after routing, before auth
6. Authentication before authorization
7. Custom middleware after framework middleware
8. Endpoints last

**Best Practices:**
- Keep middleware focused (single responsibility)
- Short-circuit early when possible
- Avoid expensive operations in middleware
- Use conditional middleware for environments
- Log requests/responses at pipeline boundaries

**Performance:**
- Static files early (short-circuits)
- Response compression after routing
- Caching before expensive operations
- Rate limiting after auth

**Security:**
- Exception handler first (catches all errors)
- HTTPS redirection early
- Authentication/authorization in correct order
- Validate inputs early

## Benefits

Composable. Build pipeline from components.

Ordered. Predictable request/response flow.

Flexible. Conditional and branching pipelines.

Performant. Short-circuits when appropriate.

## Related

- [custom-middleware.md](./custom-middleware.md) - Building middleware
- [minimal-api-basics.md](./minimal-api-basics.md) - Endpoint definition
- [cors-configuration.md](./cors-configuration.md) - CORS middleware
- [authentication.md](../../09-security/authentication.md) - Auth middleware
