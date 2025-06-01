# DynamoDB Best Practices

Optimize DynamoDB performance, cost, and design patterns.

## Principle

Design for access patterns. Use partition keys efficiently. Avoid scans. Leverage indexes wisely.

## Access Pattern Design

Start with access patterns:

```
Access Patterns:
1. Get user by ID
2. Get all loans for user
3. Get all loans for book
4. Get user by email
5. Get active loans (all users)
6. Get overdue loans (all users)

Key Design:
Pattern 1: PK=USER#{id}, SK=USER#{id}
Pattern 2: PK=USER#{id}, SK begins_with LOAN#
Pattern 3: PK=BOOK#{id}, SK begins_with LOAN#
Pattern 4: GSI1: PK=EMAIL#{email}
Pattern 5: GSI2: PK=LOAN#ACTIVE, SK=DueDate
Pattern 6: GSI2: PK=LOAN#OVERDUE, SK=DueDate
```

## Partition Key Selection

Good partition keys:

```csharp
// GOOD - High cardinality, even distribution
PK = USER#{userId}           // Millions of unique values
PK = BOOK#{bookId}           // Thousands of unique values
PK = ORDER#{orderId}         // High cardinality

// BAD - Low cardinality, hot partitions
PK = STATUS#{status}         // Only a few values (active, inactive)
PK = DATE#{date}            // All requests for today hit same partition
PK = CATEGORY#{category}    // Limited number of categories
```

Distribute writes:

```csharp
// GOOD - Distribute writes across partitions
var userId = Guid.NewGuid().ToString(); // Random distribution
PK = USER#{userId}

// BAD - All writes to single partition
PK = USERS
SK = USER#{userId}
```

## Hot Partition Mitigation

Add random suffix:

```csharp
// For low-cardinality keys, add shard suffix
var shardId = (userId.GetHashCode() % 10).ToString();
PK = $"STATUS#{status}#{shardId}"

// Query all shards
for (int shard = 0; shard < 10; shard++)
{
    var items = await QueryAsync($"STATUS#active#{shard}");
    allItems.AddRange(items);
}
```

## Sort Key Patterns

Hierarchical data:

```csharp
PK = LIBRARY#{libraryId}
SK = LIBRARY#{libraryId}        // Library itself
SK = BOOK#{bookId}              // Books in library
SK = MEMBER#{userId}            // Members in library
SK = LOAN#{loanId}              // Loans in library

// Query all entities for library
PK = LIBRARY#ABC

// Query only books
PK = LIBRARY#ABC, SK begins_with BOOK#
```

Composite sort keys:

```csharp
// Sort by multiple attributes
SK = STATUS#{status}#DATE#{date}

// Example
SK = STATUS#active#DATE#2024-01-15
SK = STATUS#active#DATE#2024-01-16
SK = STATUS#completed#DATE#2024-01-14

// Query active items sorted by date
PK = USER#123
SK between STATUS#active#DATE#2024-01-01 and STATUS#active#DATE#2024-12-31
```

## Sparse Indexes

Create GSI only for subset of items:

```csharp
// Main table - all users
PK = USER#{userId}
SK = USER#{userId}
IsActive = true/false

// GSI for active users only (sparse index)
// Only set GSI1PK for active users
if (user.IsActive)
{
    item["GSI1PK"] = new AttributeValue { S = "ACTIVE_USER" };
    item["GSI1SK"] = new AttributeValue { S = user.CreatedAt.ToString("o") };
}

// Query only includes active users
QueryRequest.IndexName = "GSI1"
QueryRequest.KeyConditionExpression = "GSI1PK = :active"
```

## Conditional Writes

Prevent overwrites:

```csharp
// Only create if doesn't exist
var request = new PutItemRequest
{
    TableName = tableName,
    Item = item,
    ConditionExpression = "attribute_not_exists(PK)"
};

try
{
    await dynamoDb.PutItemAsync(request);
}
catch (ConditionalCheckFailedException)
{
    throw new DuplicateException("User already exists");
}

// Optimistic locking with version
var request = new PutItemRequest
{
    TableName = tableName,
    Item = item,
    ConditionExpression = "Version = :currentVersion",
    ExpressionAttributeValues = new Dictionary<string, AttributeValue>
    {
        [":currentVersion"] = new AttributeValue { N = currentVersion.ToString() }
    }
};

// Increment version
item["Version"] = new AttributeValue { N = (currentVersion + 1).ToString() };
```

## Projection Expressions

Fetch only needed attributes:

```csharp
// Fetch full item (expensive)
var request = new GetItemRequest
{
    TableName = tableName,
    Key = key
};

// Fetch specific attributes only (cheaper)
var request = new GetItemRequest
{
    TableName = tableName,
    Key = key,
    ProjectionExpression = "UserId, Email, Name"
};

// Query with projection
var request = new QueryRequest
{
    TableName = tableName,
    KeyConditionExpression = "PK = :pk",
    ProjectionExpression = "UserId, Email",
    ExpressionAttributeValues = new Dictionary<string, AttributeValue>
    {
        [":pk"] = new AttributeValue { S = $"USER#{userId}" }
    }
};
```

## Batch Operations

Batch reads:

```csharp
// Up to 100 items, 16 MB
var request = new BatchGetItemRequest
{
    RequestItems = new Dictionary<string, KeysAndAttributes>
    {
        [tableName] = new KeysAndAttributes
        {
            Keys = userIds.Select(id => new Dictionary<string, AttributeValue>
            {
                ["PK"] = new AttributeValue { S = $"USER#{id}" },
                ["SK"] = new AttributeValue { S = $"USER#{id}" }
            }).ToList()
        }
    }
};

var response = await dynamoDb.BatchGetItemAsync(request);

// Handle unprocessed keys
if (response.UnprocessedKeys.Count > 0)
{
    // Retry with exponential backoff
}
```

Batch writes:

```csharp
// Up to 25 items
var request = new BatchWriteItemRequest
{
    RequestItems = new Dictionary<string, List<WriteRequest>>
    {
        [tableName] = users.Select(user => new WriteRequest
        {
            PutRequest = new PutRequest
            {
                Item = MapToItem(user)
            }
        }).ToList()
    }
};

await dynamoDb.BatchWriteItemAsync(request);
```

## TTL for Auto-Expiration

Automatic deletion:

```csharp
// Enable TTL on table (once)
// Attribute: ExpiresAt (Unix timestamp)

// Set expiration when creating item
var expiresAt = DateTimeOffset.UtcNow.AddDays(30).ToUnixTimeSeconds();

var item = new Dictionary<string, AttributeValue>
{
    ["PK"] = new AttributeValue { S = $"SESSION#{sessionId}" },
    ["SK"] = new AttributeValue { S = $"SESSION#{sessionId}" },
    ["ExpiresAt"] = new AttributeValue { N = expiresAt.ToString() },
    // ... other attributes
};

// DynamoDB automatically deletes within 48 hours of expiration
```

## Query Pagination

Efficient pagination:

```csharp
public async Task<(List<User> Items, string? NextToken)> GetUsersPagedAsync(
    string? lastEvaluatedKey,
    int limit)
{
    var request = new QueryRequest
    {
        TableName = tableName,
        KeyConditionExpression = "PK = :pk",
        Limit = limit,
        ExpressionAttributeValues = new Dictionary<string, AttributeValue>
        {
            [":pk"] = new AttributeValue { S = "USERS" }
        }
    };

    // Resume from previous page
    if (lastEvaluatedKey != null)
    {
        request.ExclusiveStartKey = DeserializeKey(lastEvaluatedKey);
    }

    var response = await dynamoDb.QueryAsync(request);

    var items = response.Items.Select(MapToUser).ToList();

    // Serialize token for next page
    var nextToken = response.LastEvaluatedKey.Count > 0
        ? SerializeKey(response.LastEvaluatedKey)
        : null;

    return (items, nextToken);
}
```

## Error Handling

Retry with exponential backoff:

```csharp
public async Task<T> RetryAsync<T>(
    Func<Task<T>> operation,
    int maxRetries = 3)
{
    for (int retry = 0; retry <= maxRetries; retry++)
    {
        try
        {
            return await operation();
        }
        catch (ProvisionedThroughputExceededException) when (retry < maxRetries)
        {
            var delay = TimeSpan.FromMilliseconds(Math.Pow(2, retry) * 100);
            await Task.Delay(delay);
        }
        catch (ThrottlingException) when (retry < maxRetries)
        {
            var delay = TimeSpan.FromMilliseconds(Math.Pow(2, retry) * 100);
            await Task.Delay(delay);
        }
    }

    throw new Exception("Max retries exceeded");
}
```

## Cost Optimization

On-demand vs provisioned:

```csharp
// On-demand: Pay per request (good for unpredictable traffic)
// Provisioned: Pay for capacity (good for steady traffic)

// Use on-demand for:
// - New applications
// - Unpredictable workloads
// - Dev/test environments

// Use provisioned for:
// - Steady, predictable traffic
// - Cost optimization at scale
// - Production with consistent load
```

Optimize read/write:

```csharp
// Eventually consistent reads (cheaper)
var request = new GetItemRequest
{
    TableName = tableName,
    Key = key,
    ConsistentRead = false  // Default, half the cost
};

// Strongly consistent reads (more expensive)
var request = new GetItemRequest
{
    TableName = tableName,
    Key = key,
    ConsistentRead = true   // Double the cost
};

// Use eventually consistent unless you need immediate consistency
```

## Guidelines

**Key Design:**
- High cardinality partition keys
- Distribute writes evenly
- Use sort keys for queries
- Design for access patterns

**Indexes:**
- GSI for alternate access patterns
- Sparse indexes for subsets
- Project only needed attributes
- Monitor index costs

**Queries:**
- Use Query over Scan
- Fetch only needed attributes
- Paginate large result sets
- Use batch operations for multiple items

**Writes:**
- Use batch writes for multiple items
- Conditional writes for safety
- Transactions for atomicity
- Consider write sharding for hot keys

**Cost:**
- On-demand for variable load
- Eventually consistent reads when possible
- Use TTL for automatic cleanup
- Monitor and optimize GSI usage

## Anti-Patterns

Avoid:

```csharp
// DON'T: Scan entire table
var request = new ScanRequest { TableName = tableName };

// DO: Query with partition key
var request = new QueryRequest
{
    KeyConditionExpression = "PK = :pk"
};

// DON'T: Store large items (>400 KB)
var item = new { Data = largeBlob }; // Use S3 instead

// DON'T: Use sequential IDs as partition key
PK = $"{sequentialId}"; // Creates hot partitions

// DO: Use random/UUID
PK = $"{Guid.NewGuid()}";
```

## Benefits

Performance. Single-digit millisecond latency.

Scalability. Unlimited storage and throughput.

Cost-effective. Pay for what you use.

Reliability. 99.99% availability SLA.

## Related

- [single-table-design.md](./single-table-design.md) - Single-table pattern
- [dynamodb-repository-pattern.md](./dynamodb-repository-pattern.md) - Repository implementation
