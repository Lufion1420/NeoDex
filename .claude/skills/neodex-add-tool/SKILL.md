---
name: neodex-add-tool
description: Checklist for adding a brand-new tool, dialog, or menu action to NeoDex end-to-end — manifest registration, both menu-registration code paths, sidebar button, and localization — so nothing ends up half-wired. Use whenever asked to add a new NeoDex tool, macroscript, menu entry, or sidebar button, or when a newly-added tool "doesn't show up".
---

# Adding a new NeoDex tool without leaving it half-registered

A new tool in this codebase touches up to **seven** separate places. Missing
any one of them makes the tool invisible or silently non-functional with no
error message, which is why this is worth a checklist rather than winging
it. Work through these in order; skip steps that genuinely don't apply
(e.g. a tool with no standalone dialog doesn't need a rollout).

1. **Logic file** — new `.ms` file in `NeoDex/pre-start-up scripts parts/`
   if other tools or the import/export pipeline need to call into it, or
   directly in the macroscript if it's fully self-contained (see
   `NeoDex/extra tools/*.ms` + their `.mcr` wrapper for that pattern).
   Follow the existing convention: `struct FooStruct ( ... )` +
   `global Foo = FooStruct()` at the end, tabs for indent, a header comment
   block naming the file and its load-order dependency.

2. **Macroscript UI** — new `.mcr` in `NeoDex/macroscripts parts/`, named
   `NeoDex Toolkit-NeoDex_<Name>.mcr`, starting with:
   ```
   macroScript NeoDex_<Name>
   buttonText:"<Display Name>"
   category:"NeoDex Toolkit"
   internalCategory:"NeoDex Toolkit"
   (
       ...
   )
   ```
   Keep the rollout/UI in the `.mcr` and delegate real work to the struct
   from step 1, matching how `NeoDex_Importer.mcr` only holds UI while
   `NeoDexImportFunctions.ms` holds logic.

3. **`PackageContents.xml` registration** — this is the step most likely to
   be forgotten, and the file won't load at all without it:
   - Logic file → add a `<ComponentEntry ModuleName="./pre-start-up scripts parts/YourFile.ms" />` inside the big pre-start-up `<Components>` block, positioned **after** anything it depends on (`NeoDexGlobals.ms`, `UtilityFunctions.ms`, etc. must stay first).
   - Macroscript → add a `<ComponentEntry ModuleName="./macroscripts parts/NeoDex Toolkit-NeoDex_<Name>.mcr" />` in the macroscripts `<Components>` block.
   - Respect the existing `<RuntimeRequirements SeriesMin=".." SeriesMax="..">` scoping if the tool is version-specific; otherwise it belongs in the all-versions (2022–2027) block.

4. **Menu entry (both code paths)** — if the tool should appear in the
   `NeoDex` top menu, update **both**:
   - `NeoDex/pre-start-up scripts parts/NeoDexMenuCallback.ms` (Max 2025–2027, CUI callback): add a new GUID constant and a `neoDexMainMenu.CreateAction "<guid>" 647394 "NeoDex_<Name>\`NeoDex Toolkit"` call (or into the `extraMenu` submenu for "Extra Tools" style entries).
   - `NeoDex/post-start-up scripts parts/NeoDexMenuInstaller.ms` (Max 2022–2024, `menuMan` API): add the matching `menuMan.createActionItem "NeoDex_<Name>" "NeoDex Toolkit"` and `addItem` call.
   Forgetting one of these means the tool works on some Max versions and is
   simply missing from the menu on others, with no error.

5. **Sidebar button (optional)** — if the tool warrants a quick-access icon,
   add a 24×24 (or the sidebar's existing size — check current buttons)
   PNG to `NeoDex/neodex_icons/` named `ndx_sb_<name>.png`, and wire a
   button into `NeoDex/post-start-up scripts parts/NeoDexSidebar.ms`
   following the existing `loadIcon` / button-click pattern.

6. **Localization** — every user-visible string (button text, tooltips,
   dialog labels, messages) must go through `::L.t "key"`, not a literal.
   Add the key to the English (`enData`) section of
   `NeoDex/pre-start-up scripts parts/NeoDexLocalization.ms` (required —
   this is the fallback), and to the other five language sections at least
   as an empty `""` placeholder (never leave English blank; other languages
   falling back to English is fine and expected for unfinished
   translations).

7. **Error/validation reporting (if applicable)** — if the tool can fail in
   ways the user needs to see in a report dialog rather than a raw
   MAXScript exception, register a category via
   `::ErrorReporter.InitiateErrorCategory "YourCategory" "description"`
   in `NeoDex/pre-start-up scripts parts/ErrorReporter.ms`, then use
   `::ErrorReporter.logError/logWarning/logInfo "YourCategory" objectName message`
   from the tool.

## After wiring it up

This isn't testable by reading the code — see `neodex-deploy` to actually
load it in 3ds Max (a `PackageContents.xml` or pre-start-up change needs a
full Max restart) and confirm the menu item, sidebar button, and dialog
all appear and function before calling the task done.
