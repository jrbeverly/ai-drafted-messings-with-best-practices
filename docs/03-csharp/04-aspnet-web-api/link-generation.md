# Link Generation

Generate URLs from routes. HATEOAS. Link helpers. Type-safe URL generation.

## Principle

Generate URLs programmatically from route names and parameters. Avoid hardcoding URLs. Enable hypermedia-driven APIs.

## CreatedAtAction

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    [HttpGet("{id}", Name = "GetUser")]
    public IActionResult GetUser(string id)
    {
        var user = GetUserById(id);
        return user != null ? Ok(user) : NotFound();
    }

    [HttpPost]
    public IActionResult CreateUser([FromBody] CreateUserRequest request)
    {
        var user = CreateNewUser(request);

        // Generate URL for newly created resource
        return CreatedAtAction(
            nameof(GetUser),
            new { id = user.Id },
            user);
    }
}

// Response:
// Status: 201 Created
// Location: https://api.example.com/api/users/user-123
// Body: { "id": "user-123", "name": "John Doe" }
```

## CreatedAtRoute

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    [HttpGet("{id}", Name = "GetUserById")]
    public IActionResult GetUser(string id) => Ok(GetUserById(id));

    [HttpPost]
    public IActionResult CreateUser([FromBody] CreateUserRequest request)
    {
        var user = CreateNewUser(request);

        // Use named route
        return CreatedAtRoute(
            "GetUserById",
            new { id = user.Id },
            user);
    }
}
```

## Url Helper

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    [HttpGet]
    public IActionResult GetUsers([FromQuery] int page = 1)
    {
        var users = GetUsersPage(page);

        // Generate pagination links
        var nextPageUrl = page < 10
            ? Url.Action(nameof(GetUsers), new { page = page + 1 })
            : null;

        var prevPageUrl = page > 1
            ? Url.Action(nameof(GetUsers), new { page = page - 1 })
            : null;

        return Ok(new
        {
            Data = users,
            Pagination = new
            {
                CurrentPage = page,
                NextPage = nextPageUrl,
                PrevPage = prevPageUrl
            }
        });
    }
}

// Response:
// {
//   "data": [...],
//   "pagination": {
//     "currentPage": 2,
//     "nextPage": "/api/users?page=3",
//     "prevPage": "/api/users?page=1"
//   }
// }
```

## LinkGenerator (Minimal API)

```csharp
app.MapGet("/users/{id}", (string id) =>
{
    var user = GetUserById(id);
    return Results.Ok(user);
})
.WithName("GetUser");

app.MapPost("/users", (
    CreateUserRequest request,
    LinkGenerator linkGenerator,
    HttpContext context) =>
{
    var user = CreateNewUser(request);

    // Generate URL
    var location = linkGenerator.GetPathByName(
        "GetUser",
        new { id = user.Id });

    return Results.Created(location, user);
});
```

## Absolute URLs

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    [HttpPost]
    public IActionResult CreateUser([FromBody] CreateUserRequest request)
    {
        var user = CreateNewUser(request);

        // Generate absolute URL
        var absoluteUrl = Url.Action(
            nameof(GetUser),
            "Users",
            new { id = user.Id },
            Request.Scheme); // Include scheme (http/https)

        return Created(absoluteUrl, user);
    }

    [HttpGet("{id}")]
    public IActionResult GetUser(string id) => Ok(GetUserById(id));
}

// Location: https://api.example.com/api/users/user-123
```

## HATEOAS Links

```csharp
// Link helper class
public record Link
{
    public string Href { get; init; } = "";
    public string Rel { get; init; } = "";
    public string Method { get; init; } = "GET";
}

// Resource with links
public record UserResponse
{
    public string Id { get; init; } = "";
    public string Name { get; init; } = "";
    public string Email { get; init; } = "";
    public List<Link> Links { get; init; } = new();
}

[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult GetUser(string id)
    {
        var user = GetUserById(id);

        if (user == null)
            return NotFound();

        var response = new UserResponse
        {
            Id = user.Id,
            Name = user.Name,
            Email = user.Email,
            Links = new List<Link>
            {
                new()
                {
                    Href = Url.Action(nameof(GetUser), new { id = user.Id })!,
                    Rel = "self",
                    Method = "GET"
                },
                new()
                {
                    Href = Url.Action(nameof(UpdateUser), new { id = user.Id })!,
                    Rel = "update",
                    Method = "PUT"
                },
                new()
                {
                    Href = Url.Action(nameof(DeleteUser), new { id = user.Id })!,
                    Rel = "delete",
                    Method = "DELETE"
                },
                new()
                {
                    Href = Url.Action("GetUserOrders", "Orders", new { userId = user.Id })!,
                    Rel = "orders",
                    Method = "GET"
                }
            }
        };

        return Ok(response);
    }

    [HttpPut("{id}")]
    public IActionResult UpdateUser(string id, [FromBody] UpdateUserRequest request)
        => NoContent();

    [HttpDelete("{id}")]
    public IActionResult DeleteUser(string id)
        => NoContent();
}

// Response:
// {
//   "id": "user-123",
//   "name": "John Doe",
//   "email": "john@example.com",
//   "links": [
//     { "href": "/api/users/user-123", "rel": "self", "method": "GET" },
//     { "href": "/api/users/user-123", "rel": "update", "method": "PUT" },
//     { "href": "/api/users/user-123", "rel": "delete", "method": "DELETE" },
//     { "href": "/api/orders?userId=user-123", "rel": "orders", "method": "GET" }
//   ]
// }
```

## Link Generator Service

```csharp
// Reusable link generator
public class ApiLinkGenerator
{
    private readonly IUrlHelper _urlHelper;

    public ApiLinkGenerator(IUrlHelper urlHelper)
    {
        _urlHelper = urlHelper;
    }

    public List<Link> GenerateUserLinks(string userId)
    {
        return new List<Link>
        {
            new()
            {
                Href = _urlHelper.Action("GetUser", "Users", new { id = userId })!,
                Rel = "self",
                Method = "GET"
            },
            new()
            {
                Href = _urlHelper.Action("UpdateUser", "Users", new { id = userId })!,
                Rel = "update",
                Method = "PUT"
            },
            new()
            {
                Href = _urlHelper.Action("DeleteUser", "Users", new { id = userId })!,
                Rel = "delete",
                Method = "DELETE"
            }
        };
    }

    public List<Link> GeneratePaginationLinks(string action, string controller, int page, int totalPages)
    {
        var links = new List<Link>
        {
            new()
            {
                Href = _urlHelper.Action(action, controller, new { page })!,
                Rel = "self",
                Method = "GET"
            }
        };

        if (page < totalPages)
        {
            links.Add(new Link
            {
                Href = _urlHelper.Action(action, controller, new { page = page + 1 })!,
                Rel = "next",
                Method = "GET"
            });
        }

        if (page > 1)
        {
            links.Add(new Link
            {
                Href = _urlHelper.Action(action, controller, new { page = page - 1 })!,
                Rel = "prev",
                Method = "GET"
            });
        }

        links.Add(new Link
        {
            Href = _urlHelper.Action(action, controller, new { page = 1 })!,
            Rel = "first",
            Method = "GET"
        });

        links.Add(new Link
        {
            Href = _urlHelper.Action(action, controller, new { page = totalPages })!,
            Rel = "last",
            Method = "GET"
        });

        return links;
    }
}
```

## Conditional Links

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult GetUser(string id)
    {
        var user = GetUserById(id);

        if (user == null)
            return NotFound();

        var links = new List<Link>
        {
            new()
            {
                Href = Url.Action(nameof(GetUser), new { id = user.Id })!,
                Rel = "self",
                Method = "GET"
            }
        };

        // Only include update/delete if user has permission
        if (User.IsInRole("Admin"))
        {
            links.Add(new Link
            {
                Href = Url.Action(nameof(UpdateUser), new { id = user.Id })!,
                Rel = "update",
                Method = "PUT"
            });

            links.Add(new Link
            {
                Href = Url.Action(nameof(DeleteUser), new { id = user.Id })!,
                Rel = "delete",
                Method = "DELETE"
            });
        }

        // Only include activation link if user is inactive
        if (!user.IsActive)
        {
            links.Add(new Link
            {
                Href = Url.Action(nameof(ActivateUser), new { id = user.Id })!,
                Rel = "activate",
                Method = "POST"
            });
        }

        return Ok(new { User = user, Links = links });
    }

    [HttpPut("{id}")]
    public IActionResult UpdateUser(string id, [FromBody] UpdateUserRequest request)
        => NoContent();

    [HttpDelete("{id}")]
    public IActionResult DeleteUser(string id)
        => NoContent();

    [HttpPost("{id}/activate")]
    public IActionResult ActivateUser(string id)
        => NoContent();
}
```

## Link Templates

```csharp
// RFC 6570 URI Templates
public record LinkTemplate
{
    public string Href { get; init; } = "";
    public string Rel { get; init; } = "";
    public string Method { get; init; } = "GET";
    public bool Templated { get; init; }
}

[HttpGet]
public IActionResult GetUsers()
{
    var users = GetUserList();

    var links = new List<LinkTemplate>
    {
        new()
        {
            Href = "/api/users",
            Rel = "self",
            Method = "GET"
        },
        new()
        {
            Href = "/api/users/{id}",
            Rel = "user",
            Method = "GET",
            Templated = true
        },
        new()
        {
            Href = "/api/users?page={page}&limit={limit}",
            Rel = "search",
            Method = "GET",
            Templated = true
        }
    };

    return Ok(new { Users = users, Links = links });
}

// Response:
// {
//   "users": [...],
//   "links": [
//     { "href": "/api/users", "rel": "self", "method": "GET", "templated": false },
//     { "href": "/api/users/{id}", "rel": "user", "method": "GET", "templated": true },
//     { "href": "/api/users?page={page}&limit={limit}", "rel": "search", "method": "GET", "templated": true }
//   ]
// }
```

## Collection Links

```csharp
public record CollectionResponse<T>
{
    public List<T> Items { get; init; } = new();
    public int TotalItems { get; init; }
    public int Page { get; init; }
    public int PageSize { get; init; }
    public List<Link> Links { get; init; } = new();
}

[HttpGet]
public IActionResult GetUsers([FromQuery] int page = 1, [FromQuery] int pageSize = 20)
{
    var users = GetUsersPage(page, pageSize);
    var totalUsers = GetTotalUserCount();
    var totalPages = (int)Math.Ceiling(totalUsers / (double)pageSize);

    var response = new CollectionResponse<User>
    {
        Items = users,
        TotalItems = totalUsers,
        Page = page,
        PageSize = pageSize,
        Links = GeneratePaginationLinks(page, totalPages)
    };

    return Ok(response);
}

private List<Link> GeneratePaginationLinks(int page, int totalPages)
{
    var links = new List<Link>
    {
        new()
        {
            Href = Url.Action(nameof(GetUsers), new { page, pageSize = 20 })!,
            Rel = "self",
            Method = "GET"
        }
    };

    if (page > 1)
    {
        links.Add(new Link
        {
            Href = Url.Action(nameof(GetUsers), new { page = page - 1, pageSize = 20 })!,
            Rel = "prev",
            Method = "GET"
        });

        links.Add(new Link
        {
            Href = Url.Action(nameof(GetUsers), new { page = 1, pageSize = 20 })!,
            Rel = "first",
            Method = "GET"
        });
    }

    if (page < totalPages)
    {
        links.Add(new Link
        {
            Href = Url.Action(nameof(GetUsers), new { page = page + 1, pageSize = 20 })!,
            Rel = "next",
            Method = "GET"
        });

        links.Add(new Link
        {
            Href = Url.Action(nameof(GetUsers), new { page = totalPages, pageSize = 20 })!,
            Rel = "last",
            Method = "GET"
        });
    }

    return links;
}
```

## Testing Link Generation

```csharp
public class LinkGenerationTests
{
    [Fact]
    public async Task CreateUser_ReturnsLocationHeader()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        var request = new CreateUserRequest { Email = "user@example.com", Name = "John" };

        // Act
        var response = await client.PostAsJsonAsync("/api/users", request);

        // Assert
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        Assert.NotNull(response.Headers.Location);
        Assert.StartsWith("/api/users/", response.Headers.Location.ToString());
    }

    [Fact]
    public async Task GetUser_ReturnsHATEOASLinks()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        // Act
        var response = await client.GetAsync("/api/users/user-123");

        // Assert
        response.EnsureSuccessStatusCode();

        var content = await response.Content.ReadFromJsonAsync<UserResponse>();
        Assert.NotNull(content);
        Assert.NotEmpty(content.Links);
        Assert.Contains(content.Links, l => l.Rel == "self");
        Assert.Contains(content.Links, l => l.Rel == "update");
    }
}
```

## Guidelines

**When to Generate Links:**
- RESTful APIs (HATEOAS)
- Pagination
- Resource creation (201 Created)
- Related resources

**Link Relations:**
- `self` - The current resource
- `next` / `prev` - Pagination
- `first` / `last` - Collection boundaries
- `edit` / `update` - Modification
- `delete` - Deletion
- Custom relations for domain-specific actions

**Best Practices:**
- Use absolute URLs for Location headers
- Include relevant links based on permissions
- Use named routes for maintainability
- Document link relations in API docs
- Test link generation

**Performance:**
- Cache generated URLs when possible
- Generate links lazily if not always needed
- Avoid generating unused links

## Benefits

Discoverable. Clients navigate via links.

Maintainable. No hardcoded URLs in clients.

Type-safe. Compile-time route checking.

RESTful. Follows hypermedia principles.

## Related

- [attribute-routing.md](./attribute-routing.md) - Route configuration
- [hateoas-rest.md](../05-aspnet-advanced/hateoas-rest.md) - HATEOAS patterns
- [api-versioning-strategies.md](./api-versioning-strategies.md) - Versioned URLs
- [pagination-patterns.md](./pagination-patterns.md) - Pagination links
