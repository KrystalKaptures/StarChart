# StarChart — a real-sky planetarium in Godot

## Context

`/Users/kkhlusko/Projects/Games/StarChart` is empty. We're building a Godot 4 planetarium
that shows the real sky as seen **from the user's house**: real stars from a real catalog,
the real Moon with its real phase, the real planets, constellation figures, and deep-sky
objects — with a time control that can scrub forward and backward.

Decisions already made with the user:

- **Observing site**: the house is the hardcoded startup default, stored in a config file,
  with an in-game settings panel to change latitude / longitude / elevation / timezone.
- **Scope**: sandbox planetarium first (free look, time scrubbing, click-to-identify, info
  panels). No objectives or scoring yet — game modes layer on later without rework.
- **Horizon**: flat ground plane with cardinal markers; sky colour and twilight driven by
  the true Sun altitude. Not a photographic silhouette of the actual yard.
- **Bodies in v1**: all four — Moon with real phase, Sun + planets, constellation
  lines/names, and deep-sky objects.

Local toolchain is confirmed: Godot **4.7.1.stable** at `/usr/local/bin/godot`, Python
**3.9.6** (no third-party packages — the data tool will use only the standard library).

**One input still needed:** the house's latitude, longitude and elevation. The plan writes
`config/location.cfg` with a clearly-marked placeholder so everything else can be built and
tested; drop the real numbers in that one file (or via the settings panel) when ready.

## Real data: sources verified by download

| What | Source | Verified |
|---|---|---|
| Stars | `astronexus/HYG-Database` → `hyg/CURRENT/hygdata_v41.csv` | 34 MB, **119,626 stars**; 8,921 to mag 6.5, 25,792 to mag 7.5; 465 proper names. Columns include `rarad`/`decrad` (J2000 radians), `mag`, `ci` (B−V colour), `pmra`/`pmdec`, `hip`, `bayer`, `flam`, `con`, `spect`, `dist`. **CC BY-SA 4.0** |
| Constellations | `Stellarium/stellarium` → `skycultures/modern/index.json` | 206 KB; **88 constellations** with `lines` as HIP-number polylines (710 distinct HIP stars), 92 asterisms, 660 HIP-keyed `common_names`, and 781 IAU boundary `edges`. **GPL-2.0-or-later** |
| Deep sky | `astronexus/HYG-Database` → `misc/dso.csv` | 227,285 rows (98 % faint galaxies). Filtering to mag ≤ 12 yields **1,991 objects**, covering all 109 Messier plus the bright NGC/IC showpieces. Has `r1`/`r2`/`angle` for real apparent size and orientation. **CC BY-SA 4.0** |
| Moon / Sun / planets | *No download.* Analytic ephemerides implemented in GDScript | ELP2000-82B truncated (Meeus ch. 47) for the Moon; truncated VSOP87D for the Sun and planets |

Ephemerides are **computed, not tabulated**, so time travel works for any date instead of
being capped by a baked table. Target accuracy: Moon ~10″, planets ~1″ — far below one
screen pixel at any sane field of view.

Two licence notes worth acting on, not blockers:

- HYG is CC BY-SA 4.0: requires attribution and share-alike **on the data file**. An
  in-game Credits screen plus a `data/LICENSES.md` satisfies this.
- Stellarium's constellation data is GPL. Irrelevant for a personal build; if this is ever
  distributed under a non-GPL licence, the swap is to rebuild the line figures from the
  public-domain IAU/Delporte boundaries instead. Keep the importer's constellation step
  isolated so that swap stays cheap.

The Stellarium IAU boundary edges are epoch **B1875** — the importer must precess them to
J2000, or constellation borders will sit visibly crooked against the stars.

## Architecture: one matrix per frame

The central performance idea. Stars are stored as J2000 unit vectors and **never moved**.
Everything between the catalogue and the sky is a rotation, so it composes into a single
`Basis` applied to one parent node each frame:

```
catalog J2000 (ICRS) unit vectors        [static, built once]
  → frame bias + IAU 2006 precession + IAU 2000B nutation   → true equator of date
  → Earth rotation by GAST + observer longitude             → hour-angle frame
  → observer latitude                                       → local horizon (alt/az)
```

`StarField`'s `MultiMeshInstance3D` lives under an `EquatorialFrame: Node3D`; per frame we
set `EquatorialFrame.transform.basis` once. 120 k stars cost zero per-star CPU work.

Two things are *not* rotations and get handled separately:

- **Proper motion** is per-star. Negligible inside a century, so rebuild the star buffer
  only when the date drifts more than ~100 years from J2000 (and do it off the main thread).
- **Atmospheric refraction** is a per-altitude nonlinearity. Apply it on the CPU for the
  Sun, Moon, planets and DSOs only; it is below a pixel for stars above ~15° and not worth
  bending the star field for.

**World frame** (document it in code, it is the #1 source of sign bugs): `+X` = East,
`+Y` = Up, `−Z` = North, matching Godot's "forward is −Z". `Camera3D` sits at the origin;
looking around is azimuth/altitude on the camera, never on the sky.

**Moon topocentric parallax is mandatory, not a refinement.** The Moon sits up to ~1° away
from its geocentric position as seen from the ground — far more than the Moon's own
diameter. Geocentric-only would be a visible, embarrassing error.

## Star rendering

A single `MultiMeshInstance3D`: 2-triangle `QuadMesh`, `use_colors = true`,
`use_custom_data = true`, one instance per star.

- Per-instance **transform** encodes the star's J2000 direction.
- Per-instance **colour** comes from B−V → RGB, converted on the CPU in the import tool.
- Per-instance **custom data** carries `(apparent_magnitude, catalog_index, 0, 0)`.
- The shader billboards manually (strip rotation from `MODELVIEW_MATRIX` — Godot has no
  billboard render mode) and maps magnitude → quad size and brightness, scaled by current
  FOV so stars grow slightly when zooming in, the way Stellarium does.

Confirmed against the Godot 4 spatial-shader reference: `INSTANCE_CUSTOM` (vec4) is
available in the vertex shader, and `POINT_SIZE`/`POINT_COORD` exist as a fallback path.
Quads are preferred over point sprites — they give proper antialiased round glow.

**Load speed**: building 120 k instances via `set_instance_transform` in a GDScript loop is
too slow. Assign `MultiMesh.buffer` (a `PackedFloat32Array`) in **one** call instead, built
from `FileAccess.get_buffer(...).to_float32_array()`. Verify the buffer's float layout
empirically with a 2-instance round-trip before trusting it — if the layout fights back,
fall back to the `set_instance_*` loop on a background thread with a loading screen.

**Click-to-identify** must not raycast 120 k quads. Convert the click to a direction, invert
the frame matrix into J2000, then query a precomputed RA/Dec bucket grid (e.g. 64 × 32) and
pick the nearest star within an angular tolerance, weighted by brightness.

## Project layout

```
StarChart/
  project.godot
  tools/build_data.py        # stdlib only: csv, json, struct, gzip, urllib
  data_src/                  # raw downloads, gitignored
  config/location.cfg        # home default; user:// override wins
  scenes/
    Main.tscn  Sky.tscn
    ui/{HUD,TimeControl,InfoPanel,SettingsPanel,Credits}.tscn
  scripts/
    core/{SkyTime,Observer,Settings}.gd      # SkyTime + Observer are autoloads
    astro/{julian,sidereal,precession,nutation,frames,refraction,
           sun,moon,vsop87,magnitudes,bodies}.gd
    sky/{StarField,ConstellationLines,SolarSystem,DsoLayer,Horizon,SkyPicker}.gd
    ui/*.gd
  shaders/{star_points,sky_gradient,dso_sprite,moon}.gdshader
  assets/moon_albedo.jpg
  data/                      # generated, committed, + LICENSES.md
  tests/                     # reference-value checks
```

## Generated data files

`tools/build_data.py --download` fetches to `data_src/`, then emits to `data/`:

- **`stars.bin`** — header + fixed records: `x,y,z` (f32 J2000 unit vector), `mag` (f32),
  `r,g,b,a` (u8 from B−V), `hip` (i32), `pmra,pmdec` (f32 rad/yr). 32 B/star ≈ 3.8 MB.
- **`star_index.bin`** — the RA/Dec bucket grid for picking.
- **`star_names.json`** — `hip → {proper, bayer, flam, con, spect, dist, lum}` for the few
  thousand stars that carry any designation; loaded for the info panel, not for rendering.
- **`constellations.json`** — 88 × `{abbr, name, lines: [[hip,…]], boundary: [[ra,dec],…]}`
  with boundaries already precessed B1875 → J2000.
- **`dso.json`** — the mag ≤ 12 cut, force-including every Messier object regardless of a
  blank `mag` field.
- **`vsop87_*.bin`** — the truncated planetary series.

## Build order

1. **Scaffold + data pipeline.** `project.godot`, `tools/build_data.py`, assert the record
   counts from the table above so a silently-changed upstream file fails loudly.
2. **Time and frames core.** `julian.gd` (calendar ↔ JD, ΔT via the Espenak–Meeus
   polynomial, UT1/TT, local time + timezone), `sidereal.gd` (GMST → GAST),
   `precession.gd`, `nutation.gd` (IAU 2000B, 77 terms), `frames.gd` (composite basis and
   its inverse), `refraction.gd` (Bennett). Unit-test this layer before anything renders.
3. **Star field + view.** `StarField.gd` and `star_points.gdshader`, mouse-look camera with
   FOV zoom, `Horizon.gd` ground plane with cardinal markers, `sky_gradient.gdshader`.
4. **Solar system.** `sun.gd`, `moon.gd` (with topocentric parallax, illuminated fraction,
   libration), `vsop87.gd`, `magnitudes.gd`, unified behind `bodies.gd`. Render the Moon as
   a textured `SphereMesh` lit by a `DirectionalLight3D` aimed from the computed Sun
   direction — real phase and terminator fall out for free, no phase shader needed.
5. **Constellations + DSO + labels.** `ConstellationLines.gd` as one `ArrayMesh` of
   `PRIMITIVE_LINES` under the same `EquatorialFrame`; `DsoLayer.gd` with real angular
   sizes; screen-space labels that cull when off-view or crowded.
6. **UI.** `TimeControl` (play/pause, speed from real-time to 1 yr/s in both directions,
   scrub, date picker, "Now"), `InfoPanel` driven by `SkyPicker`, `SettingsPanel` for the
   observing site, `Credits` carrying the required attributions.
7. **Polish.** Twilight colour ramp, light-pollution / limiting-magnitude slider, star glow
   and scintillation, equatorial and alt-az grid overlays. Optional: an ESO GigaGalaxy
   Milky Way panorama (CC BY 4.0) mapped into the equatorial frame.

Milky Way, satellite/ISS passes, eclipse and transit prediction, and the game modes are
deliberately out of v1. The architecture above leaves room for each.

## Verification

**Reference-value tests** (`tests/`, run headless with
`godot --headless --path . --script tests/run_all.gd`). Each is a published worked example
with a known answer:

- `JD(2000-01-01 12:00 TT) == 2451545.0`
- GMST, Meeus 12.a: 1987-04-10 0h UT → `13h 10m 46.3668s`
- Sun, Meeus 25.b: 1992-10-13 0h TD → apparent λ ≈ `199.90895°`
- Moon, Meeus 47.a: 1992-04-12 0h TD → λ = `133.162655°`, β = `−3.229126°`,
  Δ = `368409.7 km`
- Venus/VSOP87, Meeus 32.a + 33.a

**End-to-end cross-check.** Pick a handful of dates and compare computed alt/az, magnitude
and Moon phase against JPL Horizons (fetchable at verification time) for the configured
site. This is the test that catches sign errors the unit tests sail past.

**Geometric smoke tests** — fast, and they catch whole-frame mistakes instantly:

- Polaris sits due north at an altitude equal to the observer's latitude.
- At the equinox the Sun rises due east and sets due west.
- Local midnight puts the Sun at its lowest altitude of the day.
- Scrubbing one sidereal day returns the star field to the same pixels.

**Visual check.** Launch with `godot --path /Users/kkhlusko/Projects/Games/StarChart`, then
confirm by eye that Orion, the Big Dipper and Cassiopeia have the right shapes and sit in
the right places for the current date and time — mirrored or rotated constellations are the
unmistakable signature of a frame bug.
