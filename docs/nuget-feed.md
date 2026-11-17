# NuGet feed on GitLab

The `Spkl.OnPrem*` packages live in the NuGet package registry of the private GitLab project that publishes them (see [releasing](releasing.md)). This page covers the consuming side: developer machines, Visual Studio and the GitLab CI pipelines of your Dynamics 365 solution repositories.

The setup in short:

- Every solution repository has a `nuget.config` with the feed URL and **no credentials**. The `spkl-solution` template creates it.
- Every developer stores a GitLab token once in their **user-level** NuGet config, under the same source name `spkl`.
- CI pipelines pass the job token through an environment variable.

## Feed URL

Use the v3 URL of the GitLab project that publishes spkl (find the project id on the project's overview page, or in **Settings → General**):

```
https://<gitlab>/api/v4/projects/<project id>/packages/nuget/index.json
```

A group URL works too, and returns the NuGet packages of every project in the group and its subgroups. That is useful if more internal packages follow, e.g. the FakeXrmEasy fork:

```
https://<gitlab>/api/v4/groups/<group id>/-/packages/nuget/index.json
```

The group URL is read-only: GitLab only accepts `dotnet nuget push` on the project URL. `.gitlab-ci.yml` in this repository already pushes to the project URL.

## Solution repositories: `nuget.config`

The template asks for the URL:

```powershell
dotnet new spkl-solution -n Contoso --SolutionUniqueName ContosoCore `
  --FeedUrl "https://<gitlab>/api/v4/projects/<project id>/packages/nuget/index.json"
```

and writes it to `nuget.config` in the solution folder:

```xml
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" protocolVersion="3" />
    <add key="spkl" value="https://<gitlab>/api/v4/projects/<project id>/packages/nuget/index.json" />
  </packageSources>
  <packageSourceMapping>
    <packageSource key="nuget.org">
      <package pattern="*" />
    </packageSource>
    <packageSource key="spkl">
      <package pattern="Spkl.OnPrem*" />
    </packageSource>
  </packageSourceMapping>
</configuration>
```

For an existing solution, copy [`templates/solution/nuget.config`](../templates/solution/nuget.config) next to the `.sln` and put in the URL.

- `<clear />` makes restore independent of the sources configured on each machine.
- [Package source mapping](https://learn.microsoft.com/nuget/consume-packages/package-source-mapping) restores `Spkl.OnPrem*` only from GitLab and everything else only from nuget.org. Nobody can shadow the internal ids with a package of the same name on nuget.org. Needs NuGet 6 or later (Visual Studio 2022, .NET 6 SDK or later). If you add more internal packages, add their ids or prefixes to the `spkl` entry, or restore fails with `NU1100`.
- Keep the source name `spkl`. Developer and CI credentials are looked up by that name.
- **Never add `<packageSourceCredentials>` to this file.** It is committed.

## Developer machines

Once per developer and machine.

### 1. Create a token

In GitLab: avatar → **Edit profile → Access tokens → Add new token**, scope `read_api`, with an expiry date. If your GitLab enforces two-factor authentication, GitLab's NuGet docs ask for scope `api` instead.

You need at least the **Reporter** role in the spkl project (or its group) to download packages from a private project.

### 2. Store it in your user-level NuGet config

```powershell
dotnet nuget add source "https://<gitlab>/api/v4/projects/<project id>/packages/nuget/index.json" `
  --name spkl --username <gitlab user> --password <token> `
  --configfile "$env:APPDATA\NuGet\NuGet.Config"
```

- `--configfile` makes sure the token lands in `%APPDATA%\NuGet\NuGet.Config` and not in a `nuget.config` of the folder you're in.
- Without `--store-password-in-clear-text` the token is stored encrypted for your Windows user. Don't add that flag.
- Use exactly the name `spkl`. NuGet matches the credentials to the source in the solution's `nuget.config` by name, so the solution then restores without any extra step.

This also makes the source available outside solution folders, which `dotnet new install Spkl.OnPrem.Templates` needs.

### 3. Check it

```powershell
dotnet nuget list source
dotnet new install Spkl.OnPrem.Templates
```

Visual Studio reads the same config files, so restore works there too (restart Visual Studio after the change). If Visual Studio asks for credentials anyway, the stored token is wrong or expired.

### Token expired

```powershell
dotnet nuget update source spkl --username <gitlab user> --password <new token> `
  --configfile "$env:APPDATA\NuGet\NuGet.Config"
```

## CI of solution repositories

The pipeline of a solution repository restores with the job token. No token has to be stored.

### 1. Allow the solution project in the spkl project

GitLab only accepts another project's job token if that project is on the allowlist of the spkl project:

1. In the **spkl** GitLab project: **Settings → CI/CD → Job token permissions**.
2. Under **CI/CD job token allowlist** select **Add** and add the solution project, or the group that contains all solution repositories (GitLab 17.0 or later), so new repositories work without another change.

The job token has the permissions of the user who started the pipeline. That user needs at least the **Reporter** role in the spkl project. Developers who can restore locally already have it.

### 2. Pass the token to NuGet

NuGet reads credentials for the source `spkl` from the environment variable `NuGetPackageSourceCredentials_spkl`. A minimal `.gitlab-ci.yml` for a solution repository on a Windows shell runner (PowerShell, .NET 8 SDK, .NET Framework 4.8 Developer Pack):

```yaml
default:
  tags:
    - windows

build:
  script:
    - $env:NuGetPackageSourceCredentials_spkl = "Username=gitlab-ci-token;Password=$env:CI_JOB_TOKEN"
    - dotnet build Contoso.sln; if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
  artifacts:
    paths:
      - Plugins/bin/Debug/*.dll
      - Actions/bin/Debug/*.dll
      - Workflows/bin/Debug/*.dll
    expire_in: 1 week
```

- The variable name is case-sensitive on some platforms and must match the source name in `nuget.config` exactly.
- NuGet ignores the variable silently if the name or the `Username=…;Password=…` format is wrong; restore then fails with `401 Unauthorized`.
- `CI_JOB_TOKEN` is only valid while the job runs, so there is nothing to rotate or leak afterwards.
- The job builds `Debug`, because `Tooling\spkl.json` points to `bin\Debug`.

### Alternative: deploy token

If the allowlist can't be used, or pipelines are started by users without access to the spkl project (e.g. scheduled pipelines owned by a bot user):

1. In the spkl project or its group: **Settings → Repository → Deploy tokens → Add token**, scope `read_package_registry`.
2. In the solution project or group: **Settings → CI/CD → Variables**, add `SPKL_DEPLOY_USER` and `SPKL_DEPLOY_TOKEN` (masked, protected if only protected branches build).
3. In the pipeline:

   ```yaml
   - $env:NuGetPackageSourceCredentials_spkl = "Username=$env:SPKL_DEPLOY_USER;Password=$env:SPKL_DEPLOY_TOKEN"
   ```

The deploy token works for every pipeline regardless of who starts it, so treat it as a secret and rotate it when people leave.

## Troubleshooting

| Error | Cause |
|---|---|
| `401 Unauthorized` | No or wrong credentials for the source `spkl`: token expired, wrong scope, source stored under another name, or (CI) project not on the allowlist. |
| `403 Forbidden` or `404 Not Found` on `index.json` | Usually a wrong project or group id, a disabled package registry in the spkl project, or a user with less than Reporter. |
| `NU1100: Unable to resolve 'Spkl.OnPrem …'` | The version is not published yet, or package source mapping sends the id to the wrong source. |
| `NU1301` | The GitLab server is not reachable from this machine or runner (proxy, VPN, certificate). |

## References

- GitLab: [NuGet packages in the package registry](https://docs.gitlab.com/user/packages/nuget_repository/), [supported token types](https://docs.gitlab.com/user/packages/package_registry/supported_functionality/), [CI/CD job token](https://docs.gitlab.com/ci/jobs/ci_job_token/), [deploy tokens](https://docs.gitlab.com/user/project/deploy_tokens/)
- NuGet: [consuming packages from authenticated feeds](https://learn.microsoft.com/nuget/consume-packages/consuming-packages-authenticated-feeds), [package source mapping](https://learn.microsoft.com/nuget/consume-packages/package-source-mapping)
