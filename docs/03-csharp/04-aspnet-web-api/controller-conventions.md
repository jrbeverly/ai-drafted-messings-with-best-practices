# Controller Conventions

Standard patterns for RESTful API controllers: naming, structure, async, versioning, and error handling.

## Why It Matters

- Consistent conventions make APIs predictable and self-documenting
- Following REST standards reduces integration friction for consumers
- Proper async patterns prevent thread pool starvation under load

## Key Patterns

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase  // Plural resource name
{
    private readonly IUserService _userService; // Constructor injection

    [HttpGet]                                   // GET api/users
    public async Task<ActionResult<User[]>> GetUsers(CancellationToken ct) { }

    [HttpGet("{id}")]                           // GET api/users/{id}
    [ProducesResponseType(typeof(User), 200)]
    [ProducesResponseType(404)]
    public async Task<ActionResult<User>> GetUser(string id) { }

    [HttpPost]                                  // POST api/users -> 201
    public async Task<ActionResult<User>> CreateUser(CreateUserRequest request)
        => CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);

    [HttpPut("{id}")]                           // PUT api/users/{id} -> 204
    public async Task<IActionResult> UpdateUser(string id, UpdateUserRequest request)
        => NoContent();

    [HttpDelete("{id}")]                        // DELETE api/users/{id} -> 204
    public async Task<IActionResult> DeleteUser(string id) => NoContent();
}
```

**Naming:** `UsersController` (plural), not `UserController`. Actions: `GetUser`, `CreateUser`.

**Sub-resources:** `[HttpGet("{userId}/orders")]` for nested resources.

**Authorization:** `[Authorize]` on controller, `[AllowAnonymous]` on public actions, `[Authorize(Policy = "...")]` on specific actions.

## Pitfalls to Avoid

- Singular controller names (`UserController` instead of `UsersController`)
- Synchronous action methods (blocks thread pool threads)
- Fat controllers with business logic (delegate to services)
- Missing `CancellationToken` on long-running operations
- Using service locator pattern (`[FromServices] IServiceProvider`) instead of constructor injection
