# NuGet feeds

The `Spkl.OnPrem*` packages are published with the same versions to two feeds (see [releasing](releasing.md)):

- the NuGet package registry of the private **GitLab** project,
- **GitHub Packages** of the GitHub repository owner.

This page covers the consuming side: developer machines, Visual Studio and the CI pipelines of your Dynamics 365 solution repositories, on GitLab or on GitHub.

The setup in short:

- Every solution repository has a `nuget.config` with the feed URL and **no credentials**. The `spkl-solution` template creates it.
- Every developer stores a token once in their **user-level** NuGet config, under the same source name `spkl`.
- CI pipelines pass their job token through an environment variable.

## Which feed

Restore a solution from the feed on the platform where its CI runs: GitLab feed for solution repositories on GitLab, GitHub Packages for solution repositories on GitHub. The pipeline's own job token then works without a stored secret.

Both feeds hold identical versions, so a solution can switch by changing the URL in its `nuget.config`. Keep one feed per solution: listing both for `Spkl.OnPrem*` makes restore depend on both being reachable and authorized.

A developer who works on solutions with different feeds needs a token for each, see [Both feeds on one machine](#both-feeds-on-one-machine).

## Feed URLs

### GitLab

Use the v3 URL of the GitLab project that publishes spkl (find the project id on the project's overview page, or in **Settings → General**):

```
https://<gitlab>/api/v4/projects/<project id>/packages/nuget/index.json
```

A group URL works too, and returns the NuGet packages of every project in the group and its subgroups. That is useful if more internal packages follow, e.g. the FakeXrmEasy fork:

```
https://<gitlab>/api/v4/groups/<group id>/-/packages/nuget/index.json
```

The group URL is read-only: GitLab only accepts `dotnet nuget push` on the project URL. `.gitlab-ci.yml` in this repository already pushes to the project URL.

### GitHub Packages

```
https://nuget.pkg.github.com/lbaumdeveloper/index.json
```

The URL contains the owner of the GitHub repository, not the repository. GitHub Packages always needs a token, also for public packages.

## Solution repositories: `nuget.config`

The template asks for the URL:

```powershell
dotnet new spkl-solution -n Contoso --SolutionUniqueName ContosoCore `
  --FeedUrl "https://<gitlab>/api/v4/projects/<project id>/packages/nuget/index.json"

dotnet new spkl-solution -n Contoso --SolutionUniqueName ContosoCore `
  --FeedUrl "https://nuget.pkg.github.com/lbaumdeveloper/index.json"
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
- [Package source mapping](https://learn.microsoft.com/nuget/consume-packages/package-source-mapping) restores `Spkl.OnPrem*` only from the spkl feed and everything else only from nuget.org. Nobody can shadow the internal ids with a package of the same name on nuget.org. Needs NuGet 6 or later (Visual Studio 2022, .NET 6 SDK or later). If you add more internal packages, add their ids or prefixes to the `spkl` entry, or restore fails with `NU1100`.
- Keep the source name `spkl`. Developer and CI credentials are looked up by that name.
- **Never add `<packageSourceCredentials>` to this file.** It is committed.

## Developer machines

Once per developer and machine, for the feed your solutions use.

### GitLab

#### 1. Create a token

In GitLab: avatar → **Edit profile → Access tokens → Add new token**, scope `read_api`, with an expiry date. If your GitLab enforces two-factor authentication, GitLab's NuGet docs ask for scope `api` instead.

You need at least the **Reporter** role in the spkl project (or its group) to download packages from a private project.

#### 2. Store it in your user-level NuGet config

```powershell
dotnet nuget add source "https://<gitlab>/api/v4/projects/<project id>/packages/nuget/index.json" `
  --name spkl --username <gitlab user> --password <token> `
  --configfile "$env:APPDATA\NuGet\NuGet.Config"
```

- `--configfile` makes sure the token lands in `%APPDATA%\NuGet\NuGet.Config` and not in a `nuget.config` of the folder you're in.
- Without `--store-password-in-clear-text` the token is stored encrypted for your Windows user. Don't add that flag.
- Use exactly the name `spkl`. NuGet matches the credentials to the source in the solution's `nuget.config` by name, so the solution then restores without any extra step.

This also makes the source available outside solution folders, which `dotnet new install Spkl.OnPrem.Templates` needs.

#### 3. Check it

```powershell
dotnet nuget list source
dotnet new install Spkl.OnPrem.Templates
```

Visual Studio reads the same config files, so restore works there too (restart Visual Studio after the change). If Visual Studio asks for credentials anyway, the stored token is wrong or expired.

#### Token expired

```powershell
dotnet nuget update source spkl --username <gitlab user> --password <new token> `
  --configfile "$env:APPDATA\NuGet\NuGet.Config"
```

### GitHub Packages

#### 1. Create a token

GitHub Packages only accepts a **personal access token (classic)**. On GitHub: avatar → **Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token (classic)**, scope `read:packages`, with an expiry date.

As long as the packages are private, you also need read access to them: either access to the linked repository, or an entry under **Package settings → Manage access**.

#### 2. Store it in your user-level NuGet config

```powershell
dotnet nuget add source "https://nuget.pkg.github.com/lbaumdeveloper/index.json" `
  --name spkl --username <github user> --password <token> `
  --configfile "$env:APPDATA\NuGet\NuGet.Config"
```

The same rules as for GitLab apply: no `--store-password-in-clear-text`, and exactly the name `spkl`. Check it and renew it as for GitLab, with your GitHub user and token.

### Both feeds on one machine

NuGet looks credentials up by source name, and the user-level config holds one `spkl`. A developer with GitLab solutions and GitHub solutions therefore uses a second name for one of the feeds, e.g. `spkl_github`:

1. In the `nuget.config` of each solution on that feed, rename the source `spkl` to `spkl_github`, in `<packageSources>` and in `<packageSourceMapping>`.
2. Store the token under that name:

   ```powershell
   dotnet nuget add source "https://nuget.pkg.github.com/lbaumdeveloper/index.json" `
     --name spkl_github --username <github user> --password <token> `
     --configfile "$env:APPDATA\NuGet\NuGet.Config"
   ```

3. In the CI of those solutions, use `NuGetPackageSourceCredentials_spkl_github` instead of `NuGetPackageSourceCredentials_spkl`.

Decide the name per feed for the whole team, so every solution on the same feed uses the same name.

## CI of solution repositories

The pipeline of a solution repository restores with the job token of its platform. No token has to be stored.

NuGet reads credentials for the source `spkl` from the environment variable `NuGetPackageSourceCredentials_spkl`, in the format `Username=…;Password=…`.

### GitLab CI

#### 1. Allow the solution project in the spkl project

GitLab only accepts another project's job token if that project is on the allowlist of the spkl project:

1. In the **spkl** GitLab project: **Settings → CI/CD → Job token permissions**.
2. Under **CI/CD job token allowlist** select **Add** and add the solution project, or the group that contains all solution repositories (GitLab 17.0 or later), so new repositories work without another change.

The job token has the permissions of the user who started the pipeline. That user needs at least the **Reporter** role in the spkl project. Developers who can restore locally already have it.

#### 2. Pass the token to NuGet

A minimal `.gitlab-ci.yml` for a solution repository on a Windows shell runner (PowerShell, .NET 8 SDK, .NET Framework 4.8 Developer Pack):

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

#### Alternative: deploy token

If the allowlist can't be used, or pipelines are started by users without access to the spkl project (e.g. scheduled pipelines owned by a bot user):

1. In the spkl project or its group: **Settings → Repository → Deploy tokens → Add token**, scope `read_package_registry`.
2. In the solution project or group: **Settings → CI/CD → Variables**, add `SPKL_DEPLOY_USER` and `SPKL_DEPLOY_TOKEN` (masked, protected if only protected branches build).
3. In the pipeline:

   ```yaml
   - $env:NuGetPackageSourceCredentials_spkl = "Username=$env:SPKL_DEPLOY_USER;Password=$env:SPKL_DEPLOY_TOKEN"
   ```

The deploy token works for every pipeline regardless of who starts it, so treat it as a secret and rotate it when people leave.

### GitHub Actions

#### 1. Give the solution repository access to the packages

The job's `GITHUB_TOKEN` can read a package only if the solution repository has access to it. For each of `Spkl.OnPrem` and `Spkl.OnPrem.Attributes` (package page on GitHub): **Package settings → Manage Actions access → Add repository**, role **Read**. Repeat for every new solution repository.

#### 2. Pass the token to NuGet

A minimal `.github/workflows/build.yml` for a solution repository on a GitHub-hosted Windows runner:

```yaml
name: build

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read
  packages: read

jobs:
  build:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 8.0.x
      - name: Build
        env:
          NuGetPackageSourceCredentials_spkl: Username=${{ github.actor }};Password=${{ secrets.GITHUB_TOKEN }}
        run: dotnet build Contoso.sln
      - uses: actions/upload-artifact@v4
        with:
          name: assemblies
          path: |
            Plugins/bin/Debug/*.dll
            Actions/bin/Debug/*.dll
            Workflows/bin/Debug/*.dll
          retention-days: 7
```

- `packages: read` is required; without it the token can't read GitHub Packages.
- `windows-latest` has the .NET Framework 4.8 targeting pack. On a self-hosted runner, install the .NET 8 SDK and the .NET Framework 4.8 Developer Pack.

#### Alternative: personal access token

If the repository can't be given access in the package settings, store a personal access token (classic) with `read:packages` of a user who can read the packages as the repository secret `SPKL_PACKAGES_TOKEN`, and use:

```yaml
NuGetPackageSourceCredentials_spkl: Username=<github user>;Password=${{ secrets.SPKL_PACKAGES_TOKEN }}
```

### Restoring from the other platform's feed

Possible, but needs a stored token and network access:

- GitLab CI from GitHub Packages: a personal access token (classic) with `read:packages` as a masked CI/CD variable, passed in `NuGetPackageSourceCredentials_spkl`.
- GitHub Actions from the GitLab feed: a GitLab deploy token as repository secrets, and a runner that can reach the GitLab server. A private GitLab is usually not reachable from GitHub-hosted runners, so this needs a self-hosted runner in your network.

## Troubleshooting

| Error | Cause |
|---|---|
| `401 Unauthorized` | No or wrong credentials for the source `spkl`: token expired, wrong scope, source stored under another name, a GitLab token sent to GitHub or the other way round, or (CI) project not on the allowlist. |
| `403 Forbidden` or `404 Not Found` on `index.json` | GitLab: usually a wrong project or group id, a disabled package registry in the spkl project, or a user with less than Reporter. GitHub: a fine-grained token instead of a classic one, missing `read:packages`, or (Actions) the repository has no access to the package or the workflow lacks `packages: read`. |
| `NU1100: Unable to resolve 'Spkl.OnPrem …'` | The version is not published yet, or package source mapping sends the id to the wrong source. |
| `NU1301` | The feed is not reachable from this machine or runner (proxy, VPN, certificate). |

## References

- GitLab: [NuGet packages in the package registry](https://docs.gitlab.com/user/packages/nuget_repository/), [supported token types](https://docs.gitlab.com/user/packages/package_registry/supported_functionality/), [CI/CD job token](https://docs.gitlab.com/ci/jobs/ci_job_token/), [deploy tokens](https://docs.gitlab.com/user/project/deploy_tokens/)
- GitHub: [working with the NuGet registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-nuget-registry), [access control and visibility](https://docs.github.com/en/packages/learn-github-packages/configuring-a-packages-access-control-and-visibility), [`GITHUB_TOKEN` permissions](https://docs.github.com/en/actions/tutorials/authenticate-with-github_token)
- NuGet: [consuming packages from authenticated feeds](https://learn.microsoft.com/nuget/consume-packages/consuming-packages-authenticated-feeds), [package source mapping](https://learn.microsoft.com/nuget/consume-packages/package-source-mapping)
