# Developer Guide

Day-to-day notes for developing on LevelManager.

## Code map

| Where to look | What lives there |
| :--- | :--- |
| [src/Shared/App.cs](../src/Shared/App.cs) | Ribbon panel + button. Edit here to change icons, button label, or add panels. |
| [src/Shared/Command.cs](../src/Shared/Command.cs) | Wires up `LevelApiController`, the two event handlers, `ExternalEvent`s, and shows `MainForm`. |
| [src/Shared/Api/LevelApiController.cs](../src/Shared/Api/LevelApiController.cs) | Every Revit-API mutation. **Add new transactions here.** |
| [src/Shared/Api/EventHandlers/](../src/Shared/Api/EventHandlers/) | `IExternalEventHandler` implementations — one per mutating action. |
| [src/Shared/Domain/](../src/Shared/Domain/) | POCOs (no Revit refs except elevation as `double`). |
| [src/Shared/UI/MainForm.xaml.cs](../src/Shared/UI/MainForm.xaml.cs) | WPF code-behind. Owns the `ObservableCollection<LevelModel>`. |

## Adding a new mutating action

1. Add a method on `LevelApiController` that owns its own `Transaction`.
2. Add an `IExternalEventHandler` under `Api/EventHandlers/` that takes its `Input` from a property and invokes the controller method.
3. In `Command.Execute`, create the handler + a paired `ExternalEvent.Create(handler)` and pass both into `MainForm`'s constructor.
4. In `MainForm`, set `handler.Input = …` then call `externalEvent.Raise()` from your button handler.
5. Add the new file to [`src/Shared/Shared.projitems`](../src/Shared/Shared.projitems) so all three csprojs pick it up.

## Adding a new file to the shared project

The `.projitems` file is the source-of-truth list. Add an entry like:

```xml
<Compile Include="$(MSBuildThisFileDirectory)Path\To\NewFile.cs" />
```

For XAML pages, add both a `<Compile>` (with `<DependentUpon>`) and a `<Page>` entry. For images / icons, add a `<Resource>` entry.

## Debugging in Revit

1. Build Debug (`build.bat Debug`).
2. Copy `dist\Revit 2024\LevelManager.{dll,pdb,addin}` into `%AppData%\Autodesk\Revit\Addins\2024\`.
3. In Visual Studio, open the appropriate per-version csproj, set the project's **Start external program** to `C:\Program Files\Autodesk\Revit <year>\Revit.exe`, and press **F5**.
4. Set breakpoints in shared source — they hit regardless of which per-version assembly is loaded, because the .pdb maps to the shared `.cs`.

## Adding a new Revit version

When Autodesk releases Revit 2027:

1. Copy `src/LevelManager 2026/` to `src/LevelManager 2027/`, rename the `.csproj`, and update the `RevitAPI` / `RevitAPIUI` HintPaths to `Revit 2027\…`.
2. Add the new csproj to [`LevelManager.sln`](../LevelManager.sln) and the `SharedMSBuildProjectFiles` block.
3. Add a `call :build 2027 …` line to [`build.bat`](../build.bat).
4. Update the version matrix in [build-and-load.md](build-and-load.md) and the README badges.

## Coding conventions

- C# `latest` lang version, x64 only, AnyCPU platform.
- Domain types are immutable POCOs.
- All Revit-API calls live under `Api/`. Domain and UI layers must not reference `Autodesk.Revit.*`.
- Each mutating operation owns its own `Transaction`.
