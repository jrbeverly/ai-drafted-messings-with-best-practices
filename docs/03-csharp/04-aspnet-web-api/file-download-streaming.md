# File Download and Streaming

FileResult, stream responses. Range requests. Download attachments. Large file streaming.

## Principle

Stream files efficiently. Support range requests. Set proper headers. Memory-efficient downloads.

## Basic File Download

```csharp
// Download file as attachment
app.MapGet("/download/{id}", (string id) =>
{
    var filePath = Path.Combine("files", $"{id}.pdf");

    if (!File.Exists(filePath))
    {
        return Results.NotFound();
    }

    var bytes = File.ReadAllBytes(filePath);
    return Results.File(bytes, "application/pdf", "document.pdf");
});
```

## Stream Large Files

```csharp
// Stream file (memory efficient)
app.MapGet("/download-large/{id}", async (string id) =>
{
    var filePath = Path.Combine("files", $"{id}.zip");

    if (!File.Exists(filePath))
    {
        return Results.NotFound();
    }

    var stream = File.OpenRead(filePath);
    return Results.Stream(stream, "application/zip", "archive.zip");
});
```

## Physical File Result

```csharp
// Serve physical file
app.MapGet("/files/{filename}", (string filename) =>
{
    var filePath = Path.Combine(Directory.GetCurrentDirectory(), "uploads", filename);

    if (!File.Exists(filePath))
    {
        return Results.NotFound();
    }

    return Results.File(filePath, "application/octet-stream", filename);
});
```

## Set Content-Disposition

```csharp
// Force download (attachment) vs inline
app.MapGet("/download/attachment/{id}", (string id, HttpContext context) =>
{
    var filePath = Path.Combine("files", $"{id}.pdf");

    if (!File.Exists(filePath))
    {
        return Results.NotFound();
    }

    // Force download
    context.Response.Headers.ContentDisposition = "attachment; filename=\"document.pdf\"";

    var stream = File.OpenRead(filePath);
    return Results.Stream(stream, "application/pdf");
});

app.MapGet("/view/inline/{id}", (string id, HttpContext context) =>
{
    var filePath = Path.Combine("files", $"{id}.pdf");

    if (!File.Exists(filePath))
    {
        return Results.NotFound();
    }

    // View in browser
    context.Response.Headers.ContentDisposition = "inline; filename=\"document.pdf\"";

    var stream = File.OpenRead(filePath);
    return Results.Stream(stream, "application/pdf");
});
```

## Range Requests (Partial Content)

```csharp
app.MapGet("/stream/{id}", async (string id, HttpContext context) =>
{
    var filePath = Path.Combine("files", $"{id}.mp4");

    if (!File.Exists(filePath))
    {
        return Results.NotFound();
    }

    var fileInfo = new FileInfo(filePath);
    var fileLength = fileInfo.Length;

    // Check if client requests a range
    var rangeHeader = context.Request.Headers.Range.ToString();

    if (string.IsNullOrEmpty(rangeHeader))
    {
        // No range, send entire file
        var stream = File.OpenRead(filePath);
        return Results.Stream(stream, "video/mp4", "video.mp4", fileLength);
    }

    // Parse range header: "bytes=0-1023"
    var range = rangeHeader.Replace("bytes=", "").Split('-');
    var start = long.Parse(range[0]);
    var end = range.Length > 1 && !string.IsNullOrEmpty(range[1])
        ? long.Parse(range[1])
        : fileLength - 1;

    var contentLength = end - start + 1;

    // Set 206 Partial Content response
    context.Response.StatusCode = 206;
    context.Response.Headers.ContentType = "video/mp4";
    context.Response.Headers.ContentLength = contentLength;
    context.Response.Headers.ContentRange = $"bytes {start}-{end}/{fileLength}";
    context.Response.Headers.AcceptRanges = "bytes";

    // Stream partial content
    var fileStream = File.OpenRead(filePath);
    fileStream.Seek(start, SeekOrigin.Begin);

    var buffer = new byte[81920]; // 80 KB buffer
    var remaining = contentLength;

    while (remaining > 0)
    {
        var bytesToRead = (int)Math.Min(buffer.Length, remaining);
        var bytesRead = await fileStream.ReadAsync(buffer, 0, bytesToRead);

        if (bytesRead == 0)
            break;

        await context.Response.Body.WriteAsync(buffer, 0, bytesRead);
        remaining -= bytesRead;
    }

    return Results.Empty;
});
```

## Download from S3

```csharp
// Install: AWSSDK.S3

public class S3FileDownloadService
{
    private readonly IAmazonS3 _s3Client;
    private readonly string _bucketName;

    public S3FileDownloadService(IAmazonS3 s3Client, IConfiguration configuration)
    {
        _s3Client = s3Client;
        _bucketName = configuration["AWS:S3:BucketName"]!;
    }

    public async Task<Stream> DownloadFileAsync(string key)
    {
        var request = new GetObjectRequest
        {
            BucketName = _bucketName,
            Key = key
        };

        var response = await _s3Client.GetObjectAsync(request);
        return response.ResponseStream;
    }

    public async Task<GetObjectMetadataResponse> GetFileMetadataAsync(string key)
    {
        return await _s3Client.GetObjectMetadataAsync(_bucketName, key);
    }
}

// Register
builder.Services.AddSingleton<IAmazonS3, AmazonS3Client>();
builder.Services.AddScoped<S3FileDownloadService>();

// Usage
app.MapGet("/download-s3/{key}", async (
    string key,
    S3FileDownloadService downloadService) =>
{
    try
    {
        var metadata = await downloadService.GetFileMetadataAsync(key);
        var stream = await downloadService.DownloadFileAsync(key);

        return Results.Stream(
            stream,
            metadata.Headers.ContentType,
            Path.GetFileName(key));
    }
    catch (AmazonS3Exception)
    {
        return Results.NotFound();
    }
});
```

## Presigned URL (S3)

```csharp
public class S3PresignedUrlService
{
    private readonly IAmazonS3 _s3Client;
    private readonly string _bucketName;

    public S3PresignedUrlService(IAmazonS3 s3Client, IConfiguration configuration)
    {
        _s3Client = s3Client;
        _bucketName = configuration["AWS:S3:BucketName"]!;
    }

    public string GeneratePresignedUrl(string key, TimeSpan expiration)
    {
        var request = new GetPreSignedUrlRequest
        {
            BucketName = _bucketName,
            Key = key,
            Expires = DateTime.UtcNow.Add(expiration)
        };

        return _s3Client.GetPreSignedURL(request);
    }
}

// Usage
app.MapGet("/download-link/{key}", (
    string key,
    S3PresignedUrlService urlService) =>
{
    // Generate URL valid for 1 hour
    var url = urlService.GeneratePresignedUrl(key, TimeSpan.FromHours(1));

    return Results.Ok(new { DownloadUrl = url });
});

// Client can download directly from S3:
// GET https://bucket.s3.amazonaws.com/key?signature=...
```

## CSV Export

```csharp
app.MapGet("/export/users/csv", async (IUserService userService) =>
{
    var users = await userService.GetAllUsersAsync();

    var csv = new StringBuilder();

    // Header
    csv.AppendLine("Id,Name,Email,CreatedAt");

    // Rows
    foreach (var user in users)
    {
        csv.AppendLine($"{user.Id},{user.Name},{user.Email},{user.CreatedAt:yyyy-MM-dd}");
    }

    var bytes = Encoding.UTF8.GetBytes(csv.ToString());

    return Results.File(bytes, "text/csv", "users.csv");
});
```

## Excel Export

```csharp
// Install: EPPlus

app.MapGet("/export/users/excel", async (IUserService userService) =>
{
    var users = await userService.GetAllUsersAsync();

    using var package = new ExcelPackage();
    var worksheet = package.Workbook.Worksheets.Add("Users");

    // Header
    worksheet.Cells[1, 1].Value = "Id";
    worksheet.Cells[1, 2].Value = "Name";
    worksheet.Cells[1, 3].Value = "Email";
    worksheet.Cells[1, 4].Value = "Created At";

    // Data
    for (int i = 0; i < users.Length; i++)
    {
        var user = users[i];
        worksheet.Cells[i + 2, 1].Value = user.Id;
        worksheet.Cells[i + 2, 2].Value = user.Name;
        worksheet.Cells[i + 2, 3].Value = user.Email;
        worksheet.Cells[i + 2, 4].Value = user.CreatedAt;
    }

    // Auto-fit columns
    worksheet.Cells.AutoFitColumns();

    var bytes = package.GetAsByteArray();

    return Results.File(
        bytes,
        "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        "users.xlsx");
});
```

## PDF Generation

```csharp
// Install: QuestPDF

app.MapGet("/report/{id}/pdf", async (string id, IReportService reportService) =>
{
    var reportData = await reportService.GetReportDataAsync(id);

    var document = Document.Create(container =>
    {
        container.Page(page =>
        {
            page.Size(PageSizes.A4);
            page.Margin(2, Unit.Centimetre);

            page.Header()
                .Text("Report")
                .FontSize(20);

            page.Content()
                .Column(column =>
                {
                    column.Item().Text(reportData.Title).FontSize(16);
                    column.Item().Text(reportData.Content);
                });

            page.Footer()
                .AlignCenter()
                .Text(x =>
                {
                    x.Span("Page ");
                    x.CurrentPageNumber();
                });
        });
    });

    var bytes = document.GeneratePdf();

    return Results.File(bytes, "application/pdf", $"report-{id}.pdf");
});
```

## Zip Multiple Files

```csharp
// Install: System.IO.Compression

app.MapGet("/download/bulk", async (string[] ids, IFileService fileService) =>
{
    var memoryStream = new MemoryStream();

    using (var archive = new ZipArchive(memoryStream, ZipArchiveMode.Create, leaveOpen: true))
    {
        foreach (var id in ids)
        {
            var file = await fileService.GetFileAsync(id);

            if (file is not null)
            {
                var entry = archive.CreateEntry(file.Name);

                using var entryStream = entry.Open();
                await file.Stream.CopyToAsync(entryStream);
            }
        }
    }

    memoryStream.Seek(0, SeekOrigin.Begin);

    return Results.Stream(memoryStream, "application/zip", "files.zip");
});
```

## Throttled Download

```csharp
public class ThrottledStream : Stream
{
    private readonly Stream _baseStream;
    private readonly int _maxBytesPerSecond;
    private long _bytesTransferred;
    private readonly Stopwatch _stopwatch = Stopwatch.StartNew();

    public ThrottledStream(Stream baseStream, int maxBytesPerSecond)
    {
        _baseStream = baseStream;
        _maxBytesPerSecond = maxBytesPerSecond;
    }

    public override async Task<int> ReadAsync(byte[] buffer, int offset, int count, CancellationToken cancellationToken)
    {
        var bytesRead = await _baseStream.ReadAsync(buffer, offset, count, cancellationToken);
        _bytesTransferred += bytesRead;

        // Calculate expected time for bytes transferred
        var expectedSeconds = _bytesTransferred / (double)_maxBytesPerSecond;
        var actualSeconds = _stopwatch.Elapsed.TotalSeconds;

        if (actualSeconds < expectedSeconds)
        {
            var delay = (int)((expectedSeconds - actualSeconds) * 1000);
            await Task.Delay(delay, cancellationToken);
        }

        return bytesRead;
    }

    // Implement other Stream members...
    public override bool CanRead => _baseStream.CanRead;
    public override bool CanSeek => _baseStream.CanSeek;
    public override bool CanWrite => _baseStream.CanWrite;
    public override long Length => _baseStream.Length;
    public override long Position
    {
        get => _baseStream.Position;
        set => _baseStream.Position = value;
    }

    public override void Flush() => _baseStream.Flush();
    public override int Read(byte[] buffer, int offset, int count) => _baseStream.Read(buffer, offset, count);
    public override long Seek(long offset, SeekOrigin origin) => _baseStream.Seek(offset, origin);
    public override void SetLength(long value) => _baseStream.SetLength(value);
    public override void Write(byte[] buffer, int offset, int count) => _baseStream.Write(buffer, offset, count);
}

app.MapGet("/download/throttled/{id}", (string id) =>
{
    var filePath = Path.Combine("files", $"{id}.zip");
    var fileStream = File.OpenRead(filePath);

    // Throttle to 1 MB/s
    var throttledStream = new ThrottledStream(fileStream, 1024 * 1024);

    return Results.Stream(throttledStream, "application/zip", "download.zip");
});
```

## Cache Control for Downloads

```csharp
app.MapGet("/download/cached/{id}", (string id, HttpContext context) =>
{
    var filePath = Path.Combine("files", $"{id}.pdf");

    if (!File.Exists(filePath))
    {
        return Results.NotFound();
    }

    var fileInfo = new FileInfo(filePath);

    // Set cache headers
    context.Response.Headers.CacheControl = "public, max-age=3600";
    context.Response.Headers.ETag = $"\"{fileInfo.LastWriteTimeUtc.Ticks}\"";
    context.Response.Headers.LastModified = fileInfo.LastWriteTimeUtc.ToString("R");

    // Check if client has cached version
    var ifNoneMatch = context.Request.Headers.IfNoneMatch.ToString();
    if (ifNoneMatch == $"\"{fileInfo.LastWriteTimeUtc.Ticks}\"")
    {
        return Results.StatusCode(304); // Not Modified
    }

    var stream = File.OpenRead(filePath);
    return Results.Stream(stream, "application/pdf", "document.pdf");
});
```

## Testing File Downloads

```csharp
public class FileDownloadTests
{
    [Fact]
    public async Task DownloadFile_ValidId_ReturnsFile()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        // Act
        var response = await client.GetAsync("/download/test-file");

        // Assert
        response.EnsureSuccessStatusCode();
        Assert.Equal("application/pdf", response.Content.Headers.ContentType?.ToString());

        var bytes = await response.Content.ReadAsByteArrayAsync();
        Assert.True(bytes.Length > 0);
    }

    [Fact]
    public async Task DownloadFile_InvalidId_Returns404()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        // Act
        var response = await client.GetAsync("/download/nonexistent");

        // Assert
        Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
    }
}
```

## Guidelines

**Streaming:**
- Use Stream for large files (> 1 MB)
- Don't buffer entire file in memory
- Support range requests for video/audio
- Close streams properly (use using or await DisposeAsync)

**Headers:**
- Set Content-Type appropriately
- Use Content-Disposition for downloads
- Add Content-Length when known
- Set Cache-Control for static files

**Performance:**
- Stream directly from source (S3, database)
- Use async I/O
- Consider compression
- Implement throttling for large files

**Security:**
- Validate file paths (no path traversal)
- Check user permissions
- Don't expose internal paths
- Scan files for malware

## Benefits

Efficient. Stream large files without memory issues.

Compatible. Support range requests for media.

Flexible. Multiple file sources (disk, S3, database).

Standard. Proper HTTP headers and status codes.

## Related

- [file-upload-aspnet.md](./file-upload-aspnet.md) - File uploads
- [static-files-middleware.md](./static-files-middleware.md) - Static files
- [response-caching-middleware.md](../06-aspnet-advanced/response-caching-middleware.md) - Caching
- [s3-integration.md](../../06-aws/s3-integration.md) - AWS S3
