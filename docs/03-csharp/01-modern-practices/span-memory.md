# Span and Memory

Span<T> and Memory<T> fundamentals for high-performance, low-allocation C# code. Stack allocation, string processing, parsing, and async-compatible memory slicing.

Keywords: Span, Memory, ReadOnlySpan, ReadOnlyMemory, stackalloc, ArrayPool, MemoryPool, zero-allocation, parsing, performance

## Principle

Use Span<T> for synchronous, stack-scoped views over contiguous memory. Use Memory<T> when the reference must survive across async boundaries. Both avoid copying data by providing windows into existing buffers.

## Span Fundamentals

Span<T> is a ref struct that represents a contiguous region of memory. It can point to managed arrays, native memory, or stack-allocated memory.

```csharp
// Span over an array
int[] numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
Span<int> span = numbers.AsSpan();

// Slice without allocation
Span<int> firstHalf = span[..5];    // [1, 2, 3, 4, 5]
Span<int> lastThree = span[^3..];   // [8, 9, 10]
Span<int> middle = span[2..7];      // [3, 4, 5, 6, 7]

// Modify through span (modifies original array)
firstHalf[0] = 100;
Console.WriteLine(numbers[0]); // 100
```

## ReadOnlySpan

ReadOnlySpan<T> provides read-only access. Used heavily for string processing.

```csharp
// ReadOnlySpan over a string (no allocation)
ReadOnlySpan<char> text = "Hello, World!".AsSpan();
ReadOnlySpan<char> hello = text[..5];   // "Hello"
ReadOnlySpan<char> world = text[7..12]; // "World"

// ReadOnlySpan over an array
int[] data = [10, 20, 30, 40, 50];
ReadOnlySpan<int> readOnly = data.AsSpan();
// readOnly[0] = 99; // Compile error - read only
```

## Stack Allocation with Span

stackalloc combined with Span avoids heap allocation entirely.

```csharp
// Stack-allocate a small buffer
Span<byte> buffer = stackalloc byte[256];
buffer[0] = 0xFF;
buffer[1] = 0xAA;

// Stack-allocate for number formatting
Span<char> formatted = stackalloc char[64];
if (value.TryFormat(formatted, out int charsWritten))
{
    ReadOnlySpan<char> result = formatted[..charsWritten];
    ProcessText(result);
}

// Conditional stack vs heap allocation
int length = GetRequiredLength();
Span<byte> data = length <= 1024
    ? stackalloc byte[length]           // Stack for small
    : new byte[length];                 // Heap for large
```

## String Processing with Span (Avoiding Allocations)

Span-based string parsing eliminates Substring allocations.

```csharp
// BAD: Allocates substrings
public (string Key, string Value) ParseHeaderBad(string header)
{
    int colonIndex = header.IndexOf(':');
    string key = header.Substring(0, colonIndex);          // Allocation
    string value = header.Substring(colonIndex + 1).Trim(); // Allocation
    return (key, value);
}

// GOOD: Zero-allocation parsing
public (ReadOnlySpan<char> Key, ReadOnlySpan<char> Value) ParseHeaderGood(
    ReadOnlySpan<char> header)
{
    int colonIndex = header.IndexOf(':');
    ReadOnlySpan<char> key = header[..colonIndex];
    ReadOnlySpan<char> value = header[(colonIndex + 1)..].Trim();
    return (key, value);
}

// CSV line parsing without allocations
public static void ParseCsvLine(ReadOnlySpan<char> line)
{
    while (!line.IsEmpty)
    {
        int commaIndex = line.IndexOf(',');
        ReadOnlySpan<char> field;

        if (commaIndex >= 0)
        {
            field = line[..commaIndex];
            line = line[(commaIndex + 1)..];
        }
        else
        {
            field = line;
            line = [];
        }

        ProcessField(field.Trim());
    }
}

// Number parsing from span
public static bool TryParseCoordinate(ReadOnlySpan<char> input, out double lat, out double lon)
{
    lat = 0;
    lon = 0;

    int commaIndex = input.IndexOf(',');
    if (commaIndex < 0) return false;

    return double.TryParse(input[..commaIndex].Trim(), out lat)
        && double.TryParse(input[(commaIndex + 1)..].Trim(), out lon);
}
```

## Parsing with Span

Complex parsing scenarios benefit from Span to eliminate intermediate allocations.

```csharp
// Parse key=value pairs from query string
public static Dictionary<string, string> ParseQueryString(ReadOnlySpan<char> query)
{
    var result = new Dictionary<string, string>();

    // Skip leading '?'
    if (query.Length > 0 && query[0] == '?')
        query = query[1..];

    while (!query.IsEmpty)
    {
        ReadOnlySpan<char> pair;
        int ampIndex = query.IndexOf('&');

        if (ampIndex >= 0)
        {
            pair = query[..ampIndex];
            query = query[(ampIndex + 1)..];
        }
        else
        {
            pair = query;
            query = [];
        }

        int eqIndex = pair.IndexOf('=');
        if (eqIndex >= 0)
        {
            string key = pair[..eqIndex].ToString();
            string value = pair[(eqIndex + 1)..].ToString();
            result[key] = value;
        }
    }

    return result;
}

// Binary data parsing
public static (int Id, long Timestamp) ParseHeader(ReadOnlySpan<byte> data)
{
    int id = BitConverter.ToInt32(data[..4]);
    long timestamp = BitConverter.ToInt64(data[4..12]);
    return (id, timestamp);
}
```

## Memory for Async

Memory<T> is the async-compatible counterpart to Span<T>. It can be stored on the heap and passed across await boundaries.

```csharp
// Span cannot be used across await
// Memory<T> can
public async Task ProcessDataAsync(Memory<byte> data)
{
    // Write to memory
    data.Span[0] = 0xFF;

    // Pass across await
    await stream.WriteAsync(data);

    // Still valid after await
    int firstByte = data.Span[0];
}

// ReadOnlyMemory<T> for read-only async scenarios
public async Task<int> CountNonZeroAsync(ReadOnlyMemory<byte> data)
{
    await Task.Delay(1); // Simulating async work

    int count = 0;
    foreach (var b in data.Span)
    {
        if (b != 0) count++;
    }
    return count;
}

// Slicing Memory is also allocation-free
public async Task ProcessChunksAsync(Memory<byte> buffer, int chunkSize)
{
    int offset = 0;
    while (offset < buffer.Length)
    {
        int remaining = Math.Min(chunkSize, buffer.Length - offset);
        Memory<byte> chunk = buffer.Slice(offset, remaining);
        await ProcessChunkAsync(chunk);
        offset += remaining;
    }
}
```

## ArrayPool and MemoryPool

Rent buffers instead of allocating to reduce GC pressure.

```csharp
using System.Buffers;

// ArrayPool - rent and return arrays
public async Task<byte[]> ReadAndTransformAsync(Stream stream, int length)
{
    byte[] rented = ArrayPool<byte>.Shared.Rent(length);
    try
    {
        int bytesRead = await stream.ReadAsync(rented.AsMemory(0, length));
        Transform(rented.AsSpan(0, bytesRead));

        var result = new byte[bytesRead];
        rented.AsSpan(0, bytesRead).CopyTo(result);
        return result;
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(rented, clearArray: true);
    }
}

// MemoryPool - rent IMemoryOwner<T> with automatic disposal
public async Task ProcessStreamAsync(Stream stream)
{
    using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(4096);
    Memory<byte> buffer = owner.Memory[..4096];

    int bytesRead;
    while ((bytesRead = await stream.ReadAsync(buffer)) > 0)
    {
        ProcessChunk(buffer[..bytesRead].Span);
    }
}

// Custom pool size for specific workloads
ArrayPool<byte> customPool = ArrayPool<byte>.Create(
    maxArrayLength: 1024 * 1024,  // 1 MB max
    maxArraysPerBucket: 50);
```

## Benchmarks: Allocation Reduction

```csharp
// Scenario: Parse 10,000 "key=value" strings

// | Method               | Mean    | Allocated |
// |----------------------|---------|-----------|
// | String.Split + new   | 450 us  | 960 KB    |
// | Span-based parsing   | 120 us  | 0 B       |

// Scenario: Format 100,000 integers to strings

// | Method               | Mean    | Allocated |
// |----------------------|---------|-----------|
// | ToString()           | 8.5 ms  | 3.2 MB    |
// | TryFormat + Span     | 2.1 ms  | 0 B       |

// Scenario: Read 1 MB in 4 KB chunks

// | Method               | Mean    | Allocated |
// |----------------------|---------|-----------|
// | new byte[4096] each  | 1.2 ms  | 1.0 MB    |
// | ArrayPool rental     | 0.9 ms  | 0 B       |
```

## When to Use Span vs Memory

| Criteria | Span<T> | Memory<T> |
|---|---|---|
| Stack-only (ref struct) | Yes | No (regular struct) |
| Can cross await | No | Yes |
| Can be stored in fields | No | Yes |
| Can be in collections | No | Yes |
| Performance | Fastest | Slightly slower |
| stackalloc compatible | Yes | No |
| String slicing | ReadOnlySpan<char> | ReadOnlyMemory<char> |

Decision rule: Use Span<T> by default. Switch to Memory<T> only when you need to store the reference in a field, pass across await, or put it in a collection.

## Best Practices

**DO:**
- Use Span<T> for synchronous, short-lived views over data
- Use Memory<T> when data must cross async boundaries
- Use stackalloc for small temporary buffers (< 1 KB)
- Use ArrayPool for larger temporary buffers
- Use ReadOnlySpan<char> for string parsing to avoid Substring allocations

**DON'T:**
- Use stackalloc for large or variable-size buffers (stack overflow risk)
- Store Span<T> in fields or collections (it is a ref struct)
- Use Span<T> across await boundaries (compiler error)
- Forget to return rented arrays to ArrayPool (memory leak)
- Optimize with Span where allocation is not the bottleneck (measure first)

## Guidelines

**Essential:**
- Understand the Span (synchronous) vs Memory (async) distinction
- Use ReadOnlySpan<char> for zero-allocation string slicing and parsing
- Always return rented buffers in a finally block or using statement

**Recommended:**
- Use stackalloc + Span for small formatting buffers (< 512 bytes typical)
- Use ArrayPool for temporary buffers in loops and high-throughput paths
- Prefer ReadOnlySpan/ReadOnlyMemory when mutation is not needed

**Advanced:**
- Use MemoryPool for IMemoryOwner-based lifetime management
- Combine Span with inline arrays for zero-allocation fixed buffers
- Profile with BenchmarkDotNet to measure allocation reduction impact

## Benefits

Zero-allocation slicing. Span and Memory provide views without copying data.

Stack allocation. stackalloc with Span avoids heap entirely for small buffers.

Async compatibility. Memory<T> brings allocation-free patterns to async code paths.

Reduced GC pressure. ArrayPool and MemoryPool eliminate repeated buffer allocations in hot loops.

## Related

- [collections-performance.md](./collections-performance.md) - Collection type selection and ArrayPool
- [linq-optimization.md](./linq-optimization.md) - Span-based alternatives to LINQ in hot paths
