# CodeHaks **LevelManager** for Revit

[![Build](https://img.shields.io/badge/build-msbuild-blue?logo=.net)](docs/build-and-load.md)
[![Revit](https://img.shields.io/badge/Revit-2024%20%7C%202025%20%7C%202026-005CA9?logo=autodesk&logoColor=white)](docs/build-and-load.md#revit-version-matrix)
[![.NET](https://img.shields.io/badge/.NET-Framework%204.8%20%7C%208.0--windows-512BD4?logo=dotnet&logoColor=white)](docs/architecture.md#target-frameworks)
[![Platform](https://img.shields.io/badge/platform-Windows%20x64-0078D6?logo=windows&logoColor=white)](#requirements)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE.txt)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](docs/contributing.md)

**CodeHaks LevelManager** is a Revit add-in that lists every `Level` in the active document and lets the user create or delete levels from a modeless WPF window. Document mutations run inside Revit transactions via the `IExternalEventHandler` bridge, so the dialog stays modeless and Revit-API-thread-safe.

The codebase is built around a single shared-source project that is compiled into three per-Revit-version assemblies — one for Revit 2024 (.NET Framework 4.8) and one each for Revit 2025 / 2026 (.NET 8 on Windows).

---

## Highlights

- **Modeless WPF window** — lists existing levels (Name, Elevation, Base Point), with input validation for name and elevation.
- **Transactional mutations** — `Create Level` and `Delete Level` run inside their own `Transaction`, marshalled to Revit's API thread by `ExternalEvent`.
- **Ribbon integration** — adds a *Levels* panel with a *Manager* button (16/32-px branded icons) under the Add-Ins tab.
- **Multi-version build** — one shared project (`src/Shared/`), three thin per-version csprojs that differ only in target framework and Revit reference paths.
- **Single one-shot builder** — [`build.bat`](build.bat) compiles all targets and collects DLL + `.addin` into `dist/Revit <year>/`.

---

## Requirements

| | |
| :--- | :--- |
| **OS** | Windows 10 / 11, x64 |
| **Revit** | 2024, 2025, or 2026 (installed at `C:\Program Files\Autodesk\Revit <year>\`) |
| **.NET (build-time)** | .NET Framework 4.8 SDK *and* .NET 8 SDK |
| **Build tools** | MSBuild 17+ / Visual Studio 2022 17.8+ |

See [docs/build-and-load.md](docs/build-and-load.md) for the full version matrix.

---

## Quick start

### Build everything

```bat
build.bat              :: Release into dist\Revit 2024|2025|2026\
build.bat Debug        :: Debug build
```

Or build a single target:

```bat
msbuild "src\LevelManager 2026\LevelManager 2026.csproj" /p:Configuration=Release
```

### Load into Revit

1. Copy `dist\Revit <year>\LevelManager.dll` and `dist\Revit <year>\LevelManager.addin` into:
   `%AppData%\Autodesk\Revit\Addins\<year>\`
2. Launch Revit `<year>` and open a project.
3. Click **Add-Ins ▸ Levels ▸ Manager** on the ribbon.

Full walkthrough: [docs/build-and-load.md](docs/build-and-load.md).

---

## In-Revit commands

| Surface | Description |
| :--- | :--- |
| **Add-Ins ▸ Levels ▸ Manager** | Opens the modeless **Level Manager** window: list, create, and delete levels. |

---

## Documentation

| Document | What's inside |
| :--- | :--- |
| [docs/architecture.md](docs/architecture.md) | High-level design, layering, namespaces, Revit interop & threading notes. |
| [docs/build-and-load.md](docs/build-and-load.md) | Build pipeline, version matrix, manifest install instructions. |
| [docs/developer-guide.md](docs/developer-guide.md) | Day-to-day development workflow, code map, debugging in Revit. |
| [docs/testing.md](docs/testing.md) | How to add unit tests and integration tests against Revit APIs. |
| [docs/roadmap.md](docs/roadmap.md) | Future ideas, suggested improvements, technical-debt items. |
| [docs/contributing.md](docs/contributing.md) | Branching, commit conventions, review checklist. |

---

## Support

Open an issue on the GitHub repo, or email **[support@codehaks.com](mailto:support@codehaks.com)**.

---

## License

Released under the [MIT License](LICENSE.txt).

---

**CodeHaks LevelManager** — Manage Revit levels without leaving Revit.
