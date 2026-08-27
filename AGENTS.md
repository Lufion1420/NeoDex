# NeoDex — Agent Guide

NeoDex is a **3ds Max application plugin** (versions 2022–2027) for authoring
Warcraft III models: import/export of the MDX (binary) and MDL (text) formats,
skeletal animation tooling, materials/team-color handling, particle/ribbon
emitters, collision shapes, and a suite of scene-cleanup utilities. It is
almost entirely written in **MAXScript** (`.ms` / `.mcr`), with a small
compiled native helper for MPQ/CASC archive reading and BLP texture I/O.

This is a fork of `DennisHerrm/NeoDex` (`git remote -v` shows `origin` =
this fork, `upstream` = the original). Sync from upstream with
`git fetch upstream && git merge upstream/main` (check the actual default
branch name first).

There is **no build system, no package manager, and no automated test
suite**. "Running" the code means loading it into 3ds Max. Read
`.claude/skills/neodex-deploy/SKILL.md` before trying to test a change.

## Repo layout

Everything the plugin ships lives under `NeoDex/` (this inner folder *is*
the Autodesk "ApplicationPlugins" bundle — see Architecture below).

| Path | Contents |
|---|---|
| `NeoDex/PackageContents.xml` | The manifest. Declares every file that gets loaded and in what order/version range. **Nothing loads unless it's listed here.** |
| `NeoDex/pre-start-up scripts parts/` | Core engine: globals, utilities, the MDX/MDL reader/writer, scene parser/rebuilder, localization. Loaded first, in the exact order listed in the manifest. |
| `NeoDex/scripted plugins parts/` | Custom scene-object plugin classes (`Wc3AttachPoint`, `Wc3Light`, `Wc3Material`, `Wc3CollisionShapes`, `Wc3VertexColor`, `Wc3RefEvent`, `BlizzRibbon`, `FaceFX`, `Popcorn`, `BlizzPart1/2`). These `plugin ... (...)` definitions must exist before scene objects of that class can be created or deserialized. |
| `NeoDex/post-start-up scripts parts/` | UI-facing tools that assume the engine is ready: Sequence/GlobalSequence Manager, Node Manager, Team Color Manager, Skin Changer, Visibility Keyer, Object Settings, the sidebar, the auto-updater, the pre-2025 menu installer. |
| `NeoDex/macroscripts parts/*.mcr` | The macroScript entry points 3ds Max exposes as toolbar/menu actions (Importer, Exporter, Manager, Settings, About, KeyframeOptimizer, CellShadeCreator). UI rollouts live here; the underlying logic lives in the matching `pre-start-up` file (e.g. import logic is in `NeoDexImportFunctions.ms`, not in the `.mcr`). |
| `NeoDex/extra tools/` | Standalone-ish tools (`Keyframe_Optimizer.ms`, `Cell_Shade_Creator.ms`) wired into the menu/sidebar but kept separate from the core. |
| `NeoDex/native plugins/Max<year>/` | Prebuilt, **vendored binaries** — `blp.bmi` (native BLP bitmap I/O) per Max version 2022–2027, and `NeoDexNative.dll` (MPQ/CASC + BLP/DDS decoding) in two flavors: `Max2022-2025` (.NET Framework 4.8.1) and `Max2026+` (.NET 8, ships `Ijwhost.dll`). **No source for these is in this repo** — don't try to "fix" them here; they're loaded via `dotNet.loadAssembly` from `NeoDexNativeHelper.ms`. |
| `NeoDex/neodex_icons/` | PNG/BMP icons for the sidebar, spinner curves, and social links in the About dialog. |
| `NeoDex/maps/Neodex/TeamGlow/` | Team-color glow textures shipped as model assets, not code. |
| `NeoDex/WhiteoutTexCLI.exe` | A separate, prebuilt CLI tool (BLP↔image conversion, with AI upscaling via ncnn/Real-ESRGAN) used as a fallback where the native BLP plugin isn't available. Its source is a different project; only `THIRD_PARTY.md` license notices and the binary live here. |
| `NeoDex/NeoDex_Settings.ini` | **Runtime user state** (UTF-16), not really source — active language, sidebar dock state, updater last-check timestamp. Tracked in git already; be careful about hand-editing it or treating diffs to it as meaningful. |

Two files are stray backups accidentally left in the tree and are **not**
referenced by `PackageContents.xml`: `NeoDex/post-start-up scripts parts/NodeManager - Kopie.ms`
and `NeoDex/PackageContents - Kopie.xml` ("Kopie" = German for "copy"). Don't
edit these thinking they're live — check the manifest if unsure whether a
file is actually loaded.

## Architecture: the manifest drives everything

`PackageContents.xml` loads components in five phases, and **order within
each `<Components>` block matters** because later files reference structs/
globals defined by earlier ones:

1. **Native plugins parts** (per-Max-version `<RuntimeRequirements>` blocks) — `blp.bmi`.
2. **Pre-start-up scripts parts** — one big ordered list, `NeoDexGlobals.ms` *must* be first (defines `NeoDexHasNativeBLP` and the utility/material/renderer structs everything else needs), then the scripted-plugin class definitions, then the engine files (IO, MDX reader/writer, scene parser/rebuilder, localization, native helper).
3. **Menu registration** — version-gated: `NeoDexMenuCallback.ms` (2025–2027, CUI `cuiRegisterMenus` callback) vs `NeoDexMenuInstaller.ms` (2022–2024, `menuMan` API). **Both must be kept in sync** when adding/removing a top-level menu item — see `.claude/skills/neodex-add-tool/SKILL.md`.
4. **Post-start-up scripts parts** — UI tools and the sidebar.
5. **Macroscripts parts** — the `.mcr` toolbar actions.

If you add a new `.ms`/`.mcr` file, it **will not load** until you add a
matching `<ComponentEntry ModuleName="...">` in the right `<Components>`
block, in the right position relative to its dependencies.

## Core singleton globals

Almost every subsystem is a MAXScript `struct` instantiated once into a
global with a short name, then accessed everywhere via `::Name.method args`
(the `::` forces global scope lookup from inside rollouts/callbacks). This
is the map you need before grepping around:

| Global | Defined in | Purpose |
|---|---|---|
| `::L` | `NeoDexLocalization.ms` | `::L.t "key"` → localized string (en/zh/de/ru/ja/ko) |
| `::ErrorReporter` | `ErrorReporter.ms` | Central logging + user-facing error/warning report dialog |
| `::Utils` | `UtilityFunctions.ms` | Misc general-purpose helpers |
| `::NeoDexUtils` | `NeoDexGlobals.ms` | Base64, HSB color, CLI exec, file helpers |
| `::MaterialManager` | `NeoDexGlobals.ms` | Team colors, material tracking, callbacks |
| `::Wc3Renderer` | `NeoDexGlobals.ms` | Ribbon rendering, lighting presets |
| `::SequenceStorage` | `NeoDexGlobals.ms` | Sequence data on custom attributes |
| `::IOManager` | `NeoDexIO.ms` | Low-level MDX byte reader/writer + all chunk tag constants (`tag_*`) |
| `::NeoDexMDXReader` | `NeoDexMDXReader.ms` | Classic MDX binary → in-memory model |
| `::NeoDexMDXReaderV1200` | `NeoDexReaderReforge1200.ms` | Reforged v1200 MDX variant (adds `TANG`/`SKIN`/`CORN`/`FAFX`/`BPOS` chunks) |
| `::MDXWriter` | `NeoDexMDXWriter.ms` | In-memory model → MDX binary |
| `::MDLWriter` | `NeoDexMDLWriter.ms` | In-memory model → MDL text (**export only — there is no MDL reader**; import is MDX-only) |
| `::SceneParser` | `NeoDexSceneParser.ms` | 3ds Max scene graph → `Wc3Model` |
| `::SceneRebuilder` | `NeoDexSceneRebuilder.ms` | `Wc3Model` → 3ds Max scene graph |
| `::GeoUtils` | `GeometryUtilities.ms` | Mesh/vertex math |
| `::Progress` | `ProgressUtility.ms` | Progress bar UI wrapper |
| `::Optimizer` | `NeoDexOptimize.ms` | Keyframe/mesh optimization |
| `::ModelRelinker` | `NeoDexModelRelinker.ms` | Re-pointing texture/reference paths |
| `::NeoDexMPQ` / `::NeoDexCASC` | `NeoDexNativeHelper.ms` | MPQ/CASC archive access via `NeoDexNative.dll` |
| `::NeoDexTexBrowser` | `NeoDexNativeHelper.ms` | Texture browser backed by the archive helpers |
| `::NeoDexMenu` | `NeoDexMenuCallback.ms` | 2025+ menu bar registration |
| `::AnimationManager` | `Wc3Animation.ms` | Animation controller helpers |
| `::TeamColorManager`, `::SkinChanger`, `::SequenceManager`, `::GlobalSequenceManager`, `::NodeManager`, `::ObjectSettings`, `::ObjectManipulationTools`, `::AnimTools`, `::GridAndDummyCreator`, `::VisibilityKeyer` | respective `post-start-up` files | Each is one UI tool window's backing struct |

`Wc3Model.ms` defines the plain-data structs the reader/writer/parser all
share (`Wc3Sequence`, `Wc3Texture`, `Wc3Layer`, `Wc3Geoset`, `Wc3Bone`, etc.)
— read it first when working on anything format-related. Most structs carry
an `ex` field reserved for extra/future data that doesn't fit the standard
schema.

## Coding conventions (match the existing style)

- **Indentation**: tabs, not spaces (mixed with some post-refactor files using spaces — match whatever the surrounding file already uses).
- **File header comment**: a `/* Name.ms  ===  one-line purpose, usage example, load-order note */` block at the top of most core files.
- **Section banners** inside long files: `/* ============ SECTION N: NAME ============ */`.
- **Singleton pattern**: `struct FooStruct ( ... )` followed by `global Foo = FooStruct()` at the bottom of the file — keep new subsystems consistent with this rather than inventing a new pattern.
- **Error handling**: wrap risky calls in `try (...) catch (format "⚠ NeoDex [FnName]: %\n" (getCurrentException()))`. For user-facing/scene-validation issues (not code bugs), report through `::ErrorReporter.logError/logWarning/logInfo category object message` and the category must first exist via `InitiateErrorCategory` in `ErrorReporter.ms`.
- **No hardcoded UI strings**: any label/tooltip/message shown to the user should go through `::L.t "key"`. Adding a string means adding the key to **every** language section in `NeoDexLocalization.ms` (English is the required fallback; others can be left as `""` to fall back to English — never leave English blank).
- **.NET interop**: `dotNetClass "Namespace.Class"`, `dotNetObject "Namespace.Class" args`, `dotNet.loadAssembly path`. Native archive/image calls go through `::NeoDexNativeDLL` (the loaded `NeoDex.NeoDexNative` class), not raw interop, to stay version-agnostic between the .NET Framework and .NET 8 DLL builds.
- Comments are a mix of English and German (original author is German-speaking) — don't feel obligated to translate existing ones, just don't rely on non-English comments existing for anything essential.

## MDX/MDL format notes

- MDX is Blizzard's binary chunked format: 4-byte FourCC tag + 4-byte size, repeated. Every chunk tag is a `global tag_XXXX = 0x...` constant in `NeoDexIO.ms` — that file is the authoritative tag reference (has inline comments naming each chunk).
- MDL is the human-readable text equivalent; this codebase only **writes** MDL (`NeoDexMDLWriter.ms`), it does not read it.
- Reforged's "v1200" MDX variant adds chunks (`TANG` tangents, `SKIN` binding, `CORN` popcorn/corn emitters, `FAFX` FaceFX, `BPOS` bind pose) handled by the separate `NeoDexReaderReforge1200.ms` reader rather than branching inside the classic reader.
- Round-trip flow: `NeoDexMDXReader`/`V1200` → `Wc3Model` struct → `::SceneRebuilder` → 3ds Max scene (import). `::SceneParser` → `Wc3Model` → `::MDXWriter`/`::MDLWriter` → file (export).

## Testing a change

There's no CI and no automated tests — validation is manual, inside 3ds Max.
See `.claude/skills/neodex-deploy/SKILL.md` for the deploy/reload workflow
before claiming a MAXScript change works. Do not report a fix as verified
unless it was actually reloaded and exercised in Max (or the user confirms
it).

## Skills available in this repo

- `neodex-deploy` — get a local checkout live-testable inside 3ds Max (symlink/junction into `ApplicationPlugins`, reload without restarting Max where possible).
- `neodex-add-tool` — checklist for wiring up a brand-new tool/macroscript end-to-end (logic file, `.mcr` UI, manifest entries in both load-order sections, both menu-registration paths, sidebar button, localization keys) so nothing is silently half-registered.
- `wc3-mdx-format` — MDX/MDL chunk-tag and data-model quick reference for format-level bug fixes.
