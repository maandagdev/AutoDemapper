---
name: test-framework-adapter
description: "Adapts test code syntax for xUnit, NUnit, or MSTest based on detected framework."
---

# Test Framework Adapter

Translates test code between different .NET test frameworks while preserving test logic.

## When to Use
- Generating tests for detected framework
- Converting test examples to match project conventions
- Ensuring consistent test syntax

## Prerequisites
- Test framework identified in inventory.json

## Procedure

### Step 1: Identify Target Framework
Read from `inventory.json`:
```json
"repoFacts": {
  "testFramework": "xunit|nunit|mstest"
}
```

### Step 2: Apply Framework-Specific Syntax

#### Attributes

| Concept | xUnit | NUnit | MSTest |
|---------|-------|-------|--------|
| Test method | `[Fact]` | `[Test]` | `[TestMethod]` |
| Parameterized | `[Theory]` | `[TestCase]` | `[DataTestMethod]` |
| Category | `[Trait("Category", "X")]` | `[Category("X")]` | `[TestCategory("X")]` |
| Setup | Constructor | `[SetUp]` | `[TestInitialize]` |
| Teardown | IDisposable | `[TearDown]` | `[TestCleanup]` |
| Class setup | IClassFixture | `[OneTimeSetUp]` | `[ClassInitialize]` |
| Skip | `[Fact(Skip = "reason")]` | `[Ignore("reason")]` | `[Ignore]` |

#### Assertions

| Assertion | xUnit | NUnit | MSTest |
|-----------|-------|-------|--------|
| Equal | `Assert.Equal(e, a)` | `Assert.AreEqual(e, a)` | `Assert.AreEqual(e, a)` |
| Not equal | `Assert.NotEqual(e, a)` | `Assert.AreNotEqual(e, a)` | `Assert.AreNotEqual(e, a)` |
| Null | `Assert.Null(obj)` | `Assert.IsNull(obj)` | `Assert.IsNull(obj)` |
| Not null | `Assert.NotNull(obj)` | `Assert.IsNotNull(obj)` | `Assert.IsNotNull(obj)` |
| True | `Assert.True(b)` | `Assert.IsTrue(b)` | `Assert.IsTrue(b)` |
| False | `Assert.False(b)` | `Assert.IsFalse(b)` | `Assert.IsFalse(b)` |
| Throws | `Assert.Throws<T>(() => {})` | `Assert.Throws<T>(() => {})` | `Assert.ThrowsException<T>(() => {})` |
| Collection | `Assert.All(c, i => {})` | `Assert.Multiple(() => {})` | foreach + Assert |

#### Using Statements

| Framework | Namespace |
|-----------|-----------|
| xUnit | `using Xunit;` |
| NUnit | `using NUnit.Framework;` |
| MSTest | `using Microsoft.VisualStudio.TestTools.UnitTesting;` |

### Step 3: Class Structure

**xUnit**:
```csharp
public class MyTests
{
    private readonly IMapper _mapper;
    
    public MyTests() // Setup in constructor
    {
        _mapper = CreateMapper();
    }
    
    [Fact]
    [Trait("Category", "Characterization")]
    public void Test() { }
}
```

**NUnit**:
```csharp
[TestFixture]
[Category("Characterization")]
public class MyTests
{
    private IMapper _mapper;
    
    [SetUp]
    public void Setup()
    {
        _mapper = CreateMapper();
    }
    
    [Test]
    public void Test() { }
}
```

**MSTest**:
```csharp
[TestClass]
[TestCategory("Characterization")]
public class MyTests
{
    private IMapper _mapper;
    
    [TestInitialize]
    public void Setup()
    {
        _mapper = CreateMapper();
    }
    
    [TestMethod]
    public void Test() { }
}
```

## Output
Framework-appropriate test code syntax.

## Example Transformation

**Input (generic)**:
```
Test: Maps_User_To_UserDto
Assert: result.Name equals source.Name
Assert: result is not null
```

**Output (xUnit)**:
```csharp
[Fact]
public void Maps_User_To_UserDto()
{
    var result = _mapper.Map<UserDto>(source);
    Assert.NotNull(result);
    Assert.Equal(source.Name, result.Name);
}
```

**Output (NUnit)**:
```csharp
[Test]
public void Maps_User_To_UserDto()
{
    var result = _mapper.Map<UserDto>(source);
    Assert.IsNotNull(result);
    Assert.AreEqual(source.Name, result.Name);
}
```
