# Architecture

## High-level diagram (text)

```
                  Revit process
 ┌───────────────────────────────────────────────────────────────────┐
 │                                                                   │
 │  Ribbon panel "Levels"  ── click ──▶  LevelManager.Command        │
 │   (built in App.OnStartup)            [IExternalCommand,          │
 │                                        TransactionMode.Manual]    │
 │                                              │                    │
 │                                              ▼                    │
 │                              ┌─────────────────────────────────┐  │
 │                              │ Command.Execute                 │  │
 │                              │  - LevelApiController(doc)      │  │
 │                              │  - GetAll() → List<LevelModel>  │  │
 │                              │  - new CreateLevelEventHandler  │  │
 │                              │  - new DeleteLevelEventHandler  │  │
 │                              │  - ExternalEvent.Create(...) x2 │  │
 │                              └────────────────┬────────────────┘  │
 │                                               ▼                   │
 │                                  ┌──────────────────────────┐     │
 │                                  │   MainForm (WPF, modeless)│    │
 │                                  │   src/LevelManager/UI/    │    │
 │                                  └──────┬──────────┬─────────┘    │
 │  WPF UI thread                          │          │              │
 │  ────────────────────────────────────── │ Raise()  │ Raise() ──── │
 │  Revit API thread                       ▼          ▼              │
 │                          CreateLevelEventHandler  DeleteLevelEH   │
 │                                     │                  │          │
 │                                     ▼                  ▼          │
 │                         LevelApiController.Create / Delete        │
 │                          (opens & commits Transaction)            │
 │                                     │                             │
 │                                     ▼                             │
 │                            Autodesk.Revit.DB.Document             │
 └───────────────────────────────────────────────────────────────────┘
```

## Component responsibilities

| Component | File | Responsibility |
|---|---|---|
| `App` | `App.cs` | `IExternalApplication`. Registers the ribbon panel and push button. No domain logic. |
| `Command` | `Command.cs` | `IExternalCommand` entry point. Wires up controller, handlers, and external events; opens the window modelessly. |
| `LevelApiController` | `Api/LevelApiController.cs` | Sole owner of Revit `Document` reads and writes. Opens transactions for `Create` and `Delete`. |
| `CreateLevelEventHandler` | `Api/EventHandlers/CreateLevelEventHandler.cs` | `IExternalEventHandler` that runs `LevelApiController.Create` on Revit's API thread. |
| `DeleteLevelEventHandler` | `Api/EventHandlers/DeleteLevelEventHandler.cs` | Same, for `Delete`. Namespace is `LevelManagerApp.Windows` — see API_REFERENCE.md note. |
| `LevelModel` / `Elevation` / `BasePointType` | `Domain/*.cs` | Plain DTO + value wrappers carried between layers. No Revit dependency. |
| `MainForm` | `UI/MainForm.xaml` (+`.xaml.cs`) | Presents the level list, validates user input, raises external events, and maintains the in-memory `ObservableCollection<LevelModel>`. Acts as its own view-model (no separate VM class). |

There is **no MVVM** layer in the strict sense — `MainForm.xaml.cs`
exposes its own bindable properties (`BasePointTypeEnumValues`,
`SelectedBasePointType`, `IsAddLevelButtonEnabled`) and the XAML binds back
to the `Window` via `RelativeSource AncestorType=Window`.

## Data flow: creating a level

1. User types Name + Elevation, picks Base Point Type, clicks **Add
   Level** in `MainForm`.
2. `AddLevel_Click` validates inputs, builds a
   `LevelModel(name, new Elevation(double), basePointType)`.
3. The model is assigned to `_createLevelEventHandler.Input`; then
   `_createLevelExternalEvent.Raise()` queues the operation with Revit.
4. `MainForm` adds the new `LevelModel` to its `ObservableCollection`
   immediately (optimistic update) and clears the inputs.
5. Revit, on its next idle slice, calls `CreateLevelEventHandler.Execute`
   on its API thread.
6. The handler instantiates a new `LevelApiController` for the active
   document and calls `Create(Input)`.
7. `LevelApiController.Create` opens a `"Create Level"` `Transaction`,
   calls `Level.Create`, sets `Name`, sets
   `BuiltInParameter.LEVEL_RELATIVE_BASE_TYPE` from
   `(int)BasePointType`, and commits — or rolls back and logs to
   `Debug.WriteLine` on exception.

## Data flow: deleting a level

1. User clicks the per-row **Delete** button in `LevelsDataGrid`.
2. `DeleteRow_Click` confirms via `MessageBox`, sets
   `_deleteLevelEventHandler.Input = selectedLevel`, raises the external
   event, and removes the row from the `ObservableCollection`.
3. `DeleteLevelEventHandler.Execute` runs on Revit's API thread, calls
   `LevelApiController.Delete(input.Name)`.
4. `Delete` resolves the level by name (`FindByName`) and runs a
   `"Delete Level"` transaction. Failure surfaces via `TaskDialog` and
   `Debug.WriteLine`; the UI is not notified.

## Threading summary

- WPF UI thread: input validation, `MessageBox`/`TaskDialog` is invoked
  from both threads but always reaches the user safely.
- Revit API thread: all `Document` reads (`GetAll` runs at command
  startup, before the modeless window takes over the UI thread) and all
  writes via `IExternalEventHandler.Execute`.
- The `ExternalEvent` bridge is the only legal way to mutate the document
  from the modeless WPF window in this design.

## Lifetime

- `App.OnStartup` runs once per Revit session.
- `Command.Execute` runs once per ribbon click. It creates fresh
  handlers and external events every time, so closing and reopening the
  window starts a clean state.
- `MyMainForm_Closed` disposes both `ExternalEvent` instances. The
  handler objects themselves are garbage-collected with the window.
