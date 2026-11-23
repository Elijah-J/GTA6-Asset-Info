# 11. Vehicle paint and vehicle tables

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-21; confidence: confirmed / inferred / unknown.*

| Item | Finding | Confidence |
|---|---|---|
| colour slot `c8e8c282` (`diffusetex`) on vehicle shaders (`standard_vehicle_2lyr`, `4aa38282`) | points at an engine-default **white** texture found in no store; the body colour is a runtime paint colour, as in GTA V | confirmed |
| `f151c063` | decal / sticker layer, RGBA | inferred |
| `d37026cb`, `640111c6` | dirt / grime / wear masks | inferred |
| `84d22f74` | fine scratches and flakes | inferred |
| `b9f4bdac` | grime | inferred |
| `fb03ae36` (`dusttexture`) | green-dominant mask (mask pack or normal detail) | unknown |
| `4d56d7fe` / `2bb2ede8` (`lyr1materialatex` / `lyr0materialatex`) | BC1 atlas, normal-like | inferred |
| `84c69350` / `58b6e707` (`lyr0normaltex` / `lyr1normaltex`) | BC5 normals | confirmed |
| shader constant buffer (+0x08) | ~0x160 bytes of floats, layout not keyed by the parameter table | unknown |
| `19df25d7` | 128 × 256 R8G8B8A8 sRGB texture of horizontal colour bands; one texture (same hash) is bound by every two-layer material of a car and by the shared tyre model (12 of 12 `standard_vehicle_2lyr` materials on one SUV, plus the tyre). Probably a material palette indexed by the layer maps or constants; the lookup is unknown. *Later check (ambulance and SUV): every vehicle binds the **same** texture: row 0 R / G / B markers, rows 1–32 black, rows 33 and up ramps, so it is probably a placeholder the game replaces (inferred); this disagrees with "one per car" and is kept alongside it* | format, sharing confirmed; role inferred |
| `346394ea` / `7d4f12e2` (`lyr0diffusetex` / `lyr1diffusetex`) | **linear BC7 detail maps, not colour**: R and G grey around 0.5, B a sparse mask | confirmed (not colour; channels on one map) |
| COLOR0.a on `standard_vehicle_2lyr` | a **part id** (0–255, a few values per material), not a layer weight | confirmed (ambulance, SUV) |

### Vehicle shaders (2026-11-22)

| Shader | Role | Confidence |
|---|---|---|
| `4aa38282` `standard_vehicle_2lyr` | **not only body paint**: one SUV has 12 materials on it, the largest (82 K triangles) spanning the whole car including the interior. Rule of thumb (inferred): a material is body paint when it binds no detail texture and its meshes sit on body bones. Real per-part colours are not decoded | confirmed (coverage) |
| `99ad6fc9` `standard_vehicle_2lyr_tyre_inst` | tyres; `diffusetex` is the engine-default white | confirmed |
| `c5cdbdd2` `standard_vehicle_2lyr_wheel_inst`, `99ad6fc9` | rims and tyres embed one **129 × 1 tint palette**, `584f3a883128c1a4`, identical on every rim and tyre looked at: entries 0–7 pure red, yellow, green, blue, magenta, grey (85), orange, cyan; 8–63 shades in groups of four; 64–128 one pink. Probably a placeholder the game overwrites with the wheel colour (inferred from the marker colours and the sharing) | palette confirmed; role medium |
| `389206d1` `standard_vehicle_alpha` | lamp lenses and clear covers, alpha bucket. `diffusetex` RGB is one constant colour (128, 128, 98 on the sample), opacity in alpha (78–255); also `extinctionlut` and `refractionbumptexture` slots. 41 materials in a 20,000-model sample | confirmed on one vehicle |
| `03b27476` `standard_vehicle_2lyr_reflector` | lamp reflectors (earlier grouped with car paint) | inferred (name, position) |
| `29648098` `standard_vehicle_emissive_animated_bones`, `5a693bbe` `standard_vehicle_emissive_layer` | lamps and lit panels; `emissivetex` (`8e0d63ee`) and `diffusetex` bind the same texture | confirmed on one vehicle |
| `5a693bbe` **runtime screen overlay** | a dashboard screen is two meshes about 1 mm apart: behind, `standard_vehicle_emissive_layer` with a real colour map; in front, the same shader with `diffusetex` and `emissivetex` both `a800afccf01aa36b` (in no store), always in the cutout bucket (record flags `0003a80108ff0000`; `0003a00108ff0000` on generic models). 570 models bind it (284 `_hi`, 275 `_mid`, 11 others; also on `standard_emissive_layer`); of the 284 `_hi`, 251 in both slots, 31 have no emissive slot, 2 another emissive map, and 267 have a textured emissive mesh behind it. Probably the texture the game draws dials and navigation into at runtime, or a transparent default; what is drawn and its size are unknown | medium |
| `bd71bdbf` `windshield_wipermask` | **wiper sweep mask**: no textures, one UV set, bound only to the `windscreen` / `windscreen_r` bones; u takes only {0, 0.014, 0.5, 0.986, 1} and v runs 0–1, the sweep coordinates of a fan over the glass. 210 of 284 surveyed `_hi` vehicles have one. Not a visible surface | high |
| `fc34f08b` `vehicle_water_stencil` | no textures, opaque bucket (record flags `0004200010ff0000`, `0005200020ff0000`): a lid over a boat's hull opening (74 meshes in the surveyed boats and jet skis), probably stencil-only; how the game uses it is unknown | medium (name and shape) |
| `fa9cc069` `standard_vehicle_glass` | `diffusetex` is a 64 × 512 gradient, constant along u; v selects clear glass (alpha 128) or the dark border (alpha 255) | high (two vehicles) |

## Vehicle tables

The game-data meta resource in `packs.rpf` (type 0x7a, 26 MB) and about 330
per-vehicle pack resources with the same structs (rows repeated) hold the
vehicle tables. Field names are case-sensitive joaat and match GTA V's
`vehicles.meta` names where they exist (confirmed).

| Struct | Count (game-data resource) | Fields |
|---|---|---|
| `e59aa857` model info | 327 | `gameName` (`120e5836`, usually joaat of the model name, see below), `handlingId` (`ec41bd9c`, the model name), `txdName`, `wheelScale` (`933301be`), `wheelScaleRear` (`76706238`), `wheelType` (`a2e90c68`), `layout`, `HDTextureDist`, `lodDistances`, `vehicleClass`, `swankness`, `plateType`, `defaultBodyHealth` … (52 names resolve) |
| `51019114` configuration | 3,879 | `fb1f80ff` model (field name `handling`), **`5bdc13e3` body colour** (ARGB u32), `e89b8540` part models (joaat names, about 150: frame, bumpers, exhaust, `brakedisc_*`, `caliper_*`, and one `rim_*` and one `tyre_*` in 82 %) |
| `fc1c7000` variation set | 4,772 | `fb1f80ff` model (`handling`), **`8c866829` (`colours`) weighted paint-colour list**: 1 to 400 ARGB colours with repeats (2,246 sets; this is the pool of 17,632 colours), `59a59c3e` (`slots`) weighted part groups: `37a6f9ef` slot, `a12ff3a2` (`options`) list of {`8e6612bf` part, `dc732774` (`probability`) weight %} |

With the per-vehicle packs this covers 373 models (305 named), 4,063
configurations (3,339 with a rim and a tyre) and 2,377 colour lists. Handling is
a separate PSO ([§17](17-other.md#vehicle-handling)).

- **Joining model info to a model** (2026-11-23, high): `gameName` is not
  always the model name: 46 of 325 distinct records hold a different value
  there than in `handlingId` (case-sensitive joaat). `handlingId` is what
  configurations and variation sets use as owner, and it matches the
  drawable name. Falling back to `handlingId` when `gameName` finds nothing
  raises the models with model info from 312 to 357 of 373; every surveyed
  vehicle with wheel mounts then has a record except two damage variants
  and one train. Open: whether several models can share one `handlingId`,
  and what `gameName` means where it differs (it looks like a display or
  make name); field `1d2a64a8` also equals the model name hash on some
  records.
- **Mod parts** (2026-11-23, high: parts placed at the named bone's bind
  pose sit correctly on two cars, and kit names resolve for all 12
  two-element kits of one car) sit in the vehicle's own pack resource (the
  0x7a resource with one model info):
  - `0caef606` is a part or a kit. `e6e17f14` (`name`) is joaat of the part
    model's name or of a kit name; a kit name is its left and right models'
    names without the side word (`comet7_exhaust_histr_19` for `…_lr_19` and
    `…_rr_19`; side words `_l_`, `_r_`, `_lr_`, `_rr_`). `37a6f9ef` is the
    slot: joaat of `bonnet`, `bumper_f`, `bumper_r`, `exhaust`, `spoiler`,
    `splitter`, `diffuser`, `towing`, `chassis`, `roof` (`bc96a7e5`) or
    `livery` (`805d8f10`). `7e5d1612` (`elements`) names `77f2b89e`
    records; `61e1e572` is a list of hashes.
  - `77f2b89e` is one model of a part: the low u32 of u64 `079baba7` is
    joaat of the bone the model is authored around (`chassis` `d418625e`, the
    car's own space; `boot`, `bumper_f`, `mod_spoiler_attach`,
    `mod_spoiler_attach01`, `mod_attach_exhaust_lr` / `_rr`); `2b490ab7`
    lists the stock bones it replaces (a bumper lists `bumper_f` and its
    indicator bones); `8605bf0e` is a 64-bit value per model, meaning
    unknown.
  - One sports car has 123 records, 31 with elements, 43 elements for its 43
    part models; 323 of 373 vehicles have parts (7,915 records with
    elements). Open: slots `46268fee`, `9bbfb64c`, `ac89920d`, `5a4394ab`
    and others.
- **Material set: located, not decoded** (2026-11-23; high that this is the
  vehicle's material set, since the counts 41 and 256 match the layer
  records and the 0–255 part ids in the body's vertex colour alpha; low on
  every field's meaning):
  - A configuration's part list (`e89b8540`, 75–150 joaat names) is mostly
    not models: on one car 4 of 75 are records in its own pack, and the other
    71 are defined by name in none of the 327 vehicle resources.
  - Mod records with no elements sit in slots `46268fee` (37 on that car),
    `9bbfb64c` (27), `ac89920d` (9) and `livery` (4). Their `61e1e572`
    lists name `44cc9d8c` records (41 on that car): name, data (a hash; 41
    distinct), layer (tag 1 on 37, tag 0 on 4) and layout (`91dc6b5f` on all;
    7,908 occurrences across packs, 1,611 in `data.rpf`).
  - The data hashes are defined in a **companion 0x7a resource** per
    vehicle: 41 × `b07176ab` (one per `44cc9d8c`; each with `09bd7c2c`, 32 ×
    `fe88b23d` {`c6a46bcd` hash, floats}, three lists of `825d57ab`, a list
    `fc07418c` and two 64-bit values); three tables of exactly 256 entries
    (`f5908771`: six hashes and three 16-byte fields that look like half
    floats; `b5feb11f`; `3d9aef8b`); 3,936 `825d57ab` (41 × 96), 1,312
    `fe88b23d` (41 × 32) and a root `f14244fd`.
  - Open: how a part id picks a preset, where the table hashes become
    colours, and how a vehicle links to its companion resource.
- **Convertible roof** (2026-11-23, high): as in GTA V,
  `animConvRoofDictName` = joaat(`va_<model>`) and `animConvRoofName` =
  joaat(`roof`), lower-case (`va_sentinelclassic2` = `180f9ebf`, `roof` =
  `bc96a7e5`); the named dictionary and clip exist
  ([§17](17-other.md#convertible-roof-clips)). A non-convertible has the
  same value in both fields. `animConvRoofWindowsAffected` is a 4-element
  array on the convertibles looked at; how it is used is unknown.
- **Colours** are all dark (`phoenix`: `ff091f30`, `ff122214`), so they are
  probably linear. Converted from linear to sRGB they give plausible paints
  (#356278 steel blue, #4b664f sage). Inferred; not checked in game.
- **Colour lists** (corrected 2026-11-21; earlier read as 100-colour
  palettes): colours repeat, e.g. `phoenix` has 400 entries with 10
  distinct colours, so the repeat count reads as a draw weight (inferred).
  They match the spawn-configuration body colours in only 168 of 4,024
  configurations, so they are a separate random-paint pool.
- **Wheel scale**: the shared rim models have radius 0.5 and the shared
  tyre 0.875 (width 0.585). Scaling both by **1.625 × `wheelScale`** (front
  and rear separately) fits `phoenix` (0.37 at `wheelScale` 0.2277), `mesa`
  and `ridgway` by eye. That factor is fitted, not decoded. It puts the
  `phoenix` rims at about 14.6 inch, close to a real car's 15 inch, so the
  earlier reading of `wheelScale` as the rim radius in metres (a dirt bike
  has 0.2538 / 0.2234, which would be 21 / 18 inch) is doubtful.
  *Later measurement (2026-11-23) withdraws the 1.625 factor and first read*
  `wheelScale` / `wheelScaleRear` as the **rim radius in metres** (now
  **low**, see the arch check below) (0.217 on a BMX with 20-inch wheels, 0.303 on a
  cruiser bicycle, 0.252 / 0.195 and 0.254 / 0.223 on dirt bikes, 0.249 on a
  supercar, 0.343 on an armoured truck). Every shared `rim_*` model has a
  radius of exactly 0.5 (bounding box, 500+ models; high), which suggested a
  rim scale of `wheelScale` / 0.5 (not established as the game's rule);
  tyre models have an outer radius of 0.54 to 1.19 at that unit scale. Over
  203 `_hi` vehicles, tyre radius × 2 × `wheelScale`
  puts the tyre bottom a median 0.22 m below the body's lowest point (one
  case 4 cm above). Helicopters have `wheelScale` 0.05–0.08, no
  configuration with a rim, and `wheel_*` bones parented to skid bones.
  **Arch check** (2026-11-23, 213 `_hi` vehicles; arch radius = the nearest
  body vertex above the axle within 12 cm of the wheel plane): with scale 2 ×
  `wheelScale` the tyre's outer radius is a median 0.96 of the arch radius
  (10th–90th percentile 0.83–1.42), so on about half the fleet the tyre is
  as big as the arch; per tyre model the median is 0.92–1.11
  (`tyre_car_001` 0.96 over 92 vehicles, `tyre_lowpro_001` 1.11 over 12,
  `tyre_truck_001` 1.09 over 29). Reading `wheelScale` as the tyre's outer
  radius instead gives a median arch / `wheelScale` of 1.65, too small. A
  single factor of about 1.7 fits ordinary cars but not sports cars and
  trucks. A dirt bike's front and rear values differ (0.252 / 0.195) with
  both axles at the same height, so there `wheelScale` is not the outer
  radius. **Neither reading fits; the real rule is unknown.** The
  `tyre_profile` / `rim_scale` bones below suggest the tyre's outer radius
  is set separately from the rim.
  **Superseded by stored wheel discs** (2026-11-23): the tyre's outer
  radius and half width are stored as collision discs in the vehicle
  fragment ([§13.4](13-collision.md#vehicle-fragment-bounds-and-wheel-discs);
  high). `wheelScale` ÷ that radius runs 0.29–2.08 over the fleet, median
  0.70 (mostly 0.6–0.9), so `wheelScale` is not the tyre size; the rim
  radius remains the best guess (low). The rim's own radius is not found in
  the data.
- **Shared wheel skeleton** (2026-11-23; bones high, role inferred): every
  `rim_*` and `tyre_*` model carries one 15-bone skeleton: `wheel_lf_link`,
  `rim_offset_mover_lf`, `rim_scale_lf`, `rim_dmg_1`–`5_lf`,
  `rim_non_dmg_lf`, `tread_profile_inside` / `outside_lf`,
  `tyre_profile_inside` / `outside_lf`, `wheel_width_mover_lf` and
  `rim_inside_edge_lf`. Rims skin to the `rim_*` bones, tyres to the four
  profile bones. The game probably sets wheel width and tyre profile through
  them (not shown); no wheel width was found in model info, configurations
  or the per-vehicle singleton structs.

## Open questions

- Secondary and pearl colours, liveries; whether the colours are linear.
- What `wheelScale` measures and how the game sets the rim radius and
  tyre profile (the tyre's outer radius and width are the wheel discs): what
  drives the shared wheel skeleton's width and profile bones.
- The shader constant buffer.
- The `19df25d7` palette lookup, which two-layer materials are painted, and the channel layout of the layer maps.
- Where the rim colour comes from: vehicle configurations store one body
  colour; variation sets have a list (`0bf8dac6`) parallel to the paint
  list, holding 64-bit hashes (4 distinct in the one resource looked at),
  meaning unknown.
- Spawn configurations list about 100 part joaats; only rim, tyre, caliper, brake disc and one damage part resolve, and 0 of 218 are bone names.


---

← [10. Interiors (MLO)](10-interiors.md) · [Index](README.md) · [12. Material classes and texture slots](12-materials.md) →
