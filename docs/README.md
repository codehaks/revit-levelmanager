# LevelManager

A Revit add-in that provides a WPF-based UI for listing, creating, and deleting
Revit `Level` elements in the active document.

- Source: `src/LevelManager/`
- Add-in manifest: `src/LevelManager/LevelManager.addin`
- Target framework: .NET Framework 4.8
- Target Revit version: Revit 2024 (per `RevitAPI.dll` HintPath in
  `src/LevelManager/LevelManager.csproj`)

## Features

Read directly from the source code:

- Adds a ribbon panel named **Levels** with a single push button **Manager**
  (see `src/LevelManager/App.cs`).
- Opens a WPF window **Level Manager v1.0.0** that:
  - Lists all existing levels in the active document with columns
    *Name*, *Elevation*, *Base Point*, and a per-row *Delete* action
    (`src/LevelManager/UI/MainForm.xaml`).
  - Creates a new `Level` from user input (Name, Elevation, Base Point Type)
    via `LevelApiController.Create` in
    `src/LevelManager/Api/LevelApiController.cs`.
  - Deletes a selected level via `LevelApiController.Delete` in the same file.
- Uses Revit `ExternalEvent` handlers
  (`CreateLevelEventHandler`, `DeleteLevelEventHandler`) so document
  modifications run in a valid Revit API context while the WPF window remains
  modeless.

## Project layout

```
src/LevelManager/
├── App.cs                                # IExternalApplication: ribbon setup
├── Command.cs                            # IExternalCommand: launches MainForm
├── LevelManager.addin                    # Revit add-in manifest
├── LevelManager.csproj                   # .NET 4.8 project
├── Api/
│   ├── LevelApiController.cs             # GetAll / Create / Delete (transactions live here)
│   └── EventHandlers/
│       ├── CreateLevelEventHandler.cs    # IExternalEventHandler
│       └── DeleteLevelEventHandler.cs    # IExternalEventHandler
├── Domain/
│   ├── BasePointType.cs                  # enum: Project = 0, Shared = 1
│   ├── Elevation.cs                      # value wrapper around double
│   └── LevelModel.cs                     # DTO: Name, Elevation, BasePointType
├── UI/
│   ├── MainForm.xaml                     # WPF window
│   └── MainForm.xaml.cs                  # Code-behind (event wiring + validation)
├── Resources/
│   ├── logo_16.png
│   └── logo_32.png
└── Properties/AssemblyInfo.cs
```

## Dependencies

From `src/LevelManager/LevelManager.csproj`:

- `RevitAPI.dll` (Revit 2024)
- `RevitAPIUI.dll` (Revit 2024)
- `PresentationCore`, `PresentationFramework`, `WindowsBase`, `System.Xaml`
  (WPF)
- Standard BCL: `System`, `System.Core`, `System.Xml`, etc.

## Installation

1. Build the project (see [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md)).
2. Copy `LevelManager.addin` to one of:
   - `%ProgramData%\Autodesk\Revit\Addins\2024\`
   - `%AppData%\Autodesk\Revit\Addins\2024\`
3. Edit the `<Assembly>` element in the deployed `.addin` to point to the
   absolute path of your built `LevelManager.dll`. The shipped manifest hard-
   codes
   `C:\Projects\Revit\LevelManager\code\LevelManager\src\LevelManager\bin\Debug\LevelManager.dll`
   — update it for your machine.
4. Start Revit 2024, open or create a project, and click
   **Add-Ins → Levels → Manager**.

## Usage

See [USER_MANUAL.md](USER_MANUAL.md).

## Documentation index

- [API_REFERENCE.md](API_REFERENCE.md) — public classes and methods
- [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md) — build, debug, threading model,
  extension points
- [USER_MANUAL.md](USER_MANUAL.md) — end-user walkthrough
- [ARCHITECTURE.md](ARCHITECTURE.md) — components and data flow

