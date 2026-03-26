---
name: characterization-test
description: "Generates characterization tests that capture current AutoMapper behavior for a specific mapping."
---

# Characterization Test Generation

Creates tests that document the current behavior of AutoMapper mappings, serving as a safety net during migration.

## When to Use
- Before migrating any AutoMapper mapping
- When documenting existing mapping behavior
- Creating regression test suite

## Prerequisites
- Mapping details from inventory.json
- Test framework identified
- Test project location known

## Procedure

### Step 1: Analyze Mapping
From inventory entry, extract:
- Source type name and properties
- Destination type name and properties  
- Profile class for mapper configuration
- Features used (ForMember, AfterMap, etc.)

### Step 2: Determine Test Scenarios

**Required Tests**:
1. Basic mapping - all properties transfer correctly
2. Null source handling

**Conditional Tests**:
3. Collection mapping (if source has IEnumerable properties)
4. Nested object mapping (if destination has complex types)
5. Custom logic (if ForMember with MapFrom)
6. Converter behavior (if ConvertUsing)
7. Round-trip (if `ReverseMap()` or both directions are mapped) — verifies Domain → DAO → Domain produces equivalent object

### Step 3: Generate Test Class

**xUnit Template**:
```csharp
using AutoMapper;
using Xunit;

namespace {Namespace}.CharacterizationTests
{
    [Trait("Category", "Characterization")]
    public class {SourceType}MappingTests
    {
        private readonly IMapper _mapper;

        public {SourceType}MappingTests()
        {
            var config = new MapperConfiguration(cfg => 
                cfg.AddProfile<{ProfileClass}>());
            _mapper = config.CreateMapper();
        }

        [Fact]
        public void Maps_{SourceType}_To_{DestType}_AllProperties()
        {
            // Arrange
            var source = Create{SourceType}();

            // Act
            var result = _mapper.Map<{DestType}>(source);

            // Assert
            Assert.NotNull(result);
            // Property assertions based on type analysis
        }

        [Fact]
        public void Maps_{SourceType}_To_{DestType}_NullSource()
        {
            // Arrange
            {SourceType} source = null;

            // Act
            var result = _mapper.Map<{DestType}>(source);

            // Assert
            Assert.Null(result);
        }

        // Only generate if both directions are mapped (ReverseMap or two CreateMap declarations)
        [Fact]
        public void RoundTrip_{SourceType}_To_{DestType}_And_Back_PreservesValues()
        {
            // Arrange
            var original = Create{SourceType}();

            // Act
            var intermediate = _mapper.Map<{DestType}>(original);
            var roundTripped = _mapper.Map<{SourceType}>(intermediate);

            // Assert — only compare properties that survive the round-trip
            Assert.Equal(original.Id, roundTripped.Id);
            // Add property-by-property assertions for all non-lossy properties
        }

        private static {SourceType} Create{SourceType}()
        {
            return new {SourceType}
            {
                // Property initializers
            };
        }
    }
}
```

### Step 4: Generate Property Assertions
For each mapped property:
```csharp
Assert.Equal(source.{Property}, result.{Property});
```

For ForMember with MapFrom:
```csharp
Assert.Equal(expected_transformed_value, result.{Property});
```

### Step 5: Handle Special Cases

**Collections**:
```csharp
Assert.Equal(source.Items.Count, result.Items.Count);
Assert.All(result.Items, item => Assert.NotNull(item));
```

**Nested Objects**:
```csharp
Assert.NotNull(result.NestedObject);
Assert.Equal(source.NestedObject.Property, result.NestedObject.Property);
```

**Ignored Properties**:
```csharp
Assert.Null(result.IgnoredProperty); // or default value
```

## Output
- Test file at `{TestProject}/CharacterizationTests/{SourceType}MappingTests.cs`
- Entry in tests.json with test method names

## Example

**Mapping**:
```csharp
CreateMap<Order, OrderDto>()
    .ForMember(d => d.CustomerName, o => o.MapFrom(s => s.Customer.Name))
    .ForMember(d => d.Total, o => o.Ignore());
```

**Generated Test**:
```csharp
[Fact]
public void Maps_Order_To_OrderDto_AllProperties()
{
    var source = new Order
    {
        Id = 1,
        Customer = new Customer { Name = "John" },
        Items = new List<OrderItem> { new() }
    };

    var result = _mapper.Map<OrderDto>(source);

    Assert.Equal(1, result.Id);
    Assert.Equal("John", result.CustomerName);
    Assert.Equal(default, result.Total); // Ignored
}
```
