---
name: Characterization Test Conventions
description: Conventions for AutoDemapper characterization test classes. These tests capture current AutoMapper behavior and must never be modified to make them pass after migration — fix the mapper instead.
applyTo: '**/*MappingTests.cs'
---

# Characterization Test Conventions

## Purpose

Characterization tests document the *current* behavior of AutoMapper so regressions are caught during migration. They are never mocked — they run against real `MapperConfiguration`.

**NEVER modify these tests to make them pass after migration. Fix the mapper.**

## Required Category Marker

Every test class MUST carry the category attribute so `--filter Category=Characterization` works:

| Framework | Attribute |
|-----------|-----------|
| xUnit | `[Trait("Category", "Characterization")]` on the class |
| NUnit | `[Category("Characterization")]` on the class |
| MSTest | `[TestCategory("Characterization")]` on the class |

Test run filter: `dotnet test --filter "Category=Characterization"`

## Class Structure

```csharp
[Trait("Category", "Characterization")]   // xUnit
public class UserMappingTests
{
    private readonly IMapper _mapper;

    public UserMappingTests()
    {
        var config = new MapperConfiguration(cfg => cfg.AddProfile<UserProfile>());
        _mapper = config.CreateMapper();
    }

    // tests ...

    private static User CreateUser() => new User
    {
        Id = 1,
        Name = "Test User",
        Email = "test@example.com"
    };
}
```

## Test Method Naming

Format: `Maps_{SourceType}_To_{DestType}_{Scenario}`

Examples:
- `Maps_User_To_UserDto_AllProperties`
- `Maps_User_To_UserDto_NullSource`
- `Maps_User_To_UserDto_WithOrders`
- `Maps_DerivedEntity_To_DerivedEntityDto_InheritsBaseProperties`

## Required Scenarios Per Mapping

| Scenario | Always? | Condition |
|----------|---------|-----------|
| `_AllProperties` | Yes | every mapping |
| `_NullSource` | Yes | every mapping |
| `_WithCollection` | Conditional | source or destination has IEnumerable property |
| `_NestedObject` | Conditional | destination has complex-type property |
| `_CustomConverter` | Conditional | ConvertUsing present |
| `_RoundTrip` | Conditional | ReverseMap or both directions mapped |

## Factory Method

Every test class MUST have a `private static Create{SourceType}()` factory that returns a fully-populated instance with non-null, non-default values. This ensures assertion failures are meaningful:

```csharp
private static User CreateUser() => new User
{
    Id = 42,
    FirstName = "Jane",
    LastName = "Doe",
    Email = "jane@example.com",
    Orders = new List<Order> { new Order { Id = 1, Total = 9.99m } }
};
```

## Assertion Style

- Assert every non-ignored destination property
- Use concrete expected values, not just `Assert.NotNull(result)`
- For ignored properties: assert `null` or `default` explicitly

```csharp
Assert.NotNull(result);
Assert.Equal(source.Id, result.Id);
Assert.Equal($"{source.FirstName} {source.LastName}", result.FullName);
Assert.Null(result.IgnoredField);
```

## File Location

`{TestProject}/CharacterizationTests/{SourceType}MappingTests.cs`
