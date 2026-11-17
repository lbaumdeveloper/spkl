# spkl

Deployment task runner for Dynamics 365 / Dataverse. spkl deploys plugins, Custom APIs, workflow activities and web resources, generates early-bound classes, and packs/unpacks solutions, all from one `spkl.json` and the `[CrmPluginRegistration]` attributes in your code.

This repository contains only spkl. The original SparkleXrm framework, UI components and samples were removed.

## Documentation

| Guide | Read it when |
|---|---|
| [Getting started](docs/getting-started.md) | Setting up a solution with `dotnet new spkl-solution`, running spkl from VS Code, Visual Studio or a terminal. |
| [Existing solution](docs/existing-solution.md) | Adding spkl to a solution whose plugins, workflows and web resources are already registered in an environment. |
| [Commands](docs/commands.md) | You want to know what a task does, what it reads and what it changes or deletes. |
| [spkl.json reference](docs/spkl-json.md) | Writing or debugging `spkl.json`: every option, paths and profiles. |
| [Releasing](docs/releasing.md) | Publishing the packages on the private GitLab, first version and updates. |
| [NuGet feed on GitLab](docs/nuget-feed.md) | Adding the GitLab feed on developer machines and restoring from it in the CI of solution repositories. |
| [Registration attributes](docs/registration-attributes.md) | Registering plugin steps, images, Custom APIs and workflow activities in code. |

The templates are in [`templates`](templates): `spkl-solution` for a new solution, `spkl-tooling` for an existing one.

## Quick start

```powershell
dotnet new install Spkl.OnPrem.Templates
dotnet new spkl-solution -n Contoso --SolutionUniqueName ContosoCore --FeedUrl "<GitLab NuGet feed>"
cd Contoso; .\Tooling\New-SigningKey.ps1
.\Tooling\spkl.ps1 -Task plugins               # deploy plugins (after building)
.\Tooling\spkl.ps1 -Task webresources          # deploy web resources
```

Existing solution with components already registered:

```powershell
dotnet new spkl-tooling --SolutionUniqueName ContosoCore --FeedUrl "<GitLab NuGet feed>"
.\Tooling\spkl.ps1 -Task instrument -Path .\Contoso.Plugins   # attributes from the registered steps
.\Tooling\spkl.ps1 -Task plugins -DryRun                      # expect updates only
```

## Requirements

- Windows with .NET Framework 4.8
- Dynamics 365 v9 on-premises (AD / IFD) or online
- A Dataverse / Dynamics 365 environment

## Repository layout

| Folder | Content |
|---|---|
| `spkl/` | Source: CLI (`spkl`), tasks (`SparkleXrm.Tasks`), attribute package (`Spkl.Attributes`), tests and test projects |
| `docs/` | Documentation |
| `templates/` | `dotnet new spkl-solution` and `spkl-tooling` templates (`Spkl.OnPrem.Templates`) |

Planned work is tracked in [`todo.md`](todo.md).

License: [MIT](MIT-License.txt)
