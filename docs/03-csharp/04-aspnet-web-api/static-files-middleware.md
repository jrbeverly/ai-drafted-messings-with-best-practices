# Static Files Middleware

Serve static files. wwwroot directory. Directory browsing. File providers. Cache headers.

## Principle

Serve static files efficiently. Configure caching. Secure by default. Support SPA routing.

## Basic Static Files

```csharp
var builder = WebApplication.CreateBuilder(args);

var app = builder.Build();

// Serve files from wwwroot directory
app.UseStaticFiles();

app.MapGet("/", () => "API is running");

app.Run();

// Files served from wwwroot:
// GET /index.html → wwwroot/index.html
// GET /css/styles.css → wwwroot/css/styles.css
// GET /js/app.js → wwwroot/js/app.js
```

## Custom Directory

```csharp
// Serve files from custom directory
app.UseStaticFiles(new StaticFileOptions
{
    FileProvider = new PhysicalFileProvider(
        Path.Combine(Directory.GetCurrentDirectory(), "StaticFiles")),
    RequestPath = "/files"
});

// Files served from StaticFiles directory:
// GET /files/document.pdf → StaticFiles/document.pdf
```

## Multiple Static File Directories

```csharp
// Serve from multiple directories
app.UseStaticFiles(); // wwwroot

app.UseStaticFiles(new StaticFileOptions
{
    FileProvider = new PhysicalFileProvider(
        Path.Combine(Directory.GetCurrentDirectory(), "uploads")),
    RequestPath = "/uploads"
});

app.UseStaticFiles(new StaticFileOptions
{
    FileProvider = new PhysicalFileProvider(
        Path.Combine(Directory.GetCurrentDirectory(), "images")),
    RequestPath = "/images"
});

// GET / → wwwroot/
// GET /uploads/ → uploads/
// GET /images/ → images/
```

## Content Types

```csharp
var builder = WebApplication.CreateBuilder(args);

// Configure content type provider
var provider = new FileExtensionContentTypeProvider();
provider.Mappings[".webmanifest"] = "application/manifest+json";
provider.Mappings[".wasm"] = "application/wasm";

builder.Services.Configure<StaticFileOptions>(options =>
{
    options.ContentTypeProvider = provider;
});

var app = builder.Build();

app.UseStaticFiles(new StaticFileOptions
{
    ContentTypeProvider = provider
});

app.Run();
```

## Cache Headers

```csharp
app.UseStaticFiles(new StaticFileOptions
{
    OnPrepareResponse = ctx =>
    {
        // Cache static files for 1 year
        ctx.Context.Response.Headers.CacheControl = "public,max-age=31536000";

        // Immutable (never changes)
        ctx.Context.Response.Headers.Append("Cache-Control", "immutable");
    }
});

// Response headers:
// Cache-Control: public,max-age=31536000,immutable
```

## Conditional Caching

```csharp
app.UseStaticFiles(new StaticFileOptions
{
    OnPrepareResponse = ctx =>
    {
        var file = ctx.File;
        var path = ctx.Context.Request.Path.Value;

        // Long cache for versioned files
        if (path!.Contains(".min.") || path.Contains(".bundle."))
        {
            ctx.Context.Response.Headers.CacheControl = "public,max-age=31536000,immutable";
        }
        // Short cache for other files
        else
        {
            ctx.Context.Response.Headers.CacheControl = "public,max-age=3600";
        }
    }
});
```

## Directory Browsing

```csharp
var builder = WebApplication.CreateBuilder(args);

// Enable directory browsing
builder.Services.AddDirectoryBrowser();

var app = builder.Build();

var fileProvider = new PhysicalFileProvider(
    Path.Combine(Directory.GetCurrentDirectory(), "uploads"));

// Serve files
app.UseStaticFiles(new StaticFileOptions
{
    FileProvider = fileProvider,
    RequestPath = "/uploads"
});

// Enable browsing
app.UseDirectoryBrowser(new DirectoryBrowserOptions
{
    FileProvider = fileProvider,
    RequestPath = "/uploads"
});

app.Run();

// Browse directory: GET /uploads/
// Shows list of files and subdirectories
```

## Default Files

```csharp
// Serve default files (index.html, default.html)
app.UseDefaultFiles(); // Must be before UseStaticFiles
app.UseStaticFiles();

// GET / → serves wwwroot/index.html
// GET /about → serves wwwroot/about/index.html

// Custom default files
app.UseDefaultFiles(new DefaultFilesOptions
{
    DefaultFileNames = new List<string> { "index.html", "default.html", "home.html" }
});
app.UseStaticFiles();
```

## File Server (Combined)

```csharp
// FileServer = DefaultFiles + StaticFiles + DirectoryBrowsing
app.UseFileServer(enableDirectoryBrowsing: true);

// Or with options
app.UseFileServer(new FileServerOptions
{
    FileProvider = new PhysicalFileProvider(
        Path.Combine(Directory.GetCurrentDirectory(), "files")),
    RequestPath = "/files",
    EnableDirectoryBrowsing = true
});
```

## SPA Fallback (React, Vue, Angular)

```csharp
// Serve SPA (single-page application)
app.UseStaticFiles();

// Fallback to index.html for client-side routing
app.MapFallbackToFile("index.html");

// Or with custom path
app.MapFallbackToFile("/app/{*path:nonfile}", "index.html");

// Client-side routes work:
// GET /about → index.html (client router handles /about)
// GET /users/123 → index.html (client router handles /users/123)
```

## Authorization for Static Files

```csharp
// Secure static files
app.UseStaticFiles(new StaticFileOptions
{
    FileProvider = new PhysicalFileProvider(
        Path.Combine(Directory.GetCurrentDirectory(), "private")),
    RequestPath = "/private",
    OnPrepareResponse = ctx =>
    {
        // Check if user is authenticated
        if (!ctx.Context.User.Identity?.IsAuthenticated ?? true)
        {
            ctx.Context.Response.StatusCode = 401;
            ctx.Context.Response.ContentLength = 0;
            ctx.Context.Response.Body = Stream.Null;
        }
    }
});

// Better approach: Use endpoint for authorization
app.MapGet("/private/{filename}", async (
    string filename,
    HttpContext context) =>
{
    var filePath = Path.Combine("private", filename);

    if (!File.Exists(filePath))
    {
        return Results.NotFound();
    }

    var stream = File.OpenRead(filePath);
    return Results.Stream(stream, "application/octet-stream", filename);
})
.RequireAuthorization();
```

## Compression

```csharp
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();

    options.MimeTypes = ResponseCompressionDefaults.MimeTypes.Concat(
        new[] { "text/css", "application/javascript", "image/svg+xml" });
});

var app = builder.Build();

app.UseResponseCompression(); // Before UseStaticFiles
app.UseStaticFiles();

app.Run();
```

## HTTPS and Security Headers

```csharp
app.UseStaticFiles(new StaticFileOptions
{
    OnPrepareResponse = ctx =>
    {
        var headers = ctx.Context.Response.Headers;

        // Cache
        headers.CacheControl = "public,max-age=31536000";

        // Security headers
        headers.XContentTypeOptions = "nosniff";
        headers.XFrameOptions = "DENY";

        // CORS (if needed)
        headers.AccessControlAllowOrigin = "*";
    }
});
```

## Exclude Files

```csharp
app.UseStaticFiles(new StaticFileOptions
{
    OnPrepareResponse = ctx =>
    {
        var path = ctx.Context.Request.Path.Value;

        // Block .env, .config, etc.
        if (path!.EndsWith(".env") ||
            path.EndsWith(".config") ||
            path.Contains("/.git/"))
        {
            ctx.Context.Response.StatusCode = 404;
            ctx.Context.Response.ContentLength = 0;
            ctx.Context.Response.Body = Stream.Null;
        }
    }
});
```

## Virtual File Provider

```csharp
// Serve files from embedded resources
var embeddedProvider = new EmbeddedFileProvider(
    typeof(Program).Assembly,
    "MyApp.Resources");

app.UseStaticFiles(new StaticFileOptions
{
    FileProvider = new CompositeFileProvider(
        new PhysicalFileProvider(Path.Combine(Directory.GetCurrentDirectory(), "wwwroot")),
        embeddedProvider),
    RequestPath = ""
});

// Serves from wwwroot first, then embedded resources
```

## Range Requests

```csharp
// Static files middleware automatically supports range requests
app.UseStaticFiles();

// Client request:
// GET /video.mp4
// Range: bytes=0-1023

// Response:
// 206 Partial Content
// Content-Range: bytes 0-1023/1048576
// Content-Length: 1024
```

## Custom Middleware for Static Files

```csharp
public class CustomStaticFileMiddleware
{
    private readonly RequestDelegate _next;
    private readonly string _rootPath;

    public CustomStaticFileMiddleware(RequestDelegate next, string rootPath)
    {
        _next = next;
        _rootPath = rootPath;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var path = context.Request.Path.Value;

        if (path!.StartsWith("/static/"))
        {
            var filePath = Path.Combine(_rootPath, path[8..]); // Remove /static/

            if (File.Exists(filePath))
            {
                var contentType = GetContentType(filePath);
                context.Response.ContentType = contentType;

                await using var stream = File.OpenRead(filePath);
                await stream.CopyToAsync(context.Response.Body);

                return;
            }
        }

        await _next(context);
    }

    private string GetContentType(string path)
    {
        var ext = Path.GetExtension(path).ToLowerInvariant();
        return ext switch
        {
            ".html" => "text/html",
            ".css" => "text/css",
            ".js" => "application/javascript",
            ".json" => "application/json",
            ".png" => "image/png",
            ".jpg" or ".jpeg" => "image/jpeg",
            ".svg" => "image/svg+xml",
            _ => "application/octet-stream"
        };
    }
}
```

## Testing Static Files

```csharp
public class StaticFilesTests
{
    [Fact]
    public async Task GetStaticFile_ReturnsFile()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        // Act
        var response = await client.GetAsync("/index.html");

        // Assert
        response.EnsureSuccessStatusCode();
        Assert.Equal("text/html", response.Content.Headers.ContentType?.MediaType);
    }

    [Fact]
    public async Task GetNonexistentFile_Returns404()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        // Act
        var response = await client.GetAsync("/nonexistent.html");

        // Assert
        Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
    }
}
```

## Production Configuration

```csharp
var builder = WebApplication.CreateBuilder(args);

var app = builder.Build();

if (app.Environment.IsProduction())
{
    // Compress static files
    app.UseResponseCompression();

    // HTTPS redirect
    app.UseHttpsRedirection();

    // HSTS
    app.UseHsts();
}

// Static files with long cache
app.UseStaticFiles(new StaticFileOptions
{
    OnPrepareResponse = ctx =>
    {
        if (app.Environment.IsProduction())
        {
            ctx.Context.Response.Headers.CacheControl = "public,max-age=31536000,immutable";
        }
        else
        {
            ctx.Context.Response.Headers.CacheControl = "no-cache";
        }
    }
});

// SPA fallback
app.MapFallbackToFile("index.html");

app.Run();
```

## Guidelines

**Structure:**
- Use wwwroot for public files
- Separate directories for different file types
- Keep sensitive files outside wwwroot

**Caching:**
- Long cache for versioned/hashed files (1 year)
- Short cache for non-versioned files (1 hour)
- No cache for development
- Add immutable flag for files that never change

**Security:**
- Never serve sensitive files (.env, .config, private keys)
- Use authorization for protected files
- Add security headers
- Disable directory browsing in production

**Performance:**
- Enable compression (Brotli, gzip)
- Use CDN for production
- Support range requests for media
- Optimize images

## Benefits

Fast. Optimized for static file serving.

Cached. Long cache for better performance.

Secure. Files outside wwwroot not accessible.

Compatible. Supports SPAs and range requests.

## Related

- [file-download-streaming.md](./file-download-streaming.md) - File downloads
- [response-compression.md](../03-dotnet/response-compression.md) - Compression
- [cdn-integration.md](../../10-performance/cdn-integration.md) - CDN setup
- [spa-hosting.md](../../02-frontend/spa-hosting.md) - SPA deployment
