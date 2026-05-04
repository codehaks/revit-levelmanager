# API Reference

All types live under the `LevelManager` root namespace.

---

## `LevelManager.App`

File: `src/LevelManager/App.cs`

Implements `Autodesk.Revit.UI.IExternalApplication`. Registered in
`LevelManager.addin` as the `FullClassName`.

### Methods

```csharp
public Result OnStartup(UIControlledApplication application);
public Result OnShutdown(UIControlledApplication application);
```

`OnStartup` creates a ribbon panel `"Levels"` and adds a `PushButtonData`
for `LevelManager.Command` with `Resources/logo_16.png` and
`Resources/logo_32.png` as button images. Returns `Result.Succeeded`.

`OnShutdown` is a no-op that returns `Result.Succeeded`.

---

## `LevelManager.Command`

File: `src/LevelManager/Command.cs`

```csharp
[Transaction(TransactionMode.Manual)]
internal class Command : IExternalCommand
{
    public Result Execute(ExternalCommandData commandData,
                          ref string message,
                          ElementSet elements);
}
```

Entry point for the ribbon button. It:

1. Resolves the active `Document` from `commandData.Application.ActiveUIDocument`.
2. Builds a `LevelApiController` and calls `GetAll()` to seed the UI list.
3. Creates `CreateLevelEventHandler` and `DeleteLevelEventHandler` and wraps
   each in an `ExternalEvent` via `ExternalEvent.Create(...)`.
4. Constructs `MainForm` with the level list, both handlers, and both events,
   and shows it modelessly with `Show()`.

Always returns `Result.Succeeded`.

---

## `LevelManager.Api.LevelApiController`

File: `src/LevelManager/Api/LevelApiController.cs`

Wraps Revit document operations on `Level` elements. **All mutating methods
open and commit their own `Transaction`** — callers must invoke them from a
valid Revit API context (i.e. inside an `IExternalEventHandler.Execute`).

### Constructor

```csharp
public LevelApiController(Document document);
```

### `GetAll`

```csharp
public List<LevelModel> GetAll();
```

Uses a `FilteredElementCollector` with
`OfCategory(BuiltInCategory.OST_Levels).WhereElementIsNotElementType()` to
collect levels and projects each into a `LevelModel`. The `BasePointType` is
derived from the level **type's** `LEVEL_RELATIVE_BASE_TYPE` parameter via
the private helper `GetLevelBasePointType` — `0` ⇒ `Project`, otherwise
`Shared`.

Returns an empty list when the document has no levels.

### `Create`

```csharp
public void Create(LevelModel input);
```

Opens a `Transaction` named `"Create Level"`, calls
`Level.Create(_Document, input.Elevation.Value)`, sets `Name`, then sets
`BuiltInParameter.LEVEL_RELATIVE_BASE_TYPE` to `(int)input.BasePointType`.
On exception the transaction is rolled back and the message is logged via
`Debug.WriteLine`.

### `Delete`

```csharp
public void Delete(string levelName);
```

Looks up the level via the private `FindByName` helper. If not found, shows
`MessageBox.Show($"Level not found: {levelName}")` and returns. Otherwise
opens a `"Delete Level"` transaction and calls `_Document.Delete(level.Id)`.
On failure rolls back and shows
`TaskDialog.Show("Invalid Operation", $"Could not delete level {levelName}")`.

### Private helpers

- `Level FindByName(string levelName)` — linear scan over a
  `FilteredElementCollector.OfClass(typeof(Level))`.
- `BasePointType GetLevelBasePointType(Element level)` — reads
  `LEVEL_RELATIVE_BASE_TYPE` off the level's element type.

---

## `LevelManager.Api.EventHandlers.CreateLevelEventHandler`

File: `src/LevelManager/Api/EventHandlers/CreateLevelEventHandler.cs`

Implements `IExternalEventHandler`.

```csharp
public LevelModel Input { get; set; }
public void Execute(UIApplication app);
public string GetName(); // returns "CreateLevelEventHandler"
```

`Execute` constructs a `LevelApiController` from the active document and
calls `Create(Input)`. The caller is expected to set `Input` **before**
raising the associated `ExternalEvent`.

Example:

```csharp
_createLevelEventHandler.Input = level;
_createLevelExternalEvent.Raise();
```

---

## `LevelManagerApp.Windows.DeleteLevelEventHandler`

File: `src/LevelManager/Api/EventHandlers/DeleteLevelEventHandler.cs`

[REVIEW REQUIRED: namespace is `LevelManagerApp.Windows`, which differs from
the rest of the project's `LevelManager.*` namespaces. Likely a leftover from
an earlier refactor.]

```csharp
public LevelModel Input { get; set; }
public bool Success { get; set; } = false;
public void Execute(UIApplication app);
public string GetName(); // returns "DeleteLevelHandler"
```

`Execute` calls `LevelApiController.Delete(Input.Name)` and unconditionally
sets `Success = true` afterward (it is not flipped back to `false` on
failure inside `LevelApiController.Delete`, which catches its own
exceptions).

---

## `LevelManager.Domain.LevelModel`

File: `src/LevelManager/Domain/LevelModel.cs`

```csharp
public LevelModel(string name, Elevation elevation, BasePointType basePointType);

public string         Name           { get; }
public Elevation      Elevation      { get; }
public BasePointType  BasePointType  { get; }
```

Immutable DTO carried between the UI, the event handlers, and
`LevelApiController`.

---

## `LevelManager.Domain.Elevation`

File: `src/LevelManager/Domain/Elevation.cs`

```csharp
public Elevation(double value);

public double Value       { get; }                  // raw value, units as Revit
public double SimpleValue { get; }                  // Math.Round(Value, 2)
public double RoundedBy(int digits);                // Math.Round(Value, digits)
```

[REVIEW REQUIRED: `Value` is passed straight to `Level.Create` as the
internal-units double. Revit internal length units are decimal feet — UI
input is treated as feet without conversion.]

---

## `LevelManager.Domain.BasePointType`

File: `src/LevelManager/Domain/BasePointType.cs`

```csharp
public enum BasePointType
{
    Project = 0,
    Shared  = 1
}
```

Maps to Revit's `BuiltInParameter.LEVEL_RELATIVE_BASE_TYPE`.

---

## `LevelManager.UI.MainForm`

Files: `src/LevelManager/UI/MainForm.xaml`, `MainForm.xaml.cs`

WPF `Window` (modeless) implementing `INotifyPropertyChanged`.

### Constructor

```csharp
public MainForm(
    List<LevelModel> levelList,
    CreateLevelEventHandler createLevelEventHandler,
    DeleteLevelEventHandler deleteLevelEventHandler,
    ExternalEvent createLevelExternalEvent,
    ExternalEvent deleteLevelExternalEvent);
```

Wraps `levelList` in an `ObservableCollection<LevelModel>` bound to
`LevelsDataGrid`. Centers on screen, disables the **Add Level** button
until both name and elevation are entered, restricts elevation input to
characters matching `[0-9.-]`, and stores both event handlers + external
events for later use.

### Public properties

| Property | Type | Notes |
|---|---|---|
| `BasePointTypeEnumValues` | `IEnumerable<BasePointType>` | All enum values, bound to the `BasePointComboBox`. |
| `SelectedBasePointType` | `BasePointType` | Two-way bound to `BasePointComboBox.SelectedValue`. |
| `IsAddLevelButtonEnabled` | `bool` | Drives `AddLevelButton.IsEnabled`; raises `PropertyChanged`. |

### Event handlers (private)

- `AddLevel_Click` — validates `NameTextBox`/`ElevationTextBox`, rejects
  duplicate names, builds a `LevelModel`, sets
  `_createLevelEventHandler.Input` and raises
  `_createLevelExternalEvent`, optimistically adds the new level to
  `LevelList`, then clears the inputs. Catches `FormatException` separately
  for elevation parsing.
- `DeleteRow_Click` — confirms via `MessageBox`, sets
  `_deleteLevelEventHandler.Input`, raises `_deleteLevelExternalEvent`, and
  removes the row from `LevelList`.
- `TextBox_TextChanged` — toggles `IsAddLevelButtonEnabled`.
- `ElevationTextBox_PreviewTextInput` — blocks any character matching
  `[^0-9.-]+`.
- `MyMainForm_Closed` — disposes both `ExternalEvent` instances.

### Static helper

```csharp
public static IEnumerable<BasePointType> BasePointTypeValues();
```

Returns `Enum.GetValues(typeof(BasePointType)).Cast<BasePointType>()`.
Currently unused by the XAML (the instance property
`BasePointTypeEnumValues` is bound instead).
