# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

NWebsec is a family of security libraries for ASP.NET that set security HTTP headers
(CSP, HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, X-XSS-Protection,
X-Robots-Tag), validate redirects, and control cache headers. The same security features
are shipped for two platforms — classic **ASP.NET 4 (net45)** and **ASP.NET Core
(netcoreapp3.1)** — built from a shared source base. User docs live at https://docs.nwebsec.com/.

## Build, test, pack

Everything goes through the `dotnet` CLI against one of two solutions:

- `NWebsec-CI.sln` — used by CI. The library + test projects only. **Use this by default.**
- `NWebsec.sln` — adds the two sample web apps `src/Mvc` (ASP.NET Core) and
  `src/MvcClassic` (ASP.NET 4). `MvcClassic` is an old-style (packages.config / Global.asax)
  project that needs MSBuild + Windows classic ASP.NET tooling and will **not** build under
  the plain `dotnet` SDK — this is why it's excluded from the CI solution.

```powershell
dotnet build NWebsec-CI.sln -c Release
dotnet test  NWebsec-CI.sln -c Release --no-build
dotnet pack  NWebsec-CI.sln -c Release --no-build -o nupkgs
```

Run a single test project / a single test:

```powershell
dotnet test test/NWebsec.AspNetCore.Core.Tests
dotnet test test/NWebsec.AspNetCore.Core.Tests --filter "FullyQualifiedName~CspDirectiveOverride"
```

Notes:
- The net45 projects require building on **Windows** (.NET Framework reference assemblies).
- Release builds set `TreatWarningsAsErrors=true` and generate XML doc files, so warnings
  fail the build in Release.
- All assemblies are strong-named: product assemblies with `src/nwebsec.snk`, test
  assemblies with `test/nwebsectst.snk` (configured in `src/common.props` and per-test csproj).
- Tests use **xUnit** + **Moq**; functional tests use `Microsoft.AspNetCore.Mvc.Testing`.

## Architecture: shared source, dual-targeted

The defining pattern here is **Visual Studio shared projects** (`.shproj` + `.projitems`).
Source files live in a shared project *once* and are `<Import>`-ed into multiple real
projects that each target a different framework. **Editing a file under a shared project
changes both the net45 and the netcore assemblies that import it.**

Two shared projects hold almost all the real logic:

- `src/NWebsec.Core.Shared` — core header generation, configuration, CSP, middleware options.
- `src/NWebsec.Mvc.Common` — MVC-layer helpers, CSP override config, attribute support.

These are imported into the per-framework, packable projects. The folder names do **not**
match the NuGet package / assembly names — this mapping matters:

| Source folder | Target | NuGet package = AssemblyName | Imports shared |
|---|---|---|---|
| `src/NWebsec.AspNet.Core` | net45 | `NWebsec.Core` | Core.Shared |
| `src/NWebsec.AspNet.Classic` | net45 | `NWebsec` (HTTP modules) | — |
| `src/NWebsec.AspNet.Mvc` | net45 | `NWebsec.Mvc` | Mvc.Common |
| `src/NWebsec.AspNet.Owin` | net45 | `NWebsec.Owin` | — |
| `src/NWebsec.AspNetCore.Core` | netcoreapp3.1 | `NWebsec.AspNetCore.Core` | Core.Shared |
| `src/NWebsec.AspNetCore.Middleware` | netcoreapp3.1 | `NWebsec.AspNetCore.Middleware` | (refs Core) |
| `src/NWebsec.AspNetCore.Mvc` | netcoreapp3.1 | `NWebsec.AspNetCore.Mvc` | Mvc.Common |
| `src/NWebsec.AspNetCore.Mvc.TagHelpers` | netcoreapp3.1 | `NWebsec.AspNetCore.Mvc.TagHelpers` | (refs Core) |

So `NWebsec.AspNet.*` folders = the **ASP.NET 4 / net45** packages; `NWebsec.AspNetCore.*`
folders = the **ASP.NET Core** packages.

### The `*SharedProject` / `*CommonProject` wrappers

`src/NWebsec.Core.SharedProject` and `src/NWebsec.Mvc.CommonProject` are multi-targeted
(`net45;netcoreapp3.1`), `IsPackable=false` wrappers that also import the same shared
`.projitems`. They exist **only so the shared source can be unit-tested across both
frameworks** — the `test/*SharedProject.Tests` and `test/*CommonProject.Tests` projects
reference these, not the shipping packages. When you change shared code, these are the
projects whose tests cover it.

### Delivery mechanisms per platform

The shared core logic is exposed through framework-specific entry points: HTTP **modules**
(`NWebsec.AspNet.Classic`), **OWIN** middleware (`NWebsec.AspNet.Owin`), ASP.NET Core
**middleware** (`NWebsec.AspNetCore.Middleware`), MVC **filter attributes**
(`*.Mvc`, e.g. `XFrameOptionsAttribute`, `CspAttribute`), and **TagHelpers** for CSP
nonces (`*.Mvc.TagHelpers`).

## Versioning & release

- Common packaging props (authors, license `BSD-3-Clause`, symbol packages, signing) are
  centralized in `src/common.props`; each shipping csproj sets its own `VersionPrefix` /
  `AssemblyVersion` and `PackageReleaseNotes`.
- CI (`azure-pipelines.yml`, `appveyor.yml`) appends a `--version-suffix` from the build
  number for non-`master` branches and pushes to nuget.org only from `master`/`beta`/`rc`.
- `NuGet.Config` pins the single restore source to api.nuget.org (clears inherited sources).
