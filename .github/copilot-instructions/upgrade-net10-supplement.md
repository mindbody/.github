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

Verify companion packages are on a mutually compatible version matrix. Example: xunit and
xunit.runner.visualstudio must be compatible versions — check the xunit release notes to
confirm the correct pairing.

## Deprecated Package Verification

Before adding a deprecation comment, confirm status on nuget.org. Some packages flagged as
deprecated by assessment tools are simply old versions of actively maintained packages
(e.g., Microsoft.Identity.Client 4.33.0 is not deprecated — update to latest). If
`dotnet list package --deprecated` returns 401 Unauthorized (private NuGet feeds), verify credentials in nuget.config.

## NuGet Audit Source Configuration

If `dotnet restore` or `dotnet list package --deprecated` fails with a 401 or
authentication error that persists despite valid credentials, the cause may be NuGet
attempting to query the internal Azure DevOps feed for vulnerability audit data — a
capability it does not support. To resolve, create a nuget.config in the solution root
(or update it if one exists) to restrict audit scanning to nuget.org only:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
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
suppressing the unsupported audit query against the internal feed. Commit this
file as part of the upgrade — it is a permanent configuration fix, not scaffolding.

## Microsoft.IdentityModel Transitive Version Alignment

After updating System.IdentityModel.Tokens.Jwt, run:
`dotnet list package --include-transitive | Select-String "Microsoft.IdentityModel|System.IdentityModel"`
Verify all Microsoft.IdentityModel.* packages are at the same version. If misaligned, add
explicit PackageReference pins to the affected project. See:
<https://docs.duendesoftware.com/identityserver/troubleshooting/#microsoftidentitymodel-versions>

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
