# Single-Table Design

DynamoDB single-table design pattern. Store multiple entity types in one table for efficiency and cost savings.

## Principle

One DynamoDB table for entire application. Use partition key (PK) and sort key (SK) to organize different entity types. Related data stored together for efficient queries.

## Key Structure

Design keys for access patterns:

```
PK (Partition Key)    SK (Sort Key)           Entity Type
USER#123              USER#123                User
USER#123              LOAN#456                User's Loan
BOOK#789              BOOK#789                Book
BOOK#789              LOAN#456                Book's Loan
LIBRARY#ABC           LIBRARY#ABC             Library
LIBRARY#ABC           BOOK#789                Library's Book
LIBRARY#ABC           MEMBER#123              Library's Member
```

## Entity Design

User entity:

```csharp
public class User
{
    public string Id { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string Name { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
}

// DynamoDB mapping
PK: USER#123
SK: USER#123
EntityType: User
Email: john@example.com
Name: John Doe
CreatedAt: 2024-01-15T10:30:00Z
```

Loan entity:

```csharp
public class Loan
{
    public string Id { get; set; } = string.Empty;
    public string UserId { get; set; } = string.Empty;
    public string BookId { get; set; } = string.Empty;
    public DateTime DueDate { get; set; }
    public LoanStatus Status { get; set; }
}

// DynamoDB mapping for user's loans
PK: USER#123
SK: LOAN#456
EntityType: Loan
LoanId: 456
BookId: 789
DueDate: 2024-02-15T23:59:59Z
Status: Active

// Also stored for book's loans
PK: BOOK#789
SK: LOAN#456
EntityType: Loan
LoanId: 456
UserId: 123
DueDate: 2024-02-15T23:59:59Z
Status: Active
```

## Access Patterns

Query by partition key:

```csharp
// Get user by ID
PK = USER#123, SK = USER#123

// Get all loans for user
PK = USER#123, SK begins_with LOAN#

// Get all books in library
PK = LIBRARY#ABC, SK begins_with BOOK#
```

## Repository Implementation

Single-table repository base:

```csharp
public abstract class DynamoDbRepository
{
    protected readonly IAmazonDynamoDB DynamoDb;
    protected readonly string TableName;

    protected DynamoDbRepository(
        IAmazonDynamoDB dynamoDb,
        IOptions<DatabaseOptions> options)
    {
        DynamoDb = dynamoDb;
        TableName = options.Value.TableName;
    }

    protected async Task<Dictionary<string, AttributeValue>?> GetItemAsync(
        string pk,
        string sk)
    {
        var request = new GetItemRequest
        {
            TableName = TableName,
            Key = new Dictionary<string, AttributeValue>
            {
                ["PK"] = new AttributeValue { S = pk },
                ["SK"] = new AttributeValue { S = sk }
            }
        };

        var response = await DynamoDb.GetItemAsync(request);
        return response.Item.Count > 0 ? response.Item : null;
    }

    protected async Task PutItemAsync(Dictionary<string, AttributeValue> item)
    {
        var request = new PutItemRequest
        {
            TableName = TableName,
            Item = item
        };

        await DynamoDb.PutItemAsync(request);
    }

    protected async Task<List<Dictionary<string, AttributeValue>>> QueryAsync(
        string pk,
        string? skPrefix = null)
    {
        var request = new QueryRequest
        {
            TableName = TableName,
            KeyConditionExpression = skPrefix != null
                ? "PK = :pk AND begins_with(SK, :sk)"
                : "PK = :pk",
            ExpressionAttributeValues = new Dictionary<string, AttributeValue>
            {
                [":pk"] = new AttributeValue { S = pk }
            }
        };

        if (skPrefix != null)
        {
            request.ExpressionAttributeValues[":sk"] = new AttributeValue { S = skPrefix };
        }

        var response = await DynamoDb.QueryAsync(request);
        return response.Items;
    }
}
```

User repository:

```csharp
public class UserRepository : DynamoDbRepository, IUserRepository
{
    public UserRepository(
        IAmazonDynamoDB dynamoDb,
        IOptions<DatabaseOptions> options)
        : base(dynamoDb, options)
    {
    }

    public async Task<User?> GetByIdAsync(string userId)
    {
        var item = await GetItemAsync($"USER#{userId}", $"USER#{userId}");
        return item != null ? MapToUser(item) : null;
    }

    public async Task<Loan[]> GetUserLoansAsync(string userId)
    {
        var items = await QueryAsync($"USER#{userId}", "LOAN#");
        return items.Select(MapToLoan).ToArray();
    }

    public async Task SaveAsync(User user)
    {
        var item = new Dictionary<string, AttributeValue>
        {
            ["PK"] = new AttributeValue { S = $"USER#{user.Id}" },
            ["SK"] = new AttributeValue { S = $"USER#{user.Id}" },
            ["EntityType"] = new AttributeValue { S = "User" },
            ["UserId"] = new AttributeValue { S = user.Id },
            ["Email"] = new AttributeValue { S = user.Email },
            ["Name"] = new AttributeValue { S = user.Name },
            ["CreatedAt"] = new AttributeValue { S = user.CreatedAt.ToString("o") }
        };

        await PutItemAsync(item);
    }

    private User MapToUser(Dictionary<string, AttributeValue> item)
    {
        return new User
        {
            Id = item["UserId"].S,
            Email = item["Email"].S,
            Name = item["Name"].S,
            CreatedAt = DateTime.Parse(item["CreatedAt"].S)
        };
    }

    private Loan MapToLoan(Dictionary<string, AttributeValue> item)
    {
        return new Loan
        {
            Id = item["LoanId"].S,
            UserId = item["UserId"].S,
            BookId = item["BookId"].S,
            DueDate = DateTime.Parse(item["DueDate"].S),
            Status = Enum.Parse<LoanStatus>(item["Status"].S)
        };
    }
}
```

## Composite Entities

Store loan with both user and book relationships:

```csharp
public class LoanRepository : DynamoDbRepository, ILoanRepository
{
    public async Task SaveAsync(Loan loan)
    {
        // Store under user
        var userItem = new Dictionary<string, AttributeValue>
        {
            ["PK"] = new AttributeValue { S = $"USER#{loan.UserId}" },
            ["SK"] = new AttributeValue { S = $"LOAN#{loan.Id}" },
            ["EntityType"] = new AttributeValue { S = "Loan" },
            ["LoanId"] = new AttributeValue { S = loan.Id },
            ["UserId"] = new AttributeValue { S = loan.UserId },
            ["BookId"] = new AttributeValue { S = loan.BookId },
            ["DueDate"] = new AttributeValue { S = loan.DueDate.ToString("o") },
            ["Status"] = new AttributeValue { S = loan.Status.ToString() }
        };

        // Store under book
        var bookItem = new Dictionary<string, AttributeValue>
        {
            ["PK"] = new AttributeValue { S = $"BOOK#{loan.BookId}" },
            ["SK"] = new AttributeValue { S = $"LOAN#{loan.Id}" },
            ["EntityType"] = new AttributeValue { S = "Loan" },
            ["LoanId"] = new AttributeValue { S = loan.Id },
            ["UserId"] = new AttributeValue { S = loan.UserId },
            ["BookId"] = new AttributeValue { S = loan.BookId },
            ["DueDate"] = new AttributeValue { S = loan.DueDate.ToString("o") },
            ["Status"] = new AttributeValue { S = loan.Status.ToString() }
        };

        // Write both (consider using TransactWriteItems for atomicity)
        await PutItemAsync(userItem);
        await PutItemAsync(bookItem);
    }

    public async Task<Loan[]> GetByUserIdAsync(string userId)
    {
        var items = await QueryAsync($"USER#{userId}", "LOAN#");
        return items.Select(MapToLoan).ToArray();
    }

    public async Task<Loan[]> GetByBookIdAsync(string bookId)
    {
        var items = await QueryAsync($"BOOK#{bookId}", "LOAN#");
        return items.Select(MapToLoan).ToArray();
    }
}
```

## Global Secondary Index

Add GSI for alternate access patterns:

```
Table:
PK                    SK                    GSI1PK              GSI1SK
USER#123              USER#123              EMAIL#john@ex.com   USER#123
USER#123              LOAN#456              -                   -
BOOK#789              BOOK#789              ISBN#978-123        BOOK#789
```

Query by email:

```csharp
public async Task<User?> GetByEmailAsync(string email)
{
    var request = new QueryRequest
    {
        TableName = TableName,
        IndexName = "GSI1",
        KeyConditionExpression = "GSI1PK = :email",
        ExpressionAttributeValues = new Dictionary<string, AttributeValue>
        {
            [":email"] = new AttributeValue { S = $"EMAIL#{email}" }
        }
    };

    var response = await DynamoDb.QueryAsync(request);

    return response.Items.Count > 0
        ? MapToUser(response.Items[0])
        : null;
}
```

## Transactions

Atomic multi-item operations:

```csharp
public async Task CreateLoanAsync(Loan loan)
{
    var userItem = CreateLoanItem($"USER#{loan.UserId}", loan);
    var bookItem = CreateLoanItem($"BOOK#{loan.BookId}", loan);

    var request = new TransactWriteItemsRequest
    {
        TransactItems = new List<TransactWriteItem>
        {
            new TransactWriteItem
            {
                Put = new Put
                {
                    TableName = TableName,
                    Item = userItem
                }
            },
            new TransactWriteItem
            {
                Put = new Put
                {
                    TableName = TableName,
                    Item = bookItem
                }
            }
        }
    };

    await DynamoDb.TransactWriteItemsAsync(request);
}
```

## Batch Operations

Read multiple items efficiently:

```csharp
public async Task<User[]> GetByIdsAsync(string[] userIds)
{
    var keys = userIds.Select(id => new Dictionary<string, AttributeValue>
    {
        ["PK"] = new AttributeValue { S = $"USER#{id}" },
        ["SK"] = new AttributeValue { S = $"USER#{id}" }
    }).ToList();

    var request = new BatchGetItemRequest
    {
        RequestItems = new Dictionary<string, KeysAndAttributes>
        {
            [TableName] = new KeysAndAttributes { Keys = keys }
        }
    };

    var response = await DynamoDb.BatchGetItemAsync(request);

    return response.Responses[TableName]
        .Select(MapToUser)
        .ToArray();
}
```

## Table Design

CloudFormation template:

```yaml
Resources:
  MainTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: library-service
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: PK
          AttributeType: S
        - AttributeName: SK
          AttributeType: S
        - AttributeName: GSI1PK
          AttributeType: S
        - AttributeName: GSI1SK
          AttributeType: S
      KeySchema:
        - AttributeName: PK
          KeyType: HASH
        - AttributeName: SK
          KeyType: RANGE
      GlobalSecondaryIndexes:
        - IndexName: GSI1
          KeySchema:
            - AttributeName: GSI1PK
              KeyType: HASH
            - AttributeName: GSI1SK
              KeyType: RANGE
          Projection:
            ProjectionType: ALL
```

## Guidelines

**Key Design:**
- Use meaningful prefixes (USER#, BOOK#, LOAN#)
- PK groups related items
- SK enables range queries
- Include entity type attribute

**Access Patterns:**
- Design keys for query patterns first
- Duplicate data for multiple access patterns
- Use GSI for alternate queries
- Avoid scans

**Data Duplication:**
- Duplicate when needed for access patterns
- Trade storage for query performance
- Keep duplicates in sync
- Consider eventual consistency

**Transactions:**
- Use for atomic multi-item writes
- Limited to 100 items
- Higher cost than regular operations
- Use when consistency critical

## Benefits

Cost-effective. One table, lower costs.

Performance. Related data retrieved in single query.

Scalability. Unlimited storage and throughput.

Simplicity. Single table to manage.

## Related

- [dynamodb-repository-pattern.md](./dynamodb-repository-pattern.md) - Repository implementation
- [dynamodb-best-practices.md](./dynamodb-best-practices.md) - Optimization patterns
- [repository-pattern.md](../../07-architecture/repository-pattern.md) - General repository pattern
