Perform a .NET major version upgrade (e.g., net8.0 to net10.0). Follow these rules precisely.

## Project Files
- Multi-targeting libraries: replace old netX.0 with new, keep netstandard2.x targets.
- Single-target netstandard libraries: no TargetFramework change needed.
- Test/app projects: update singular <TargetFramework> directly to new version.
- If global.json exists, update SDK version and preserve rollForward before any dotnet CLI
  calls (prevents NETSDK1045). If global.json does not exist, create it pinning the target
  SDK version with rollForward: latestFeature.

## Package Version Management
Check for Directory.Packages.props in the solution root before updating any packages.

**If Directory.Packages.props EXISTS (CPM):**
- Update ALL package versions in Directory.Packages.props only, as <PackageVersion> entries.
- .csproj files must NOT have Version= attributes on <PackageReference> elements.
- Use VersionOverride= in a .csproj only for deliberate per-project exceptions.
- Do not use `dotnet add package` to update versions — edit Directory.Packages.props directly.

**If Directory.Packages.props does NOT exist:**
- Update Version= attributes directly in each .csproj file.
- Use explicit version numbers — no wildcards.
- Do not use `dotnet add package` to update versions — edit .csproj files directly.

## Packages (both CPM and non-CPM)
- Microsoft.Extensions.*: update ALL to the SAME version matching the new .NET version (e.g., 10.0.x).
- System.Text.Json: update to match .NET version.
- Run `dotnet list package --outdated` and `--deprecated` to identify packages needing updates.
- Update ALL packages consistently across EVERY project including test projects — no project
  may be left at its old version.
- Verify companion packages are on a mutually compatible version matrix (e.g., xunit and
  xunit.runner.visualstudio).
- Third-party: update to latest stable. If NU1605 downgrade errors appear, update the package
  in ALL projects in the solution.
- ⚠️ FluentAssertions: NEVER upgrade to 8.0+ (license changed to proprietary). If below
  7.2.2, upgrade to 7.2.2. Add XML comment explaining the version cap.
- Microsoft.NET.Test.Sdk: update to latest stable; do not downgrade.
- Deprecated packages: update to latest stable if available. If no replacement exists, add an
  inline comment referencing the recommended migration path (e.g., packages deprecated in
  favor of OpenTelemetry should note that).
- Mindbody.*: update to latest stable if available.

## .NET 10 Breaking Changes
- X509Certificate2 constructors obsolete (SYSLIB0057). Replace with X509CertificateLoader:
  - new X509Certificate2(byte[]) -> X509CertificateLoader.LoadCertificate(byte[])
  - new X509Certificate2(path, pwd) -> X509CertificateLoader.LoadPkcs12FromFile(path, pwd)
- AuthenticationHandler subclasses: remove deprecated ISystemClock parameter; use the
  3-argument constructor.
- TimeSpan.FromSeconds(int): resolve integer-overload ambiguity by passing a double literal
  (e.g., TimeSpan.FromSeconds(30.0)).
- WebHostBuilder (ASPDEPR004/ASPDEPR008): suppress with #pragma on affected lines; create
  follow-up work item to migrate to WebApplicationBuilder.

## aws-lambda-tools-default.json
For each aws-lambda-tools-default.json found in the solution:
- Set "framework" to "net10.0".
- Set "function-runtime" to "dotnet10".

## launchSettings.json
For each launchSettings.json found in the solution:
- In "executablePath": replace dotnet-lambda-test-tool-{old}.exe with
  dotnet-lambda-test-tool-10.0.exe. Preserve the rest of the path.
- In "workingDirectory": replace net{old} with net10.0. Preserve $(Configuration) exactly.

## Dockerfiles
- Update FROM dotnet/sdk:{old} and dotnet/aspnet:{old} to new version.
- Sync ARG version values (e.g., NewRelic) to match updated NuGet references.
- Update pinned `dotnet tool install --version` values to latest.

## infrastructure.yaml
If infrastructure.yaml has a "lambdaFunctions" section, for each function:
- Set "runtime" to "dotnet10".
- Increase "memorySize" by 50%, capping at 3008.
- If "layers" contains "arn:aws:lambda:us-west-2:451483290750:layer:NewRelicDotnet:*",
  update trailing version number to "49".

## Validation
1. `dotnet restore` — no errors
2. `dotnet build --configuration Release` — 0 errors
3. `dotnet test --configuration Release` — 100% pass rate
4. `dotnet list package --deprecated` — none remaining (or documented)

## Scope Discipline
Do NOT stage: .github/upgrades/, tool-generated artifacts, assessment files, execution logs,
plans, or scaffolding. PR diff must contain only production code, configuration,
infrastructure, and documentation changes.

## Documentation
Update README.md: SDK version, IDE version, target frameworks, key dependency versions,
build/Docker instructions.

## Commit
Commit message must list: framework changes, package updates, deprecated packages resolved.

## Pull Request
- Title must start with "Upgrade to .NET 10".
- Append " +semver:breaking" if: any .csproj has GeneratePackageOnBuild=true or explicit
  IsPackable=true, a Dockerfile contains 'dotnet pack', or any ci/ file contains
  'ci/public/nugetJobs.yml'.
- Use .github/pull_request_template.md if present, otherwise use
  https://raw.githubusercontent.com/mindbody/.github/main/.github/pull_request_template.md
- PR description must include a checklist distinguishing: (a) verified locally — dotnet
  restore, dotnet build, dotnet test; (b) requires CI or live-environment validation —
  Docker builds, Lambda deployment to dev.