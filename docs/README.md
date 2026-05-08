# LevelManager — Developer Documentation

This folder contains the engineering documentation for **CodeHaks LevelManager**. The user-facing entry point lives in the repository [README.md](../README.md); this index is for contributors and maintainers.

## Index

| Document | Audience | Purpose |
| :--- | :--- | :--- |
| [architecture.md](architecture.md) | New contributors | Layered design, namespaces, Revit interop & threading notes. |
| [build-and-load.md](build-and-load.md) | Anyone building locally | Per-version build matrix, `build.bat`, addin install workflow. |
| [developer-guide.md](developer-guide.md) | Active contributors | Daily workflow, code map, debugging tips. |
| [testing.md](testing.md) | Contributors adding tests | How to introduce unit tests and Revit-integration tests. |
| [roadmap.md](roadmap.md) | Maintainers, planners | Suggested improvements and future features. |
| [contributing.md](contributing.md) | External contributors | Branching, commit style, review expectations. |

## Conventions

- All paths in these docs are written relative to the repository root.
- "the shared project" = [src/Shared/](../src/Shared/).
- "per-version csprojs" = [src/LevelManager 2024/](../src/LevelManager%202024/), [src/LevelManager 2025/](../src/LevelManager%202025/), [src/LevelManager 2026/](../src/LevelManager%202026/).
- The C# root namespace is `LevelManager` (the assembly is `LevelManager.dll`).
