# Policy Services

Centralized authorization and validation logic injected into handlers. Policies check conditions and throw domain exceptions on failure (fail-fast pattern).

## Why It Matters

- Eliminates duplicated authorization logic across endpoints
- Handlers read like business requirements -- no nested if/else
- Easily testable in isolation via mocking

## Interface Pattern

All methods start with `Require` and throw on failure:

```csharp
public interface IBookPolicy
{
    Task<Book> RequireBookExistsAsync(string libraryId, string bookId);
    void RequireBookBelongsToLibrary(Book book, string libraryId);
    void RequireBookIsAvailable(Book book);
}
```

## Naming Convention

```
Task<T> Require{Entity}ExistsAsync(...)    // fetch + validate existence
void Require{Entity}BelongsTo{Owner}(...)  // validate ownership
void Require{Entity}{State}(...)           // validate state
void RequireUser{Permission}(...)          // validate permission
```

## Common Policy Types

- **IAuthContextPolicy** -- extract/validate user identity from `ClaimsPrincipal`
- **ILibraryPolicy** -- verify library exists, active membership, role checks
- **IBookPolicy** -- verify book exists, ownership, availability

## Usage in Handlers

Call policies in order: general to specific, cheap checks first.

```csharp
var userId = authPolicy.RequireUserId(user);
var membership = await libraryPolicy.RequireActiveMembershipAsync(userId, libraryId);
libraryPolicy.RequireUserIsOwnerOrAdmin(membership);
var book = await bookPolicy.RequireBookExistsAsync(libraryId, bookId);
// ... business logic
```

## Registration

Policies are stateless -- register as **singleton**:

```csharp
services.AddSingleton<IAuthContextPolicy, AuthContextPolicy>();
services.AddSingleton<ILibraryPolicy, LibraryPolicy>();
```

## Testing

```csharp
mockLibraryPolicy
    .Setup(p => p.RequireActiveMembershipAsync("user-123", "lib-456"))
    .ThrowsAsync(new ForbiddenException("User is not a member"));
```

## Key Recommendations

- Return the validated entity from `Require*ExistsAsync` methods
- Throw specific exceptions: `NotFoundException`, `ForbiddenException`, etc.
- Include entity type and ID in error messages
- No business logic in policies -- only validation and authorization

## When to Create vs. Not

| Create a policy | Keep inline |
|-----------------|-------------|
| Auth logic used by 2+ endpoints | Simple null check |
| Complex validation with data access | Endpoint-specific validation |
| Consistent error messages needed | No data access required |

## Pitfalls to Avoid

- Returning booleans instead of throwing (defeats fail-fast pattern)
- Putting business logic in policy services
- Registering as scoped/transient when policies are stateless
- Forgetting to include meaningful error messages

## Related

- [domain-exceptions.md](./domain-exceptions.md) -- exception types thrown by policies
- [endpoint-organization.md](./endpoint-organization.md) -- handler structure
