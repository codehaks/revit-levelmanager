# Contributing

Thanks for taking an interest in LevelManager. This document is short on purpose; ask if anything's unclear.

## Setup

1. Install Revit 2024, 2025, and/or 2026 (you only need the years you intend to test against).
2. Install Visual Studio 2022 17.8+ with the **Desktop development with .NET** workload, or MSBuild 17+ on PATH.
3. Install the .NET 8 SDK.
4. Clone the repo and run `build.bat Debug` to verify everything resolves.

## Branching

- `master` — always builds; should reflect the most recent published version.
- Feature branches off `master`: `feature/<short-name>` or `fix/<short-name>`.

## Commits

- One logical change per commit.
- Commit subject in imperative mood: *"Add X"*, *"Fix Y"*, *"Refactor Z"*.
- Reference the issue number where relevant.

## Pull requests

Before opening a PR, please:

1. **Build all three versions** (`build.bat`) — the shared project means a typo in one place breaks every assembly.
2. **Smoke-test in Revit** for at least one version. Note in the PR description which Revit you tested.
3. **Update [Shared.projitems](../src/Shared/Shared.projitems)** if you added or moved source files.
4. **Update docs** if you changed the public surface (ribbon, commands, manifest, dependencies).

## Adding a new feature

If your feature involves Revit document mutations, follow the existing pattern:

- New `Transaction` lives in `LevelApiController` (one transaction per logical action).
- New `IExternalEventHandler` lives in `Api/EventHandlers/`.
- The WPF code-behind raises the external event after setting the handler's `Input`.

See [developer-guide.md](developer-guide.md#adding-a-new-mutating-action) for the full recipe.

## Reporting bugs

GitHub Issues. Please include:

- Revit version (2024 / 2025 / 2026).
- Build configuration (`Debug` / `Release`).
- Steps to reproduce.
- Expected vs actual behaviour.
- Stack trace or `Debug.WriteLine` output if available.
