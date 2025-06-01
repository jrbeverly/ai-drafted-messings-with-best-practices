# Raw String Literals

C# 11 triple-quoted strings for multi-line content without escape sequences.

## Why It Matters

- No escaping needed for quotes, backslashes, or special characters
- Copy-paste friendly for JSON, SQL, HTML, and regex
- Indentation controlled by closing `"""` position

## Key Recommendations

**Use `"""` for multi-line strings with embedded quotes:**
```csharp
string json = """
    {
      "name": "John",
      "email": "john@example.com"
    }
    """;
```
Closing `"""` position determines base indentation (content indented relative to it).

**Interpolation with `$$` for JSON (double braces):**
```csharp
string json = $$"""
    {
      "name": "{{name}}",
      "email": "{{email}}"
    }
    """;
```
Number of `$` signs determines how many braces trigger interpolation.

**SQL queries:**
```csharp
var sql = """
    SELECT UserId, Name FROM Users
    WHERE Name LIKE @Search ORDER BY Name
    """;
```

**Test assertions against expected output:**
```csharp
var expected = """
    { "status": "ok" }
    """;
Assert.Equal(expected, actual);
```

**Regex patterns without double-escaping:**
```csharp
var regex = new Regex("""\d{3}-\d{2}-\d{4}""");
```

## Pitfalls to Avoid

- Using for simple one-line strings (regular `""` is cleaner)
- Misunderstanding indentation rules (closing `"""` sets the baseline)
- Forgetting `$$` when the string contains literal `{` braces and also needs interpolation
