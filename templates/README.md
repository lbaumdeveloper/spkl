# Templates

Both templates are packed by `Spkl.OnPrem.Templates.csproj`.

| Folder | Template | Use |
|---|---|---|
| `solution/` | `dotnet new spkl-solution` | New solution. See [getting started](../docs/getting-started.md). |
| `tooling/` | `dotnet new spkl-tooling` | Existing solution. See [existing solution](../docs/existing-solution.md). |

`tooling/` only contains its own `template.json` and `spkl.json`. `Tooling.csproj`, `spkl.ps1`, `.vscode/tasks.json` and `nuget.config` are packed from `solution/`, so change them there.
