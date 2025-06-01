# Request/Response Pipeline

Understand middleware execution order, short-circuiting, and the HttpContext lifecycle.

## Why It Matters

- Middleware order determines security, performance, and correctness
- Incorrect ordering causes silent failures (CORS not applied, auth skipped)
- Short-circuiting avoids unnecessary work for static files and cached responses

## Correct Middleware Order

```csharp
app.UseExceptionHandler("/error");   // 1. Catch all exceptions
app.UseHsts();                       // 2. HTTPS enforcement
app.UseHttpsRedirection();           // 3. Redirect HTTP to HTTPS
app.UseStaticFiles();                // 4. Short-circuit for files
app.UseRouting();                    // 5. Match endpoint
app.UseCors();                       // 6. After routing (needs endpoint metadata)
app.UseAuthentication();             // 7. Identify user
app.UseAuthorization();              // 8. Check permissions
app.UseRateLimiter();                // 9. Throttle
app.UseOutputCache();                // 10. Serve cached responses
app.UseSession();                    // 11. Load session
// Custom middleware here             // 12.
app.MapControllers();                // 13. Terminal -- execute endpoint
```

## Execution Model

Middleware runs in order on the way in and **reverse order** on the way out:

```
Request → M1 → M2 → M3 → Endpoint → M3 → M2 → M1 → Response
```

Each middleware can **short-circuit** by not calling `next()` (e.g., static files, auth failures, cached responses).

## Key Recommendations

- **Exception handler must be first** -- otherwise exceptions in earlier middleware are uncaught
- **Static files before routing** -- avoids running auth/CORS for CSS/JS/images
- **Authentication before authorization** -- identity must exist before permission checks
- **Custom logging middleware**: place after `UseAuthentication` (to log user info) but before endpoints
- **Branch pipelines** with `app.Map("/api", ...)` for path-specific middleware stacks
- **Terminal middleware** (`app.Run(...)`) does not call `next()` -- nothing registered after it executes

## HttpContext Lifecycle

- Created when request arrives, disposed when response completes
- `HttpContext.Items` dictionary lives for one request only
- Access from services via `IHttpContextAccessor` (scoped lifetime)
- Never store references to `HttpContext` beyond the request

## Pitfalls to Avoid

- CORS after endpoints (never executes)
- Authentication after authorization (identity is null)
- Exception handler not first (middleware exceptions uncaught)
- Adding middleware after `app.Run()` or `app.MapControllers()` (unreachable)
