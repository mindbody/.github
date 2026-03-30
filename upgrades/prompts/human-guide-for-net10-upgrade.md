# Custom Upgrade Instructions: .NET Version Upgrade

## Scenario

Upgrading a .NET solution to a newer major version (e.g., .NET 8 → .NET 10, .NET 9 → .NET 11).

## Scope

This instruction applies to:
- ✅ Solutions with class libraries and test projects
- ✅ Projects using Microsoft.Extensions.* packages
- ✅ Projects with Dockerfiles
- ✅ Projects requiring documentation updates
- ✅ Projects with or without code changes
- ✅ Projects that publish NuGet packages (optional metadata enhancement)

## Prerequisites

Before starting upgrade:
1. Verify target .NET SDK installed: `dotnet --version`
2. Create upgrade branch: `git checkout -b upgrade-to-NET{version}`
3. Ensure no pending uncommitted changes
4. Run assessment to identify breaking changes

## Upgrade Strategy Selection

Based on solution complexity:

**Small Solutions** (≤5 projects, simple dependencies):
- Use **All-At-Once Strategy**: Update all projects simultaneously
- Faster completion, single validation cycle

**Large Solutions** (>5 projects, complex dependencies):
- Use **Incremental Strategy**: Update projects in topological order
- Safer for complex dependency chains, allows gradual validation

## Step-by-Step Instructions

### Phase 1: Update Project Files

#### 1.1 Update TargetFramework(s) in Project Files

**For Class Libraries** (maintain backward compatibility):
- Use multi-targeting only if project already uses multi-targeting
  - change "net8.0" to "net10.0"
  - keep "netstandard2.0" or "netstandard2.1"
- Keep existing "netstandard" targets for backward compatibility

**Example**:
```xml
<!-- Before -->
<TargetFrameworks>netstandard2.1;net8.0</TargetFrameworks>

<!-- After (remove net8.0 and add net10.0) -->
<TargetFrameworks>netstandard2.1;net10.0</TargetFrameworks>
```

**Example**:
```xml
<!-- Before -->
<TargetFramework>netstandard2.1</TargetFramework>

<!-- After (no change) -->
<TargetFramework>netstandard2.1</TargetFramework>
```

**For Test Projects** (no backward compatibility needed):
- Directly upgrade to new target
- Use singular `<TargetFramework>` (not plural)

**Example**:
```xml
<!-- Before -->
<TargetFramework>net8.0</TargetFramework>

<!-- After -->
<TargetFramework>net10.0</TargetFramework>
```

**For Web/Console Applications** (single target):
- Directly upgrade to new target

**Example**:
```xml
<!-- Before -->
<TargetFramework>net8.0</TargetFramework>

<!-- After -->
<TargetFramework>net10.0</TargetFramework>
```

---

### Phase 2: Update Package Dependencies

#### 2.1 Update Microsoft.Extensions.* Packages

**Rule**: Keep all Microsoft.Extensions.* packages at the **same version** for consistency.

**Find these packages** (common ones):
- Microsoft.Extensions.Configuration
- Microsoft.Extensions.Configuration.Binder
- Microsoft.Extensions.DependencyInjection
- Microsoft.Extensions.Hosting
- Microsoft.Extensions.Hosting.Abstractions
- Microsoft.Extensions.Logging
- Microsoft.Extensions.Logging.Abstractions
- Microsoft.Extensions.Options

**Update all to target .NET version** (e.g., for .NET 10):
```xml
<PackageReference Include="Microsoft.Extensions.Configuration" Version="10.0.5" />
<PackageReference Include="Microsoft.Extensions.Logging" Version="10.0.5" />
<!-- Update ALL Microsoft.Extensions.* to same version -->
```

#### 2.2 Update System.Text.Json

```xml
<!-- Before -->
<PackageReference Include="System.Text.Json" Version="9.0.2" />

<!-- After (match .NET version) -->
<PackageReference Include="System.Text.Json" Version="10.0.5" />
```

#### 2.3 Update Third-Party Packages

**Check for latest compatible versions** using:
```sh
dotnet list package --outdated
```

**Check for deprecated packages**:
```sh
dotnet list package --deprecated
```

**Example** (NewRelic APM):
```xml
<!-- Before -->
<PackageReference Include="NewRelic.Agent.Api" Version="10.36.0" />

<!-- After (check latest version) -->
<PackageReference Include="NewRelic.Agent.Api" Version="10.49.0" />
```

##### 2.3.1 Handling Deprecated Packages

**If assessment shows deprecated packages**, determine upgrade strategy:

**Step 1: Identify Deprecated Packages**

Assessment will flag packages with deprecation warnings:
```
⚠️ NuGet package is deprecated
```

**Step 2: Check for Latest Stable Version**

Use `dotnet add` to query available versions:
```sh
# This will query NuGet and show latest available version
dotnet add package <PackageName>
# Review output to find latest stable version
```

**Step 3: Decision Tree**

```
Is package critical for functionality?
├─ YES → Check if newer version exists that's not deprecated
│   ├─ Newer version exists → Update to latest stable
│   └─ No newer version → Keep current, document deprecation, plan future migration
└─ NO → Consider removing if not needed
```

**Example: NewRelic.Agent.Api Upgrade**

**Scenario**: Assessment shows NewRelic.Agent.Api 10.29.0 is deprecated

**Action**:
1. Query latest version:
   ```sh
   dotnet add package NewRelic.Agent.Api
   # Shows: Latest version is 10.49.0
   ```

2. Update package in all affected projects:
   ```xml
   <!-- Before (deprecated) -->
   <PackageReference Include="NewRelic.Agent.Api" Version="10.29.0" />

   <!-- After (latest stable) -->
   <PackageReference Include="NewRelic.Agent.Api" Version="10.49.0" />
   ```

3. Verify build and functionality:
   ```sh
   dotnet restore
   dotnet build
   # Verify APM still functions correctly
   ```

**Common Deprecated Packages and Strategies**:

| Package | Typical Issue | Strategy |
|---------|---------------|----------|
| **NewRelic.Agent.Api** | Older versions deprecated | Update to latest 10.x version (10.49.0+) |
| **System.Data.SqlClient** | Deprecated in favor of Microsoft.Data.SqlClient | Migrate to Microsoft.Data.SqlClient |
| **Microsoft.AspNetCore.Mvc.NewtonsoftJson** | May show deprecation warnings but still supported | Keep if using Newtonsoft.Json, consider System.Text.Json for new code |
| **Logging frameworks** | Older versions may be deprecated | Update to latest stable within same major version |

**Best Practices**:
- ✅ Always check for non-deprecated alternatives before keeping deprecated packages
- ✅ Update deprecated packages to latest stable version if available
- ✅ Document any deprecated packages you must keep (with reason)
- ✅ Plan migration away from deprecated packages in next development cycle
- ❌ Don't ignore deprecation warnings - they indicate future breaking changes

**When to Keep Deprecated Package**:
- No non-deprecated alternative exists
- Migration requires extensive code changes beyond upgrade scope
- Package still functions correctly in new .NET version
- Migration can be planned for separate work item

**Document Deprecation**:
```xml
<!-- Package deprecated but functional - plan migration in Q2 2024 -->
<PackageReference Include="OldPackage" Version="1.2.3" />
```

#### 2.4 ⚠️ CRITICAL: FluentAssertions Version Constraint

**DO NOT upgrade FluentAssertions beyond 7.2.1**

**Reason**: License changed from Apache 2.0 to proprietary starting with version 8.0.0

**Correct**:
```xml
<PackageReference Include="FluentAssertions" Version="7.2.1" />
```

**INCORRECT (Do not use)**:
```xml
<PackageReference Include="FluentAssertions" Version="8.0.0" />
<PackageReference Include="FluentAssertions" Version="8.1.0" />
<!-- Any 8.x or higher version has license restrictions -->
```

**If assessment suggests upgrading FluentAssertions**:
- ✅ Keep at 7.2.1 (latest Apache 2.0 licensed version)
- ✅ Document this constraint in comments
- ✅ Consider alternatives if 8.x features needed: XUnit.Assert, Shouldly, NUnit Constraints

**Add comment in project file**:
```xml
<!-- FluentAssertions pinned at 7.2.1 - do not upgrade to 8.x (license changed to proprietary) -->
<PackageReference Include="FluentAssertions" Version="7.2.1" />
```

#### 2.5 Update Microsoft.NET.Test.Sdk

**Rule**: **MUST** update to latest 17.x version for best compatibility with .NET 10.

**Minimum Required Version**: 17.14.1 or later (latest 17.x)

**Why this update is mandatory**:
- Older 17.x versions (like 17.11.0 or 17.13.0) may have bugs or compatibility issues
- Version 17.14.1+ includes important fixes for .NET 10 compatibility
- Version 18.x introduces breaking changes for some test frameworks (avoid)
- Version 17.x is the stable LTS line for .NET 10

**Update example**:
```xml
<!-- Before (older 17.x - MUST UPDATE) -->
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.11.0" />
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.13.0" />

<!-- After (latest 17.x - REQUIRED) -->
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.14.1" />
```

**How to find and update to latest 17.x version**:

**Step 1: Check current version**:
```sh
# Check what version you currently have
dotnet list package | Select-String "Microsoft.NET.Test.Sdk"
```

**Step 2: Query latest 17.x version**:
```sh
# This queries NuGet for latest 17.x version
dotnet add package Microsoft.NET.Test.Sdk --version "17.*"
# Review output to see resolved version (e.g., 17.14.1)
```

**Step 3: Update all test projects explicitly**:
```sh
# Update each test project to explicit version (not wildcard)
dotnet add Tests.Unit/Tests.Unit.csproj package Microsoft.NET.Test.Sdk --version 17.14.1
dotnet add Tests.Behavioral/Tests.Behavioral.csproj package Microsoft.NET.Test.Sdk --version 17.14.1
# Repeat for all test projects
```

**⚠️ Important: Use explicit version, not wildcard**:
```xml
<!-- ❌ INCORRECT - Don't use wildcard in committed code -->
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.*" />

<!-- ✅ CORRECT - Use explicit version -->
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.14.1" />
```

**Verification**:
```sh
# After update, verify all test projects have 17.14.1+
Get-ChildItem -Recurse -Filter "*.csproj" | Select-String "Microsoft.NET.Test.Sdk"

# Expected: All test projects show Version="17.14.1" or later
```

**Common mistake to avoid**:
- ❌ Leaving test projects at 17.11.0 or 17.13.0 because "they still work"
- ✅ Always update to latest 17.x even if older version appears compatible
- ✅ Include Microsoft.NET.Test.Sdk update in commit message

**Affected project types**:
- All test projects (*.Tests.csproj, *.Test.csproj, projects with `<IsTestProject>true</IsTestProject>`)
- Typically: Unit tests, Integration tests, Behavioral tests, Contract tests, End-to-end tests

#### 2.6 Packages That Usually Don't Need Updates

These packages are typically framework-agnostic:
- coverlet.msbuild (code coverage)
- xunit.runner.visualstudio (usually already compatible)
- Moq (mocking framework - version 4.x compatible)

---

### Phase 3: Update Dockerfiles

#### 3.1 Update Base Image Versions

**Find and replace SDK images**:

```dockerfile
# Before
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build

# After
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
```

**Find and replace runtime images** (if used):

```dockerfile
# Before
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime

# After
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
```

#### 3.2 Update Version Pins for .NET Tools

Look for comments indicating version pins are out of date:

```dockerfile
# Before
# When we would upgrade this image to the fresh .NET Core, we can remove this specific version
RUN dotnet tool install -g dotnet-reportgenerator-globaltool --version 5.1.26

# After
# Updated for .NET 10 - using latest reportgenerator version
RUN dotnet tool install -g dotnet-reportgenerator-globaltool --version 5.5.3
```

#### 3.3 Validate Dockerfile Changes

After updating:
```sh
# Build each Dockerfile to verify
docker build -f path/to/Dockerfile .
```

---

### Phase 3.5: Add NuGet Package Metadata (Optional - For Library Projects)

**When to Apply**: If upgrading a project that publishes NuGet packages (has `GeneratePackageOnBuild` or is packaged).

**Purpose**: Ensure proper package discoverability and professional presentation in NuGet feeds.

#### 3.5.1 Check if Project Generates Packages

Look for these indicators in the `.csproj` file:

```xml
<GeneratePackageOnBuild>true</GeneratePackageOnBuild>
```

Or if project is explicitly packed via CI/CD pipeline.

#### 3.5.2 Add Minimal NuGet Metadata

**Recommended properties** to add if missing:

```xml
<PropertyGroup>
  <TargetFramework>netstandard2.1</TargetFramework>
  <GeneratePackageOnBuild>true</GeneratePackageOnBuild>
  <Version>1.0.0</Version>

  <!-- NuGet Package Metadata -->
  <PackageId>YourCompany.YourPackage</PackageId>
  <Authors>Your Company Name</Authors>
  <Company>Your Company Name</Company>
  <Product>YourCompany.YourPackage</Product>
  <Description>Brief description of what this package does. Include key features and use cases.</Description>
  <PackageTags>keywords;separated;by;semicolon</PackageTags>
  <Copyright>Copyright (c) Your Company 2026</Copyright>
  <PackageProjectUrl>https://github.com/yourorg/yourrepo</PackageProjectUrl>
  <RepositoryUrl>https://github.com/yourorg/yourrepo</RepositoryUrl>
  <RepositoryType>git</RepositoryType>
  <PackageReadmeFile>README.md</PackageReadmeFile>
</PropertyGroup>
```

#### 3.5.3 Include README in Package

Add an `ItemGroup` to include the README file:

```xml
<ItemGroup>
  <!-- Include README in NuGet package -->
  <None Include="..\README.md" Pack="true" PackagePath="\" />
</ItemGroup>
```

Or if README is in the project directory:
```xml
<ItemGroup>
  <None Include="README.md" Pack="true" PackagePath="\" />
</ItemGroup>
```

#### 3.5.4 Essential vs. Optional Metadata

**Essential** (minimum for good package):
- `PackageId` - Unique package identifier
- `Version` - Semantic version number
- `Authors` - Package authors
- `Description` - Clear description of functionality

**Recommended**:
- `PackageTags` - Keywords for discoverability
- `RepositoryUrl` - Source code location
- `PackageReadmeFile` - Include README for context

#### 3.5.6 Validate Package Metadata

After adding metadata:

```sh
# Build and pack the project
dotnet pack ProjectName.csproj --configuration Release

# Verify package was created with metadata
# Check the .nupkg file in bin/Release/
```

**Best Practices**:
- ✅ Keep descriptions clear and concise (1-2 sentences)
- ✅ Use relevant, searchable tags
- ✅ Include README with usage examples
- ✅ Link to source repository
- ❌ Don't use excessive tags (5-8 is enough)
- ❌ Don't leave default "Package Description" text
- ❌ Don't specify license (package is for company use only)

---

### Phase 4: Update Documentation

#### 4.1 Update README.md

**Required SDK Version**:
```markdown
# Before
- [.NET Core 8.0](https://dotnet.microsoft.com/en-us/download/dotnet/8.0)

# After
- [.NET 10.0 SDK](https://dotnet.microsoft.com/en-us/download/dotnet/10.0) (LTS)
```

**Update Target Frameworks Section** (if present):
```markdown
## Target Frameworks

**ProjectName** (Library):
- .NET Standard 2.1

**ProjectName.Tests** (Tests):
- .NET 10.0
```

**Add/Update Key Dependencies Section**:
```markdown
## Key Dependencies

- Microsoft.Extensions.* packages: 10.0.5
- System.Text.Json: 10.0.5
- [List other major dependencies with versions]
```

**Add Build Instructions** (if not present):
```markdown
## Using .NET CLI
\`\`\`bash
# Restore dependencies
dotnet restore Solution.sln

# Build solution
dotnet build Solution.sln --configuration Release

# Run tests
dotnet test Solution.sln
\`\`\`
```

**Add Docker Instructions** (if Dockerfiles present):
```markdown
## Using Docker
\`\`\`bash
# Build application image
docker build -t appname:latest -f path/to/Dockerfile .

# Run tests in container
docker build -t appname-tests:latest -f path/to/Tests.Dockerfile .
docker run --rm appname-tests:latest
\`\`\`
```

#### 4.2 Update Visual Studio Requirements

```markdown
# Before
- Visual Studio 2017 v15.7 or newer

# After
- Visual Studio 2022 or newer (or any IDE with .NET 10 support)
```

#### 4.3 Update Extension Links

Remove version-specific references:
```markdown
# Before
[Extension Name, VS2017 extension](url)

# After
[Extension Name](url)
```

---

### Phase 5: Validate Changes

#### 5.1 Restore and Build

```sh
# Clean previous artifacts
dotnet clean Solution.sln

# Restore packages
dotnet restore Solution.sln

# Build all configurations
dotnet build Solution.sln --configuration Release --no-restore

# Verify build succeeds with 0 errors
```

#### 5.2 Validate Multi-Targeting

**For multi-targeted projects**, verify each framework builds:

```sh
# Build for each target framework
dotnet build Project.csproj --framework netstandard2.1
dotnet build Project.csproj --framework net10.0

# Verify output assemblies exist
ls Project/bin/Release/netstandard2.1/
ls Project/bin/Release/net10.0/
```

#### 5.3 Run All Tests

```sh
# Run test suite
dotnet test Solution.sln --configuration Release --logger "console;verbosity=normal"

# Verify:
# - All tests execute (no infrastructure failures)
# - 100% pass rate (0 failures)
# - No unexpected skipped tests
```

#### 5.4 Validate Docker Builds

```sh
# Test each Dockerfile
docker build -f Dockerfile1 .
docker build -f Dockerfile2 .

# Run containerized tests if applicable
docker run --rm <test-image>
```

#### 5.5 Verify Deprecated Packages Addressed

**Check for remaining deprecated packages**:
```sh
dotnet list package --deprecated
```

**Expected outcome**: 
- ✅ No deprecated packages remaining (ideal)
- ✅ OR deprecated packages documented with migration plan

**If deprecated packages remain**:
1. Review each package individually
2. Verify it was addressed per §2.3.1 strategy
3. Document reason for keeping (no alternative, future migration planned)
4. Add comment to .csproj explaining deprecation

**Example validation**:
```sh
# Before upgrade
dotnet list package --deprecated
# Output: NewRelic.Agent.Api 10.29.0 (deprecated)

# After upgrade
dotnet list package --deprecated
# Output: No deprecated packages found ✅
```

**If assessment identified deprecated packages but none found after upgrade**:
- ✅ Deprecation successfully resolved by updating to latest version
- ✅ Document updated version in commit message

#### 5.6 Verify Microsoft.NET.Test.Sdk Version

**Check all test projects have latest 17.x version**:
```sh
# Check all test project versions
Get-ChildItem -Recurse -Filter "*Test*.csproj" | Select-String "Microsoft.NET.Test.Sdk"

# Or using dotnet CLI
dotnet list package | Select-String "Microsoft.NET.Test.Sdk"
```

**Expected outcome**:
- ✅ All test projects show **17.14.1 or later** (NOT 17.11.0, 17.13.0, or 18.x)
- ✅ All test projects have **same version** (consistency)
- ✅ No test projects left on older 17.x versions

**Common issue to catch**:
```sh
# ❌ BAD - Older 17.x version missed during upgrade
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.11.0" />

# ✅ GOOD - Latest 17.x version
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.14.1" />
```

**If older version found**:
1. Update to 17.14.1 per §2.5
2. Restore and rebuild
3. Re-run tests to verify compatibility
4. Include update in commit message

---

### Phase 6: Handle Code Changes

#### 6.1 API Breaking Changes

**If assessment detects breaking changes**:

1. **Review compiler errors** after build
2. **Consult breaking changes documentation**: 
   - https://learn.microsoft.com/en-us/dotnet/core/compatibility/10.0
3. **Common patterns to fix**:

**Obsolete APIs**:
```csharp
// Before (obsolete)
var result = OldApiMethod();

// After (replacement)
var result = NewApiMethod();
```

**Changed Method Signatures**:
```csharp
// Before
void Method(string param)

// After (parameter added)
void Method(string param, CancellationToken cancellationToken = default)
```

**Namespace Changes**:
```csharp
// Before
using Old.Namespace;

// After
using New.Namespace;
```

#### 6.2 No Code Changes Scenario

**If assessment shows 0 breaking changes**:
- ✅ No code modifications needed
- ✅ Only project files and packages updated
- ✅ Validate with build and tests

---

### Phase 7: Commit Changes

#### 7.1 Stage Files

Do NOT stage files under .github/upgrades/

```sh
# Stage project files
git add **/*.csproj

# Stage Dockerfiles
git add **/Dockerfile

# Stage documentation
git add README.md

# Verify staged files
git status
git diff --cached
```

#### 7.2 Commit with Descriptive Message

**Template**:
```
Upgrade solution to .NET {version} LTS

- Update {Project1} to multi-target netstandard2.1;net{version}
- Update {Project2} to target net{version}
- Upgrade Microsoft.Extensions.* packages from {old} to {new}
- Upgrade System.Text.Json from {old} to {new}
- Update Dockerfiles to use .NET {version} SDK
- Update README with .NET {version} requirements

Assessment Results:
- {X} API breaking changes detected
- All tests passing (100% pass rate)

Validated:
- ✅ Solution builds successfully for all target frameworks
- ✅ All unit tests pass
- ✅ Docker containers build successfully
```

---

## Package-Specific Constraints

### FluentAssertions License Constraint

**⚠️ CRITICAL CONSTRAINT: FluentAssertions ≤ 7.2.1 ONLY**

**Rule**: Never upgrade FluentAssertions beyond version 7.2.1

**Reason**: 
- **Version 7.2.1 and earlier**: Apache License 2.0 (permissive open source)
- **Version 8.0.0 and later**: Proprietary license (Community License with restrictions)

**License Changes**:
- FluentAssertions 8.x requires compliance with new Community License
- Commercial use may require purchasing a license
- Apache 2.0 versions (≤7.2.1) have no such restrictions

**Implementation**:

1. **Pin version in project files**:
```xml
<!-- FluentAssertions pinned at 7.2.1 - do not upgrade to 8.x (license changed to proprietary) -->
<PackageReference Include="FluentAssertions" Version="7.2.1" />
```

2. **If assessment suggests upgrading**:
   - ✅ Override recommendation - keep at 7.2.1
   - ✅ Document reason in upgrade notes
   - ✅ Add comment in project file

3. **If 8.x features are needed**:
   - Consider alternatives: XUnit.Assert, Shouldly, NUnit Constraints
   - Or obtain commercial license for FluentAssertions 8.x
   - Evaluate license compliance with legal team

4. **During package updates**:
   - Always check FluentAssertions version after `dotnet restore`
   - Verify it remains at 7.2.1
   - Add explicit version constraint if needed

**Validation**:
```sh
# After restore, verify FluentAssertions version
dotnet list package | Select-String "FluentAssertions"

# Output should show: FluentAssertions 7.2.1 (not 8.x)
```

---

## Dockerfile Update Patterns

### Standard SDK Image Update

**Pattern**: Replace SDK version in FROM statements

```dockerfile
# Before
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build

# After
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
```

### Standard Runtime Image Update

```dockerfile
# Before
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime

# After
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
```

### Multi-Stage Dockerfile Pattern

```dockerfile
# Build stage - uses SDK
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY . .
RUN dotnet restore
RUN dotnet build -c Release
RUN dotnet publish -c Release -o /app

# Runtime stage - uses smaller runtime image
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "YourApp.dll"]
```

### Update Version Pins for .NET Tools

**Pattern**: Update `--version X.Y.Z` when encountered and newer minor version available

```dockerfile
# Before
# When we would upgrade this image to the fresh .NET Core, we can remove this specific version
RUN dotnet tool install -g some-tool --version 5.1.26

# After
# Updated for .NET {version} - using latest tool version
RUN dotnet tool install -g some-tool --version 5.5.3
```

**Common Tools**:
- dotnet-reportgenerator-globaltool
- dotnet-ef
- dotnet-format

---

## Dependency Update Strategy

### Coordinated Package Updates

**Rule**: Update related packages together in a single operation

**Microsoft.Extensions.* Suite** (7-15 packages typically):
- All updated to same version (e.g., 10.0.5)
- Updated simultaneously to avoid version conflicts
- Maintains internal compatibility

**Example batch update**:
```xml
<!-- Update all together -->
<PackageReference Include="Microsoft.Extensions.Configuration" Version="10.0.5" />
<PackageReference Include="Microsoft.Extensions.Logging" Version="10.0.5" />
<PackageReference Include="Microsoft.Extensions.Options" Version="10.0.5" />
<!-- All set to 10.0.5 -->
```

### Independent Package Updates

**Third-party packages**: Update to latest compatible version independently

**Check latest version**:
```sh
dotnet list package --outdated
```

**Or query specific package**:
```sh
# Check NuGet.org for latest version
dotnet add package NewRelic.Agent.Api --version 10.49.0 --no-restore
# Then verify in .csproj
```

---

## Build Validation

### Multi-Framework Build Validation

**Verify all target frameworks build successfully**:

```sh
# For multi-targeted projects, build each framework
dotnet build Project.csproj --framework netstandard2.1 --no-restore
dotnet build Project.csproj --framework net10.0 --no-restore

# Verify output assemblies exist
ls Project/bin/Release/netstandard2.1/Project.dll
ls Project/bin/Release/net10.0/Project.dll
```

### Success Criteria

**Build must**:
- ✅ Complete with **0 errors**
- ✅ Produce assemblies for all target frameworks
- ✅ Have acceptable warning count (document new warnings)
- ✅ No package downgrade warnings
- ✅ No package conflict errors

---

## Test Validation

### Test Execution

```sh
# Run full test suite
dotnet test Solution.sln --configuration Release --logger "console;verbosity=normal"

# Verify:
# - Total tests: N
# - Passed: N
# - Failed: 0
# - Skipped: 0 (or expected skips documented)
```

### Expected Test Behavior

**No code changes scenario** (0 breaking changes):
- ✅ All tests should pass unchanged
- ✅ 100% pass rate expected
- ✅ If tests fail, investigate framework behavior changes

**With code changes scenario** (breaking changes detected):
- ⚠️ Some tests may need updates
- ⚠️ Review test failures for expected behavior changes
- ⚠️ Update test expectations if new framework behavior is correct

### Test Framework Compatibility

**xunit**: Usually compatible, no update needed (2.9.x supports .NET 10)  
**NUnit**: Usually compatible, update to latest 3.x or 4.x  
**MSTest**: Usually compatible, update to latest if needed

---

## Documentation Updates Checklist

### README.md Updates

- [ ] Update required .NET SDK version
- [ ] Update Visual Studio / IDE requirements
- [ ] Add/update Target Frameworks section
- [ ] Add/update Key Dependencies section with package versions
- [ ] Add/update build instructions (.NET CLI commands)
- [ ] Add/update Docker build instructions (if Dockerfiles present)
- [ ] Update extension links (remove version-specific references)
- [ ] Update copyright year if needed

### Optional Documentation

- [ ] CHANGELOG.md - Add entry for .NET version upgrade
- [ ] CONTRIBUTING.md - Update development environment requirements
- [ ] Developer setup guides - Update .NET SDK installation steps
- [ ] Deployment documentation - Update runtime requirements

---

## Source Control Strategy

### Commit Strategy

**Single Atomic Commit** (recommended for small solutions):
```sh
git add **/*.csproj **/Dockerfile README.md
git commit -m "Upgrade solution to .NET {version} LTS"
```

**Multiple Commits** (for complex solutions or team preference):
```sh
# Commit 1: Project files
git add **/*.csproj
git commit -m "Update target frameworks to .NET {version}"

# Commit 2: Packages
git add **/*.csproj
git commit -m "Upgrade packages for .NET {version}"

# Commit 3: Dockerfiles
git add **/Dockerfile
git commit -m "Update Dockerfiles to .NET {version}"

# Commit 4: Documentation
git add README.md
git commit -m "Update documentation for .NET {version}"
```

### Commit Message Template

```
Upgrade solution to .NET {version} LTS

- Update {ProjectName} to multi-target netstandard2.1;net8.0;net{version}
- Update {TestProjectName} to target net{version}
- Upgrade Microsoft.Extensions.* packages from {oldVersion} to {newVersion}
- Upgrade System.Text.Json from {oldVersion} to {newVersion}
- Upgrade Microsoft.NET.Test.Sdk from 17.11.0 to 17.14.1 (latest 17.x)
- [List other significant package updates]
- Update Dockerfiles to use .NET {version} SDK
- Update README with .NET {version} requirements

Deprecated Package Updates:
- Upgrade NewRelic.Agent.Api from 10.29.0 to 10.49.0 (resolved deprecation)
- [List other deprecated packages updated]

Assessment Results:
- {X} API breaking changes detected
- All {N} APIs analyzed for compatibility
- {Y} deprecated packages identified and resolved
- All tests passing (100% pass rate)

Validated:
- ✅ Solution builds successfully for all target frameworks
- ✅ All unit tests pass
- ✅ Docker containers build successfully
- ✅ Multi-targeting works correctly
- ✅ No deprecated packages remaining (or documented if kept)
- ✅ Microsoft.NET.Test.Sdk at 17.14.1+ in all test projects
```

---

## Common Issues and Solutions

### Issue 1: Package Conflict After Restore

**Symptom**: Version conflict errors during `dotnet restore`

**Solution**:
1. Check for mismatched Microsoft.Extensions.* versions
2. Ensure all are at same version (e.g., all 10.0.5)
3. Clear NuGet cache: `dotnet nuget locals all --clear`
4. Restore again: `dotnet restore`

### Issue 2: Build Errors After Framework Update

**Symptom**: Compilation errors referencing missing APIs

**Solution**:
1. Check .NET breaking changes documentation
2. Search for API in documentation: https://learn.microsoft.com/en-us/dotnet/core/compatibility/
3. Use suggested replacement APIs
4. Add conditional compilation if multi-targeting requires it:
   ```csharp
   #if NET10_0_OR_GREATER
       // Use new API
   #else
       // Use old API
   #endif
   ```

### Issue 3: Test Failures After Upgrade

**Symptom**: Previously passing tests now fail

**Solution**:
1. Determine if behavior change is expected in new framework
2. Check release notes for behavioral changes
3. Update test expectations if new behavior is correct
4. Fix application code if test reveals actual bug

### Issue 4: Docker Build Fails

**Symptom**: Docker build fails with "image not found" or similar

**Solution**:
1. Verify base image exists: Check https://mcr.microsoft.com/
2. Pull image manually: `docker pull mcr.microsoft.com/dotnet/sdk:10.0`
3. Check internet connectivity / proxy settings
4. Try alternative tags: `10.0-alpine`, `10.0-jammy`

### Issue 5: FluentAssertions Upgraded to 8.x Accidentally

**Symptom**: `dotnet restore` or build upgrades FluentAssertions to 8.x

**Solution**:
1. Immediately downgrade to 7.2.1:
   ```sh
   dotnet add package FluentAssertions --version 7.2.1
   ```
2. Add version constraint comment in .csproj
3. Inform team of license restriction
4. Review license implications if 8.x already in use

### Issue 6: Deprecated Package Warning After Upgrade

**Symptom**: Assessment or `dotnet list package --deprecated` shows deprecated packages

**Solution**:
1. **Check for latest stable version**:
   ```sh
   dotnet add package <PackageName>
   # Review output to see latest available version
   ```

2. **Determine if newer non-deprecated version exists**:
   - **If YES**: Update to latest version per §2.3.1
   - **If NO**: Keep current version, document reason

3. **Example - NewRelic.Agent.Api**:
   ```sh
   # Assessment shows: NewRelic.Agent.Api 10.29.0 (deprecated)

   # Update to latest
   dotnet add package NewRelic.Agent.Api --version 10.49.0

   # Verify deprecation resolved
   dotnet list package --deprecated
   # Should no longer show NewRelic.Agent.Api
   ```

4. **If must keep deprecated package**:
   - Add explanatory comment in .csproj:
     ```xml
     <!-- Package deprecated but no alternative exists - monitoring for replacement -->
     <PackageReference Include="OldPackage" Version="1.2.3" />
     ```
   - Document in commit message
   - Create backlog item to investigate alternatives

**Prevention**:
- ✅ Always run `dotnet list package --deprecated` before AND after upgrade
- ✅ Address deprecations during upgrade (don't defer)
- ✅ Check package release notes for deprecation notices

### Issue 7: Microsoft.NET.Test.Sdk Left at Older 17.x Version

**Symptom**: Test projects still at 17.11.0 or 17.13.0 after upgrade, even though solution appears to work

**Root Cause**: Older 17.x versions are technically compatible with .NET 10, so upgrade wasn't flagged as required

**Problem**: 
- Missing important bug fixes and improvements in 17.14.1+
- Not following organizational standard (§2.5 requires latest 17.x)
- Creates inconsistency across projects

**Solution**:
1. **Check current versions across all test projects**:
   ```sh
   Get-ChildItem -Recurse -Filter "*Test*.csproj" | Select-String "Microsoft.NET.Test.Sdk"
   ```

2. **If any show older than 17.14.1, update all test projects**:
   ```sh
   # Find latest 17.x version
   dotnet add package Microsoft.NET.Test.Sdk --version "17.*"
   # Note the resolved version (e.g., 17.14.1)

   # Update each test project explicitly
   dotnet add Tests.Unit/Tests.Unit.csproj package Microsoft.NET.Test.Sdk --version 17.14.1
   # Repeat for all test projects
   ```

3. **Verify all test projects updated**:
   ```sh
   Get-ChildItem -Recurse -Filter "*Test*.csproj" | Select-String "Microsoft.NET.Test.Sdk"
   # All should show 17.14.1
   ```

4. **Test after update**:
   ```sh
   dotnet restore
   dotnet build
   dotnet test
   # Verify all tests still pass
   ```

5. **Include in commit**:
   - Update commit message to mention: "Upgrade Microsoft.NET.Test.Sdk from 17.11.0 to 17.14.1 (latest 17.x)"

**Prevention**:
- ✅ Always include §5.6 "Verify Microsoft.NET.Test.Sdk Version" in validation
- ✅ Add to pre-commit checklist: "Microsoft.NET.Test.Sdk at 17.14.1+"
- ✅ Don't assume "compatible" means "recommended" - always use latest stable

---

## Validation Checklist

Use this checklist before committing upgrade:

### Project Files
- [ ] All TargetFramework(s) properties updated
- [ ] Multi-targeting configured correctly (plural `TargetFrameworks` for libraries)
- [ ] Single targeting for apps/tests (singular `TargetFramework`)
- [ ] No typos in framework monikers (e.g., `net10.0` not `net10`)

### Package Updates
- [ ] All Microsoft.Extensions.* at same version
- [ ] System.Text.Json updated to match .NET version
- [ ] Third-party packages updated to latest compatible versions
- [ ] ⚠️ FluentAssertions pinned at 7.2.1 (if present)
- [ ] **Microsoft.NET.Test.Sdk updated to 17.14.1+** (latest 17.x, not 18.x)
- [ ] **Deprecated packages addressed** (`dotnet list package --deprecated` shows none or documented)
- [ ] Deprecated package updates included in commit message if any were resolved

### Dockerfiles
- [ ] All FROM statements updated to new SDK version
- [ ] Runtime images updated (if applicable)
- [ ] Version pins removed where indicated by comments
- [ ] Dockerfiles build successfully

### Documentation
- [ ] README.md updated with new .NET version
- [ ] Target frameworks documented
- [ ] Key dependencies listed with versions
- [ ] Build instructions present and accurate
- [ ] Docker instructions present (if applicable)

### Build & Test
- [ ] `dotnet restore` completes without errors
- [ ] `dotnet build` completes with 0 errors
- [ ] All target frameworks build successfully
- [ ] All tests pass (100% pass rate)
- [ ] Docker containers build successfully

### Source Control
- [ ] All changes staged
- [ ] No .github/upgrades/scenarios/ files committed
- [ ] No build artifacts committed (bin/, obj/)
- [ ] Commit message descriptive and complete
- [ ] Changes on appropriate branch

---

## Post-Upgrade Actions

### Immediate (Same Day)

1. **Push upgrade branch**:
   ```sh
   git push origin upgrade-to-NET{version}
   ```

2. **Create Pull Request** with:
   - Summary of changes
   - Test results
   - Any caveats or notes

3. **Validate CI/CD**:
   - Ensure build agents have new .NET SDK
   - Verify pipeline builds successfully
   - Check Docker image builds in CI

### Short-Term (Within 1 Week)

1. **Monitor application** after merge:
   - Check logs for unexpected errors
   - Validate in staging environment
   - Monitor performance metrics

2. **Update team**:
   - Notify of new .NET version requirement
   - Share upgrade documentation
   - Update onboarding guides

3. **Environment updates**:
   - Update developer machines with new SDK
   - Update build servers
   - Update deployment targets (if needed)

### Long-Term (Within 1 Month)

1. **Review for optimizations**:
   - Explore new framework features
   - Consider .NET version-specific performance improvements
   - Update coding patterns to leverage new APIs

2. **Dependency maintenance**:
   - Check for package updates
   - Review security advisories
   - Plan next upgrade cycle

---

## Examples from Actual Upgrade

### Example 1: Library Project Update

**Project**: Mindbody.Messaging.Core.csproj

**Changes Applied**:
```xml
<!-- TargetFramework update -->
<TargetFramework>netstandard2.1</TargetFramework>

<!-- Package updates -->
<PackageReference Include="Microsoft.Extensions.Configuration" Version="10.0.5" />
<PackageReference Include="Microsoft.Extensions.Logging" Version="10.0.5" />
<PackageReference Include="NewRelic.Agent.Api" Version="10.49.0" />
<PackageReference Include="System.Text.Json" Version="10.0.5" />
```

**Build Result**: ✅ Succeeded for all 3 targets (netstandard2.1, net8.0, net10.0)

---

### Example 2: Test Project Update

**Project**: Mindbody.Messaging.Core.Tests.csproj

**Changes Applied**:
```xml
<!-- TargetFramework update (singular) -->
<TargetFramework>net10.0</TargetFramework>

<!-- Packages - no updates needed -->
<PackageReference Include="xunit" Version="2.9.3" />
<PackageReference Include="FluentAssertions" Version="7.2.1" />
<!-- FluentAssertions kept at 7.2.1 due to license -->
```

**Test Result**: ✅ 1 test passed, 0 failed (100% pass rate)

---

### Example 3: Dockerfile Update

**File**: Mindbody.Messaging.Core/Dockerfile

**Change**:
```dockerfile
# Before
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build

# After
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
```

**Build Result**: ✅ Docker build succeeded

---

### Example 4: Test Dockerfile with Tool Update

**File**: Tests.Unit.Dockerfile

**Changes**:
```dockerfile
# Before
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
# When we would upgrade this image to the fresh .NET Core, we can remove this specific version
RUN dotnet tool install -g dotnet-reportgenerator-globaltool --version 5.1.26

# After
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
# Updated for .NET 10 - using latest reportgenerator version
RUN dotnet tool install -g dotnet-reportgenerator-globaltool --version 5.5.3
```

**Build Result**: ✅ Docker build succeeded, tests passed in container

---

## Special Considerations

### APM / Observability Libraries

**NewRelic.Agent.Api, Application Insights, etc.**:
- Update to latest version compatible with new framework
- Test distributed tracing functionality after upgrade
- Verify APM dashboard shows traces correctly
- Check for framework-specific instrumentation updates

**Post-upgrade validation**:
```csharp
// Verify tracing still works
var agent = NewRelic.Api.Agent.NewRelic.GetAgent();
// Test transaction creation
// Validate distributed trace headers
```

### Code Analysis Tools

**Tools affected by framework upgrade**:
- SonarAnalyzer
- StyleCop
- Roslyn analyzers

**Actions**:
- Update analyzer packages if available
- Review new analyzer warnings
- Suppress false positives if needed

### Package Transitivity

**Watch for transitive dependencies**:
- Direct package updates may pull in updated transitive dependencies
- Verify no version conflicts
- Check `obj/project.assets.json` if conflicts suspected

---

## Quality Gates

### Before Committing

- [ ] ✅ Solution builds with 0 errors
- [ ] ✅ All tests pass (100% pass rate)
- [ ] ✅ All target frameworks validated
- [ ] ✅ Dockerfiles build successfully
- [ ] ✅ Documentation updated
- [ ] ✅ No build artifacts in staging area
- [ ] ⚠️ FluentAssertions at 7.2.1 (if present)

### Before Merging to Main

- [ ] ✅ PR created with complete description
- [ ] ✅ Code review completed
- [ ] ✅ CI/CD pipeline passes
- [ ] ✅ Breaking changes documented (if any)
- [ ] ✅ Team notified of new requirements
- [ ] ✅ Assessment and plan documents deleted

---

## Template: Quick Reference

### Minimal Upgrade Commands

```sh
# 1. Create branch
git checkout -b upgrade-to-NET{version}

# 2. Update project files (manually)
# - Update <TargetFramework(s)> properties
# - Update package versions

# 3. Update Dockerfiles (manually)
# - Update FROM mcr.microsoft.com/dotnet/sdk:{version}

# 4. Update README (manually)
# - Update required SDK version

# 5. Build and test
dotnet restore Solution.sln
dotnet build Solution.sln --configuration Release
dotnet test Solution.sln --configuration Release

# 6. Validate Docker
docker build -f Dockerfile .

# 7. Commit
git add **/*.csproj **/Dockerfile README.md
git commit -m "Upgrade solution to .NET {version} LTS"

# 8. Push
git push origin upgrade-to-NET{version}
```

---

## Version History

This instruction was created based on:
- **Source Solution**: Mindbody.Messaging.Core
- **Upgrade**: .NET 8 / .NET Standard 2.1 → .NET Standard 2.1
- **Date**: March 2026
- **Outcome**: ✅ Successful (0 errors, 0 test failures)
- **Complexity**: Low (2 projects, 8 package updates, 2 Dockerfiles)

---

## Notes

- These instructions prioritize **safety** (multi-targeting for libraries)
- Emphasize **validation** at each step
- Assume **All-At-Once strategy** for small solutions
- Can be adapted for **Incremental strategy** for larger solutions
- **FluentAssertions constraint** is critical and non-negotiable for license compliance

---

## Resources

- [.NET Release Schedule](https://dotnet.microsoft.com/platform/support/policy)
- [.NET Breaking Changes](https://learn.microsoft.com/en-us/dotnet/core/compatibility/)
- [Docker .NET Images](https://mcr.microsoft.com/product/dotnet/sdk/about)
- [NuGet Package Metadata](https://learn.microsoft.com/en-us/nuget/create-packages/package-authoring-best-practices)
- [FluentAssertions License Information](https://github.com/fluentassertions/fluentassertions/blob/develop/LICENSE)
