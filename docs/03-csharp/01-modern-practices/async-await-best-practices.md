# Async/Await Best Practices

Asynchronous programming patterns for scalable, non-blocking I/O operations in C#.

## Why It Matters

- Frees threads during I/O waits (database, HTTP, file system), improving scalability
- Built-in cancellation support for graceful shutdown
- Enables parallel execution of independent operations

## Key Recommendations

**Async all the way** -- never block on async code:
```csharp
// WRONG: deadlock risk
var user = GetUserAsync(id).Result;

// CORRECT: async through entire call stack
var user = await GetUserAsync(id);
```

**Suffix async methods with "Async"**, return `Task<T>` or `Task` (never `async void` except event handlers).

**Accept and propagate `CancellationToken`:**
```csharp
public async Task<User> GetUserAsync(int id, CancellationToken ct = default)
{
    var user = await _repository.GetByIdAsync(id, ct);
    ct.ThrowIfCancellationRequested();
    return user;
}
```

**Run independent operations in parallel:**
```csharp
var userTask = _userRepo.GetByIdAsync(userId);
var ordersTask = _orderRepo.GetByUserIdAsync(userId);
await Task.WhenAll(userTask, ordersTask);
```

**Use `ValueTask<T>` for hot paths** where results are frequently cached/synchronous.

**Use `ConfigureAwait(false)` in library code** (not needed in ASP.NET Core apps).

**Stream large collections with `IAsyncEnumerable<T>`:**
```csharp
await foreach (var user in GetUsersStreamAsync(ct)) { ProcessUser(user); }
```

## Pitfalls to Avoid

- Calling `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()` on async code
- Using `async void` (exceptions are unobservable)
- Wrapping I/O in `Task.Run` (wastes a thread pool thread)
- Unnecessary async/await when you can return the Task directly
- Wrapping synchronous code in `Task.FromResult` and marking it async
