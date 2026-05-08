# Build & Load

This document covers building the per-version assemblies and loading them into Revit.

## Revit version matrix

| Revit | Target framework | csproj | Output DLL path |
| :--- | :--- | :--- | :--- |
| 2024 | `.NET Framework 4.8` | [src/LevelManager 2024/LevelManager 2024.csproj](../src/LevelManager%202024/LevelManager%202024.csproj) | `src\LevelManager 2024\bin\<Config>\LevelManager.dll` |
| 2025 | `net8.0-windows` | [src/LevelManager 2025/LevelManager 2025.csproj](../src/LevelManager%202025/LevelManager%202025.csproj) | `src\LevelManager 2025\bin\<Config>\net8.0-windows\LevelManager.dll` |
| 2026 | `net8.0-windows` | [src/LevelManager 2026/LevelManager 2026.csproj](../src/LevelManager%202026/LevelManager%202026.csproj) | `src\LevelManager 2026\bin\<Config>\net8.0-windows\LevelManager.dll` |

`<Config>` is `Debug` or `Release`. All three build outputs share the assembly name **`LevelManager.dll`**.

## Prerequisites

- Windows 10 / 11, x64.
- **Visual Studio 2022 17.8+** *or* MSBuild 17+ on PATH. .NET Framework 4.8 build tools must be present (installed via the "Desktop development with .NET" workload, or the .NET Framework 4.8 targeting pack).
- **.NET 8 SDK** (`dotnet --list-sdks` should show 8.x).
- The Revit versions you want to target installed at the standard locations:
  - `C:\Program Files\Autodesk\Revit 2024\`
  - `C:\Program Files\Autodesk\Revit 2025\`
  - `C:\Program Files\Autodesk\Revit 2026\`

If a referenced Revit version isn't installed, that csproj will fail to resolve `RevitAPI.dll` / `RevitAPIUI.dll`. Either install the matching Revit release, or unload that project from the solution.

## Build

### One-shot (recommended)

```bat
build.bat              :: Release (default)
build.bat Debug        :: Debug
```

[build.bat](../build.bat) builds all three csprojs sequentially and copies each `LevelManager.dll` (and `.pdb`, `.addin`) into `dist\Revit <year>\` so you don't have to chase three different output paths.

Layout after a successful run:

```text
dist/
├── Revit 2024/
│   ├── LevelManager.dll
│   └── LevelManager.addin
├── Revit 2025/
│   ├── LevelManager.dll
│   └── LevelManager.addin
└── Revit 2026/
    ├── LevelManager.dll
    └── LevelManager.addin
```

### Whole solution

```bat
msbuild LevelManager.sln /p:Configuration=Release
```

Builds all three projects but leaves outputs in their respective `bin\<Config>\…\` folders.

### Single target

```bat
msbuild "src\LevelManager 2024\LevelManager 2024.csproj" /p:Configuration=Release
msbuild "src\LevelManager 2025\LevelManager 2025.csproj" /p:Configuration=Release
msbuild "src\LevelManager 2026\LevelManager 2026.csproj" /p:Configuration=Release
```

The 2025 / 2026 projects can also be built with `dotnet build`:

```bat
dotnet build "src\LevelManager 2026\LevelManager 2026.csproj" -c Release
```

(`dotnet build` will *not* build the 2024 project — `.NET Framework 4.8` requires `msbuild`.)

## Load into Revit

1. Locate Revit's per-user addin folder for the target year:
   - `%AppData%\Autodesk\Revit\Addins\2024\`
   - `%AppData%\Autodesk\Revit\Addins\2025\`
   - `%AppData%\Autodesk\Revit\Addins\2026\`
2. Copy both files from `dist\Revit <year>\` into that folder:
   - `LevelManager.dll`
   - `LevelManager.addin`
3. The shipped `.addin` uses `<Assembly>LevelManager.dll</Assembly>` (relative path), so the manifest resolves the DLL from the same folder — no editing required.
4. Launch Revit, open a project. The **Levels** ribbon panel with a *Manager* button appears under the **Add-Ins** tab.

## Troubleshooting

| Symptom | Likely cause |
| :--- | :--- |
| `error MSB3245: Could not resolve this reference … RevitAPI` | Matching Revit version not installed at the standard path. |
| Ribbon button missing on launch | The `.addin` was copied to the wrong year's folder, or the DLL is blocked (right-click → Properties → Unblock). |
| Add-in fails to load with security prompt | Revit blocks unsigned DLLs from network paths. Install from a local drive. |
| `FileLoadException` for `LevelManager.dll` after a code change | Old DLL is locked by a running Revit instance. Close all Revit windows before rebuilding. |
| 2025 or 2026 build fails resolving WPF | Ensure .NET 8 SDK includes the Windows Desktop runtime (`dotnet --list-sdks`, `dotnet --list-runtimes`). |
