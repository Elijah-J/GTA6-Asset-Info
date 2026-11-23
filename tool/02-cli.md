# CLI

*Part of the [SixNG tool guide](README.md). Numbers and timings as of 2026-11-21.*

## CLI basics

- Every command prints **one JSON document on stdout**. `--pretty` indents
  it.
- Progress goes to **stderr as NDJSON**:
  `{"event":"progress","stage":…,"done":…,"total":…,"pct":…}`. `--quiet`
  turns it off.
- On error the CLI prints `{"ok":false,"error":"…"}` and exits with code 1.
- **Entry ids** are `<archive>@<hex offset>` (e.g. `levels.rpf@10f2d2d010`).
  A member of a drawable dictionary is `<entry id>#<member name>`; `extract`
  and `inspect` accept these, and each member exports as its own model
  (2026-11-22).
  They are stable for one install and can be passed to `--id`.
- **Archives** are named by their stem (`levels`, `data`, `navmeshes` …) or
  by path for stores (`data/ttdb/vehicles/ttdb_vehicles.ttpf`).

### Filter flags

Used by `list`, `search`, `extract` and the world commands.

| Flag | Meaning |
|---|---|
| `--kind a,b` | entry kinds: `resource`, `rageobj`, `xml`, `rbf`, `pso`, `gfx`, `dds`, `png`, `text`, `bin` |
| `--rtype a,b` | resource types from names: `dr` (drawable), `ft` (fragment), `dd` (drawable dictionary), `txd` … |
| `--name <regex>` | regex on the recovered name (or id) |
| `--exclude-name <regex>` | regex on names to leave out |
| `--id a,b` | exact entry ids or hex offsets |
| `--id-file ids.txt` | ids from a file, one per line (`#` comments allowed); combines with `--id`. Use it for long lists (added 2026-11-20) |
| `--min-size <bytes>` | minimum decompressed size |
| `--limit <n>` | at most n entries |
| `--bbox=minx,miny,maxx,maxy` | world region in RAGE metres, for `placements`, `collision`, `navmesh`, `lights`. Write it with `=` so negative values parse. |

### Output flags

Used by `extract` and `extract-category`.

| Flag | Meaning |
|---|---|
| `-o, --out <dir>` | output directory (required) |
| `--tex png\|dds\|both\|none` | texture files to write (default `png`). DDS keeps the original BCn data with a mip chain. `none` writes no texture files (geometry-only re-exports). |
| `--raw` | also write the decompressed resource bytes |
| `--force` | redo items that already have outputs |
| `--no-embedded-textures` | skip textures embedded in models |
| `--dry-run` | print the selection count and decompressed size; write nothing |
| `--body-shape "<control>=<v>,…"` | ped clothing: pose the shared ped skeleton with the game's body-shape rig, e.g. `gcweightbelly=0.5,gcmusclechest=1`. Controls: `gcweight<region>` −1..1, `gcmuscle<region>` 0..1 (back, belly, butt, calf, chest, forearm, hip, muffin, neck, shoulder, thigh, upperarm), `gcbutt` 0..1, `gcbreasts` −1..1; `allweight=v` / `allmuscle=v` set a whole group. The shape goes into the joint node transforms (inverse bind matrices stay neutral). Needs `packs` and `anim` indexed; re-index `anim` with SixNG 2026-11-21 or later (its index format changed) |
| `--head-skin N` | ped heads: which face to use (they bind no textures). Heads a material preset names use that face; others get a face of the same sex, body type and ethnicity code (`fr1_lat` in `head_fr1_lat_018_m_full`) from the linked heads, preferring the same age letter |

**Resumable runs.** Each item writes a marker when it finishes, and items
with outputs are skipped. To continue a stopped run, run the same command
again. Every run appends one line per item to `<out>/manifest.jsonl`. The
summary JSON has `exported`, `skipped`, `error_count`, `errors`,
`warning_count` and `warnings` (lists capped at 50; the manifest has all).

## Commands

### Inspecting the install

```powershell
# Archives in the install, their headers and index status
sixng info --pretty

# Build carving indexes and recover names. Resumable. No arguments = every RPF9 archive.
# levels.rpf takes about 3–6 minutes.
sixng index levels textures packs data common navmeshes
sixng index levels --force        # rebuild
sixng index levels --no-names     # skip the name pass

# List entries of one archive
sixng list levels --rtype txd --limit 20 --pretty
sixng list levels --rtype ft --name '^phoenix' --ndjson
sixng list levels --rtype dr --name 'road' --streams     # include stream lists

# Search names across all indexed archives
sixng search 'elegy|buffalo' --archive levels

# Describe one entry without writing files
sixng inspect 'levels.rpf@10f2d2d010' --pretty
```

`list` prints a JSON array (or NDJSON with `--ndjson`) of entries with
`id`, `offset`, `csize`, `usize`, `kind`, `sub`, `ext`, `name`, `rtype`,
`nstreams`.

### Extracting models, textures and files

```powershell
# By filter
sixng extract levels --rtype txd --name '^island' -o out --tex both
sixng extract levels --id 'levels.rpf@10f266d010' -o out --tex none

# Built-in categories: textures | map | vehicles | meta
sixng extract-category vehicles --name '^phoenix' -o out --tex both
sixng extract-category map --limit 200 -o out --dry-run
```

| Category | Selection | Approx. size |
|---|---|---|
| `textures` | every texture dictionary (2,486) | ~5.4 GB PNG |
| `map` | `levels.rpf` drawables plus PSO/RBF/XML (183,200) | ~93 GB raw, ~180 GB with GLBs, ~100 GB more of linked PNG |
| `vehicles` | fragments not matching a prop/character prefix list (37,056); a name heuristic, narrow it with `--name` | ~15 GB with GLBs |
| `meta` | XML/PSO/RBF/text from `data` and `common` (3,763) | ~0.2 GB |

One vehicle (`--name '^phoenix'`) is 248 parts and about 100 MB.

`--paint RRGGBB` (or `AARRGGBB`, sRGB) sets the colour of car-paint
materials (linear `baseColorFactor`, and `paint.colour` in `model.json`);
`sixng vehicles` gives each model's colours. Materials of emissive shaders
(`standard_emissive*` and the known emissive families) get their colour
texture as `emissiveTexture`, so lamps, neon and screens light up.

### Characters and animals

```powershell
# Every character and animal outfit can build (327 characters, 263 animals)
sixng characters --kind animal --name deer
# A character or animal as one rigged GLB; several clips, one glTF animation each
sixng outfit crb_deer_whitetailed_m_00 -o out/deer.glb --clip "gameplay@creatures@land@mammal@deer@normal@run@fast/move,gameplay@creatures@land@mammal@deer@idle/idle"
# Skeleton and animations only, to pair with the full model (same joint names)
sixng outfit crb_deer_whitetailed_m_00 -o out/deer_walk.glb --anim-only --clip "gameplay@creatures@land@mammal@deer@normal@walk@slow/move"
# Any name plus --parts (regexes over every drawable) builds an ad-hoc character
sixng outfit hiker_m -o out/hiker.glb --parts '^head_mr1_cau_000_m_full','^hand_mr1_000_tshirt','^feet_mr1_000_shortsmid','^tshirtcrewneck_mr1_000.*_full$','^shortscargo_mr1_000','^bootshiking_mr1','^backpackhiking'
```

`outfit` skins a character's parts (its default look, or `--parts`; `--add`
adds parts) to the shared ped skeleton plus their own helper bones, or an
animal's parts to its root fragment's skeleton
([§7.5](../formats/07-models.md#animals-and-one-bone-parts-2026-11-22)), and
writes **one GLB with one skin**; textures go to `<glb dir>/textures/`.
Joint lists are merged by name (a helper bone two parts define differently
is kept twice, the second as `<name>.<part>`). One-bone parts are attached
to their bone; fur placeholders are left out unless `--add fur`. `--clip
<dictionary>/<clip>` adds bone tracks as glTF animations (root motion and
float tracks are left out). The report on stdout gives unresolved joints
and how many tracks land on a joint. Not done: root motion, facial
animation, cloth.

`ped-variations` (0.7.0) lists a wardrobe's variations and their parts
([§12](../formats/12-materials.md#material-presets-peds)). In a fresh work
folder it needs two indexes first:

```powershell
# 1. The names dictionary; without it every variation's scale is null (the field is read by name)
sixng index common
sixng names
# 2. The texture index, which links an item's texture hashes to its drawable
sixng texture-users packs
```

For the two zombie wardrobes (`packs.rpf@2b21010` and `@2b34010`),
indexing `packs` alone gives the same 43 and 75 variations, parts and
scales as indexing every archive (compared record by record); other
wardrobes may reference drawables outside `packs` (not checked). Timings:
names about 4 s, `texture-users packs` about 32 s. The model's variation
list names only clothing sets, so head sets probably pair with any of them
(medium; see [variation lists](../formats/12-materials.md#material-presets-peds)).

### Effects and fonts

```powershell
# Particle effect dictionaries: names, rule names; -o adds the keyframe curves
sixng ptfx data --name scr_arc -o out
# The 29 UI fonts as TrueType and three.js typeface JSON (~70 MB)
sixng fonts data -o out
```

`ptfx` writes `<out>/ptfx/data/index.json` and one `<offset>.json` per
dictionary ([§17](../formats/17-other.md#particle-effect-dictionaries)).
`fonts` writes `<out>/fonts/<name>.ttf`, `<name>.typeface.json` and
`fonts.json` ([§17](../formats/17-other.md#scaleform-ui-fonts)). The fonts
belong to their owners; use them locally with your own install. All added
2026-11-22.

### World data

```powershell
# Map placements: one JSON per map file (~1.5 min, ~1.3 GB for all of levels.rpf)
sixng placements levels -o out
# Instanced grass and prop batches (~2.5 min, ~5.8 GB); --batches adds them to a normal run
sixng placements levels -o out --batches-only

# Collision: one GLB per bounds file, one primitive per surface id (~15 GB)
sixng collision levels -o out
# Navmesh tiles; --roads-only keeps road polygons only (~0.25 GB vs ~25 GB)
sixng navmesh --roads-only -o out
# Road graph with junctions and traffic-light phases (~40 MB)
sixng paths levels -o out
# Placed lights (~1–1.5 min, ~0.7 GB)
sixng lights levels -o out
# Interior instances, entity lists and linked collision (run after collision; ~15 MB)
sixng interiors levels -o out
# Inferred collision surface categories (after collision, paths, placements --batches; ~3 min)
sixng surfaces levels -o out
sixng surfaces levels -o out --navmeshes none     # skip the navmesh road cue

# Every world step in order: placements, batches, collision, navmesh, paths, lights, interiors, surfaces
sixng world all -o out
sixng world collision -o out                      # or one step

# One region only (RAGE metres)
sixng collision levels -o region --bbox=-920,-190,-420,310
sixng placements levels -o region --bbox=-920,-190,-420,310 --batches
sixng lights levels -o region --bbox=-920,-190,-420,310
sixng navmesh -o region --bbox=-920,-190,-420,310
```

A 500 m box gives about 110 MB of collision. `paths` and `surfaces` always
cover the whole map; filter their output by position.

**Order matters:** `interiors` needs `collision`; `surfaces` needs
`collision` and uses `paths`, `placements --batches` and the navmesh
archive when present. `world all` runs them in the right order.

### Linking and census commands

```powershell
# Link archetypes whose drawable has no recovered name (bbox + archive-order match).
# Run once after indexing (~9 min). Writes <work>/index/archetype_links.json,
# which placements, lights and extract then use automatically.
sixng link-archetypes levels

# Archetype LOD distances and resolved LOD parents (~2.5 min). Writes
# <work>/index/archetype_lod.jsonl and lod_parents.jsonl; re-run placements to pick them up.
sixng lod-links levels

# Interior membership: per map file its space (world / interior-local / group-local),
# per entity its interior, room and entity set (~1.5 min). Writes <work>/index/interior_entities.jsonl.
sixng interior-entities levels

# Name dictionary for shader, parameter and meta field hashes, built from the install
# (~2 s, needs the common index). Writes <work>/names/names.json; exports and the app use it.
sixng names
sixng names --lookup bcf03fc1,c8e8c282

# BCPack texture gaps: every material with flat or low-mip textures (~6 min).
# Uses placements, batches, collision and surfaces from the same output dir.
sixng texture-gaps levels -o out --file out/texture_gaps.json --placements out/placements/levels

# Schema index of every meta (0x7a) resource; --dump adds a decoded JSON per resource
sixng meta data -o out
sixng meta levels -o out --dump --limit 100

# Shader census: per shader hash and name, use count, parameters, texture formats, record flags
sixng shaders levels -o shaders.json
```

### Metadata and vehicles

```powershell
# PSO files to JSON: <out>/pso/data/<offset>.json (a few seconds)
sixng pso data --out out
# Vehicle handling records: <out>/pso/handling.json
sixng handling data --out out
# Vehicle model info (wheel scale and type), spawn configurations (colour, rim, tyre, parts)
# and weighted paint-colour lists: <out>/vehicles/vehicles.json
# (format 6 adds display_name and make per vehicle, null when absent)
sixng vehicles packs --out out
```

### Texture stores

```powershell
# List a ttpf store
sixng ttpf vehicles/ttdb_vehicles.ttpf --pretty
# Decode textures by hash, or the first N, at a mip level
sixng ttpf vehicles/ttdb_vehicles.ttpf --hash 0123456789abcdef --out tex
sixng ttpf maps --first 20 --mip 1 --out tex
# Texture arrays and cube maps: every slice / face as <hash>_m<mip>_s<slice>.png
# (without --all-slices only slice 0 is written)
sixng ttpf lighting --hash 0123456789abcdef --all-slices --out tex
# A whole store in parallel, with index.json marking each texture complete, partial
# (part of the data in BCPack chunks other than BC1) or bcpack_only (no PNG). Use the full store id
# when a short name matches several stores. The UI store takes about 10 s.
sixng ttpf data/ttdb/ui/ttdb_ui.ttpf --all --out out/ui/ttdb_ui
```

### Video and UI graphics

```powershell
# Bink videos: carve by header (~7 s) and list them in <out>/movies/movies.json
sixng movies movies --out out
# Write the .bk2 files as they are (all of them: 8.8 GB), or only some
sixng movies movies --out out --extract
sixng movies movies --out out --extract --only 143205000
# Named UI dictionaries in data.rpf (loading screens, HUD, gauges; 880 textures, 108 MB)
sixng extract data --rtype txd -o out
```

### Audio

```powershell
# Sound banks in audio.rpf: scan once (~2.5 min, index ~30 MB), then list
sixng audio --min-seconds 30 --limit 10 --pretty
# Indexes from before 2026-11-22 read the codec from the wrong byte: rescan
sixng audio --rescan
# Export a bank's streams (--stream <hex> for one, --all for every bank)
sixng audio-export --bank 12a2000 --format both -o work/out/audio
```

The scan finds 7,160 banks with 1.6 M streams (about 1,300 hours). Export
by codec ([structure/03](../structure/03-audio-awc.md)):

| Codec | Export |
|---|---|
| Opus | Ogg Opus (lossless remux) or WAV (decoded, 16-bit, 48 kHz); the encoder delay is taken as 312 samples and the end is trimmed to the stream's sample count |
| Opus music (1–8 sub-streams) | one `.opus` / `.wav` per sub-stream, named `<bank>_<stream>_<sub-stream id>` |
| PCM16, float32 PCM | WAV, samples as stored |
| XMA2 | `.xma`: RIFF WAVE with an XMA2 format header, a lossless remux of the 2 KiB packets |
| any | `--format awc`: the raw bank |

Stream names are 29-bit hashes and are not recovered; streams are listed by
bank offset and stream id.

### Text

```powershell
# Every localised text table in data.rpf to JSON (~21 s)
sixng text data -o work/out
# One label's string in every table (every language), without markup
sixng text pack__shared_minigame000__shared_minigame000 --label AG_SUR_GEAR_FIRST1_01 --plain
```

Each table becomes `<offset>.json` (label hash → string), listed in
`index.jsonl`. `--plain` drops the inline markup (subtitle timing,
colours, line breaks, button icons). Which table is which language is not
stored. Layout: [§17](../formats/17-other.md#text-gxt). Added 2026-11-22.

### Hair, cloth and head deltas

```powershell
# Every strand groom in packs.rpf as a GLB of lines (~0.8 GB, under a minute)
sixng grooms packs --out work/out/grooms
# Crossed ribbon meshes instead of lines (width in mm; render in any viewer),
# scalp triangles between the roots, some grooms only, or just the catalogue
sixng grooms packs --out work/out/grooms --ribbons 0.5 --roots --only 10950d010,f14aa010
sixng grooms packs --out work/out/grooms --list
# Cloth and soft bodies: particles and constraint edges as GLB lines (~30 s)
sixng cloth packs --out work/out/cloth
# Head identity deltas; --build writes the heads that exist only as deltas (~40 s, 0.5 GB)
sixng head-deltas packs --out work/out/head_deltas --build
```

- **Grooms** (20,010; [§19.2](../formats/19-ped-physics-resources.md#192-strand-hair-type-3)):
  one polyline per guide strand in model space, so they sit on the head
  drawables. Lines carry the point width (`_WIDTH`) and the root UV
  (`TEXCOORD_0`; both inferred). `--list` writes `catalog.json`. No
  resource says which groom belongs to which ped.
- **Cloth** (13,067; [§19.3](../formats/19-ped-physics-resources.md#193-cloth-type-2)):
  both particle / edge sets of each resource.
- **Head deltas** (6,757; [§19.4](../formats/19-ped-physics-resources.md#194-head-identity-deltas)):
  each delta matched to its head; `--build` writes the 673 heads that have
  deltas but no drawable (the base head moved by the delta, or the head's
  own drawable where the name follows from the order). They are
  untextured: the base heads have no colour texture in the install.

All added 2026-11-22.

### Research helpers

`sixng find-u32 <archive> --values <file>` searches meta struct instances
for u32 values (hex, one per line) and reports struct and offset hits.
Useful when identifying fields.

---

← [Build and install](01-install.md) · [Index](README.md) · [GUI](03-gui.md) →
