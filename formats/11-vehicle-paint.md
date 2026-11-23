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
| `e59aa857` model info | 327 | `gameName` (`120e5836`, joaat of the model name), `handlingId` (`ec41bd9c`), `txdName`, `wheelScale` (`933301be`), `wheelScaleRear` (`76706238`), `wheelType` (`a2e90c68`), `layout`, `HDTextureDist`, `lodDistances`, `vehicleClass`, `swankness`, `plateType`, `defaultBodyHealth` … (52 names resolve) |
| `51019114` configuration | 3,879 | `fb1f80ff` model (field name `handling`), **`5bdc13e3` body colour** (ARGB u32), `e89b8540` part models (joaat names, about 150: frame, bumpers, exhaust, `brakedisc_*`, `caliper_*`, and one `rim_*` and one `tyre_*` in 82 %) |
| `fc1c7000` variation set | 4,772 | `fb1f80ff` model (`handling`), **`8c866829` weighted paint-colour list**: 1 to 400 ARGB colours with repeats (2,246 sets; this is the pool of 17,632 colours), `59a59c3e` (`slots`) weighted part groups: `37a6f9ef` slot, `a12ff3a2` (`options`) list of {`8e6612bf` part, `dc732774` (`probability`) weight %} |

With the per-vehicle packs this covers 373 models (305 named), 4,063
configurations (3,339 with a rim and a tyre) and 2,377 colour lists. Handling is
a separate PSO ([§17](17-other.md#vehicle-handling)).

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

## Open questions

- Secondary and pearl colours, liveries; whether the colours are linear.
- The game's rule for sizing the shared rim and tyre from `wheelScale`
  (1.625 × is fitted); the tyre width.
- The shader constant buffer.
- The `19df25d7` palette lookup, which two-layer materials are painted, and the channel layout of the layer maps.
- Where the rim colour comes from: vehicle configurations store one body
  colour; variation sets have a list (`0bf8dac6`) parallel to the paint
  list, holding 64-bit hashes (4 distinct in the one resource looked at),
  meaning unknown.
- Spawn configurations list about 100 part joaats; only rim, tyre, caliper, brake disc and one damage part resolve, and 0 of 218 are bone names.


---

← [10. Interiors (MLO)](10-interiors.md) · [Index](README.md) · [12. Material classes and texture slots](12-materials.md) →
