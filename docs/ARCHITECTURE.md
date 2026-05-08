# Architecture

LevelManager is a small, layered Revit add-in compiled into three per-version assemblies from a single shared-source project.

## Target frameworks

| Revit | Framework | csproj style |
| :--- | :--- | :--- |
| 2024 | `.NET Framework 4.8` | Old-style (legacy `.csproj`, `<TargetFrameworkVersion>v4.8`) |
| 2025 | `net8.0-windows` | SDK-style (`<Project Sdk="Microsoft.NET.Sdk">`, `<UseWPF>true</UseWPF>`) |
| 2026 | `net8.0-windows` | SDK-style |

All three produce **`LevelManager.dll`**; the assembly name is identical so the same `.addin` manifest works against every version.

## Source layout

```
src/
├── Shared/                          (compiled into all three assemblies)
│   ├── Shared.shproj                (.shproj — shared-project metadata)
│   ├── Shared.projitems             (lists every .cs / .xaml / resource)
│   ├── App.cs                       (IExternalApplication — ribbon registration)
│   ├── Command.cs                   (IExternalCommand — opens MainForm)
│   ├── Api/
│   │   ├── LevelApiController.cs    (Revit-API ops: GetAll / Create / Delete)
│   │   └── EventHandlers/
│   │       ├── CreateLevelEventHandler.cs   (IExternalEventHandler)
│   │       └── DeleteLevelEventHandler.cs   (IExternalEventHandler)
│   ├── Domain/                      (POCOs — no Revit refs except elevation units)
│   │   ├── BasePointType.cs         (enum: Project = 0, Shared = 1)
│   │   ├── Elevation.cs             (double wrapper with rounding)
│   │   └── LevelModel.cs            (Name, Elevation, BasePointType)
│   ├── UI/
│   │   ├── MainForm.xaml            (modeless WPF window)
│   │   └── MainForm.xaml.cs         (code-behind, INotifyPropertyChanged)
│   └── Resources/                   (logo_16.png, logo_32.png — pack:// URIs)
├── LevelManager 2024/               (thin csproj for Revit 2024)
├── LevelManager 2025/               (thin csproj for Revit 2025)
└── LevelManager 2026/               (thin csproj for Revit 2026)
```

## Layering

```
┌─────────────────────────────────────────────────────────────┐
│ Revit host (UI thread + API thread)                         │
└──────────────┬───────────────────────────┬──────────────────┘
               │                           │
   ┌───────────▼─────────┐    ┌────────────▼─────────────┐
   │ App (Ribbon)        │    │ Command (IExternalCmd)   │
   └─────────────────────┘    └────────────┬─────────────┘
                                            │ shows
                                ┌───────────▼─────────────┐
                                │ UI/MainForm (WPF)       │
                                └───────────┬─────────────┘
                                            │ Raise()
                                ┌───────────▼─────────────┐
                                │ Api/EventHandlers/*     │   API thread
                                └───────────┬─────────────┘
                                            │ delegates to
                                ┌───────────▼─────────────┐
                                │ Api/LevelApiController  │   transactions
                                └───────────┬─────────────┘
                                            │ reads/writes
                                ┌───────────▼─────────────┐
                                │ Revit Document          │
                                └─────────────────────────┘
```

## Threading model

The Revit API may only be called from Revit's API thread, inside an `IExternalCommand.Execute` or `IExternalEventHandler.Execute`. The MainForm is **modeless** (`Window.Show()`), so its event handlers run on the WPF UI thread — *not* on Revit's API thread.

To bridge this:
1. `Command.Execute` constructs `CreateLevelEventHandler` and `DeleteLevelEventHandler`, pairs each with `ExternalEvent.Create(...)`, and passes both into `MainForm`.
2. When the user clicks **Add Level** or **Delete**, the WPF code-behind sets the handler's `Input` property and calls `event.Raise()`.
3. Revit invokes `Execute(UIApplication)` on the API thread at the next idle slice; the handler delegates to `LevelApiController`, which owns the `Transaction`.
4. `ExternalEvent` instances are disposed when the window closes (`MyMainForm_Closed`).

## Revit API surface used

| API | Purpose | Stable across 2024/2025/2026 |
| :--- | :--- | :--- |
| `FilteredElementCollector` + `OfCategory(BuiltInCategory.OST_Levels)` | Enumerate levels | ✓ |
| `Level.Create(Document, double)` | Create a level | ✓ |
| `Level.Name`, `Level.Elevation` | Read level metadata | ✓ |
| `BuiltInParameter.LEVEL_RELATIVE_BASE_TYPE` | Project vs Shared base | ✓ |
| `Document.Delete(ElementId)` | Remove level | ✓ |
| `Transaction` | Mutation atomicity | ✓ |
| `ExternalEvent` / `IExternalEventHandler` | Cross-thread marshalling | ✓ |

No version-specific `#if` directives are needed; the same shared source compiles cleanly against all three Revit API surfaces.
