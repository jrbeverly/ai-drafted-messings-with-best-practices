# ASP.NET Core Hosting

Configure the application host, services, middleware pipeline, and environments using the minimal hosting model.

## Why It Matters

- The hosting model defines how services are registered, middleware is ordered, and the app runs
- Correct middleware ordering prevents subtle security and routing bugs
- Environment-specific config keeps dev flexible and prod secure

## Minimal Hosting Model (.NET 6+)

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();          // Register services
var app = builder.Build();
app.UseRouting();                           // Configure pipeline
app.MapControllers();                       // Map endpoints
app.Run();
```

## Key Recommendations

- **Configuration precedence** (last wins): `appsettings.json` < `appsettings.{Env}.json` < User Secrets (dev) < Environment variables < Command-line args
- **Service lifetimes**: Singleton (caches, config), Scoped (per-request repos/services), Transient (lightweight utilities)
- **Middleware order matters**:
  1. `UseExceptionHandler` -- first (catches everything)
  2. `UseHsts` / `UseHttpsRedirection`
  3. `UseStaticFiles`
  4. `UseRouting`
  5. `UseCors`
  6. `UseAuthentication` then `UseAuthorization`
  7. Endpoints last
- **Environment-specific middleware**: gate Swagger/DevExceptionPage behind `IsDevelopment()`
- **Startup actions** (migrations, seeding): create a scope before `app.Run()`:
  ```csharp
  using var scope = app.Services.CreateScope();
  var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
  await db.Database.MigrateAsync();
  ```
- **Use `MapGroup`** to organize and apply shared metadata to endpoint groups
- **Integration tests**: override services in `WebApplicationFactory<Program>.ConfigureWebHost`

## Pitfalls to Avoid

- Placing `UseAuthentication` after `UseAuthorization` (auth never happens)
- Adding code after `app.Run()` (never reached -- it blocks)
- Mixing legacy `Startup.cs` and minimal hosting in the same project
- Injecting scoped services from the root provider outside of a scope
