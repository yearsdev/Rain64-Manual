# Rain64 Retro Shader

A drop-in retro-console post-process for Unreal Engine. Rain64 recreates the look
of mid-90s 3D consoles — low internal resolution, 15-bit color with ordered
dithering, the era's signature de-dither softness, composite-video artefacts, and
CRT scanlines — as a custom render path injected after the tonemapper.

**No material changes. No content migration.** Enable the plugin, pick a preset,
and your existing project renders like 1996.

## Features

- **Low internal resolution** — native buffer presets (256×224, 320×240, 640×480)
  or any custom width, with pixel-aspect and window-relative scaling options.
- **Render-scale mode** — rasterize the 3D scene itself at a fraction of display
  resolution: a real GPU saving *plus* authentic native-resolution aliasing.
- **15-bit color quantization** with ordered dithering — magic-square (the classic
  hardware pattern), Bayer 4×4, or static noise; adjustable strength and cell size.
- **Indexed palette mode** — snap the whole image to an art-authored palette
  (up to 64 colors), dithered between the two nearest entries.
- **Ten built-in palettes** — from a 16-color horror ramp to full 64-color sets,
  hot-swappable at runtime with one console command (`r.Rain64.PalettePreset`).
- **World-space dither** (experimental) — anchor the dither pattern to 3D surfaces
  via scene depth instead of the screen.
- **De-dither / AA filter** — the horizontal-biased tent blur that dissolved dither
  into smooth gradients on real hardware, with stray-pixel cleanup ("divot") and an
  optional unsharp-mask sharpen.
- **Neutral tonemapper** — flattens UE's ACES film curve so color reaches the chain
  direct and punchy, like a raw framebuffer.
- **Highlight rolloff & ceiling** — hue-preserving soft shoulder for bright values,
  plus a hard cap so all-white surfaces read as light grey instead of blasting.
- **NTSC composite artefacts** — a cheap single-pass mode, or a full two-pass QAM
  encode/decode with animated dot crawl.
- **CRT display stage** — resolution-independent scanlines and an aperture-grille
  mask, computed in output space like a real display.
- **Three one-command presets** and a full Project Settings page that persists your
  custom look to config.

## Requirements

- **Unreal Engine 5.4, 5.5, or 5.6.** Shader Model 5+.
- **Platforms: Windows 64-bit** — precompiled binaries included; developed and
  runtime-tested on Windows/DirectX. Mobile and VR are not supported.
- Works in Blueprint-only or C++ projects (full source included).
- SDR output only — disable HDR display output when using the plugin.

## Installation

1. Install to your engine from Fab / the Epic Games Launcher.
2. In your project: **Edit → Plugins**, search "Rain64", enable, restart the editor.

## Quick start

1. Press **Play**. By default the effect renders in game and PIE only — your editor
   viewports, material previews, and thumbnails stay clean. To preview the look in
   the level-editor viewport too, enable **Apply In Editor** in Project Settings →
   Plugins → Rain64 Retro Shader (or `r.Rain64.ApplyInEditor 1`).
2. Try the presets from the console:
   - `r.Rain64.Preset 1` — **Authentic**: native 320×240, full dither and de-dither
     blur, NTSC composite, 240p scanlines. The real thing.
   - `r.Rain64.Preset 2` — **Enhanced**: 720px internal, half the blur, light
     composite, no scanlines. The look as you *remember* it.
   - `r.Rain64.Preset 3` — **Modern**: ~960px effective, subtle Bayer dither,
     highlight rolloff, gentle sharpen. Retro flavor for projects that want to stay
     crisp.
3. Try the built-in palettes (indexed-color mode):
   - `r.Rain64.ColorMode 1` switches to palette snapping, then
     `r.Rain64.PalettePreset gloom16` (or `ember24`, `arcade30`, `dread32`,
     `frontier46`, `vivid60`, `painter64`, `wilds64`, `desktop64`, `cube64`) loads
     a palette live. Run `r.Rain64.PalettePreset` with no argument to list them.
4. Presets and palette presets are runtime-only. To make a look stick, dial it in
   **Project Settings → Plugins → Rain64 Retro Shader** — every property mirrors a
   console variable and saves to your project's `DefaultGame.ini`.

## How it works

A single `SceneViewExtension` hooks the frame right after the tonemapper and runs
a small chain of passes, all at the low internal resolution:

1. **Downscale** — box average (smooth) or point sampling (preserves aliasing).
2. **Grade + quantize** — saturation/contrast/highlight grade, then color
   quantization with ordered dither (or palette snap).
3. **De-dither filter** — the tent blur + divot + optional sharpen.
4. **Composite** (optional) — NTSC chroma bleed, fringing, dot crawl.
5. **Present** — point-upscale to the viewport, output gamma, scanlines/mask.

Because the hook runs after the tonemapper, the chain operates on final
display-encoded LDR color — what a console actually wrote to its framebuffer.

## Console variables

Everything is driven by `r.Rain64.*` console variables; the Project Settings page
mirrors them. Defaults below are the plugin's out-of-the-box (Authentic-style) look.

| CVar | Default | Notes |
|---|---|---|
| `r.Rain64.Enable` | 1 | Master on/off for the render path. |
| `r.Rain64.ApplyInEditor` | 0 | Also run in level-editor viewports. Game/PIE always render; preview scenes never do. |
| `r.Rain64.Preset` | — | Command: apply preset 1 (Authentic), 2 (Enhanced), or 3 (Modern). Runtime-only. |
| `r.Rain64.PalettePreset` | — | Command: load a built-in indexed palette (no argument lists all ten). Runtime-only. |
| `r.Rain64.ResPreset` | 2 | 0 = Custom (use Width), 1 = 256×224, 2 = 320×240, 3 = 640×480. |
| `r.Rain64.Width` | 320 | Custom-mode internal width; height follows the viewport aspect. |
| `r.Rain64.PixelAspect` | 1.0 | Custom-mode height multiplier (<1 = taller pixels). |
| `r.Rain64.ResScale` | 0.0 | Custom-mode window-relative width. 0 = absolute Width; >0 = fraction of viewport. |
| `r.Rain64.RenderScale` | 0.0 | Rasterize the 3D **scene** at this fraction of display res. 0 = off. The one setting that saves GPU — see Performance. |
| `r.Rain64.DownscaleFilter` | 0 | 0 = 2×2 box (smooth), 1 = point (keeps native aliasing; pair with RenderScale). |
| `r.Rain64.Dither` | 1 | 0 = hard quantize (banding), 1 = ordered dither. |
| `r.Rain64.DitherPattern` | 0 | 0 = magic square, 1 = Bayer 4×4, 2 = static noise. |
| `r.Rain64.DitherStrength` | 1.0 | Dither amplitude: 1 = full pattern, 0 = plain banding. |
| `r.Rain64.DitherScale` | 1.0 | Dither cell size in texels (≥1). |
| `r.Rain64.DitherSpace` | 0 | Experimental. 0 = screen space, 1 = world space (pattern sticks to surfaces). |
| `r.Rain64.DitherWorldScale` | 4.0 | World-space mode: world units (cm) per dither cell. |
| `r.Rain64.ColorMode` | 0 | 0 = per-channel quantize, 1 = snap to the indexed Palette (Project Settings). |
| `r.Rain64.ColorLevels` | 31 | Quantize levels per channel. 31 = 15-bit color, 255 = no visible banding. |
| `r.Rain64.Saturation` | 2.0 | Grade saturation, before quantize. |
| `r.Rain64.Contrast` | 1.0 | Grade contrast around mid-grey, before quantize. |
| `r.Rain64.HighlightRolloff` | 0.0 | 0 = hard clip (era-accurate), 1 = full soft shoulder below the ceiling. |
| `r.Rain64.HighlightWhitePoint` | 2.0 | Luminance that maps to the ceiling. Lower = punchier, higher = softer. |
| `r.Rain64.HighlightCeiling` | 1.0 | Brightest output luminance. Lower to make pure-white surfaces settle to grey. |
| `r.Rain64.VIFilter` | 1 | Enable the de-dither/AA tent filter. Must be on for Sharpen. |
| `r.Rain64.Divot` | 1 | Median-3 stray-pixel cleanup at edges. |
| `r.Rain64.VIFilterStrength` | 1.0 | 1 = full era-accurate blur, 0 = crisp, >1 = softer. |
| `r.Rain64.Sharpen` | 0.0 | Unsharp mask after the filter. ~0.2–0.6 = subtle to strong. |
| `r.Rain64.Gamma` | 1.0 | Extra display gamma at the present stage. |
| `r.Rain64.NeutralTonemapper` | 1 | Flatten UE's ACES filmic tonemapper for direct framebuffer-style color. |
| `r.Rain64.Composite` | 0.5 | Composite-video strength. 0 = off (stage skipped). |
| `r.Rain64.CompositeMode` | 2 | 1 = cheap single-pass, 2 = full two-pass NTSC with animated dot crawl. |
| `r.Rain64.CompositeBleed` | 1.5 | Chroma smear width in texels. |
| `r.Rain64.CompositeArtifact` | 0.15 | Dot-crawl / rainbow fringe (single-pass mode only). |
| `r.Rain64.Scanlines` | 0.35 | CRT scanline darkening. 0 = off. |
| `r.Rain64.ScanlineCount` | 240 | Scanlines across the output height, independent of internal res. |
| `r.Rain64.Mask` | 0.0 | Aperture-grille RGB mask strength. |

The indexed **Palette** (used when `ColorMode` = 1) holds up to 64 colors. Author
your own in Project Settings, or load one of the ten built-ins with
`r.Rain64.PalettePreset` — sets larger than 64 entries reduce by even stride on
apply. While the palette is empty, palette mode falls back to per-channel.

## Performance

The post chain itself is *overhead*, not a saving — it renders your scene normally,
then downsamples. It's a handful of cheap low-resolution passes, but it won't make
a heavy scene faster on its own.

The one real GPU saving is **`r.Rain64.RenderScale`**: it installs a low primary
screen percentage so the expensive per-pixel scene rendering happens at a fraction
of display resolution. As a bonus it's *more* authentic — you get true native-res
edge aliasing instead of a smoothed supersampled image. For the full effect, pair:

```
r.Rain64.RenderScale 0.4
r.AntiAliasingMethod 0
r.Rain64.DownscaleFilter 1
```

(No AA so the aliasing survives, point downscale so it isn't averaged away.)

## Tips & limitations

- **Screen-space UMG is not affected** — it draws after post-processing. To
  retro-fy UI, wrap widgets in a Retainer Box with a pixelate/dither effect
  material. World-space widget components *are* affected automatically.
- **Temporal upscalers (TSR / TAA upsample) are fully supported.**
- **SDR only** — the chain assumes standard display-encoded output.
- **Scene alpha is not preserved** through the chain (output alpha is 1).
- Point-sampled, low-resolution textures on your meshes complete the look — the
  plugin handles the framebuffer, but texture crunch is an art choice.
- Fixed resolution presets (256×224 etc.) stretch to fill the viewport; use Custom
  mode if you need the internal aspect to match your display exactly.

## Troubleshooting

- **"I see no effect in the editor viewport"** — by design; the effect renders in
  game/PIE. Enable **Apply In Editor** to preview it in the level editor.
- **"My material preview looks normal"** — also by design; preview scenes and
  thumbnails are always excluded.
- **"My preset choice didn't survive a restart"** — `r.Rain64.Preset` is
  runtime-only. Save looks via Project Settings.
- **"Everything got darker/brighter than vanilla"** — the neutral tonemapper
  intentionally removes UE's filmic curve. Compensate with `r.Rain64.Gamma`,
  `Saturation`, and `Contrast`, or set `r.Rain64.NeutralTonemapper 0`.

## Support

- Discord: https://discord.gg/RsA8K8uAvc
- Or open an issue at https://github.com/yearsdev/Rain64-Manual.

## Version history

See `CHANGELOG.md`, included with the plugin.

---

© YEARS Dev Ltd. Distributed under the Fab EULA.
