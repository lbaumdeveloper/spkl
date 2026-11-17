# Adopting spkl in an existing solution

This guide moves an existing Visual Studio solution to spkl when its plugins, workflow activities and web resources are already registered in a running Dynamics 365 development environment. Nothing is registered twice: spkl finds existing components by name and updates them.

The steps assume an unmanaged solution in the environment that already contains your components, for example `ContosoCore`. spkl adds everything it deploys to that solution and never to the default solution.

## How spkl matches what is already registered

| Component | Matched by | Not in code → |
|---|---|---|
| Plugin assembly | Assembly name | – |
| Plugin type / workflow activity | Full class name (`Namespace.Class`) | Deleted with its steps, if the class is no longer in the assembly |
| Step | `Id` in the attribute, otherwise step name + message | Deleted, if its class has at least one attribute |
| Image | Alias + image type | Deleted |
| Custom API | Unique name (only the plugin type is set) | – |
| Web resource | Unique name | Kept, unless `deleteaction` says otherwise |

What spkl never writes, and so keeps as it is on the server: secure configuration, the impersonating user ("Run in user's context") and the Custom API definition. What it does overwrite: the name and friendly name of plugin types become the full class name.

Two rules follow from the table:

- **Every step you want to keep needs an attribute** on a class that has attributes. `instrument` (step 4) writes them for you.
- **Classes without any attribute are left alone.** Their steps stay untouched, so you can migrate one class at a time.

## 1. Add the Tooling

Run the `spkl-tooling` template in the folder that contains the `.sln`. Set up the feed on your machine first, see [NuGet feeds](nuget-feed.md#developer-machines).

```powershell
dotnet new install Spkl.OnPrem.Templates
cd C:\src\Contoso
dotnet new spkl-tooling --SolutionUniqueName ContosoCore `
  --FeedUrl "https://<gitlab>/api/v4/projects/<project id>/packages/nuget/index.json"
```

It creates:

```
Contoso/
├── nuget.config          ← nuget.org + the spkl feed, no credentials
├── .vscode/tasks.json    ← VS Code commands
└── Tooling/
    ├── Tooling.csproj    ← references Spkl.OnPrem, added to the .sln
    ├── spkl.json         ← edit in step 3
    └── spkl.ps1
```

If the solution already has a `nuget.config` or `.vscode/tasks.json`, pass `--NuGetConfig false` or `--VsCodeTasks false` and copy the `spkl` source or the `spkl: …` tasks from [`templates/solution`](../templates/solution) into your files. If the template cannot find exactly one `.sln`, add the project yourself with `dotnet sln add Tooling\Tooling.csproj`.

Unlike `spkl-solution`, this template adds no `Directory.Build.props` and no signing key, because your projects already have both.

## 2. Reference the attributes

Install `Spkl.OnPrem.Attributes` into every project that contains plugins, Custom API plugins or workflow activities.

- **packages.config**: the file `CrmPluginRegistrationAttribute.cs` is added to the project.
- **PackageReference**: it is compiled in without appearing in the project.

If the project already has a `CrmPluginRegistrationAttribute.cs` from the original spkl package, delete it and remove the `spkl` package, otherwise the class exists twice. `SecureConfiguration` and `Action` no longer exist on the attribute; remove them from existing attributes. Secure configuration stays on the server.

## 3. Edit `Tooling/spkl.json`

Replace the placeholders with your projects. Paths are relative to the `Tooling` folder.

```json
{
  "plugins": [
    {
      "profile": "default,plugins",
      "solution": "ContosoCore",
      "assemblypath": "..\\Contoso.Plugins\\bin\\Debug\\Contoso.Plugins.dll"
    },
    {
      "profile": "default,workflows",
      "solution": "ContosoCore",
      "assemblypath": "..\\Contoso.Workflows\\bin\\Debug\\Contoso.Workflows.dll"
    }
  ],
  "webresources": [
    {
      "profile": "default",
      "solution": "ContosoCore",
      "root": "..\\Webresources\\",
      "autodetect": "yes"
    }
  ]
}
```

Use one entry per assembly. A profile per assembly lets you deploy them separately. Every option is described in the [spkl.json reference](spkl-json.md).

Keep two things as they are registered today:

- **The signing key.** Don't run `New-SigningKey.ps1`: a new key changes the public key token, and the assembly no longer matches the registered one.
- **The assembly name and the major and minor version.** spkl finds the assembly by name. Only change the build or revision part of the version.

## 4. Generate the attributes with `instrument`

`instrument` reads the registered steps, images and workflow activities from the environment and writes matching `[CrmPluginRegistration]` attributes above your classes. Each step attribute gets the step's `Id`, so later deploys update exactly that step, even if you rename it.

Commit or stash first, then run it once per project folder:

```powershell
.\Tooling\spkl.ps1 -Task instrument -Path .\Contoso.Plugins
.\Tooling\spkl.ps1 -Task instrument -Path .\Contoso.Workflows
```

- It scans every `.cs` file below `-Path` and replaces existing `[CrmPluginRegistration]` attributes on the classes it finds.
- It finds `public` or `public sealed` classes that inherit `IPlugin`, a base class whose name starts with `Plugin` (e.g. `PluginBase`), or `CodeActivity`. For another base class, put a `spkl.json` with only `{ "plugins": [ { "classRegex": "…" } ] }` into the scanned folder for this run and delete it afterwards. The default regex is in the [spkl.json reference](spkl-json.md#plugins).
- It stops if a class has two steps with the same name or a step has more than two images. Fix those in the Plugin Registration Tool and run it again.
- It writes a `.bak` copy next to every changed file. Review the diff, then delete the `.bak` files.
- It does not change `spkl.json`.

Classes that are not registered get no attribute and are skipped on deploy until you add one.

## 5. Map the web resources

Pick one of the two ways.

**A. Download them** (recommended when the repository has no web resource files yet, or they are outdated):

```powershell
.\Tooling\spkl.ps1 -Task download-webresources
```

Every web resource in the `solution` is written below `root`, with its name as the path: `new_/js/account.js` → `Webresources\new_\js\account.js`. With `autodetect` this is exactly the layout spkl deploys from. Existing files are kept unless you add `-Overwrite`.

**B. Keep your folder layout**: remove `autodetect`, point `root` to your folder and let spkl list the matches in `files`:

```powershell
.\Tooling\spkl.ps1 -Task get-webresources -Prefix new
```

`-Prefix` is your publisher prefix. Each local file whose path contains a web resource's name without the prefix is added to `files` with that unique name. Check the list; files that matched nothing are not deployed.

## 6. Preview, then deploy

Build the solution, then run every deploy as a dry run first:

```powershell
.\Tooling\spkl.ps1 -Task plugins -DryRun
.\Tooling\spkl.ps1 -Task workflow -Profile workflows -DryRun
.\Tooling\spkl.ps1 -Task webresources -DryRun
```

For a clean migration you see only updates and `AddSolutionComponent` requests:

```
[dry-run] Would update pluginassembly 'Contoso.Plugins' 3f2c...
[dry-run] Would update plugintype 'Contoso.Plugins.AccountPreCreate' ...
[dry-run] Would update sdkmessageprocessingstep 'Contoso.Plugins.AccountPreCreate: Create of account' ...
[dry-run] 0 create(s), 7 update(s), 0 delete(s), 3 other request(s). Nothing was written.
```

| You see | Usual cause |
|---|---|
| `create` of a step that already exists | The class has an attribute without `Id`, and its name or message differs from the registered step. Run `instrument` again or copy the `Id`. |
| `delete` of a step | A step is registered on the class but has no attribute. Add it, or let it be deleted if it is obsolete. |
| `delete` of a plugin type | The class was renamed or removed from the assembly. |
| `create` of a plugin assembly | `assemblypath` points to a DLL whose name differs from the registered assembly. |
| No web resource lines | All files are unchanged, which is expected right after `download-webresources`. |

When the dry run looks right, deploy without `-DryRun`. From now on the attributes and `spkl.json` are the source of truth: register new steps in code, not in the Plugin Registration Tool, because spkl deletes steps it does not find in code.

## Afterwards

- Add `*.bak` to `.gitignore`.
- Commit `Tooling/`, `nuget.config`, `.vscode/tasks.json` and the changed `.cs` files.
- Set up Visual Studio External Tools and connections as described in [getting started](getting-started.md#3-run-commands).
- For CI restores on GitLab or GitHub, see [NuGet feeds](nuget-feed.md#ci-of-solution-repositories).
