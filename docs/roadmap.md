# Roadmap

Loose backlog of ideas and known gaps. Not a commitment — listed here so contributors know what's already on the radar.

## Near-term

- **Settle UI updates after transaction commit.** Today the WPF list is updated optimistically; if `LevelApiController.Create` rolls back, the grid will diverge from the document. Drive the grid off a re-collection after each `ExternalEvent` completes (or use the success flag already wired into `DeleteLevelEventHandler`).
- **Unit conversion for elevation.** Inputs go straight into `Level.Create` as Revit internal units (decimal feet). Add a unit-aware text box that respects the document's `DisplayUnitType` / `ForgeTypeId`.
- **Namespace cleanup.** `DeleteLevelEventHandler` currently lives under `LevelManagerApp.Windows`; rename to `LevelManager.Api.EventHandlers` to match its sibling.

## Medium-term

- **Bundle packaging.** Ship a `.bundle` folder with `PackageContents.xml` so users can drop one folder into `%AppData%\Autodesk\ApplicationPlugins\` and pick up all three Revit-version variants automatically.
- **Test project.** See [testing.md](testing.md) for the proposed layout — a small `net8.0` test project covering the `Domain/` POCOs.
- **Localization.** Externalize UI strings ("Add Level", "Delete", error messages) into `.resx` files.

## Long-term

- **Revit 2027 support.** Add `src/LevelManager 2027/` once Autodesk publishes the SDK; the [developer guide](developer-guide.md#adding-a-new-revit-version) describes the steps.
- **CI build.** GitHub Actions matrix: `windows-latest` × {2024, 2025, 2026} → upload `dist/Revit <year>/` as artifacts.
