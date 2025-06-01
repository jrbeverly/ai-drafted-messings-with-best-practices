# DynamoDB Data Modeling Patterns

Common data modeling patterns for DynamoDB. Design for access patterns, not normalization.

## Principle

Model data based on how you query it. Denormalize and duplicate data when needed. Use composite keys for flexible access.

## One-to-Many Relationships

Parent-child relationship with partition key:

```csharp
// User has many loans
PK: USER#123
SK: USER#123              // User entity
SK: LOAN#456              // Loan 1
SK: LOAN#789              // Loan 2
SK: LOAN#012              // Loan 3

// Query user and all their loans
PK = USER#123             // Returns user + all loans

// Query only user
PK = USER#123, SK = USER#123

// Query only loans
PK = USER#123, SK begins_with LOAN#
```

Implementation:

```csharp
public class UserWithLoans
{
    public User User { get; set; } = null!;
    public List<Loan> Loans { get; set; } = new();
}

public async Task<UserWithLoans> GetUserWithLoansAsync(string userId)
{
    var items = await QueryAsync($"USER#{userId}");

    var result = new UserWithLoans();

    foreach (var item in items)
    {
        var entityType = item["EntityType"].S;

        if (entityType == "User")
        {
            result.User = MapToUser(item);
        }
        else if (entityType == "Loan")
        {
            result.Loans.Add(MapToLoan(item));
        }
    }

    return result;
}
```

## Many-to-Many Relationships

Duplicate data in both directions:

```csharp
// Books can have multiple authors
// Authors can have multiple books

// Book perspective
PK: BOOK#123
SK: BOOK#123              // Book entity
SK: AUTHOR#456            // Author relationship
SK: AUTHOR#789            // Author relationship

// Author perspective
PK: AUTHOR#456
SK: AUTHOR#456            // Author entity
SK: BOOK#123              // Book relationship
SK: BOOK#999              // Book relationship

// Query: Get book with authors
PK = BOOK#123

// Query: Get author with books
PK = AUTHOR#456
```

Implementation:

```csharp
public async Task AddAuthorToBookAsync(string bookId, string authorId)
{
    var bookAuthorItem = new Dictionary<string, AttributeValue>
    {
        ["PK"] = new AttributeValue { S = $"BOOK#{bookId}" },
        ["SK"] = new AttributeValue { S = $"AUTHOR#{authorId}" },
        ["EntityType"] = new AttributeValue { S = "BookAuthor" },
        ["BookId"] = new AttributeValue { S = bookId },
        ["AuthorId"] = new AttributeValue { S = authorId }
    };

    var authorBookItem = new Dictionary<string, AttributeValue>
    {
        ["PK"] = new AttributeValue { S = $"AUTHOR#{authorId}" },
        ["SK"] = new AttributeValue { S = $"BOOK#{bookId}" },
        ["EntityType"] = new AttributeValue { S = "AuthorBook" },
        ["BookId"] = new AttributeValue { S = bookId },
        ["AuthorId"] = new AttributeValue { S = authorId }
    };

    // Use transaction for atomicity
    await TransactWriteItemsAsync(new[]
    {
        new TransactWriteItem { Put = new Put { Item = bookAuthorItem } },
        new TransactWriteItem { Put = new Put { Item = authorBookItem } }
    });
}
```

## Hierarchical Data

Multi-level hierarchy with composite sort key:

```csharp
// Organization > Department > Team > Employee

PK: ORG#123
SK: ORG#123                               // Organization
SK: DEPT#456                              // Department
SK: DEPT#456#TEAM#789                     // Team
SK: DEPT#456#TEAM#789#EMP#001             // Employee
SK: DEPT#456#TEAM#789#EMP#002             // Employee
SK: DEPT#456#TEAM#999                     // Another team
SK: DEPT#456#TEAM#999#EMP#003             // Employee

// Query all departments
PK = ORG#123, SK begins_with DEPT#

// Query all teams in department
PK = ORG#123, SK begins_with DEPT#456#TEAM#

// Query all employees in team
PK = ORG#123, SK begins_with DEPT#456#TEAM#789#EMP#
```

## Time Series Data

Sort by timestamp for chronological queries:

```csharp
// Sensor readings over time
PK: SENSOR#device_123
SK: 2024-01-15T10:30:00Z              // Reading at specific time
SK: 2024-01-15T10:31:00Z
SK: 2024-01-15T10:32:00Z

// Query readings in time range
PK = SENSOR#device_123
SK between 2024-01-15T10:00:00Z and 2024-01-15T11:00:00Z
```

Implementation:

```csharp
public record SensorReading(
    string DeviceId,
    DateTime Timestamp,
    double Temperature,
    double Humidity);

public async Task SaveReadingAsync(SensorReading reading)
{
    var item = new Dictionary<string, AttributeValue>
    {
        ["PK"] = new AttributeValue { S = $"SENSOR#{reading.DeviceId}" },
        ["SK"] = new AttributeValue { S = reading.Timestamp.ToString("o") },
        ["EntityType"] = new AttributeValue { S = "SensorReading" },
        ["Temperature"] = new AttributeValue { N = reading.Temperature.ToString() },
        ["Humidity"] = new AttributeValue { N = reading.Humidity.ToString() }
    };

    await PutItemAsync(item);
}

public async Task<SensorReading[]> GetReadingsInRangeAsync(
    string deviceId,
    DateTime startTime,
    DateTime endTime)
{
    var request = new QueryRequest
    {
        TableName = tableName,
        KeyConditionExpression = "PK = :pk AND SK BETWEEN :start AND :end",
        ExpressionAttributeValues = new Dictionary<string, AttributeValue>
        {
            [":pk"] = new AttributeValue { S = $"SENSOR#{deviceId}" },
            [":start"] = new AttributeValue { S = startTime.ToString("o") },
            [":end"] = new AttributeValue { S = endTime.ToString("o") }
        }
    };

    var response = await dynamoDb.QueryAsync(request);
    return response.Items.Select(MapToSensorReading).ToArray();
}
```

## Adjacency List Pattern

Generic graph structure:

```csharp
// Flexible relationship modeling
PK: NODE#user_123
SK: NODE#user_123                     // User entity
SK: FOLLOWS#user_456                  // User follows user_456
SK: FOLLOWS#user_789                  // User follows user_789
SK: MEMBER_OF#org_999                 // User member of org_999

PK: NODE#org_999
SK: NODE#org_999                      // Organization entity
SK: HAS_MEMBER#user_123               // Org has member user_123
SK: HAS_MEMBER#user_456               // Org has member user_456

// Query: Who does user_123 follow?
PK = NODE#user_123, SK begins_with FOLLOWS#

// Query: Who are members of org_999?
PK = NODE#org_999, SK begins_with HAS_MEMBER#
```

## Inverted Index with GSI

Query by attribute values:

```csharp
// Main table
PK: USER#123
SK: USER#123
Email: john@example.com
Status: Active

// GSI1 for email lookup
GSI1PK: EMAIL#john@example.com
GSI1SK: USER#123

// GSI2 for status queries
GSI2PK: STATUS#Active
GSI2SK: USER#123

// Query by email
QueryIndexAsync("GSI1", "EMAIL#john@example.com")

// Query active users
QueryIndexAsync("GSI2", "STATUS#Active")
```

Implementation:

```csharp
public async Task SaveUserAsync(User user)
{
    var item = new Dictionary<string, AttributeValue>
    {
        ["PK"] = new AttributeValue { S = $"USER#{user.Id}" },
        ["SK"] = new AttributeValue { S = $"USER#{user.Id}" },
        ["EntityType"] = new AttributeValue { S = "User" },
        ["Email"] = new AttributeValue { S = user.Email },
        ["Status"] = new AttributeValue { S = user.Status.ToString() },

        // GSI1 for email lookup
        ["GSI1PK"] = new AttributeValue { S = $"EMAIL#{user.Email}" },
        ["GSI1SK"] = new AttributeValue { S = $"USER#{user.Id}" },

        // GSI2 for status queries
        ["GSI2PK"] = new AttributeValue { S = $"STATUS#{user.Status}" },
        ["GSI2SK"] = new AttributeValue { S = user.CreatedAt.ToString("o") }
    };

    await PutItemAsync(item);
}
```

## Composite Sort Key Pattern

Multiple query dimensions:

```csharp
// Query by status and date
SK: STATUS#Active#DATE#2024-01-15
SK: STATUS#Active#DATE#2024-01-16
SK: STATUS#Completed#DATE#2024-01-14

// Query active items
SK begins_with STATUS#Active#

// Query active items in date range
SK between STATUS#Active#DATE#2024-01-01 and STATUS#Active#DATE#2024-12-31
```

Implementation:

```csharp
public async Task SaveOrderAsync(Order order)
{
    var item = new Dictionary<string, AttributeValue>
    {
        ["PK"] = new AttributeValue { S = $"CUSTOMER#{order.CustomerId}" },
        ["SK"] = new AttributeValue
        {
            S = $"STATUS#{order.Status}#DATE#{order.CreatedAt:yyyy-MM-dd}#ORDER#{order.Id}"
        },
        ["EntityType"] = new AttributeValue { S = "Order" },
        ["OrderId"] = new AttributeValue { S = order.Id },
        ["Status"] = new AttributeValue { S = order.Status.ToString() },
        ["Total"] = new AttributeValue { N = order.Total.ToString() }
    };

    await PutItemAsync(item);
}

public async Task<Order[]> GetActiveOrdersAsync(string customerId)
{
    var items = await QueryAsync(
        $"CUSTOMER#{customerId}",
        $"STATUS#{OrderStatus.Active}#");

    return items.Select(MapToOrder).ToArray();
}
```

## Sparse Index Pattern

Index only subset of items:

```csharp
// Main table - all items
PK: ITEM#123
SK: ITEM#123
IsFeatured: true

PK: ITEM#456
SK: ITEM#456
IsFeatured: false        // No GSI attributes

// GSI for featured items only
GSI1PK: FEATURED
GSI1SK: ITEM#123         // Only set if IsFeatured = true

// Query featured items (sparse index includes only featured)
QueryIndexAsync("GSI1", "FEATURED")
```

Implementation:

```csharp
public async Task SaveProductAsync(Product product)
{
    var item = new Dictionary<string, AttributeValue>
    {
        ["PK"] = new AttributeValue { S = $"PRODUCT#{product.Id}" },
        ["SK"] = new AttributeValue { S = $"PRODUCT#{product.Id}" },
        ["EntityType"] = new AttributeValue { S = "Product" },
        ["Name"] = new AttributeValue { S = product.Name },
        ["IsFeatured"] = new AttributeValue { BOOL = product.IsFeatured }
    };

    // Only set GSI attributes for featured products
    if (product.IsFeatured)
    {
        item["GSI1PK"] = new AttributeValue { S = "FEATURED" };
        item["GSI1SK"] = new AttributeValue { S = $"PRODUCT#{product.Id}" };
    }

    await PutItemAsync(item);
}

public async Task<Product[]> GetFeaturedProductsAsync()
{
    var items = await QueryIndexAsync("GSI1", "FEATURED");
    return items.Select(MapToProduct).ToArray();
}
```

## Materialized Aggregations

Pre-compute totals and counts:

```csharp
// Store aggregated data alongside detail records
PK: CUSTOMER#123
SK: CUSTOMER#123                      // Customer entity
SK: AGGREGATE#ORDERS                  // Order count and total
SK: ORDER#2024-01-15#456              // Individual order
SK: ORDER#2024-01-16#789              // Individual order

// Aggregate item structure
{
  "PK": "CUSTOMER#123",
  "SK": "AGGREGATE#ORDERS",
  "EntityType": "OrderAggregate",
  "TotalOrders": 150,
  "TotalSpent": 4500.00,
  "LastOrderDate": "2024-01-16T14:30:00Z"
}
```

Implementation:

```csharp
public async Task CreateOrderAsync(Order order)
{
    var orderItem = MapOrderToItem(order);

    // Update aggregate using UpdateItem with ADD
    var updateRequest = new UpdateItemRequest
    {
        TableName = tableName,
        Key = new Dictionary<string, AttributeValue>
        {
            ["PK"] = new AttributeValue { S = $"CUSTOMER#{order.CustomerId}" },
            ["SK"] = new AttributeValue { S = "AGGREGATE#ORDERS" }
        },
        UpdateExpression = "ADD TotalOrders :one, TotalSpent :amount SET LastOrderDate = :date",
        ExpressionAttributeValues = new Dictionary<string, AttributeValue>
        {
            [":one"] = new AttributeValue { N = "1" },
            [":amount"] = new AttributeValue { N = order.Total.ToString() },
            [":date"] = new AttributeValue { S = order.CreatedAt.ToString("o") }
        }
    };

    // Use transaction to ensure both succeed
    await TransactWriteItemsAsync(new[]
    {
        new TransactWriteItem { Put = new Put { Item = orderItem } },
        new TransactWriteItem { Update = updateRequest }
    });
}
```

## Versioning Pattern

Track entity history:

```csharp
// Current version
PK: USER#123
SK: USER#123
Version: 5

// Version history
PK: USER#123
SK: VERSION#1#2024-01-10T10:00:00Z
SK: VERSION#2#2024-01-11T14:30:00Z
SK: VERSION#3#2024-01-12T09:15:00Z

// Query version history
PK = USER#123, SK begins_with VERSION#
```

Implementation:

```csharp
public async Task UpdateUserWithVersioningAsync(User user)
{
    var currentItem = await GetItemAsync($"USER#{user.Id}", $"USER#{user.Id}");
    var currentVersion = int.Parse(currentItem?["Version"]?.N ?? "0");
    var newVersion = currentVersion + 1;

    var versionItem = new Dictionary<string, AttributeValue>
    {
        ["PK"] = new AttributeValue { S = $"USER#{user.Id}" },
        ["SK"] = new AttributeValue { S = $"VERSION#{newVersion}#{DateTime.UtcNow:o}" },
        ["EntityType"] = new AttributeValue { S = "UserVersion" },
        ["Version"] = new AttributeValue { N = newVersion.ToString() },
        ["Email"] = new AttributeValue { S = user.Email },
        ["Name"] = new AttributeValue { S = user.Name }
    };

    var updateRequest = new UpdateItemRequest
    {
        TableName = tableName,
        Key = new Dictionary<string, AttributeValue>
        {
            ["PK"] = new AttributeValue { S = $"USER#{user.Id}" },
            ["SK"] = new AttributeValue { S = $"USER#{user.Id}" }
        },
        UpdateExpression = "SET #email = :email, #name = :name, #version = :version",
        ConditionExpression = "#version = :currentVersion",
        ExpressionAttributeNames = new Dictionary<string, string>
        {
            ["#email"] = "Email",
            ["#name"] = "Name",
            ["#version"] = "Version"
        },
        ExpressionAttributeValues = new Dictionary<string, AttributeValue>
        {
            [":email"] = new AttributeValue { S = user.Email },
            [":name"] = new AttributeValue { S = user.Name },
            [":version"] = new AttributeValue { N = newVersion.ToString() },
            [":currentVersion"] = new AttributeValue { N = currentVersion.ToString() }
        }
    };

    await TransactWriteItemsAsync(new[]
    {
        new TransactWriteItem { Put = new Put { Item = versionItem } },
        new TransactWriteItem { Update = updateRequest }
    });
}
```

## Guidelines

**Access Pattern First:**
- List all access patterns before modeling
- Design keys to support queries efficiently
- Avoid scans at all costs

**Denormalization:**
- Duplicate data when needed for query efficiency
- Trade storage for query performance
- Keep duplicates in sync with transactions

**Composite Keys:**
- Use separators (#, |, :) in keys
- Order components by query frequency
- Enable begins_with and between queries

**GSI Usage:**
- Create indexes for alternate access patterns
- Use sparse indexes to save costs
- Project only needed attributes

**Data Types:**
- Store dates as ISO 8601 strings for sorting
- Use numbers for numerical operations
- Use sets for multi-value attributes

## Benefits

Query efficiency. Single query returns all needed data.

Scalability. Designed for high throughput.

Flexibility. Composite keys support multiple access patterns.

Cost optimization. Minimal indexes and scans.

## Related

- [single-table-design.md](./single-table-design.md) - Single-table pattern
- [dynamodb-best-practices.md](./dynamodb-best-practices.md) - Optimization patterns
- [dynamodb-repository-pattern.md](./dynamodb-repository-pattern.md) - Implementation details
