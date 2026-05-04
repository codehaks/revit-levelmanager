# Developer Guide

## Prerequisites

- Visual Studio 2022 (or any IDE that builds .NET Framework 4.8 projects)
- Autodesk Revit 2024 installed at `C:\Program Files\Autodesk\Revit 2024\`
  — the `RevitAPI`/`RevitAPIUI` references in
  `src/LevelManager/LevelManager.csproj` are resolved via `HintPath`
  relative to that location.
- .NET Framework 4.8 targeting pack.

## Build

```powershell
# from repo root
msbuild src\LevelManager\LevelManager.csproj /p:Configuration=Debug
```

Output lands in `src/LevelManager/bin/Debug/`. The project copies
`LevelManager.addin` to the output folder
(`<CopyToOutputDirectory>Always</CopyToOutputDirectory>`).

## Run / debug in Revit

1. Edit the deployed `LevelManager.addin` so that `<Assembly>` points to your
   absolute build output path. The version checked in points at
   `C:\Projects\Revit\LevelManager\code\...` — adjust for your machine.
2. Place the manifest in `%AppData%\Autodesk\Revit\Addins\2024\`.
3. In Visual Studio set the project's **Start external program** to
   `C:\Program Files\Autodesk\Revit 2024\Revit.exe` and press **F5**.
4. Open any project, then click **Add-Ins → Levels → Manager**.

[REVIEW REQUIRED: there is no `.csproj` post-build step that auto-deploys
the DLL or `.addin` to the user's add-ins folder. Confirm whether one is
desired before adding it.]

## WPF + Revit threading model used in this code

Revit only allows document modifications on its own API thread, and only
inside an `IExternalCommand.Execute` or an `IExternalEventHandler.Execute`
callback. This add-in opens a **modeless** WPF window, so any code that runs
from a WPF event handler (button click, etc.) is on the WPF UI thread, not
on Revit's API thread. Calling Revit document APIs directly from a button
click would throw an "invalid API context" exception.

The chosen pattern is the standard Revit `ExternalEvent` bridge:

1. `Command.Execute` (a valid API context) creates the two
   `IExternalEventHandler` instances and wraps each with
   `ExternalEvent.Create(...)`.
2. Both handler instances and both `ExternalEvent` instances are passed
   into `MainForm`'s constructor.
3. When the user clicks **Add Level** or **Delete**, `MainForm` writes the
   request payload onto the handler's `Input` property, then calls
   `event.Raise()`. Revit invokes the handler back on the API thread on its
   next idle slice.
4. When the window closes (`MyMainForm_Closed`), both `ExternalEvent`s are
   disposed.

The UI updates its `ObservableCollection<LevelModel>` **optimistically** —
it adds/removes rows immediately, before the Revit-side transaction has
actually committed. The handler does not call back into the UI; if the
transaction rolls back inside `LevelApiController` the grid will be out of
sync with the document until the window is reopened.
[REVIEW REQUIRED: confirm this optimistic behavior is intentional.]

## Transaction strategy

Transactions are owned by `LevelApiController`, **not** by the event
handlers and **not** by the WPF code-behind:

- `Command` is decorated with `[Transaction(TransactionMode.Manual)]`
  (`src/LevelManager/Command.cs`) so the add-in is responsible for opening
  its own transactions.
- `LevelApiController.Create` opens
  `new Transaction(_Document, "Create Level")`, calls `Start()`, performs
  the work, and either `Commit()`s on success or `RollBack()`s in the
  `catch` block.
- `LevelApiController.Delete` follows the same pattern with a
  `"Delete Level"` transaction.
- `GetAll` is read-only and uses no transaction.

Errors are not surfaced to the UI: `Create` only writes to `Debug.WriteLine`
on failure; `Delete` shows a `TaskDialog` and writes to `Debug.WriteLine`.

## Adding a new WPF window

The existing `MainForm` is the template. To add another window:

1. Create `src/LevelManager/UI/YourWindow.xaml` and `YourWindow.xaml.cs`
   under namespace `LevelManager.UI`.
2. Register both in `LevelManager.csproj` — the `.xaml` as a `<Page>` with
   `Generator>MSBuild:Compile`, and the `.xaml.cs` as `<Compile>` with
   `<DependentUpon>YourWindow.xaml</DependentUpon>` (mirror how
   `MainForm` is wired in the existing csproj).
3. If the window mutates Revit data, follow the same `ExternalEvent`
   bridge: write a new `IExternalEventHandler` per operation under
   `Api/EventHandlers/`, instantiate it and its `ExternalEvent` from inside
   an `IExternalCommand.Execute`, and pass them into the window.
4. Either invoke the window from the existing `Command` or add a new
   `IExternalCommand` and a new `PushButtonData` in
   `App.OnStartup`.

## Extending functionality

- **New level operation (e.g. rename):** add a method to
  `LevelApiController` that owns its own `Transaction`, then add a matching
  `IExternalEventHandler` whose `Input` is whatever DTO it needs.
- **New domain field:** extend `LevelModel` (and `LevelApiController.GetAll`
  to populate it). Add a column to `LevelsDataGrid` in
  `MainForm.xaml`. The grid uses `Mode=OneTime` bindings — switch to
  `OneWay` if the new field should refresh after edits.
- **Different unit handling for elevation:** today
  `LevelApiController.Create` passes `input.Elevation.Value` straight into
  `Level.Create`, which expects Revit internal units (decimal feet).
  Convert in the UI or in `Elevation` before calling `Create` if you need
  a different input unit.
