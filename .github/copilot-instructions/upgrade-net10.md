Perform a .NET major version upgrade (e.g., net8.0 to net10.0). Follow these rules precisely.

If you encounter deprecated package verification issues, NuGet 401 errors, Microsoft.IdentityModel misalignment, .NET 10 breaking changes, or tool-generated files in commits, fetch <https://raw.githubusercontent.com/mindbody/.github/main/.github/copilot-instructions/upgrade-net10-supplement.md> and apply it before continuing.

## Scope Discipline

Stage only pre-existing files; the only exception is global.json if absent. Never stage or commit plans, assessments, logs, or scaffolding.

## Project Files

- Multi-targeting libraries: replace old netX.0 with new; keep netstandard2.x targets.
- Single-target netstandard libraries: no TargetFramework change needed.
- Test/app projects: update <TargetFramework> to new version.
- global.json: update SDK version and preserve rollForward before any dotnet CLI calls (prevents NETSDK1045); create if absent with rollForward: latestMinor.

## Package Version Management

Never use `dotnet add package` — edit files directly. Check for Directory.Packages.props in the solution root.

**If Directory.Packages.props EXISTS (CPM):**

- Update versions in Directory.Packages.props only, as <PackageVersion> entries.
- .csproj files must NOT have Version= on <PackageReference> elements.
- Use VersionOverride= in a .csproj only for deliberate per-project exceptions.

**If Directory.Packages.props does NOT exist:**

- Update Version= in each .csproj. Explicit versions only — no wildcards.

## Packages

- Microsoft.Extensions.*: update ALL to the SAME .NET version (e.g., 10.0.x).
- System.Text.Json: match new .NET version.
- Microsoft.NET.Test.Sdk: update to latest stable; do not downgrade.
- Mindbody.*: update to latest stable.
- Run `dotnet list package --outdated` and `--deprecated`. Update every deprecated package to latest stable; if no replacement, add inline comment with migration path.
- Update ALL packages in EVERY project including test projects; none may remain at old version.
- Third-party: update to latest stable. NU1605 errors: update the package in ALL projects.
- ⚠️ FluentAssertions: NEVER upgrade to 8.0+. If below 7.2.2, upgrade to 7.2.2. Add XML comment explaining the version cap.
- Replace Kralizek.Extensions.Configuration.AWSSecretsManager with the latest AWSSecretsManager.Provider

## Dockerfiles

- Update FROM dotnet/sdk:{old} and dotnet/aspnet:{old} to new version.
- Sync ARG version values (e.g., NewRelic) to match updated NuGet references.
- Update pinned `dotnet tool install --version` values to latest.

## aws-lambda-tools-default.json

For each found: set "framework" to "net10.0" and "function-runtime" to "dotnet10".

## launchSettings.json

For each found:

- "executablePath": replace dotnet-lambda-test-tool-{old}.exe with dotnet-lambda-test-tool-10.0.exe. Preserve remaining path.
- "workingDirectory": replace net{old} with net10.0. Preserve $(Configuration) exactly.

## infrastructure.yaml

If infrastructure.yaml has a "lambdaFunctions" section, for each function:

- Set "runtime" to "dotnet10".
- Increase "memorySize" by 50%, capping at 3008.
- If "layers" contains "arn:aws:lambda:us-west-2:451483290750:layer:NewRelicDotnet:*", update trailing version to "49".

## Validation

1. `dotnet restore` — no errors
2. `dotnet build --configuration Release` — 0 errors
3. `dotnet test --configuration Release` — 100% pass rate
4. `dotnet list package --deprecated` — none actionable

## Documentation

Update README.md: SDK version, IDE version, target frameworks, key dependencies, build/Docker instructions.

## Commit

Commit message: framework changes, package updates, deprecated packages resolved.

## Pull Request

- Title must start with "Upgrade to .NET 10".
- Append " +semver:breaking" if: any .csproj has GeneratePackageOnBuild=true or explicit IsPackable=true, a Dockerfile contains 'dotnet pack', or any ci/ file contains 'ci/public/nugetJobs.yml'.
- Use .github/pull_request_template.md if present, otherwise <https://raw.githubusercontent.com/mindbody/.github/main/.github/pull_request_template.md>
- Include checklist: locally verified (restore/build/test) vs needs CI/live (Docker, Lambda dev deploy).
