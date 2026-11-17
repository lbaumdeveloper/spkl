# Releasing

spkl ships as three NuGet packages with the same version:

| Package | Used by |
|---|---|
| `Spkl.OnPrem` | `Tooling` project: `spkl.exe` |
| `Spkl.OnPrem.Attributes` | `Plugins`, `Actions`, `Workflows`: `CrmPluginRegistrationAttribute.cs` |
| `Spkl.OnPrem.Templates` | `dotnet new spkl-solution` |

Both platforms publish the same packages from the same tag:

| Platform | Pipeline | Feed | Runner |
|---|---|---|---|
| GitHub | `.github/workflows/build.yml` | GitHub Packages of the repository owner, plus the `.nupkg` files on the GitHub release | `windows-latest`, or the runner in the repository variable `SPKL_RUNS_ON` |
| GitLab | `.gitlab-ci.yml` | NuGet package registry of the GitLab project | Windows shell runner with the tag in `SPKL_RUNNER_TAG` |

Both pipelines build and test on every push and publish only on a tag `v<Version>`. Neither needs a stored secret: GitHub pushes with `GITHUB_TOKEN`, GitLab with `CI_JOB_TOKEN`.

GitHub is the main repository: pull requests and reviews happen there. GitLab gets a copy of `main` and the tags. Which feed a solution restores from is described in [NuGet feeds](nuget-feed.md).

## First-time setup

### 1. One clone that pushes to both

Give `origin` a second push URL. `git fetch` and `git pull` keep using GitHub, `git push origin …` goes to GitHub and GitLab:

```powershell
git remote set-url --add --push origin https://github.com/lbaumdeveloper/spkl.git
git remote set-url --add --push origin https://<gitlab>/tooling/spkl.git
git remote -v
```

`git remote -v` should list one fetch URL (GitHub) and two push URLs. The first `--add --push` is needed, because once a push URL is set, the fetch URL is no longer used for pushing.

Pull requests are merged on GitHub, so GitLab only gets the merge commit when someone pushes `main` again:

```powershell
git checkout main
git pull
git push origin main
```

If a push to one of them fails, git still pushes to the other. Fix the cause and push again; pushing what is already there is a no-op.

Alternatively let GitLab pull from GitHub with repository mirroring, if your GitLab edition offers pull mirroring. Enable **Trigger pipelines for mirror updates**, otherwise tags arriving through the mirror don't publish.

### 2. GitHub

Nothing to configure for `windows-latest`. The publish job asks for `packages: write` and `contents: write` itself.

- After the first tag, the three packages appear under **Packages** on the owner's profile and on the repository page. They are linked to the repository through `RepositoryUrl` in `Directory.Build.props`, which must point to this repository.
- GitHub creates a new package as **private**, even if the repository is public. To make it public: package page → **Package settings → Change visibility**, for each of the three packages. Restoring still needs a token, see [NuGet feeds](nuget-feed.md#github-packages).
- The `.nupkg` files on the GitHub release can be downloaded without a token, e.g. to put them on a file share or into another feed.

To run on your own Windows machine instead, register a self-hosted runner (repository **Settings → Actions → Runners → New self-hosted runner**) with the .NET 8 SDK, the .NET Framework 4.8 Developer Pack, PowerShell 7 (`pwsh`) and the GitHub CLI (`gh`). Then set the repository variable `SPKL_RUNS_ON` (**Settings → Secrets and variables → Actions → Variables**) to its labels as JSON, e.g. `["self-hosted","windows"]`. The same runner then builds pull requests too. If the repository is public, don't do that without requiring approval for workflows from forks (**Settings → Actions → General → Fork pull request workflows**), because anyone could run code on that machine through a pull request.

### 3. GitLab

#### Package registry

Make sure the project's package registry is enabled (in the project settings under visibility / project features; the exact menu names depend on the GitLab version).

#### Windows runner

The pipeline needs Windows, because spkl targets .NET Framework 4.8.

On a Windows build machine:

1. Install the .NET 8 SDK and the .NET Framework 4.8 Developer Pack.
2. Install GitLab Runner and register it for the project (Settings → CI/CD → Runners → New project runner) with the **shell** executor and a tag, e.g. `windows`.
3. In the runner's `config.toml`, set `shell = "pwsh"` (or `"powershell"`) for this runner.

If the tag is not `windows`, set the CI/CD variable `SPKL_RUNNER_TAG` (Settings → CI/CD → Variables) to it.

Optional: `SPKL_FEED_URL` pushes to another GitLab NuGet registry instead of the project's own, e.g. a group-level project. The default is `<gitlab>/api/v4/projects/<project id>/packages/nuget/index.json`, and the job pushes with its own job token, so no secret is needed.

### 4. Publish the first version

The repository is at version `1.0.0` (`<Version>` in `Directory.Build.props`).

```powershell
git tag v1.0.0
git push origin main v1.0.0
```

This starts both tag pipelines. Each one:

1. builds the test projects, runs the unit tests and runs `build/Pack.ps1`, which packs the three packages (GitHub also builds a solution from the template against them),
2. checks that the tag equals `v<Version>`, then pushes the packages to its feed. A version that already exists is skipped.

Check both feeds: `Spkl.OnPrem`, `Spkl.OnPrem.Attributes` and `Spkl.OnPrem.Templates` should be listed with `1.0.0` on GitHub (**Packages**) and in GitLab (**Deploy → Package registry** in recent GitLab versions).

### 5. Set up developer machines and solution pipelines

See [NuGet feeds](nuget-feed.md): the feed URLs, a token per developer, `nuget.config` in the solution repositories and restoring in their CI pipelines on GitLab or GitHub.

## Releasing an update

### 1. Raise the version

Set the new version in **all** of these places. `build/Pack.ps1` stops the build if they differ:

| File | What |
|---|---|
| `Directory.Build.props` | `<Version>` |
| `templates/solution/Tooling/Tooling.csproj` | `Spkl.OnPrem` version |
| `templates/solution/Plugins/Contoso.Plugins.csproj` | `Spkl.OnPrem.Attributes` version |
| `templates/solution/Actions/Contoso.Actions.csproj` | `Spkl.OnPrem.Attributes` version |
| `templates/solution/Workflows/Contoso.Workflows.csproj` | `Spkl.OnPrem.Attributes` version |

Use [semantic versioning](https://semver.org): patch for fixes (`1.0.1`), minor for new features (`1.1.0`), major for breaking changes such as removed attribute properties or spkl.json options (`2.0.0`).

Merge the change on GitHub as usual. GitHub CI builds a solution from the template against the new packages, so a mismatch shows up there first.

### 2. Tag and push to both

```powershell
git checkout main
git pull
git tag v1.0.1
git push origin main v1.0.1
```

Both feeds must always hold the same build of a version number:

- Tag once and push the same tag to both. Never publish a version on only one side on purpose.
- If one pipeline fails for a reason outside the code (runner offline, registry down), re-run that pipeline for the same tag. Packages that were already pushed are skipped.
- If the code is broken, raise the version again and tag the fix. Don't move or delete the tag, and don't delete a published version from one feed: a solution that already restored it would get a different package from the other feed.

### 3. Update existing solutions

The template only affects new solutions. In an existing solution:

1. Raise `Spkl.OnPrem` in `Tooling\Tooling.csproj` and `Spkl.OnPrem.Attributes` in the plugin, action and workflow projects, e.g. with **Manage NuGet Packages for Solution → Updates** in Visual Studio.
2. Build. `spkl.ps1` always runs the highest restored `spkl.exe` version.
3. Files that came from the template, such as `Tooling\spkl.ps1`, `.vscode\tasks.json` or `Directory.Build.props`, are not updated by the packages. Check `git diff v<old>..v<new> -- templates/solution` and copy changed files by hand.

Developers update the template itself with:

```powershell
dotnet new update
```

## Packing locally

```powershell
.\build\Pack.ps1 -OutputDirectory .\artifacts
```

This writes the three `.nupkg` files to `artifacts\`, e.g. to test a change locally with `dotnet nuget add source .\artifacts --name local`.
