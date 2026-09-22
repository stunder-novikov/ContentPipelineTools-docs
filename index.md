---
title: Content Pipeline Tools
description: Bake a material subgraph to a texture, batch repeated props into instanced components in one culling bound, merge collision, override bounds.
---

# Content Pipeline Tools

## On this page

- [What each tool is for, in numbers](#what-each-tool-is-for-in-numbers)
- [Requirements](#requirements)
- [Installation](#installation)
- [Material Bake — a texture instead of a subgraph](#material-bake--a-texture-instead-of-a-subgraph)
- [Wrap & Batch — instancing with one culling bound](#wrap--batch--instancing-with-one-culling-bound)
- [Collision Body — one physics body for a cluster](#collision-body--one-physics-body-for-a-cluster)
- [Culling Bounds — bounds that do not follow the geometry](#culling-bounds--bounds-that-do-not-follow-the-geometry)
- [Three utility windows](#three-utility-windows)
- [The example](#the-example)
- [Limitations](#limitations)
- [Support](#support)
- [Licensing](#licensing)


Five editor tools that take work off a content pipeline: baking a material subgraph down to a
texture, turning a field of repeated props into instanced components inside one culling bound,
collapsing a cluster of collision bodies into one, overriding the bounds the engine culls and
streams an actor by, and three utility windows for texture and navmesh chores.

The tools themselves run in the editor and change assets, so what your game loads afterwards is
ordinary engine content. Two of the pieces - the culling-bounds and collision-body components - are
placed on actors and therefore live in your packaged build as well, with a Blueprint API you can use
there if you want to.

## What each tool is for, in numbers

Measured on Unreal Engine 5.8.2 from the launcher, on scenes built by a script rather than placed by
hand, so the conditions below are what the numbers mean. Where a tool costs something, the cost is in
the table too - you should know which side of each trade your content is on before you use it.

| Tool | What it changes | What it costs | Condition |
|---|---|---|---|
| Material Bake | pixel-shader instructions **583 → 268**, texture samples **6 → 1** | one texture asset per bake | a subgraph of three Noise nodes at six octaves, baked to 512x512 in 0.9 s |
| Wrap & Batch | actors **12 006 → 188**, components **12 001 → 728**, sections **12 001 → 729**, draw calls at the camera roughly halved | on this content the batched map measured about **2 ms slower** on the GPU | 12 000 placed props in 80 clusters over two kilometres |
| Collision merge | collision-enabled components **2 001 → 2**, every one of 4 000 rays still hits | a ray that hits the cluster **0.0193 → 0.0682 ms**; a ray that misses is unchanged | 2 000 small collision boxes |
| Culling Bounds | decides what the engine culls and streams an actor by | nothing to measure: it changes correctness, not cost | - |

**Why the middle row says what it says.** UE 5.8 already instances identical static meshes through
GPUScene, so batching a field of repeats does not make the frame faster - it makes the LEVEL smaller,
which is what an outliner, a save, a streaming cell and a memory budget pay for. On enclosed
room-and-corridor levels a shared culling bound was worse again, two to ten times more primitives drawn,
and that is what the bounds mode below is for. Measure your own level; this page will not pretend for
you.

---

## Requirements

| | |
|---|---|
| Engine | Unreal Engine **5.8** |
| Platform | Windows 64-bit |
| Project type | C++ or Blueprint. The plugin brings its own modules, so a Blueprint-only project is asked to build them once |
| Enabled for you | `EditorScriptingUtilities` and `ProceduralMeshComponent`, both shipped with the engine |

`EditorScriptingUtilities` is marked **Beta** by Epic in 5.8. The label is theirs; the plugin uses it
for two library calls the utility widgets make.

## Installation

1. Install from Fab into the engine, or copy the folder into `<YourProject>/Plugins/`.
2. Open the project and let it build the two modules.
3. **Edit → Plugins → Editor → Content Pipeline Tools** — enabled by default once installed.

Everything below appears in menus you already use. There is no separate window to find.

---

## Material Bake — a texture instead of a subgraph

A `Baked Texture` node placed in a material graph marks the subgraph feeding it. The tool renders
that subgraph into a texture, writes the texture into a texture parameter on the material, and walks
the whole tree of material instances beneath it so each one gets the result too.

**Where it lives**

- the node: right-click in a material graph, *Baked Texture*
- one material: right-click the asset in the Content Browser, **Texture baker → Bake textures**
- the whole project: **Build → Texture processing → Rebuild baked textures**
- settings: **Project Settings → Material Bake → General**

**The node's inputs.** The mode decides how many pins it has and what they mean: an RGBA float4, a
normal packed into red and green with two spare channels, two normals packed into one texture, and
single-channel variants. The output is a texture object, or a sampled colour when *Sampler Output*
is on, which is what you want when the node feeds Base Color directly.

**What it writes.** `Parameter Name` is the texture parameter it creates and fills, so a material
instance can override it later. `Default Size` is the resolution, unless you connect a texture to
`Size Reference`, in which case the bake matches that texture's size.

**Reuse by content, not by name.** Every baked texture carries a hash of its pixels as an asset tag.
A second material that bakes to the same image is pointed at the texture that already exists rather
than producing a copy, and the match is confirmed by comparing the images, so a hash collision
cannot hand back the wrong picture. The tags are read back through the asset registry at startup, so
the reuse works across sessions and across a whole library.

**Where the textures land.** `/Game/BakedTextures`, in a subfolder named after the material's own
folder, as `<Material>_<Parameter>_<Hash>`. The hash in the name is what keeps a rebake from
overwriting a texture another material is still using: change the material, and the next bake writes
a new asset instead of quietly changing the old one under everybody's feet. Both the folder and the
name template are settings.

**What it saves.** On a subgraph of three Noise nodes at six octaves the material went from 583
pixel-shader instructions and six texture samples to 268 and one - 54 % of the pixel shader. Adding
octaves grows the first number and leaves the second alone: after the bake the material samples one
texture, whatever fed it. The example inside this plugin is exactly that pair of materials, so the
comparison is one you can repeat. A material with an unbaked node reports the cheap number already,
because the node samples its placeholder texture until the bake fills a real one in - so judge a bake by
the texture it produced, not by the statistics panel.

**What it does not save.** Bake something cheap and you pay for it: a gradient of two constants mixed by
a texture coordinate measured 268 instructions before the bake and 268 after, with one sampler and one
texture added. The tool is for subgraphs that cost real pixel work and never change - noise, layered
masks, expensive blends - not for tidying a graph up.

**Worth knowing.** The bake runs on the editor's tick and reports progress in the corner. A material
that cannot compile is refused with the compiler's own message rather than waited on. By default the
tool shows the list of assets it is about to touch before it starts; turn that dialog off in the
settings for unattended runs.

## Wrap & Batch — instancing with one culling bound

An extra tab inside the engine's own **Merge Actors** window. It turns a selection into a single
actor carrying instanced static mesh components, wrapped in one culling bound.

**Bounds mode is the setting that decides whether this helps.**

- *Shared wrapper bounds* (the default) culls the whole batch in one test. That is the point of the
  wrapper in an open level: a distant cluster is rejected once and the engine never looks inside it.
- *Per-component bounds* leaves every instanced component its own box, so per-component frustum and
  occlusion culling keep working.

Enclosed levels need the second. Measured on room-and-corridor maps, a shared bound drew two to ten
times more primitives and cost up to half again as much GPU scene time, because the whole cell is
drawn the moment any corner of it is visible. The same batch in per-component mode measured within
run-to-run noise. Pick by the shape of your level, not by the default.

**What it saves, and what it does not.** A field of 12 000 placed props becomes 188 actors and 728
instanced components holding the same instances, material sections drop from 12 001 to 729, and draw
calls at the camera roughly halve. The frame does not get faster on that content: measured as a game, the
batched field cost about two milliseconds more on the GPU, because UE 5.8 already instances identical
static meshes on its own. Use this where the count is the problem - a level that has become slow to open,
save or stream - and measure before you use it for anything else.

**It is extensible from Blueprint.** `UWrappedInstancingBuilder` is a Blueprintable class with five
events: `ShouldHarvest` (refuse a component), `GetActorClassToUse`, `InitializeActor`,
`InitializeComponent` and `FinalizeActor`. Subclass it, set it as the builder in the tool's
settings, and the merge asks your rules about every component it is about to take.

**`ReferenceAwareMeshBuilder`** is that hook, already written: name the assets that refer to placed
actors — quest data, a level Blueprint, a data table — and any actor they point at is left alone
instead of being batched away and silently breaking the reference. The same builder can collapse the
batch's collision into a single body, described next.

**From a script.** *Wrap And Batch Actors* is a Blueprint and Python callable that performs the same
merge without the window, so a pipeline can batch a level unattended.

## Collision Body — one physics body for a cluster

`UMergedCollisionBodyComponent` gathers the collision shapes of many components into one aggregate
body, so twenty props that answered a trace separately become one body that answers the same way.

It only merges what is safe to merge: shapes are taken from components whose collision profile
matches, and the odd one out keeps its own body rather than being folded into an answer it never
gave. Instanced components contribute one shape per instance, at the instance transforms.

**What it saves, and what it costs.** 2 000 small collision boxes become 2 collision-enabled components
holding the same 2 000 shapes, and every one of 4 000 test rays that hit before still hits. A ray that
misses the cluster costs what it always did. A ray that hits it costs three and a half times more
(0.0193 ms to 0.0682 ms), because one aggregate body is tested shape by shape where the broadphase used
to reject two thousand separate bodies.

So this is for clutter you count in thousands and do not trace every frame - not for the floor the player
walks on. Measured on a map carrying hundreds of such actors, the difference did not show up in a frame
at all.

## Culling Bounds — bounds that do not follow the geometry

`UCullingBoundsComponent` decides the box the engine culls and streams an actor by, instead of
inheriting it from the geometry. Two ways to use it:

- **Override**: type a box, in Local, World or Actor space. The space matters when the actor moves —
  a World box stays where you put it, a Local one travels with the actor.
- **Capture**: press *Use Child Bounds* and the component gathers the box from everything attached
  to it, with an expansion margin and filters for non-colliding and editor-only components.

`ACullingBoundsWrapperActor` is a ready-made actor with the component as its root and a billboard so
you can find it in a busy level. Wrap & Batch spawns exactly this actor for each batch.

## Three utility windows

**Tools → Content Pipeline Tools** opens them:

| | |
|---|---|
| **Set Texture Channels** | assemble a texture from single channels of others: pick the source for R, G, B and A and write the result |
| **Draw Texture** | paint into a render target laid over a level through a plane, and save the result as a texture |
| **Bake Possible NavMesh To Texture** | rasterise where the navigation mesh reaches into a texture, for tools that need to ask "is this point walkable" cheaply |

They are Editor Utility Widgets, so you can duplicate one and change it for your own pipeline.

The two library functions the navmesh tool needs are exposed to Blueprint as well: *Pixels To Image
(Texture2D)* builds a texture from an array of colours, and *Image (Texture2D) To Bytes* encodes a
texture's source art to PNG, JPEG, BMP or EXR.

**Two migration helpers, and a warning about them.** The widgets above began life in UE 4.24, and two
functions written to carry them forward are callable from your own scripts under *Content Pipeline
Tools*:

| | |
|---|---|
| *Repair Widget Variable Writes* | finds Set nodes writing to a designer widget variable, which UE 5 forbids and which keep an old Widget Blueprint from compiling |
| *Retarget Deprecated Editor Scripting Calls* | re-points calls to the removed Editor Scripting Utilities functions at the stand-ins this plugin provides |

Both take a `bApply` argument: with it **false** they only count and log what they found, which is how
to use them first. With it true they edit the Blueprint and recompile it. They change somebody's asset,
so run them on a checked-in tree and read the log before believing the result.

---

## The example

`Content/Example` inside the plugin:

- `Maps/L_ContentPipelineToolsExample` — three clusters of forty repeated props, a cluster of twenty
  small collision boxes, and one assembly already wrapped in a culling bound.
- `Materials/M_ExampleSubgraph` — three Noise nodes at six octaves wired straight into Base Color:
  the material as an artist would leave it, and the **before** half of the comparison.
- `Materials/M_ExampleBake` and `MI_ExampleBake` — the same subgraph behind a `Baked Texture` node,
  and an instance under it. Deliberately **not** baked: pressing
  **Build → Texture processing → Rebuild baked textures** is the demonstration, and the texture
  appears in `/Game/BakedTextures`.

**The bake, in the engine's own numbers, on the content you have.** Open `M_ExampleSubgraph`, turn on
**Window → Stats**: 583 pixel-shader instructions and 6 texture samples. Open `M_ExampleBake` after the
rebuild: 268 and 1. Those are the figures this page quotes, measured on these two assets rather than on
something you cannot open.

To try Wrap & Batch: open the map, select one cluster, open Merge Actors, choose the Wrap & Batch
tab, and watch the outliner.

**To put numbers on your own level** rather than on this one: the outliner's count and
`stat scenerendering` (primitives drawn, draw calls) before and after a batch, and `ProfileGPU` for
the frame. Do that before you batch anything you cannot undo — the middle row of the table at the top
of this page is what happens when the frame was never the problem.

## Limitations

- **The tools are editor-only.** Baking, batching, collision merging and the three windows all run
  in the editor and change assets; none of them executes while your game runs.
- **The two components do have a runtime API.** `UCullingBoundsComponent` and
  `UMergedCollisionBodyComponent` live in a Runtime module and expose Blueprint functions that work
  in a packaged build: set or capture a bound, gather collision from an actor, count the shapes.
  Building
  collision at runtime cooks physics data, which costs time on the frame it happens.
- **Unreal Engine 5.8, Windows 64-bit.** Every declared version and platform owes a full
  verification cycle, so support is claimed only where it has been run.
- **Baking needs a live editor.** It advances on the editor tick, so a commandlet cannot drive it.
- **Wrap & Batch with a shared bound can cost more than it saves** in enclosed levels. See the
  bounds mode above; the setting exists because of it.
- **Collision merging only folds together what shares a collision profile.** Anything else keeps its
  own body, by design.

## Support

Questions, bug reports and requests: the support link on the product page. Include the engine
version and, for a bake problem, the log — the tools name the material and quote the compiler when
they refuse to do something.

## Licensing

The plugin bundles no third-party libraries. Twelve of its files are derived from Unreal Engine's own
editor sources, because the Wrap & Batch tab is built from the panels and the tool interface the
engine's own merge tools use, and the baked-texture node's details panel from the customisation the
engine applies to its texture expressions. Each of the twelve keeps Epic's copyright line with ours
beneath it, and `THIRD_PARTY_LICENSES.md` beside this file lists every one against the engine file it
came from.
