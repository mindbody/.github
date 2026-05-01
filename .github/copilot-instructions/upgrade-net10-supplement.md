# .NET 10 Upgrade — Supplemental Instructions

## Assessment Compatibility Ratings

Assessment tools report whether a package works with the target framework — not whether it is
up to date. These ratings do not replace explicit update directives. Update all packages as
instructed regardless of compatibility assessment results.

## Scope Discipline — Pre-Push Verification

Before pushing, run `git show --stat HEAD`. If tool-generated files are present (e.g.,
tasks.md, .claude/, scenario files), run `git reset --soft HEAD~N` and recommit with only
production files staged.

## Package Companion Compatibility

Verify companion packages are on a mutually compatible version matrix. If staying on xunit v2,
xunit and xunit.runner.visualstudio must be compatible versions — check the xunit release notes
to confirm the correct pairing. For xunit v3 migration, see the dedicated section below.

## Deprecated Package Verification

Before adding a deprecation comment, confirm status on nuget.org. Some packages flagged as
deprecated by assessment tools are simply old versions of actively maintained packages
(e.g., Microsoft.Identity.Client 4.33.0 is not deprecated — update to latest). If
`dotnet list package --deprecated` returns 401 Unauthorized (private NuGet feeds), verify credentials in nuget.config.

## NuGet Audit Source Configuration

If `dotnet restore` produces NU1900 warnings ("Error occurred while getting package vulnerability data: Unable to load the service index for source"), a 401 or authentication error that persists despite valid credentials, or if the solution does not already define package sources/package source mapping consistently, create or update the solution-root nuget.config to standardize package sources and audit behavior. Azure DevOps feeds do not support the NuGet vulnerability audit API, so audit sources must be restricted to nuget.org.

Standardize on these conventions:

- Internal feed name: Mindbody-Nuget
- Internal feed URL: <https://pkgs.dev.azure.com/mindbody/_packaging/Mindbody-Nuget/nuget/v3/index.json>
- Public feed: <https://api.nuget.org/v3/index.json>

If legacy internal feed names or URLs are found (for example mindbody.pkgs.visualstudio.com or other older aliases), replace them with the standard Mindbody-Nuget / pkgs.dev.azure.com form.

Important:

- If no repo-specific NuGet source configuration exists, you may create a standardized nuget.config using the example below.
- If a nuget.config already exists and contains other required repo-specific settings, merge carefully instead of replacing unrelated configuration.
- Only use <clear /> in packageSources when intentionally standardizing sources for the repo and no additional required feeds need to be preserved.

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear /> <!-- Intentionally standardizes sources for this repo  -->
    <add key="Mindbody-Nuget" value="https://pkgs.dev.azure.com/mindbody/_packaging/Mindbody-Nuget/nuget/v3/index.json" protocolVersion="3" />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" protocolVersion="3" />
  </packageSources>
  <packageSourceMapping>
    <!-- Prefer internal package prefixes on the internal feed -->
    <packageSource key="Mindbody-Nuget">
      <package pattern="DotNet.IdentityLegacyGateway.ApiClient" />
      <package pattern="DotNet.IdentityUserGateway.ApiClient" />
      <package pattern="IAM.*" />
      <package pattern="Identity.*" />
      <package pattern="MbCore.*" />
      <package pattern="Mindbody.*" />
      <package pattern="Permissions.*" />
      <package pattern="Kralizek.*" />
    </packageSource>
    <!-- Allow all other packages from nuget.org -->
    <packageSource key="nuget.org">
      <package pattern="*" />
    </packageSource>
  </packageSourceMapping>
  <config>
    <add key="audit" value="true" />
  </config>
  <auditSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
  </auditSources>
</configuration>
```

This preserves vulnerability scanning for public packages via nuget.org while
suppressing unsupported audit queries against the internal Azure DevOps feed. Commit this
file as part of the upgrade — it is a permanent configuration fix, not scaffolding.

## Microsoft.IdentityModel Transitive Version Alignment

After updating System.IdentityModel.Tokens.Jwt, run:
`dotnet list package --include-transitive | Select-String "Microsoft.IdentityModel|System.IdentityModel"`
Verify all Microsoft.IdentityModel.* packages are at the same version. If misaligned, add explicit PackageReference pins to the affected project. See:
<https://docs.duendesoftware.com/identityserver/troubleshooting/#microsoftidentitymodel-versions>

## xUnit v3 Migration

If upgrading to xunit.v3 (v3.x), replace the three v2-era packages with a single package:

- Remove `xunit`, `xunit.runner.visualstudio`, and `Microsoft.NET.Test.Sdk`.
- Add `xunit.v3` (latest stable, currently 3.2.2). The runner and test SDK are built in.
- Add `<OutputType>Exe</OutputType>` to each test project's `<PropertyGroup>`. xunit v3 test projects must be executable.
- Check for transitive dependency breakage. Packages that were pulled in transitively by xunit v2 (e.g., Newtonsoft.Json) will no longer be available. If build errors reference missing types, replace usage with built-in alternatives (e.g., `System.Text.Json`) or add an explicit PackageReference.
- After migration, run `dotnet test --configuration Release`. If zero tests are discovered,
  `<OutputType>Exe</OutputType>` is missing from one or more test projects.
- If any test project cannot migrate to v3 due to a dependency that requires xunit v2 abstractions, keep that project on the latest xunit v2 and add an inline comment in the project file explaining why.

## .NET 10 Breaking Changes

- X509Certificate2 constructors obsolete (SYSLIB0057). Replace with X509CertificateLoader:
  - new X509Certificate2(byte[]) → X509CertificateLoader.LoadCertificate(byte[])
  - new X509Certificate2(path, pwd) → X509CertificateLoader.LoadPkcs12FromFile(path, pwd)
- AuthenticationHandler subclasses: remove deprecated ISystemClock parameter; use the
  3-argument constructor.
- TimeSpan.FromSeconds(int): cast to float to resolve overload ambiguity and preserve underlying behavior (e.g., TimeSpan.FromSeconds(30.0)).
- Microsoft.IdentityModel v8+: stricter iat/exp JWT validation. If test code constructs
  JWTs manually (e.g., SecurityTokenDescriptor), ensure IssuedAt and Expires are explicitly
  set. See <https://docs.duendesoftware.com/identityserver/troubleshooting/#microsoftidentitymodel-versions>
- WebHostBuilder (ASPDEPR004/ASPDEPR008): suppress with #pragma on affected lines; create
  follow-up work item to migrate to WebApplicationBuilder.
