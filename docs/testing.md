# Testing

LevelManager does not yet ship a test project. This document captures the intended approach.

## Unit-testable surface

The pure parts of the codebase are easy to cover with a standalone test project that does **not** reference `RevitAPI.dll`:

- `Domain/Elevation.cs` — `SimpleValue`, `RoundedBy(int)` rounding behaviour.
- `Domain/LevelModel.cs` — constructor / property invariants.
- `Domain/BasePointType.cs` — enum integer mapping (must stay `Project = 0`, `Shared = 1`).

Suggested layout:

```
tests/
└── LevelManager.Test/
    ├── LevelManager.Test.csproj   (net8.0, references Shared.projitems)
    ├── ElevationTests.cs
    ├── LevelModelTests.cs
    └── BasePointTypeTests.cs
```

Adding the test project to the shared `Shared.projitems` import only picks up the domain files; you can use `<Compile Remove="..." />` to exclude the Revit-coupled files (App, Command, Api/*, UI/*).

## Integration tests against Revit

Revit-API integration tests need to run **inside** a Revit process. Two practical options:

1. **`RevitTestFramework`** (community) — runs xUnit/NUnit tests inside a Revit instance.
2. **Manual smoke tests** — documented step list run before tagging a release. See [build-and-load.md](build-and-load.md#load-into-revit).

For now, the project relies on manual smoke tests against Revit 2024 / 2025 / 2026:

1. Build all three (`build.bat`).
2. Install each into the matching `%AppData%\Autodesk\Revit\Addins\<year>\` folder.
3. Open a sample project; verify the ribbon button.
4. Add a level (e.g. `L99` at `30.0`); confirm it appears in the Project Browser and the grid.
5. Delete that level; confirm it disappears.
6. Close the window — verify Revit does not throw on subsequent commands (catches `ExternalEvent` lifetime bugs).
