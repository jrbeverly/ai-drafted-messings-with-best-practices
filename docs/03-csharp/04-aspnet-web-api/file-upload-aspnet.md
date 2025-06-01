# File Upload in ASP.NET Core

IFormFile, multipart form data. Streaming uploads. Large file handling. File validation.

## Principle

Handle file uploads securely. Stream large files. Validate before processing. Store safely.

## Basic File Upload

```csharp
// Single file upload
app.MapPost("/upload", async (IFormFile file) =>
{
    if (file is null || file.Length == 0)
    {
        return Results.BadRequest("No file uploaded");
    }

    var uploadsPath = Path.Combine(Directory.GetCurrentDirectory(), "uploads");
    Directory.CreateDirectory(uploadsPath);

    var filePath = Path.Combine(uploadsPath, file.FileName);

    using (var stream = File.Create(filePath))
    {
        await file.CopyToAsync(stream);
    }

    return Results.Ok(new
    {
        FileName = file.FileName,
        Size = file.Length,
        ContentType = file.ContentType
    });
});
```

## Multiple Files Upload

```csharp
app.MapPost("/upload-multiple", async (IFormFileCollection files) =>
{
    if (files is null || files.Count == 0)
    {
        return Results.BadRequest("No files uploaded");
    }

    var uploadsPath = Path.Combine(Directory.GetCurrentDirectory(), "uploads");
    Directory.CreateDirectory(uploadsPath);

    var uploadedFiles = new List<object>();

    foreach (var file in files)
    {
        var filePath = Path.Combine(uploadsPath, file.FileName);

        using (var stream = File.Create(filePath))
        {
            await file.CopyToAsync(stream);
        }

        uploadedFiles.Add(new
        {
            FileName = file.FileName,
            Size = file.Length
        });
    }

    return Results.Ok(new
    {
        Count = files.Count,
        Files = uploadedFiles
    });
});
```

## File with Additional Data

```csharp
public record UploadRequest
{
    public required IFormFile File { get; init; }
    public required string Title { get; init; }
    public required string Description { get; init; }
}

app.MapPost("/upload-with-data", async ([FromForm] UploadRequest request) =>
{
    if (request.File is null || request.File.Length == 0)
    {
        return Results.BadRequest("No file uploaded");
    }

    var uploadsPath = Path.Combine(Directory.GetCurrentDirectory(), "uploads");
    Directory.CreateDirectory(uploadsPath);

    var filePath = Path.Combine(uploadsPath, request.File.FileName);

    using (var stream = File.Create(filePath))
    {
        await request.File.CopyToAsync(stream);
    }

    return Results.Ok(new
    {
        request.Title,
        request.Description,
        FileName = request.File.FileName,
        Size = request.File.Length
    });
});
```

## File Validation

```csharp
public class FileUploadValidator
{
    private static readonly string[] AllowedExtensions = { ".jpg", ".jpeg", ".png", ".pdf" };
    private const long MaxFileSize = 10 * 1024 * 1024; // 10 MB

    public static (bool IsValid, string? Error) ValidateFile(IFormFile file)
    {
        if (file is null || file.Length == 0)
        {
            return (false, "No file provided");
        }

        // Check file size
        if (file.Length > MaxFileSize)
        {
            return (false, $"File size exceeds {MaxFileSize / 1024 / 1024} MB limit");
        }

        // Check file extension
        var extension = Path.GetExtension(file.FileName).ToLowerInvariant();
        if (!AllowedExtensions.Contains(extension))
        {
            return (false, $"File type {extension} not allowed. Allowed types: {string.Join(", ", AllowedExtensions)}");
        }

        // Check content type
        if (!IsContentTypeValid(file.ContentType, extension))
        {
            return (false, "File content type does not match extension");
        }

        return (true, null);
    }

    private static bool IsContentTypeValid(string contentType, string extension)
    {
        return extension switch
        {
            ".jpg" or ".jpeg" => contentType == "image/jpeg",
            ".png" => contentType == "image/png",
            ".pdf" => contentType == "application/pdf",
            _ => false
        };
    }
}

app.MapPost("/upload-validated", async (IFormFile file) =>
{
    var (isValid, error) = FileUploadValidator.ValidateFile(file);

    if (!isValid)
    {
        return Results.BadRequest(new { Error = error });
    }

    // Process file...
    var uploadsPath = Path.Combine(Directory.GetCurrentDirectory(), "uploads");
    Directory.CreateDirectory(uploadsPath);

    var filePath = Path.Combine(uploadsPath, file.FileName);

    using (var stream = File.Create(filePath))
    {
        await file.CopyToAsync(stream);
    }

    return Results.Ok(new { FileName = file.FileName });
});
```

## Stream Large Files

```csharp
// For large files, stream directly to storage
app.MapPost("/upload-stream", async (HttpContext context) =>
{
    if (!context.Request.HasFormContentType)
    {
        return Results.BadRequest("Not a multipart form");
    }

    var form = await context.Request.ReadFormAsync();
    var file = form.Files.FirstOrDefault();

    if (file is null || file.Length == 0)
    {
        return Results.BadRequest("No file uploaded");
    }

    var uploadsPath = Path.Combine(Directory.GetCurrentDirectory(), "uploads");
    Directory.CreateDirectory(uploadsPath);

    var filePath = Path.Combine(uploadsPath, file.FileName);

    // Stream directly (memory efficient)
    await using var fileStream = File.Create(filePath);
    await using var uploadStream = file.OpenReadStream();
    await uploadStream.CopyToAsync(fileStream);

    return Results.Ok(new
    {
        FileName = file.FileName,
        Size = file.Length
    });
});
```

## Disable Request Size Limits

```csharp
// Increase request body size limit
builder.Services.Configure<FormOptions>(options =>
{
    options.ValueLengthLimit = int.MaxValue;
    options.MultipartBodyLengthLimit = long.MaxValue; // 2 GB default
    options.MultipartHeadersLengthLimit = int.MaxValue;
});

builder.WebHost.ConfigureKestrel(options =>
{
    options.Limits.MaxRequestBodySize = 100 * 1024 * 1024; // 100 MB
});

// Per-endpoint disable
app.MapPost("/upload-large", async (IFormFile file) =>
{
    // File processing...
    return Results.Ok();
})
.DisableRequestSizeLimit(); // No limit for this endpoint
```

## Secure File Names

```csharp
public static class FileHelper
{
    public static string GetSafeFileName(string fileName)
    {
        // Remove path information
        fileName = Path.GetFileName(fileName);

        // Remove invalid characters
        var invalidChars = Path.GetInvalidFileNameChars();
        fileName = string.Join("_", fileName.Split(invalidChars));

        // Generate unique name to prevent overwriting
        var extension = Path.GetExtension(fileName);
        var nameWithoutExtension = Path.GetFileNameWithoutExtension(fileName);
        var uniqueName = $"{nameWithoutExtension}_{Guid.NewGuid()}{extension}";

        return uniqueName;
    }
}

app.MapPost("/upload-secure", async (IFormFile file) =>
{
    var safeFileName = FileHelper.GetSafeFileName(file.FileName);

    var uploadsPath = Path.Combine(Directory.GetCurrentDirectory(), "uploads");
    Directory.CreateDirectory(uploadsPath);

    var filePath = Path.Combine(uploadsPath, safeFileName);

    using (var stream = File.Create(filePath))
    {
        await file.CopyToAsync(stream);
    }

    return Results.Ok(new { FileName = safeFileName });
});
```

## Upload to S3 (AWS)

```csharp
// Install: AWSSDK.S3

public class S3FileUploadService
{
    private readonly IAmazonS3 _s3Client;
    private readonly string _bucketName;

    public S3FileUploadService(IAmazonS3 s3Client, IConfiguration configuration)
    {
        _s3Client = s3Client;
        _bucketName = configuration["AWS:S3:BucketName"]!;
    }

    public async Task<string> UploadFileAsync(IFormFile file)
    {
        var key = $"uploads/{Guid.NewGuid()}/{file.FileName}";

        using var stream = file.OpenReadStream();

        var request = new PutObjectRequest
        {
            BucketName = _bucketName,
            Key = key,
            InputStream = stream,
            ContentType = file.ContentType
        };

        await _s3Client.PutObjectAsync(request);

        return key;
    }
}

// Register
builder.Services.AddSingleton<IAmazonS3, AmazonS3Client>();
builder.Services.AddScoped<S3FileUploadService>();

// Usage
app.MapPost("/upload-s3", async (
    IFormFile file,
    S3FileUploadService uploadService) =>
{
    var key = await uploadService.UploadFileAsync(file);

    return Results.Ok(new { Key = key });
});
```

## Chunked Upload (Large Files)

```csharp
public record ChunkUploadRequest
{
    public required int ChunkNumber { get; init; }
    public required int TotalChunks { get; init; }
    public required string FileId { get; init; }
    public required IFormFile Chunk { get; init; }
}

app.MapPost("/upload-chunk", async ([FromForm] ChunkUploadRequest request) =>
{
    var uploadsPath = Path.Combine(Directory.GetCurrentDirectory(), "uploads", "chunks");
    Directory.CreateDirectory(uploadsPath);

    // Save chunk
    var chunkPath = Path.Combine(uploadsPath, $"{request.FileId}_chunk_{request.ChunkNumber}");

    using (var stream = File.Create(chunkPath))
    {
        await request.Chunk.CopyToAsync(stream);
    }

    // If last chunk, combine all chunks
    if (request.ChunkNumber == request.TotalChunks)
    {
        var finalPath = Path.Combine(Directory.GetCurrentDirectory(), "uploads", request.FileId);

        using var finalStream = File.Create(finalPath);

        for (int i = 1; i <= request.TotalChunks; i++)
        {
            var chunkFile = Path.Combine(uploadsPath, $"{request.FileId}_chunk_{i}");

            using var chunkStream = File.OpenRead(chunkFile);
            await chunkStream.CopyToAsync(finalStream);

            // Delete chunk after combining
            File.Delete(chunkFile);
        }

        return Results.Ok(new { Message = "Upload complete", FileId = request.FileId });
    }

    return Results.Ok(new { Message = $"Chunk {request.ChunkNumber}/{request.TotalChunks} uploaded" });
});
```

## Progress Tracking

```csharp
public class FileUploadProgress
{
    public long BytesRead { get; set; }
    public long TotalBytes { get; set; }
    public int PercentComplete => (int)((double)BytesRead / TotalBytes * 100);
}

public class ProgressTrackingStream : Stream
{
    private readonly Stream _innerStream;
    private readonly FileUploadProgress _progress;

    public ProgressTrackingStream(Stream innerStream, FileUploadProgress progress)
    {
        _innerStream = innerStream;
        _progress = progress;
    }

    public override async Task<int> ReadAsync(byte[] buffer, int offset, int count, CancellationToken cancellationToken)
    {
        var bytesRead = await _innerStream.ReadAsync(buffer, offset, count, cancellationToken);
        _progress.BytesRead += bytesRead;
        return bytesRead;
    }

    // Implement other Stream members...
    public override bool CanRead => _innerStream.CanRead;
    public override bool CanSeek => _innerStream.CanSeek;
    public override bool CanWrite => _innerStream.CanWrite;
    public override long Length => _innerStream.Length;
    public override long Position
    {
        get => _innerStream.Position;
        set => _innerStream.Position = value;
    }

    public override void Flush() => _innerStream.Flush();
    public override int Read(byte[] buffer, int offset, int count) => _innerStream.Read(buffer, offset, count);
    public override long Seek(long offset, SeekOrigin origin) => _innerStream.Seek(offset, origin);
    public override void SetLength(long value) => _innerStream.SetLength(value);
    public override void Write(byte[] buffer, int offset, int count) => _innerStream.Write(buffer, offset, count);
}
```

## Testing File Uploads

```csharp
public class FileUploadTests
{
    [Fact]
    public async Task UploadFile_ValidFile_Returns200()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        var content = new MultipartFormDataContent();
        var fileContent = new ByteArrayContent(Encoding.UTF8.GetBytes("test file content"));
        fileContent.Headers.ContentType = new System.Net.Http.Headers.MediaTypeHeaderValue("text/plain");
        content.Add(fileContent, "file", "test.txt");

        // Act
        var response = await client.PostAsync("/upload", content);

        // Assert
        response.EnsureSuccessStatusCode();
    }

    [Fact]
    public async Task UploadFile_NoFile_Returns400()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        var content = new MultipartFormDataContent();

        // Act
        var response = await client.PostAsync("/upload", content);

        // Assert
        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
    }
}
```

## Guidelines

**Security:**
- Validate file size and type
- Use safe file names (no path traversal)
- Scan for viruses/malware
- Store outside web root
- Set Content-Disposition header

**Performance:**
- Stream large files (don't buffer in memory)
- Use async I/O
- Limit concurrent uploads
- Consider chunked uploads for very large files

**Storage:**
- Use cloud storage (S3, Azure Blob) for production
- Generate unique file names
- Organize by date/user
- Implement cleanup for old files

**Validation:**
- Check file extension
- Verify content type
- Validate file size
- Check file signature (magic numbers)

## Benefits

Flexible. Multiple upload strategies.

Secure. Validation and safe storage.

Efficient. Streaming for large files.

Scalable. Cloud storage integration.

## Related

- [file-download-streaming.md](./file-download-streaming.md) - File downloads
- [static-files-middleware.md](./static-files-middleware.md) - Serving static files
- [multipart-form-data.md](./multipart-form-data.md) - Form data handling
- [s3-integration.md](../../06-aws/s3-integration.md) - S3 uploads
