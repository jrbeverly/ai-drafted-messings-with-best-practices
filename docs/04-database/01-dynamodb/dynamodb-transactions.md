# DynamoDB Transactions

Atomic operations across multiple items in DynamoDB. Ensure data consistency with ACID guarantees.

## Principle

Use transactions when multiple items must be updated atomically. All operations succeed or all fail. Maximum 100 items per transaction.

## TransactWriteItems

Atomic writes (Put, Update, Delete, ConditionCheck):

```csharp
public async Task CreateLoanAsync(Loan loan)
{
    var request = new TransactWriteItemsRequest
    {
        TransactItems = new List<TransactWriteItem>
        {
            // Store loan under user
            new TransactWriteItem
            {
                Put = new Put
                {
                    TableName = tableName,
                    Item = CreateLoanItem($"USER#{loan.UserId}", loan),
                    ConditionExpression = "attribute_not_exists(PK)"
                }
            },
            // Store loan under book
            new TransactWriteItem
            {
                Put = new Put
                {
                    TableName = tableName,
                    Item = CreateLoanItem($"BOOK#{loan.BookId}", loan),
                    ConditionExpression = "attribute_not_exists(PK)"
                }
            },
            // Update book availability
            new TransactWriteItem
            {
                Update = new Update
                {
                    TableName = tableName,
                    Key = new Dictionary<string, AttributeValue>
                    {
                        ["PK"] = new AttributeValue { S = $"BOOK#{loan.BookId}" },
                        ["SK"] = new AttributeValue { S = $"BOOK#{loan.BookId}" }
                    },
                    UpdateExpression = "SET IsAvailable = :false, ActiveLoanId = :loanId",
                    ConditionExpression = "IsAvailable = :true",
                    ExpressionAttributeValues = new Dictionary<string, AttributeValue>
                    {
                        [":false"] = new AttributeValue { BOOL = false },
                        [":true"] = new AttributeValue { BOOL = true },
                        [":loanId"] = new AttributeValue { S = loan.Id }
                    }
                }
            }
        }
    };

    try
    {
        await dynamoDb.TransactWriteItemsAsync(request);
    }
    catch (TransactionCanceledException ex)
    {
        // Check which condition failed
        foreach (var reason in ex.CancellationReasons)
        {
            if (reason.Code == "ConditionalCheckFailed")
            {
                throw new InvalidOperationException("Book is not available or loan already exists");
            }
        }
        throw;
    }
}
```

## TransactGetItems

Atomic reads across multiple items:

```csharp
public async Task<(User User, Book Book, Loan? ActiveLoan)> GetUserLoanContextAsync(
    string userId,
    string bookId)
{
    var request = new TransactGetItemsRequest
    {
        TransactItems = new List<TransactGetItem>
        {
            // Get user
            new TransactGetItem
            {
                Get = new Get
                {
                    TableName = tableName,
                    Key = new Dictionary<string, AttributeValue>
                    {
                        ["PK"] = new AttributeValue { S = $"USER#{userId}" },
                        ["SK"] = new AttributeValue { S = $"USER#{userId}" }
                    }
                }
            },
            // Get book
            new TransactGetItem
            {
                Get = new Get
                {
                    TableName = tableName,
                    Key = new Dictionary<string, AttributeValue>
                    {
                        ["PK"] = new AttributeValue { S = $"BOOK#{bookId}" },
                        ["SK"] = new AttributeValue { S = $"BOOK#{bookId}" }
                    }
                }
            }
        }
    };

    var response = await dynamoDb.TransactGetItemsAsync(request);

    var user = MapToUser(response.Responses[0].Item);
    var book = MapToBook(response.Responses[1].Item);

    // Check if book has active loan
    Loan? activeLoan = null;
    if (book.ActiveLoanId != null)
    {
        var loanItem = await GetItemAsync($"BOOK#{bookId}", $"LOAN#{book.ActiveLoanId}");
        activeLoan = loanItem != null ? MapToLoan(loanItem) : null;
    }

    return (user, book, activeLoan);
}
```

## Conditional Checks

Enforce business rules in transactions:

```csharp
public async Task ReturnLoanAsync(string loanId)
{
    // First, get the loan to know userId and bookId
    var loan = await GetLoanByIdAsync(loanId);
    if (loan == null)
        throw new NotFoundException("Loan", loanId);

    var request = new TransactWriteItemsRequest
    {
        TransactItems = new List<TransactWriteItem>
        {
            // Update loan status under user
            new TransactWriteItem
            {
                Update = new Update
                {
                    TableName = tableName,
                    Key = new Dictionary<string, AttributeValue>
                    {
                        ["PK"] = new AttributeValue { S = $"USER#{loan.UserId}" },
                        ["SK"] = new AttributeValue { S = $"LOAN#{loanId}" }
                    },
                    UpdateExpression = "SET #status = :returned, ReturnedAt = :now",
                    ConditionExpression = "#status = :active",
                    ExpressionAttributeNames = new Dictionary<string, string>
                    {
                        ["#status"] = "Status"
                    },
                    ExpressionAttributeValues = new Dictionary<string, AttributeValue>
                    {
                        [":returned"] = new AttributeValue { S = "Returned" },
                        [":active"] = new AttributeValue { S = "Active" },
                        [":now"] = new AttributeValue { S = DateTime.UtcNow.ToString("o") }
                    }
                }
            },
            // Update loan status under book
            new TransactWriteItem
            {
                Update = new Update
                {
                    TableName = tableName,
                    Key = new Dictionary<string, AttributeValue>
                    {
                        ["PK"] = new AttributeValue { S = $"BOOK#{loan.BookId}" },
                        ["SK"] = new AttributeValue { S = $"LOAN#{loanId}" }
                    },
                    UpdateExpression = "SET #status = :returned, ReturnedAt = :now",
                    ConditionExpression = "#status = :active",
                    ExpressionAttributeNames = new Dictionary<string, string>
                    {
                        ["#status"] = "Status"
                    },
                    ExpressionAttributeValues = new Dictionary<string, AttributeValue>
                    {
                        [":returned"] = new AttributeValue { S = "Returned" },
                        [":active"] = new AttributeValue { S = "Active" },
                        [":now"] = new AttributeValue { S = DateTime.UtcNow.ToString("o") }
                    }
                }
            },
            // Make book available again
            new TransactWriteItem
            {
                Update = new Update
                {
                    TableName = tableName,
                    Key = new Dictionary<string, AttributeValue>
                    {
                        ["PK"] = new AttributeValue { S = $"BOOK#{loan.BookId}" },
                        ["SK"] = new AttributeValue { S = $"BOOK#{loan.BookId}" }
                    },
                    UpdateExpression = "SET IsAvailable = :true REMOVE ActiveLoanId",
                    ConditionExpression = "ActiveLoanId = :loanId",
                    ExpressionAttributeValues = new Dictionary<string, AttributeValue>
                    {
                        [":true"] = new AttributeValue { BOOL = true },
                        [":loanId"] = new AttributeValue { S = loanId }
                    }
                }
            }
        }
    };

    await dynamoDb.TransactWriteItemsAsync(request);
}
```

## Idempotent Writes

Prevent duplicate operations:

```csharp
public async Task CreateUserAsync(User user, string idempotencyToken)
{
    var request = new TransactWriteItemsRequest
    {
        TransactItems = new List<TransactWriteItem>
        {
            // Create user
            new TransactWriteItem
            {
                Put = new Put
                {
                    TableName = tableName,
                    Item = MapToItem(user),
                    ConditionExpression = "attribute_not_exists(PK)"
                }
            },
            // Store idempotency token
            new TransactWriteItem
            {
                Put = new Put
                {
                    TableName = tableName,
                    Item = new Dictionary<string, AttributeValue>
                    {
                        ["PK"] = new AttributeValue { S = $"IDEMPOTENCY#{idempotencyToken}" },
                        ["SK"] = new AttributeValue { S = $"IDEMPOTENCY#{idempotencyToken}" },
                        ["EntityType"] = new AttributeValue { S = "IdempotencyToken" },
                        ["UserId"] = new AttributeValue { S = user.Id },
                        ["CreatedAt"] = new AttributeValue { S = DateTime.UtcNow.ToString("o") },
                        // TTL: expire after 24 hours
                        ["ExpiresAt"] = new AttributeValue
                        {
                            N = DateTimeOffset.UtcNow.AddHours(24).ToUnixTimeSeconds().ToString()
                        }
                    },
                    ConditionExpression = "attribute_not_exists(PK)"
                }
            }
        }
    };

    try
    {
        await dynamoDb.TransactWriteItemsAsync(request);
    }
    catch (TransactionCanceledException ex)
    {
        if (ex.CancellationReasons.Any(r => r.Code == "ConditionalCheckFailed"))
        {
            // Either user or idempotency token already exists
            // Check if token exists to determine if it's a duplicate request
            var tokenItem = await GetItemAsync(
                $"IDEMPOTENCY#{idempotencyToken}",
                $"IDEMPOTENCY#{idempotencyToken}");

            if (tokenItem != null)
            {
                // Duplicate request - return existing user
                var existingUserId = tokenItem["UserId"].S;
                var existingUser = await GetUserByIdAsync(existingUserId);
                return existingUser;
            }

            throw new DuplicateException("User already exists");
        }
        throw;
    }
}
```

## Optimistic Locking

Version-based concurrency control:

```csharp
public async Task UpdateUserWithVersionAsync(User user, int expectedVersion)
{
    var request = new TransactWriteItemsRequest
    {
        TransactItems = new List<TransactWriteItem>
        {
            new TransactWriteItem
            {
                Update = new Update
                {
                    TableName = tableName,
                    Key = new Dictionary<string, AttributeValue>
                    {
                        ["PK"] = new AttributeValue { S = $"USER#{user.Id}" },
                        ["SK"] = new AttributeValue { S = $"USER#{user.Id}" }
                    },
                    UpdateExpression = "SET #name = :name, #email = :email, #version = :newVersion, UpdatedAt = :now",
                    ConditionExpression = "#version = :expectedVersion",
                    ExpressionAttributeNames = new Dictionary<string, string>
                    {
                        ["#name"] = "Name",
                        ["#email"] = "Email",
                        ["#version"] = "Version"
                    },
                    ExpressionAttributeValues = new Dictionary<string, AttributeValue>
                    {
                        [":name"] = new AttributeValue { S = user.Name },
                        [":email"] = new AttributeValue { S = user.Email },
                        [":expectedVersion"] = new AttributeValue { N = expectedVersion.ToString() },
                        [":newVersion"] = new AttributeValue { N = (expectedVersion + 1).ToString() },
                        [":now"] = new AttributeValue { S = DateTime.UtcNow.ToString("o") }
                    }
                }
            }
        }
    };

    try
    {
        await dynamoDb.TransactWriteItemsAsync(request);
    }
    catch (TransactionCanceledException)
    {
        throw new ConcurrencyException(
            $"User {user.Id} was modified by another process. Expected version {expectedVersion}.");
    }
}
```

## Error Handling

Handle transaction failures:

```csharp
public async Task ExecuteTransactionWithRetryAsync(
    TransactWriteItemsRequest request,
    int maxRetries = 3)
{
    for (int retry = 0; retry <= maxRetries; retry++)
    {
        try
        {
            await dynamoDb.TransactWriteItemsAsync(request);
            return; // Success
        }
        catch (TransactionCanceledException ex)
        {
            // Check cancellation reasons
            var hasConditionalCheckFailed = ex.CancellationReasons
                .Any(r => r.Code == "ConditionalCheckFailed");

            if (hasConditionalCheckFailed)
            {
                // Business logic failure - don't retry
                throw new InvalidOperationException(
                    "Transaction condition check failed", ex);
            }

            // Other failures might be retriable
            if (retry == maxRetries)
                throw;

            var delay = TimeSpan.FromMilliseconds(Math.Pow(2, retry) * 100);
            await Task.Delay(delay);
        }
        catch (ProvisionedThroughputExceededException) when (retry < maxRetries)
        {
            var delay = TimeSpan.FromMilliseconds(Math.Pow(2, retry) * 100);
            await Task.Delay(delay);
        }
    }
}
```

## Transaction Isolation

Understand isolation levels:

```csharp
// DynamoDB transactions provide SERIALIZABLE isolation
// Reads within a transaction see a consistent snapshot

public async Task TransferItemBetweenUsersAsync(
    string fromUserId,
    string toUserId,
    string itemId)
{
    var request = new TransactWriteItemsRequest
    {
        TransactItems = new List<TransactWriteItem>
        {
            // Check item belongs to fromUser
            new TransactWriteItem
            {
                ConditionCheck = new ConditionCheck
                {
                    TableName = tableName,
                    Key = new Dictionary<string, AttributeValue>
                    {
                        ["PK"] = new AttributeValue { S = $"USER#{fromUserId}" },
                        ["SK"] = new AttributeValue { S = $"ITEM#{itemId}" }
                    },
                    ConditionExpression = "attribute_exists(PK)"
                }
            },
            // Remove from fromUser
            new TransactWriteItem
            {
                Delete = new Delete
                {
                    TableName = tableName,
                    Key = new Dictionary<string, AttributeValue>
                    {
                        ["PK"] = new AttributeValue { S = $"USER#{fromUserId}" },
                        ["SK"] = new AttributeValue { S = $"ITEM#{itemId}" }
                    }
                }
            },
            // Add to toUser
            new TransactWriteItem
            {
                Put = new Put
                {
                    TableName = tableName,
                    Item = new Dictionary<string, AttributeValue>
                    {
                        ["PK"] = new AttributeValue { S = $"USER#{toUserId}" },
                        ["SK"] = new AttributeValue { S = $"ITEM#{itemId}" },
                        ["EntityType"] = new AttributeValue { S = "Item" },
                        ["ItemId"] = new AttributeValue { S = itemId },
                        ["TransferredAt"] = new AttributeValue { S = DateTime.UtcNow.ToString("o") }
                    }
                }
            }
        }
    };

    await dynamoDb.TransactWriteItemsAsync(request);
}
```

## Performance Considerations

Transaction costs and limits:

```csharp
// Transactions consume 2x write capacity units
// Example: Transaction with 3 items = 6 WCUs

// Limit to 100 items per transaction
public async Task BatchUpdateInTransactionsAsync(List<User> users)
{
    if (users.Count > 100)
    {
        throw new ArgumentException("Cannot update more than 100 items in a single transaction");
    }

    // Alternative: Split into multiple transactions
    var batches = users.Chunk(100);
    foreach (var batch in batches)
    {
        var request = new TransactWriteItemsRequest
        {
            TransactItems = batch.Select(user => new TransactWriteItem
            {
                Put = new Put
                {
                    TableName = tableName,
                    Item = MapToItem(user)
                }
            }).ToList()
        };

        await dynamoDb.TransactWriteItemsAsync(request);
    }
}
```

## Testing Transactions

Unit test with mocked client:

```csharp
[Fact]
public async Task CreateLoan_Success_CallsTransactWriteItems()
{
    // Arrange
    var dynamoDb = Substitute.For<IAmazonDynamoDB>();
    var service = new LoanService(dynamoDb);

    var loan = new Loan
    {
        Id = "loan_123",
        UserId = "user_456",
        BookId = "book_789",
        DueDate = DateTime.UtcNow.AddDays(14)
    };

    // Act
    await service.CreateLoanAsync(loan);

    // Assert
    await dynamoDb.Received(1).TransactWriteItemsAsync(
        Arg.Is<TransactWriteItemsRequest>(r =>
            r.TransactItems.Count == 3 &&
            r.TransactItems[0].Put != null &&
            r.TransactItems[1].Put != null &&
            r.TransactItems[2].Update != null),
        Arg.Any<CancellationToken>());
}

[Fact]
public async Task CreateLoan_BookNotAvailable_ThrowsException()
{
    // Arrange
    var dynamoDb = Substitute.For<IAmazonDynamoDB>();
    var service = new LoanService(dynamoDb);

    var loan = new Loan
    {
        Id = "loan_123",
        UserId = "user_456",
        BookId = "book_789",
        DueDate = DateTime.UtcNow.AddDays(14)
    };

    dynamoDb.TransactWriteItemsAsync(
        Arg.Any<TransactWriteItemsRequest>(),
        Arg.Any<CancellationToken>())
        .ThrowsAsync(new TransactionCanceledException("Condition check failed")
        {
            CancellationReasons = new List<CancellationReason>
            {
                new CancellationReason { Code = "None" },
                new CancellationReason { Code = "None" },
                new CancellationReason { Code = "ConditionalCheckFailed" }
            }
        });

    // Act & Assert
    await Assert.ThrowsAsync<InvalidOperationException>(
        () => service.CreateLoanAsync(loan));
}
```

## Guidelines

**When to Use Transactions:**
- Multiple items must be updated atomically
- Conditional checks across multiple items
- Preventing race conditions
- Idempotent operations

**When NOT to Use Transactions:**
- Single item operations (use PutItem, UpdateItem)
- More than 100 items (split into batches)
- Read-only operations (use BatchGetItem)
- High-throughput operations (transactions are 2x cost)

**Best Practices:**
- Keep transactions small (fewer items = better performance)
- Use conditional checks to enforce business rules
- Handle TransactionCanceledException gracefully
- Implement retry logic for throttling
- Use optimistic locking for concurrent updates

**Error Handling:**
- Distinguish between retriable and non-retriable errors
- ConditionalCheckFailed = business logic failure (don't retry)
- ProvisionedThroughputExceeded = retry with backoff
- Log transaction failures with context

## Benefits

Atomicity. All operations succeed or all fail.

Consistency. Enforce business rules with conditions.

Isolation. Serializable isolation level.

Safety. Prevent race conditions and data corruption.

## Related

- [dynamodb-best-practices.md](./dynamodb-best-practices.md) - General optimization
- [dynamodb-repository-pattern.md](./dynamodb-repository-pattern.md) - Repository implementation
- [single-table-design.md](./single-table-design.md) - Table structure
