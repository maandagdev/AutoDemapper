---
name: callsite-rewrite
description: "Updates all call sites from IMapper.Map<T>() to new manual mapper extension methods."
---

# Call Site Rewrite

Finds and updates all usages of AutoMapper's IMapper.Map to use the new manual mapping methods.

## When to Use
- After creating manual mapper implementation
- Part of Stage 5 migration

## Prerequisites
- Manual mapper already created
- Source and destination types known
- Mapping ID for tracking

## Procedure

### Step 1: Find All Call Sites

Search patterns for AutoMapper usage:

**Direct Map call**:
```regex
_mapper\.Map<{DestType}>\s*\(\s*(\w+)\s*\)
```

**Projected Map**:
```regex
\.Map<{DestType}>\s*\(\s*([^)]+)\s*\)
```

**Map with explicit source type**:
```regex
_mapper\.Map<{SourceType},\s*{DestType}>\s*\(
```

### Step 2: Identify Variable Names

From the match, extract the source variable name:
```csharp
var dto = _mapper.Map<UserDto>(user);
//                             ^^^^
//                             source variable = "user"
```

### Step 3: Generate Replacement

**Before**:
```csharp
var dto = _mapper.Map<UserDto>(user);
```

**After**:
```csharp
var dto = user.ToUserDto();
```

### Step 4: Handle Variations

**Inline mapping**:
```csharp
// Before
return Ok(_mapper.Map<UserDto>(await _repo.GetUser(id)));

// After
return Ok((await _repo.GetUser(id)).ToUserDto());
```

**Collection mapping**:
```csharp
// Before
var dtos = _mapper.Map<List<UserDto>>(users);

// After
var dtos = users.Select(u => u.ToUserDto()).ToList();
```

**Nullable source**:
```csharp
// Before
var dto = _mapper.Map<UserDto>(maybeUser);

// After  
var dto = maybeUser?.ToUserDto();
```

### Step 5: Track Changes

Record each rewrite:
```json
{
  "file": "src/Services/UserService.cs",
  "line": 45,
  "original": "_mapper.Map<UserDto>(user)",
  "replacement": "user.ToUserDto()"
}
```

### Step 6: Handle Edge Cases

**Ternary expressions**:
```csharp
// Before
var dto = condition ? _mapper.Map<UserDto>(user) : null;

// After
var dto = condition ? user.ToUserDto() : null;
```

**Method chaining**:
```csharp
// Before
var result = _mapper.Map<UserDto>(user).Name;

// After
var result = user.ToUserDto().Name;
```

**As parameter**:
```csharp
// Before
DoSomething(_mapper.Map<UserDto>(user));

// After
DoSomething(user.ToUserDto());
```

## Search Commands

```bash
# Find all Map<DestType> usages
grep -rn "\.Map<{DestType}>" --include="*.cs"

# Find IMapper injections (to verify scope)
grep -rn "IMapper" --include="*.cs"
```

## Output

```json
{
  "mappingId": "uuid",
  "callSitesUpdated": 12,
  "callSiteFiles": [
    "src/Services/UserService.cs:45",
    "src/Services/UserService.cs:78",
    "src/Controllers/UserController.cs:23"
  ]
}
```

## Validation

After rewriting, verify:
1. File still compiles
2. No remaining Map<DestType> calls for this mapping
3. Using statement for mapper namespace added if needed

## Adding Using Statements

If mapper is in different namespace, add:
```csharp
using {Namespace}.Mappers;
```

Check if using exists before adding.
