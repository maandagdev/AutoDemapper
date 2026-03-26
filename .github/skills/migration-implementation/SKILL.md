---
name: migration-implementation
description: "Implements mapping methods to replace AutoMapper. Supports manual extension methods (default) and Mapperly source generators."
---

# Migration Implementation

Creates mapping code that replicates AutoMapper behavior exactly. Supports two approaches:
- **Manual** — static extension methods, zero dependencies, full control
- **Mapperly** — source-generated at compile time, less boilerplate

## When to Use
- Stage 5: Implementing migrations
- Converting individual mappings from AutoMapper to manual or Mapperly code

## Approach Selection

Before proceeding, read `mappingApproach` from inventory.json:

| Value | Action |
|-------|--------|
| `"manual"` | Follow the [Manual Mapping](#manual-mapping) section |
| `"mapperly"` | Follow the [Mapperly](#mapperly) section |
| Not set | Default to Manual Mapping |

## Prerequisites
- Mapping details from inventory.json
- Baseline established (tests passing)
- Source and destination type definitions accessible

---

## Manual Mapping

### Step 1: Analyze Mapping Configuration

From inventory entry:
```json
{
  "sourceType": "Order",
  "destType": "OrderDto",
  "tier": "formember",
  "features": {
    "hasForMember": true,
    "hasIgnore": true
  }
}
```

### Step 2: Analyze Types

Read source and destination class definitions:
- Property names and types
- Nullability
- Collections
- Nested types

### Step 2b: Detect Private Setters

Scan the destination type declaration for properties with `private set` or `private init`:

```regex
public\s+\w+\??\s+\w+\s*\{\s*get;\s*private\s+(set|init)
```

If any exist, the mapper **cannot** use a plain object initializer for those properties. On .NET 8+ the correct approach is `UnsafeAccessorAttribute`, which provides zero-cost access with no reflection overhead and is AOT-compatible.

```csharp
using System.Runtime.CompilerServices;

// Declare one accessor per private-setter property.
// The first parameter MUST be the declaring type of the member
// (which may be a base class, not the concrete type).
[UnsafeAccessor(UnsafeAccessorKind.Method, Name = "set_Status")]
private static extern void SetStatus(Order instance, OrderStatus value);

[UnsafeAccessor(UnsafeAccessorKind.Method, Name = "set_TotalAmount")]
private static extern void SetTotalAmount(Order instance, decimal value);

public static Order ToDomain(this OrderDao dao)
{
    var order = new Order { Id = dao.Id }; // public-init properties via initializer
    SetStatus(order, dao.Status);          // private set via UnsafeAccessor
    SetTotalAmount(order, dao.TotalAmount);
    return order;
}
```

> If the private setter is on a **base class**, declare the accessor with the base class as the first parameter type. Using the wrong type causes a runtime `MissingMethodException`.

### Step 3: Generate Mapper Class

**Naming convention**: Method names must follow `^To(Dao|Domain|(\w*Dto))$`. The class is a `static class` in the Mappers namespace.

**File**: `src/Mappers/{SourceType}Mapper.cs`

**Structure**:
```csharp
namespace {Namespace}.Mappers
{
    public static class {SourceType}Mapper
    {
        public static {DestType} ToDto(this {SourceType} source)
        {
            if (source == null) return null;
            
            return new {DestType}
            {
                // Property mappings
            };
        }
    }
}
```

### Step 4: Map Properties

**Direct mapping** (same name, same type):
```csharp
Id = source.Id,
```

**ForMember with MapFrom**:
```csharp
// Original: .ForMember(d => d.FullName, o => o.MapFrom(s => $"{s.First} {s.Last}"))
FullName = $"{source.First} {source.Last}",
```

**Ignore**:
```csharp
// Property not assigned, uses default
```

**Flattening** (e.g., Customer.Name → CustomerName):
```csharp
CustomerName = source.Customer?.Name,
```

### Step 5: Handle Collections

```csharp
Items = source.Items?.Select(i => i.ToItemDto()).ToList(),
```

Ensure nested type mappers exist first (dependency order).

### Step 6: Handle Hierarchy (Include/IncludeBase)

**Base mapper**:
```csharp
public static class BaseEntityMapper
{
    public static void MapBaseProperties(BaseEntity source, BaseEntityDto dest)
    {
        dest.Id = source.Id;
        dest.CreatedAt = source.CreatedAt;
    }
}
```

**Derived mapper**:
```csharp
public static DerivedDto ToDto(this Derived source)
{
    if (source == null) return null;
    
    var dto = new DerivedDto();
    BaseEntityMapper.MapBaseProperties(source, dto);
    dto.DerivedProp = source.DerivedProp;
    return dto;
}
```

### Step 7: Handle Converters

**Simple converter**:
```csharp
public static MoneyDto ToDto(this Money source)
{
    return new MoneyDto
    {
        Amount = source.Amount,
        CurrencyCode = source.Currency.IsoCode
    };
}
```

### Step 7b: Extract Common/Shared Mappers

When the same type conversion appears across multiple mappers (e.g., `Money ↔ decimal`, `DateTime ↔ DateOnly`), extract it into a shared static class rather than duplicating logic:

```csharp
// Shared conversions reused across all mappers
public static class CommonMappings
{
    public static DateOnly ToDateOnly(this DateTime dt) => DateOnly.FromDateTime(dt);
    public static DateTime ToDateTime(this DateOnly d) => d.ToDateTime(TimeOnly.MinValue, DateTimeKind.Utc);

    // Nullable variants
    public static DateOnly? ToDateOnly(this DateTime? dt) => dt.HasValue ? DateOnly.FromDateTime(dt.Value) : null;
    public static DateTime? ToDateTime(this DateOnly? d) => d?.ToDateTime(TimeOnly.MinValue, DateTimeKind.Utc);
}
```

Call these from individual mappers:
```csharp
public static OrderDto ToDto(this Order source)
{
    return new OrderDto
    {
        OrderDate = source.OrderDate.ToDateOnly(), // ← shared conversion
    };
}
```

This keeps individual mappers clean and ensures type conversions are tested in one place.

### Step 8: Handle AfterMap/BeforeMap

**AfterMap**:
```csharp
public static OrderDto ToDto(this Order source)
{
    var dto = new OrderDto { ... };
    
    // AfterMap logic
    dto.CalculatedField = CalculateValue(source);
    
    return dto;
}
```

## Output Templates

### Extension Method Style
```csharp
public static {DestType} To{DestType}(this {SourceType} source)
```

### Collection Extension
```csharp
public static IEnumerable<{DestType}> To{DestType}s(this IEnumerable<{SourceType}> source)
{
    return source?.Select(x => x.To{DestType}());
}
```

## Null Handling Patterns

**Nullable source**:
```csharp
if (source == null) return null;
```

**Nullable properties**:
```csharp
Address = source.Address?.ToDto(),
```

**Collections**:
```csharp
Items = source.Items?.Select(i => i.ToDto()).ToList() ?? new List<ItemDto>(),
```

## Example

**AutoMapper**:
```csharp
CreateMap<Order, OrderDto>()
    .ForMember(d => d.CustomerName, o => o.MapFrom(s => s.Customer.Name))
    .ForMember(d => d.InternalCode, o => o.Ignore());
```

**Manual**:
```csharp
public static OrderDto ToDto(this Order source)
{
    if (source == null) return null;
    
    return new OrderDto
    {
        Id = source.Id,
        OrderDate = source.OrderDate,
        CustomerName = source.Customer?.Name,
        // InternalCode intentionally not mapped (was Ignore())
    };
}
```

---

## Mapperly

### Step 1: Add Package Reference

```xml
<PackageReference Include="Riok.Mapperly" Version="4.*" PrivateAssets="all" />
```

`PrivateAssets="all"` makes it a build-only dependency with no runtime footprint.

### Step 2: Scan for Private Setters

Same check as Manual (see above). Mapperly's `MemberVisibility.All` only handles fully-private properties. For the common `public get; private set` pattern, Mapperly does **not** generate `UnsafeAccessorAttribute` and leaves the setter unset. You still need manual accessor declarations for these properties.

### Step 3: Generate Mapper Class

```csharp
using Riok.Mapperly.Abstractions;

[Mapper(EnumMappingStrategy = EnumMappingStrategy.ByName)]
public static partial class OrderMapper
{
    // Mapperly generates the implementation from the signature alone
    public static partial OrderDto ToDto(this Order source);
}
```

### Step 4: Override Conventions When Needed

**Property name mismatch (flattening)**:
```csharp
[MapProperty(nameof(Order.Customer.Name), nameof(OrderDto.CustomerName))]
public static partial OrderDto ToDto(this Order source);
```

**Ignore a target property**:
```csharp
[MapperIgnoreTarget(nameof(OrderDto.InternalCode))]
public static partial OrderDto ToDto(this Order source);
```

**Custom type conversion** (auto-discovered by Mapperly when defined in the same class):
```csharp
private static DateOnly DateTimeToDateOnly(DateTime dt) => DateOnly.FromDateTime(dt);
private static DateTime DateOnlyToDateTime(DateOnly d) => d.ToDateTime(TimeOnly.MinValue, DateTimeKind.Utc);
```

### Step 5: Handle Private Setters

For properties Mapperly cannot set, wrap the generated method and apply `UnsafeAccessorAttribute` manually:

```csharp
[UnsafeAccessor(UnsafeAccessorKind.Method, Name = "set_Status")]
private static extern void SetStatus(Order instance, OrderStatus value);

// Generated partial handles all public properties
[MapperIgnoreTarget(nameof(Order.Status))]
private static partial Order ToDomainInternal(OrderDao source);

// Public entry point wires the private setter
public static Order ToDomain(this OrderDao dao)
{
    var order = ToDomainInternal(dao);
    SetStatus(order, dao.Status);
    return order;
}
```

### Step 6: Handle ConstructUsing

If the domain type has no public default constructor, write the entry point manually and call Mapperly for the property population:

```csharp
public static Order ToDomain(this OrderDao dao)
{
    var order = Order.Create(dao.OrderNumber); // factory method or custom constructor
    // Mapperly cannot instantiate it, but you can populate remaining properties manually
    order.Description = dao.Description;
    return order;
}
```

### Step 7: Validate at Build Time

Mapperly produces compile-time diagnostics:
- `RMG020` — unmapped target property
- `RMG012` — source property not found

To treat unmapped properties as errors:
```csharp
[Mapper(RequiredMappingStrategy = RequiredMappingStrategy.Target)]
public static partial class OrderMapper { ... }
```

### Example

**AutoMapper**:
```csharp
CreateMap<Order, OrderDto>()
    .ForMember(d => d.CustomerName, o => o.MapFrom(s => s.Customer.Name))
    .ForMember(d => d.InternalCode, o => o.Ignore());
```

**Mapperly**:
```csharp
[Mapper]
public static partial class OrderMapper
{
    [MapProperty(nameof(Order.Customer.Name), nameof(OrderDto.CustomerName))]
    [MapperIgnoreTarget(nameof(OrderDto.InternalCode))]
    public static partial OrderDto ToDto(this Order source);
}
```
