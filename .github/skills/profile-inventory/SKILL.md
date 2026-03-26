---
name: profile-inventory
description: "Scans .NET codebase for AutoMapper Profile classes and extracts all CreateMap declarations with feature usage."
---

# AutoMapper Profile Inventory

Systematically discovers and catalogs all AutoMapper mapping configurations in a .NET codebase.

## When to Use
- Starting a new AutoMapper migration
- Auditing existing mapping configurations
- Building dependency graphs for mappings

## Prerequisites
- Access to .NET source code
- grep/glob tools available

## Procedure

### Step 1: Find All Profile Classes
```bash
grep -r "class.*:.*Profile" --include="*.cs" -l
```

Or search for:
```regex
class\s+(\w+)\s*:\s*Profile
```

### Step 2: Extract CreateMap Declarations
For each profile file, extract:
```regex
CreateMap<(\w+(?:<[^>]+>)?),\s*(\w+(?:<[^>]+>)?)>\s*\(
```

### Step 3: Detect Features Per Mapping
Scan the method chain following each CreateMap for:

| Pattern | Feature |
|---------|----------|
| `.ForMember(` | ForMember configuration |
| `.Ignore()` | Property ignored |
| `.Include<` | Derived type inclusion |
| `.IncludeBase<` | Base type inheritance |
| `.ConvertUsing` | Custom converter |
| `.AfterMap(` | Post-mapping hook |
| `.BeforeMap(` | Pre-mapping hook |
| `.ProjectTo<` | EF projection |
| `.ReverseMap()` | Bidirectional mapping |
| `.PreCondition(` | Conditional mapping |
| `IValueResolver` | Custom resolver |
| `.ConstructUsing(` | Custom construction (no default constructor or factory method) |
| `.NullSubstitute(` | Null replacement with a default value |

### Step 3b: Detect Private Setters on Destination Types

For each destination type, scan its class definition for `private set` or `private init` properties:

```regex
public\s+\w+\??\s+\w+\s*\{\s*get;\s*private\s+(set|init)
```

Record the count and property names. These require `UnsafeAccessorAttribute` in the manual mapper and significantly affect migration complexity.

### Step 4: Build Dependency Graph
For mappings using `Include<>` or `IncludeBase<>`:
1. Identify parent/child relationships
2. Create directed graph
3. Topologically sort for migration order

### Step 5: Classify Tiers
```
tier = 
  hasProjection ? 'projection' :
  hasAfterMap || hasBeforeMap || hasValueResolver ? 'complex' :
  hasInclude || hasIncludeBase ? 'hierarchy' :
  hasConvertUsing ? 'converter' :
  hasConstructUsing ? 'converter' :   // also needs special handling
  hasForMember ? 'formember' :
  'leaf'
```

Additionally flag:
- `privateSetter: true` — destination has ≥1 private-setter property (affects implementation regardless of tier)
- `nullSubstitute: true` — any `.NullSubstitute()` usage (needs explicit null-check logic in manual mapper)

## Output
JSON array of mappings with:
- Profile class name and file path
- Source and destination types
- Line number
- Features detected (boolean flags)
- Complexity tier
- Dependencies array

## Example

**Input (UserProfile.cs)**:
```csharp
public class UserProfile : Profile
{
    public UserProfile()
    {
        CreateMap<User, UserDto>()
            .ForMember(d => d.FullName, opt => opt.MapFrom(s => $"{s.FirstName} {s.LastName}"));
    }
}
```

**Output**:
```json
{
  "id": "uuid",
  "profilePath": "src/Profiles/UserProfile.cs",
  "profileClass": "UserProfile",
  "sourceType": "User",
  "destType": "UserDto",
  "lineNumber": 5,
  "tier": "formember",
  "features": {
    "hasForMember": true,
    "hasIgnore": false
  },
  "dependencies": []
}
```
