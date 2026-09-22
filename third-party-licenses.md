# Third-party licenses

Content Pipeline Tools bundles **no third-party libraries**: nothing is vendored, nothing is
downloaded at build time, and there is no `ThirdParty` folder. It builds against Unreal Engine
modules and nothing else, and it ships no asset-store content, no fonts and no sampled audio.

Two things still need saying plainly.

---

## Code derived from Unreal Engine

Twelve files are derived from Unreal Engine's own editor sources. The plugin adds a tab to the
engine's Merge Actors window and a details panel to its material editor, and both are built the way
the engine builds its own: from the same panels, the same tool interface and the same customisation
shape. Each of the twelve keeps Epic's copyright line at the top with a second line recording that it
has been modified.

| File, under `Source/ContentPipelineToolsEditor/` | Derived from, under `Engine/Source/` |
|---|---|
| `Private/WrappedInstancing/MergeProxyUtils/Utils.h` and `.cpp` | `Editor/MergeActors/Private/MergeProxyUtils/` |
| `Private/WrappedInstancing/MergeProxyUtils/SWrappedProxyCommonDialog.h` and `.cpp` | `Editor/MergeActors/Private/MergeProxyUtils/SMeshProxyCommonDialog.*` |
| `Private/WrappedInstancing/WrappedInstancingTool.h` and `.cpp` | `Editor/MergeActors/Private/MergeActorsTool.*` and `MeshInstancingTool/MeshInstancingTool.*` |
| `Private/WrappedInstancing/SWrappedInstancingDialog.h` and `.cpp` | `Editor/MergeActors/Private/MeshInstancingTool/SMeshInstancingDialog.*` |
| `Private/WrappedInstancing/WrappedInstancingUtilities.cpp` | `Developer/MeshMergeUtilities/Private/MeshMergeUtilities.cpp` |
| `Public/WrappedInstancing/WrappedInstancingSettings.h` | `Runtime/Engine/Public/MeshMerge/MeshInstancingSettings.h` |
| `Private/MaterialBake/MaterialBakeExpressionCustomization.cpp` and `Public/.../MaterialBakeExpressionCustomization.h` | `Editor/DetailCustomizations/Private/MaterialExpressionTextureBaseDetails.*` |

The largest of them, `WrappedInstancingUtilities.cpp`, carries
`FMeshMergeUtilitiesEx::WrapComponentsToInstances` - the engine's `MergeComponentsToInstances` with
the builder extension points cut into it, which is what makes the tool extensible from Blueprint at
all.

This list was produced by comparing every file in the plugin against the engine line by line rather
than by remembering which ones were copied.

The types inside these files were renamed into this plugin's own vocabulary on 2026-09-14, so a
line-by-line comparison with the engine files above now differs by those names as well as by the
changes described here. That renaming is part of what the second copyright line on each file records.


All of this is Unreal Engine source, used under the **Unreal Engine EULA**. Every buyer of this
plugin is an Unreal Engine licensee by definition, and the engine code is not redistributed in any
form other than as part of a plugin that only runs inside the engine.

---

## Unreal Engine modules

The plugin links `Core`, `CoreUObject`, `Engine`, `UnrealEd`, `Slate`, `SlateCore`,
`EditorStyle`-era editor modules, `MaterialEditor`, `MeshMergeUtilities`, `MergeActors`,
`PhysicsCore`, `RenderCore`, `RHI`, `ImageWrapper`, `AssetRegistry`, `Blutility`, `UMG` and
`ProceduralMeshComponent`. They are part of Unreal Engine, used under its EULA, and not
redistributed by this plugin.

The two plugin dependencies declared in the descriptor, `EditorScriptingUtilities` and
`ProceduralMeshComponent`, ship with the engine and are enabled for you when this plugin is
enabled. `EditorScriptingUtilities` is marked **Beta** by Epic in 5.8; the label is theirs.

---

## No obligations to pass on

Nothing above requires a notice to travel with a product built using this plugin. There is no MIT,
BSD or Apache text to reproduce, because no such library is included.
