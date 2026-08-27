---
name: wc3-mdx-format
description: Quick reference for the Warcraft III MDX/MDL model format's chunk tags and NeoDex's in-memory Wc3Model data structs. Use when reading, fixing, or extending the model reader/writer/scene-parser code (NeoDexIO.ms, NeoDexMDXReader.ms, NeoDexMDXWriter.ms, NeoDexMDLWriter.ms, NeoDexReaderReforge1200.ms, Wc3Model.ms, NeoDexSceneParser.ms, NeoDexSceneRebuilder.ms) or debugging a corrupt/misread model.
---

# Warcraft III MDX/MDL format, as NeoDex implements it

MDX is Blizzard's binary format: a flat sequence of chunks, each
`[4-byte FourCC tag][4-byte chunk size][chunk data...]`. MDL is the
human-readable text equivalent of the same data — **NeoDex only writes
MDL, it never reads it**; all import goes through the binary MDX readers.

Every tag constant lives in `NeoDex/pre-start-up scripts parts/NeoDexIO.ms`
as `global tag_XXXX = 0x...` with an inline comment naming the chunk — that
file is the single source of truth for the tag list; don't hardcode a raw
FourCC value elsewhere.

## Chunk map (classic MDX, versions up to 800)

| Tag | Chunk | Notes |
|---|---|---|
| `MDLX` | file identifier | first 4 bytes of every `.mdx` |
| `VERS` | format version | 800 = classic, see Reforged section below for 1200 |
| `MODL` | model info | name, extents |
| `SEQS` | animation sequences | → `Wc3Sequence` |
| `GLBS` | global sequences | shared animation length pools |
| `TEXS` | texture definitions | → `Wc3Texture` |
| `LAYS` / `KMTF` / `KMTA` | material layers + texture-id/alpha animation | → `Wc3Layer` |
| `MTLS` | materials | → `Wc3Material` |
| `SNDS` | sound definitions | |
| `TXAN` / `KTAT` / `KTAR` / `KTAS` | texture animations (translation/rotation/scale of UVs) | → `Wc3TvertexAnim` |
| `GEOS` + `VRTX`/`NRMS`/`PTYP`/`PCNT`/`PVTX`/`GNDX`/`MTGC`/`MATS`/`UVAS`/`UVBS` | geoset (mesh) data — vertices, normals, faces, bone/matrix groups, UV sets | → `Wc3Geoset` |
| `GEOA` + `KGAO`/`KGAC` | geoset animation (visibility alpha / color tint) | → `Wc3GeosetAnim` |
| `BONE` | bones | → `Wc3Bone`; animated via generic `KGTR`/`KGRT`/`KGSC` (translation/rotation/scale) tracks shared with helpers/objects |
| `HELP` | helper objects | → `Wc3Helper` |
| `LITE` + `KLAS`/`KLAE`/`KLAC`/`KLAI`/`KLBI`/`KLBC`/`KLAV` | lights (attenuation, color, intensity, ambient, visibility) | → `Wc3Light` |
| `ATCH` + `KATV` | attachment points | → `Wc3Attachment` |
| `PIVT` | pivot points | one per object, indexed by object id |
| `PREM` + `KPEE`/`KPEG`/`KPLN`/`KPEL`/`KPES`/`KPEV` | particle emitter 1 (MDL-style) | → `Wc3ParticleEmitter` (`emissionType: #Mdl`) |
| `PRE2` + `KP2S`/`KP2R`/`KP2L`/`KP2G`/`KP2E`/`KP2N`/`KP2W`/`KP2V` | particle emitter 2 | → `Wc3ParticleEmitter` (`emissionType: #Mdl2` or similar — check `NeoDexMDXReader.ms`) |
| `RIBB` + `KRHA`/`KRHB`/`KRAL`/`KRCO`/`KRTX`/`KRVS` | ribbon emitters | handled together with `BlizzRibbon.ms` scene plugin |
| `EVTS` + `KEVT` | event objects (sound/effect triggers) | paired with `Wc3RefEvent.ms` scene plugin |
| `CAMS` + `KCTR`/`KTTR`/`KCRL` | cameras | → `Wc3Camera` |
| `CLID` | collision shapes | paired with `Wc3CollisionShapes.ms` scene plugin |
| `SNEM` + `KESK` | sound emitter ("Corn") | paired with `Popcorn.ms` |

## Reforged v1200 additions

`NeoDex/pre-start-up scripts parts/NeoDexReaderReforge1200.ms` handles
`VERS` 1200 with a **separate reader**, not a branch inside the classic
one. Extra tags it defines:

| Tag | Chunk |
|---|---|
| `TANG` | vertex tangents |
| `SKIN` | bone-weight skinning data (replaces/augments `GNDX`/`MATS` groups) |
| `CORN` | popcorn/corn particle emitters |
| `FAFX` | FaceFX data (paired with `FaceFX.ms` scene plugin) |
| `BPOS` | bind pose matrices |

Reforged also adds fields to existing structs rather than new chunks —
see the `-- Reforged` comments in `Wc3Model.ms`: `Wc3Layer` gets PBR texture
slots (`normalTexId`, `ormTexId`, `emissiveTexId`, `teamColorTexId`,
`envMapTexId`) plus `emissiveGain`/`fresnelColor`/`fresnelOpacity`/
`fresnelTeamColor`; `Wc3Geoset` gets `tangents`/`skinData`/`lod`/`lodName`;
`Wc3Sequence` gets `sharedGroup` (name of a master sequence when animations
are shared to save space).

## In-memory model: `Wc3Model.ms`

The reader/parser produce, and the writer/rebuilder consume, a single
`Wc3Model` struct holding arrays of the structs above (`sequences`,
`globalSequences`, `textures`, `materials`, `tvertexAnims`, `geosets`,
`geosetAnims`, `bones`, `helpers`, `lights`, `attachments`,
`particleEmitters1`, `particleEmitters2`, `ribbonEmitters`, `cameras`,
`collisionShapes`, `events`). Almost every struct carries an `ex` field —
a catch-all slot for extra/format-specific data that doesn't have a
dedicated field; check how existing code reads/writes `ex` for that struct
before adding a new field, since a new named field vs. stuffing into `ex`
is a real design choice made per-struct.

There are **two** near-identical geoset/mesh structs — `Wc3Geoset` (format
data, used by reader/writer) and `MaxGeoset` (scene-side working data with
`tvfaces`, used by parser/rebuilder while manipulating the actual Max mesh)
— don't confuse them when tracing a bug through the parser/rebuilder vs.
the reader/writer.

## Data flow

```
Import:  .mdx file → ::IOManager (byte-level read)
                    → ::NeoDexMDXReader / ::NeoDexMDXReaderV1200 (chunk parse)
                    → Wc3Model
                    → ::SceneRebuilder → 3ds Max scene

Export:  3ds Max scene → ::SceneParser → Wc3Model
                    → ::MDXWriter → .mdx   (binary)
                    → ::MDLWriter → .mdl   (text, export-only)
```

When a bug report is "model X doesn't import/export correctly", first
narrow down which stage: does the raw chunk data look right (add a
`format` dump in the reader), does `Wc3Model` hold the right values after
parsing, or does the scene-side rebuild/parse step lose/mangle it. Don't
guess at the binary layout — `NeoDexIO.ms` and the tag table above are
authoritative.
