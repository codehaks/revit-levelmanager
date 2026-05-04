# User Manual

This manual reflects the UI as defined in
`src/LevelManager/UI/MainForm.xaml` and the behavior in `MainForm.xaml.cs`
and `Api/LevelApiController.cs`.

## Launching the tool

1. Open Revit 2024 with a project loaded.
2. On the ribbon go to **Add-Ins → Levels → Manager**.
   The button image is the LevelManager logo
   (`Resources/logo_16.png` / `logo_32.png`).
3. The window **Level Manager v1.0.0** opens centered on screen
   (400 × 600, non-resizable). Existing project levels populate the grid
   immediately.

## Window layout

Top input panel:

| Label | Control | Source |
|---|---|---|
| **Name:** | `NameTextBox` (max 100 chars) | Free text. |
| **Elevation:** | `ElevationTextBox` (digits, `.`, `-` only) | Numeric, parsed as `double`. |
| **Base Point Type:** | `BasePointComboBox` | Values: `Project`, `Shared`. |
| — | `AddLevelButton` ("Add Level") | Disabled until both Name and Elevation contain text. |

Bottom grid `LevelsDataGrid` columns (read-only):

| Column | Source |
|---|---|
| **Name** | `LevelModel.Name` |
| **Elevation** | `LevelModel.Elevation.SimpleValue` (rounded to 2 decimals) |
| **Base Point** | `LevelModel.BasePointType` (`Project` / `Shared`) |
| **Actions** | Per-row **Delete** button |

## Creating a level

1. Type a unique name into **Name**.
2. Type an elevation into **Elevation**. Only digits, `.`, and `-` are
   accepted; other keystrokes are silently ignored.
   The value is sent to Revit's `Level.Create` as-is, in Revit internal
   units (decimal feet).
3. Pick **Base Point Type** (`Project` or `Shared`).
4. Click **Add Level**. The new row appears in the grid and the input
   fields are cleared.

## Deleting a level

1. Click the **Delete** button on the row you want to remove.
2. A confirmation dialog asks
   *"Are you sure you want to delete the level '&lt;name&gt;'?"*.
3. Click **OK** to delete or **Cancel** to keep it.

## Closing the window

Click the window's close button. Both `ExternalEvent` instances are
disposed in `MyMainForm_Closed`. Any data already committed to the Revit
document remains; nothing is rolled back.

## Messages you may see

All strings come from `MainForm.xaml.cs` and `LevelApiController.cs`.

| Message | Where | Meaning / fix |
|---|---|---|
| `Level data not valid` | Add Level click, Name or Elevation empty | Fill in both fields. |
| `Level name already exists` | Add Level click, name collides with a row in the grid | Choose a different name. |
| `Invalid elevation format. Please enter a valid number.` | Add Level click, `Convert.ToDouble` failed | Enter a valid number (e.g. `12.5`, `-3`). |
| `An error occurred: <message>` | Add Level click, generic exception | Read the inner message; check the Revit journal / Visual Studio Debug output. |
| `Please select a row to delete.` | Delete with no row selected | Click a row first. [REVIEW REQUIRED: code only shows the message — it then continues and casts `null` to `LevelModel`, which would `NullReferenceException` if `SelectedItem` is truly null at that point.] |
| `Level not found: <name>` (`MessageBox`) | Backend lookup failed inside `LevelApiController.Delete` | The level was already removed in Revit; refresh by closing and reopening the window. |
| `Invalid Operation` / `Could not delete level <name>` (`TaskDialog`) | Revit refused the delete | Likely the level is referenced (views, datums). Resolve dependencies in Revit and retry. |

## Limitations / gotchas

- The grid is **not** refreshed after a Revit-side transaction failure;
  optimistic UI may temporarily diverge from the document.
- Elevation is interpreted in Revit internal units (decimal feet). There
  is no unit conversion or display formatting beyond two-decimal rounding.
- The window is modeless: you can keep using Revit while it is open. Each
  Add/Delete is committed in its own transaction.
