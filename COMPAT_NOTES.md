# VRAM Write Filtering + FF7/LoD Compat Mode — Branch Notes

Custom DuckStation branch (`vram-write-filtering`) built on top of upstream's **Filter Framebuffer
Uploads** feature (upstream commits `ac0e3a5c` and `aaa8a68d`, September 2026). Upstream now
filters pre-rendered backgrounds itself; this branch only adds the pieces upstream does not have.

Maintained with the help of Claude (Anthropic). This file documents everything needed to
understand and rebase the branch without any other context. New pushes to the branch build
automatically via the GitHub Actions workflow in `.github/workflows/windows-x64-custom.yml`
(artifact `duckstation-windows-x64-vram-write-filtering`, downloadable from each Actions run for
90 days).

## What comes from upstream (not part of this branch any more)

- Setting `GPU / FilterFramebufferUploads` (+ `FilterFramebufferUploadsMinimumWidth/Height`),
  checkbox *"Filter Framebuffer Uploads"* in Graphics → Rendering → Advanced (Qt and Big Picture).
  Applies the **sprite texture filter** to CPU→VRAM uploads at least as large as the minimum size.
  Requires resolution scale > 1 and a sprite filter other than Nearest.
- Gamedb trait `FilterFramebufferUploads` / setting `gpuFilterFramebufferUploads: WxH` that forces
  it on per game (Resident Evil 1-3, Dino Crisis 2, Evil Dead, ...).
- The VRAM write shader keeps the "canonical" (top-left) subpixel of every block bit-exact so
  palettes, textures and 24-bit scanout stay intact. Savestate loads and full VRAM re-uploads are
  never filtered.

The old branch-specific `GPU / FilterVRAMWrites` setting no longer exists; tick *Filter
Framebuffer Uploads* instead.

## What this branch adds on top

All of it is keyed off the upstream option: it is active only when `FilterFramebufferUploads` is
enabled, the resolution scale is above 1x and the sprite filter is not Nearest
(`GetFramebufferUploadFilter()` in `gpu_hw.cpp`, cached in `m_framebuffer_upload_filtering`).

### 1. Exact VRAM readback (`GenerateVRAMReadFragmentShader`, `point_sample`)

Upstream's readback shader box-filters the upscaled VRAM block, so CPU reads of a filtered area
would return averaged filtered colors. With the option on, the readback point-samples the block
corner (kept bit-exact by upstream's write shader), so CPU-visible VRAM round-trips stay exact.

### 2. 24-bit display filtering (`GenerateDisplay24FilterFragmentShader`, `m_display_24bit_filter_pipeline`)

24-bit displays (FMVs, static images) are extracted at 1x by DuckStation. A separate pass in
`UpdateDisplay` upscales the extracted texture with the sprite filter into
`m_display_filter_texture`, skipping interlaced / line-skip modes. Post-processing then runs on
the filtered texture.

### 3. FF7 / Legend of Dragoon compat mode (gamedb-gated)

Only active when a game has the `DisableSpriteTextureFiltering` gamedb trait (FF7 and Legend of
Dragoon, all discs/regions) **and** `FilterFramebufferUploads` is enabled with a non-Nearest
sprite filter. `game_database.cpp` then skips the trait's sprite-filter disable and sets the
runtime-only flag `Settings::gpu_sprite_nearest_coverage` (not persisted; reset in
`Settings::Load`). `gpu_hw.cpp` passes it into `BatchFragmentShaderSelector` as
`filter_nearest_coverage` / `filter_chroma_key` (macros `FILTER_NEAREST_COVERAGE` /
`FILTER_CHROMA_KEY`), **only for opaque render passes** (`TransparencyDisabled` / `OnlyOpaque`);
semitransparent passes (FF7's dithered shadow layers, glows) keep stock filtering.

Why: FF7/LoD compose fields from layered tiles whose textures contain **chroma matte garbage**
(pure green, blue, cyan, red, magenta; varies per scene) next to real content. Invisible with
nearest sampling (draw order covers it), but any texture filter blends it into content edges as
colored dashes/lines, and silhouette erosion/growth can reveal or extend it. This is exactly why
upstream's gamedb disables sprite filtering for these games.

Mechanisms (all in the `TEXTURE_FILTERING` branch of the batch fragment shader,
`gpu_hw_shadergen.cpp`), each one exists because a specific artifact was observed:

| Mechanism | Prevents |
|-----------|----------|
| Chroma key: saturated cold/pure hues (5 families: green, blue, red, cyan, magenta; dominant ≥0.55, others ≤0.28, ≥2× dominance) treated as transparent in filter taps (`SampleFromVRAM` wrapper over `SampleFromVRAMRaw`) | Colored dashes/tints along content edges |
| Exact-coverage base: discard iff exact center texel (`ncol`) is transparent — silhouettes can never erode | Colored dots from occluded matte revealed at concave silhouette spots |
| Supported outward growth: center-transparent pixels kept only if `ialpha ≥ 0.45` and filtered luminance sum ≥ 0.3 | Black dots from dark art outlines being extended onto layers behind, while keeping smooth cut-out silhouettes |
| Luminance floor: center-opaque pixels whose filtered color sum < 0.15 (near-pure black) and < 50% of the exact texel's sum snap back to the exact color | Dark hairline seams along layer cuts |
| Opaque writes (`ialpha = 1`, `texcol.a = ncol.a`) | Filtered edges alpha-blending over occluded matte |

Tunable thresholds if a game shows residuals: 0.15/0.5 (luminance floor), 0.45/0.3 (growth
support), 0.55/0.28/2× (chroma families).

## Files touched (conflict hotspots for rebases)

- `src/core/gpu_hw_shadergen.cpp` / `.h` — compat blocks in the batch fragment shader, the
  `SampleFromVRAM` wrapper, `POINT_SAMPLE` in the readback shader, the 24-bit filter shader,
  two extra bits in `BatchFragmentShaderSelector`.
- `src/core/gpu_hw.cpp` / `.h` — `GetFramebufferUploadFilter`, `m_framebuffer_upload_filtering`,
  compat selector bits in `CompilePipelines`, readback/24-bit pipelines in
  `CompileResolutionDependentPipelines`, the 24-bit pass in `UpdateDisplay`.
- `src/core/settings.h` / `settings.cpp` — `gpu_sprite_nearest_coverage`.
- `src/core/system.cpp` — settings-change detection line.
- `src/core/game_database.cpp` — `DisableSpriteTextureFiltering` trait handling.
- `src/core/shader_cache_version.h` — bumped (42).
- `src/duckstation-qt/graphicssettingswidget.cpp` — help text of the upstream checkbox.
- `.github/workflows/windows-x64-custom.yml` — CI (x64 only).

## How to update from upstream

```bash
git clone <url-of-this-repo>
cd <repo-dir>
git remote add upstream https://github.com/stenzek/duckstation.git

git fetch upstream
git checkout vram-write-filtering
git merge upstream/master        # or: git rebase upstream/master
# resolve conflicts (most likely in gpu_hw_shadergen.cpp; re-apply the blocks guarded by
# FILTER_NEAREST_COVERAGE / FILTER_CHROMA_KEY, the SampleFromVRAM wrapper and POINT_SAMPLE)
git push origin vram-write-filtering
```

If upstream renames `SampleFromVRAM` or restructures the batch fragment shader, re-apply the
concepts from the table above rather than the literal diff.

## Verifying after an update

Test with FF7 (SCES-20900 etc.), settings: `SpriteTextureFilter = xBR`,
`FilterFramebufferUploads = true`, resolution scale ≥ 4x. Check: (1) pre-rendered fields show
**no** colored dashes/dots or dark hairlines along layer contours; (2) cut-out objects (foliage,
cursors, exit arrows) keep smooth silhouettes; (3) other games (Resident Evil 2 backgrounds,
FMVs) still get filtered; (4) save states and 3D textures stay intact.

## Known limitations

- Compat mode trades a small amount of edge smoothing at layer cuts for artifact-free output;
  it is intentionally FF7/LoD-only via the gamedb trait.
- Semitransparent passes keep stock filtering by design.
- Upstream's minimum upload size defaults to 1x1 for the global option; raise it in the advanced
  settings if a game shows corrupted textures.
