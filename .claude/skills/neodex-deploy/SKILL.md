---
name: neodex-deploy
description: Deploy this repo's NeoDex folder into 3ds Max's ApplicationPlugins directory (via a directory junction) so edits can be reloaded and tested live in Max. Use whenever a MAXScript change needs to actually be tested, or when the user asks to "install", "deploy", "test in Max", or "reload" NeoDex.
---

# Deploying NeoDex for live testing

NeoDex has no build step. 3ds Max discovers Autodesk "Application Plugins"
by scanning `%APPDATA%\Autodesk\ApplicationPlugins\` (per-user) and
`%ALLUSERSPROFILE%\Autodesk\ApplicationPlugins\` (per-machine) for folders
containing a `PackageContents.xml`. The `NeoDex/` folder in this repo *is*
one of those bundle folders already — it doesn't need to be copied into a
different shape, just made visible at one of those two locations.

## One-time setup: link, don't copy

Copying the folder means every edit has to be copied again before it's
testable. Use a directory junction instead so the live plugin folder and
the git checkout are the same files on disk:

```powershell
# Remove any previous install of NeoDex at this location first (check what's
# there before deleting — it may be the user's existing real install).
$target = "$env:APPDATA\Autodesk\ApplicationPlugins\NeoDex"
if (Test-Path $target) {
    Get-Item $target | Format-List Name, LinkType, Target
    # If it's a real directory (not already a link) and has content the user
    # cares about, STOP and ask before removing it.
}
cmd /c mklink /J "$target" "<repo>\NeoDex"
```

`mklink /J` (junction) works without admin rights and without Developer
Mode, unlike `/D` (symlink) — prefer it. Confirm with the user before
running this if a real (non-link) `NeoDex` folder already exists at the
target path, since it likely represents their existing production install.

## After every MAXScript edit

3ds Max does not hot-reload `ApplicationPlugins` content. To pick up changes
to `pre-start-up scripts parts/*.ms`, `scripted plugins parts/*.ms`, or
`PackageContents.xml` itself, **3ds Max must be restarted**.

Two narrower cases don't require a full restart:

- **A single `.mcr` macroscript's UI/logic** (files loaded from
  `macroscripts parts/`): re-run via 3ds Max's MAXScript menu →
  *Run Script...* pointed at the `.mcr` file, or evaluate it in the
  MAXScript Listener. This re-registers just that macroScript.
- **A `post-start-up scripts parts/*.ms` file**: same — it can usually be
  re-run standalone via *Run Script...* since these mostly just build a
  rollout/struct and assign it to a global, as long as the pre-start-up
  globals it depends on haven't changed shape.

If you changed anything in `pre-start-up scripts parts/`, a struct
definition, or `NeoDexIO.ms` tag constants, don't try to shortcut it —
restart Max.

## Verifying the load actually worked

On startup, several core files `format` a line to the MAXScript Listener —
e.g. `NeoDexGlobals.ms` prints `NeoDex: Native BLP support detected` when
applicable, and `NeoDexNativeHelper.ms` prints which native DLL it loaded
or a `WARNING` if it didn't find one. Open the Listener (F11 or the
bottom-left mini window) after restart and check for `NeoDex WARNING` lines
before assuming a change loaded cleanly — a MAXScript syntax error in a
pre-start-up file will typically abort loading of that file silently past
the point of the error, with the only signal being a stack trace in the
Listener.

## Reporting results

Don't claim a fix "works" from reading the code alone. Either walk the user
through triggering the exact tool/menu item affected and report what
happened, or explicitly tell the user the change is untested and needs a
Max restart + manual check, per the repo-wide rule that UI/feature claims
need real verification, not just successful parsing.
