# LevelManager

[![Build](https://img.shields.io/badge/build-passing-brightgreen)](#building-from-source)
[![License](https://img.shields.io/badge/license-TBD-lightgrey)](#license)
[![Revit](https://img.shields.io/badge/Revit-2024-blue)](#prerequisites)
[![.NET](https://img.shields.io/badge/.NET%20Framework-4.8-512BD4)](#prerequisites)

A Revit 2024 add-in with a WPF interface for managing project levels.
Lists every `Level` in the active document and lets the user create or
delete levels without leaving Revit. Document changes run inside Revit
transactions via the `ExternalEvent` bridge so the WPF window can stay
modeless.

## Features

- Modeless WPF window listing all levels (Name, Elevation, Base Point).
- Create a new level with name, elevation, and Project/Shared base point.
- Delete a level with confirmation and graceful error handling.
- Ribbon integration under **Add-Ins → Levels → Manager** with branded
  16/32-px icons.
- Input validation: numeric-only elevation, duplicate-name guard,
  Add-button disabled until both inputs are filled.

## Prerequisites

| Requirement | Version |
|---|---|
| Autodesk Revit | 2024 |
| .NET Framework | 4.8 (targeting pack required) |
| Visual Studio | 2022 (or any IDE with MSBuild + .NET 4.8 tooling) |
| OS | Windows 10/11 |

The project references `RevitAPI.dll` and `RevitAPIUI.dll` from
`C:\Program Files\Autodesk\Revit 2024\` via `HintPath`.

## Installation (end users)

1. Download or build `LevelManager.dll` and `LevelManager.addin`.
2. Place `LevelManager.addin` in one of:
   - `%AppData%\Autodesk\Revit\Addins\2024\` (current user), or
   - `%ProgramData%\Autodesk\Revit\Addins\2024\` (all users).
3. Edit the `<Assembly>` element inside the deployed `.addin` so it
   points to the absolute path of `LevelManager.dll` on your machine.
4. Start Revit 2024, open a project, then click
   **Add-Ins → Levels → Manager**.

## Building from source

```powershell
git clone <repo-url>
cd LevelManager
msbuild src\LevelManager\LevelManager.csproj /p:Configuration=Debug
```

Output: `src/LevelManager/bin/Debug/LevelManager.dll`. The
`LevelManager.addin` manifest is copied to the output directory on every
build.

To debug inside Revit, set the project's **Start external program** to
`C:\Program Files\Autodesk\Revit 2024\Revit.exe` and press **F5**.

## Usage

1. Click **Add-Ins → Levels → Manager** in the Revit ribbon.
2. The **Level Manager v1.0.0** window opens with all existing levels.

   [screenshot_here]

3. To add a level: enter a unique **Name**, type an **Elevation**
   (decimal feet — Revit internal units), pick a **Base Point Type**,
   and click **Add Level**.

   [screenshot_here]

4. To remove a level: click **Delete** on its row and confirm.

   [screenshot_here]

See [docs/USER_MANUAL.md](docs/USER_MANUAL.md) for every UI element
and error message.

## WPF + Revit API notes

- The Revit API may only be called from Revit's API thread, inside an
  `IExternalCommand.Execute` or `IExternalEventHandler.Execute`.
- The window is **modeless** (`Window.Show()`), so its event handlers
  run on the WPF UI thread — not on Revit's API thread.
- Each mutating operation (create, delete) is therefore wrapped in an
  `IExternalEventHandler` paired with an
  `ExternalEvent.Create(...)`. The UI sets the handler's `Input` and
  calls `event.Raise()`; Revit invokes the handler on its next idle
  slice.
- Transactions are owned by `LevelApiController` (`Create Level`,
  `Delete Level`). The command class is decorated with
  `[Transaction(TransactionMode.Manual)]`.
- `ExternalEvent` instances are disposed when the window closes
  (`MyMainForm_Closed`).

Architecture details: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Configuration

None. There are no settings files, JSON configs, or environment
variables. The only configurable item is the `<Assembly>` path inside
`LevelManager.addin` (see [Installation](#installation-end-users)).

## Known issues / limitations

- **Revit 2024 only.** No multi-version targeting.
- **Optimistic UI updates.** Rows are added/removed before the Revit
  transaction commits; if the transaction rolls back, the grid may
  drift from the document until the window is reopened.
- **Elevation units.** Input is passed straight to `Level.Create`,
  which expects Revit internal units (decimal feet). No unit
  conversion or display formatting beyond two-decimal rounding.
- **Hard-coded assembly path** in the shipped `.addin` file must be
  edited per machine.
- `DeleteLevelEventHandler` lives under namespace
  `LevelManagerApp.Windows` while the rest of the project uses
  `LevelManager.*`.
- Error reporting from background transactions is limited to
  `Debug.WriteLine` and a single `TaskDialog` for delete failures.

## Contributing

1. Fork the repo and create a feature branch from `master`.
2. Keep the project structure under `src/LevelManager/` (Domain, Api,
   UI, EventHandlers).
3. Follow the existing patterns:
   - Document mutations belong in `LevelApiController`, each in its
     own `Transaction`.
   - WPF → Revit calls go through a dedicated `IExternalEventHandler`.
4. Build with MSBuild and smoke-test inside Revit 2024 before opening
   a PR.
5. Open a pull request describing the change and the test you ran in
   Revit.

Issues and feature requests are welcome via GitHub Issues.

## License

**All rights reserved**
