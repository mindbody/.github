You are performing a .NET major version upgrade (e.g., net8.0 → net10.0). Follow these rules precisely.

## Project Files
- Class libraries already multi-targeting: replace old netX.0 with new, keep netstandard2.x targets.
- Class libraries single-targeting netstandard: no TargetFramework change needed.
- Test/app projects: update singular <TargetFramework> directly to new version.
- Check global.json in solution root FIRST. Update SDK version before any dotnet CLI calls
  or you'll get NETSDK1045 errors. Preserve existing rollForward policy.

## Package Version Management
Before updating any packages, check whether a Directory.Packages.props file exists in the solution root.

**If Directory.Packages.props EXISTS (Central Package Management):**
- Update ALL package versions in Directory.Packages.props only, as <PackageVersion> entries.
- .csproj files must NOT have Version= attributes on <PackageReference> elements.
- Use VersionOverride= in a .csproj only for deliberate per-project exceptions.
- Do not use `dotnet add package` to update versions — edit Directory.Packages.props directly.

**If Directory.Packages.props does NOT exist:**
- Update Version= attributes directly in each .csproj file.
- Use explicit version numbers — no wildcards.
- Do not use `dotnet add package` to update versions — edit .csproj files directly.

## Packages (applies to both CPM and non-CPM)
- Microsoft.Extensions.*: update ALL to the SAME version matching the new .NET version (e.g., 10.0.x).
- System.Text.Json: update to match .NET version.
- Run `dotnet list package --outdated` and `--deprecated` to identify packages needing updates.
- Third-party: update to latest stable. If NU1605 downgrade errors appear, update the package
  in ALL projects in the solution.
- ⚠️ FluentAssertions: NEVER upgrade to 8.0 or higher (license changed from Apache 2.0 to proprietary). If below 7.2.2, upgrade to 7.2.2. Add an XML comment in .csproj or Directory.Packages.props explaining the version cap.
- Microsoft.NET.Test.Sdk: Update to latest stable. Do not downgrade.
- Deprecated packages: update to latest stable if available; if not, add XML comment with migration plan.
- Mindbody.*: update to latest stable if available.

## Dockerfiles
- Update all FROM mcr.microsoft.com/dotnet/sdk:{old} and dotnet/aspnet:{old} to new version.
- Sync any ARG version values (e.g., NewRelic agent) to match updated NuGet references.
- Update pinned `dotnet tool install --version` values to latest available.

## infrastructure.yaml
If infrastructure.yaml exists and has a "lambdaFunctions" section, for each function in the section:
- Update "runtime" value to "dotnet10".
- Increase the value for "memorySize" by half, capping at 3008.
- If "layers" contains a value matching "arn:aws:lambda:us-west-2:451483290750:layer:NewRelicDotnet:*",
  update the trailing version number to "49".

## .NET 10 Breaking Changes
- X509Certificate2 constructors are obsolete (SYSLIB0057). Replace with X509CertificateLoader:
  - new X509Certificate2(byte[]) → X509CertificateLoader.LoadCertificate(byte[])
  - new X509Certificate2(path, pwd) → X509CertificateLoader.LoadPkcs12FromFile(path, pwd)
- WebHostBuilder deprecated (ASPDEPR004/ASPDEPR008): suppress with #pragma warnings on the
  affected lines; create follow-up work item for full WebApplication migration.

## Validation (in order)
1. `dotnet restore` — no errors
2. `dotnet build --configuration Release` — 0 errors
3. `dotnet test --configuration Release` — 100% pass rate
4. `docker build` each Dockerfile — succeeds
5. `dotnet list package --deprecated` — none remaining (or documented)

## Documentation
Update README.md: required SDK version, IDE version, target frameworks, key dependency versions,
build/Docker instructions.

## Commit
- Do NOT stage .github/upgrades/ files.
- Commit message must list: framework changes, package updates, deprecated packages resolved.

## Pull Request
- Title must start with "Upgrade to .NET 10".
- If any .csproj has <GeneratePackageOnBuild>true</GeneratePackageOnBuild> or explicit <IsPackable>true</IsPackable>, a Dockerfile contains 'dotnet pack', or any file in ci/ contains the text "ci/public/nugetJobs.yml", title must end with " +semver:breaking".
- Use template in .github/pull_request_template.md if present.
