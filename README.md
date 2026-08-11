# Rain64 Retro Shader

Post-process rendering plugin for Unreal Engine. It reproduces the output
characteristics of mid-1990s game consoles — low internal resolution, 15-bit
color with ordered dithering, de-dither filtering, NTSC composite-video
artefacts, and CRT scanlines — as a render path injected after the tonemapper
via a `SceneViewExtension`. Project materials and content are not modified.

## Features

- **Low internal resolution** — preset buffer sizes (256×224, 320×240, 640×480)
  or any custom width, with pixel-aspect and window-relative scaling options.
- **Render-scale mode** — rasterizes the 3D scene at a fraction of display
  resolution, reducing GPU cost and producing native-resolution edge aliasing.
- **15-bit color quantization** with ordered dithering — magic-square,
  Bayer 4×4, or static noise pattern; adjustable strength and cell size.
- **Indexed palette mode** — quantizes the image to a user-defined palette
  (up to 64 colors), dithering between the two nearest entries.
- **Ten built-in palettes** — 16 to 64 entries each, switchable at runtime with
  a console command (`r.Rain64.PalettePreset`).
- **World-space dither** (experimental) — anchors the dither pattern to 3D
  surfaces via scene depth instead of screen coordinates.
- **De-dither / AA filter** — a horizontally biased tent blur, with stray-pixel
  cleanup ("divot") and an optional unsharp-mask sharpen.
- **Neutral tonemapper** — replaces UE's ACES film curve with a neutral
  response so color reaches the chain unmodified.
- **Highlight rolloff and ceiling** — hue-preserving soft shoulder for bright
  values, and a configurable maximum output luminance.
- **NTSC composite artefacts** — a single-pass approximation, or a two-pass QAM
  encode/decode with animated dot crawl.
- **CRT display stage** — scanlines and an aperture-grille mask, computed in
  output space independent of internal resolution.
- **Three presets** and a Project Settings page that persists configuration to
  project config files.

## Requirements

- **Unreal Engine 5.4 – 5.8.** Shader Model 5+.
- **Platforms: Windows 64-bit** — precompiled binaries included; developed and
  runtime-tested on Windows/DirectX. Mobile and VR are not supported.
- Works in Blueprint-only or C++ projects (full source included).
- SDR output only — disable HDR display output when using the plugin.

## Installation

1. Install to your engine from Fab / the Epic Games Launcher.
2. In your project: **Edit → Plugins**, search "Rain64", enable, restart the editor.

## Quick start

1. Press **Play**. By default the effect renders in game and PIE only;
   level-editor viewports, material previews, and thumbnails are unaffected. To
   render in the level-editor viewport as well, enable **Apply In Editor** in
   Project Settings → Plugins → Rain64 Retro Shader (or `r.Rain64.ApplyInEditor 1`).
2. Apply a preset from the console:
   - `r.Rain64.Preset 1` — **Authentic**: 320×240 internal resolution, full
     dither and de-dither blur, NTSC composite, 240 scanlines.
   - `r.Rain64.Preset 2` — **Enhanced**: 720 px internal width, reduced blur,
     light composite, no scanlines.
   - `r.Rain64.Preset 3` — **Modern**: ~960 px effective width, low-strength
     Bayer dither, highlight rolloff, mild sharpen.
3. Load a built-in palette (indexed-color mode):
   - `r.Rain64.ColorMode 1` switches to palette snapping, then
     `r.Rain64.PalettePreset gloom16` (or `ember24`, `arcade30`, `dread32`,
     `frontier46`, `vivid60`, `painter64`, `wilds64`, `desktop64`, `cube64`)
     loads a palette at runtime. Run `r.Rain64.PalettePreset` with no argument
     to list them.
   - Alternatively, in Project Settings set **Color Mode** to Palette and choose
     an entry from **Load Built-In Palette** — it fills the Palette list with the
     preset's colors (editable afterwards) and saves with the project.
4. `r.Rain64.Preset` and the `r.Rain64.PalettePreset` console command are
   runtime-only. To persist a configuration, set it in **Project Settings →
   Plugins → Rain64 Retro Shader** — every property mirrors a console variable
   and saves to the project's `DefaultGame.ini`.

## How it works

A single `SceneViewExtension` hooks the frame after the tonemapper and runs a
chain of passes at the low internal resolution:

1. **Downscale** — box average (smooth) or point sampling (preserves aliasing).
2. **Grade + quantize** — saturation/contrast/highlight grade, then color
   quantization with ordered dither (or palette snap).
3. **De-dither filter** — the tent blur + divot + optional sharpen.
4. **Composite** (optional) — NTSC chroma bleed, fringing, dot crawl.
5. **Present** — point-upscale to the viewport, output gamma, scanlines/mask.

Because the hook runs after the tonemapper, the chain operates on final
display-encoded LDR color.

## Console variables

Everything is driven by `r.Rain64.*` console variables; the Project Settings
page mirrors them. Defaults below are the plugin's out-of-the-box values.

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
| `r.Rain64.RenderScale` | 0.0 | Rasterize the 3D **scene** at this fraction of display res. 0 = off. Reduces GPU cost — see Performance. |
| `r.Rain64.DownscaleFilter` | 0 | 0 = 2×2 box (smooth), 1 = point (keeps native aliasing; pair with RenderScale). |
| `r.Rain64.Dither` | 1 | 0 = hard quantize (banding), 1 = ordered dither. |
| `r.Rain64.DitherPattern` | 0 | 0 = magic square, 1 = Bayer 4×4, 2 = static noise. |
| `r.Rain64.DitherStrength` | 1.0 | Dither amplitude: 1 = full pattern, 0 = plain banding. |
| `r.Rain64.DitherScale` | 1.0 | Dither cell size in texels (≥1). |
| `r.Rain64.DitherSpace` | 0 | Experimental. 0 = screen space, 1 = world space (pattern anchors to surfaces). |
| `r.Rain64.DitherWorldScale` | 4.0 | World-space mode: world units (cm) per dither cell. |
| `r.Rain64.ColorMode` | 0 | 0 = per-channel quantize, 1 = snap to the indexed Palette (Project Settings). |
| `r.Rain64.ColorLevels` | 31 | Quantize levels per channel. 31 = 15-bit color, 255 = no visible banding. |
| `r.Rain64.Saturation` | 2.0 | Grade saturation, applied before quantize. |
| `r.Rain64.Contrast` | 1.0 | Grade contrast around mid-grey, applied before quantize. |
| `r.Rain64.HighlightRolloff` | 0.0 | 0 = hard clip, 1 = full soft shoulder below the ceiling. |
| `r.Rain64.HighlightWhitePoint` | 2.0 | Luminance that maps to the ceiling. Lower = harder clipping, higher = softer. |
| `r.Rain64.HighlightCeiling` | 1.0 | Brightest output luminance. Lower to cap pure-white surfaces below full white. |
| `r.Rain64.VIFilter` | 1 | Enable the de-dither/AA tent filter. Must be on for Sharpen. |
| `r.Rain64.Divot` | 1 | Median-3 stray-pixel cleanup at edges. |
| `r.Rain64.VIFilterStrength` | 1.0 | 1 = full-strength blur, 0 = none, >1 = stronger. |
| `r.Rain64.Sharpen` | 0.0 | Unsharp mask after the filter. ~0.2–0.6 = subtle to strong. |
| `r.Rain64.Gamma` | 1.0 | Additional display gamma at the present stage. |
| `r.Rain64.NeutralTonemapper` | 1 | Replace UE's ACES filmic tonemapper with a neutral response. |
| `r.Rain64.Composite` | 0.5 | Composite-video strength. 0 = off (stage skipped). |
| `r.Rain64.CompositeMode` | 2 | 1 = single-pass, 2 = two-pass NTSC with animated dot crawl. |
| `r.Rain64.CompositeBleed` | 1.5 | Chroma smear width in texels. |
| `r.Rain64.CompositeArtifact` | 0.15 | Dot-crawl / rainbow-fringe amount (single-pass mode only). |
| `r.Rain64.Scanlines` | 0.35 | CRT scanline darkening. 0 = off. |
| `r.Rain64.ScanlineCount` | 240 | Scanlines across the output height, independent of internal res. |
| `r.Rain64.Mask` | 0.0 | Aperture-grille RGB mask strength. |

The indexed **Palette** (used when `ColorMode` = 1) holds up to 64 colors.
Author your own in Project Settings, load a built-in there via **Load Built-In
Palette** (persists with the project), or load one at runtime with
`r.Rain64.PalettePreset` — sets larger than 64 entries reduce by even stride on
apply. While the palette is empty, palette mode falls back to per-channel
quantization.

## Performance

The post chain itself is overhead, not a saving — the scene renders normally
and is then downsampled. The chain consists of a small number of low-resolution
passes, but it does not reduce scene rendering cost on its own.

**`r.Rain64.RenderScale`** is the setting that reduces GPU cost: it installs a
low primary screen percentage, so per-pixel scene rendering happens at a
fraction of display resolution. This also produces native-resolution edge
aliasing rather than a downsampled supersampled image. Typical pairing:

```
r.Rain64.RenderScale 0.4
r.AntiAliasingMethod 0
r.Rain64.DownscaleFilter 1
```

(Disabling AA preserves the edge aliasing; point downscale avoids averaging it
away.)

## Tips & limitations

- **Screen-space UMG is not affected** — it draws after post-processing. To
  apply an effect to UI, wrap widgets in a Retainer Box with an effect
  material. World-space widget components are affected automatically.
- **Temporal upscalers (TSR / TAA upsample) are supported.**
- **SDR only** — the chain assumes standard display-encoded output.
- **Scene alpha is not preserved** through the chain (output alpha is 1).
- Texture resolution and filtering are content-side choices; the plugin only
  affects the framebuffer.
- Fixed resolution presets (256×224 etc.) stretch to fill the viewport; use
  Custom mode if the internal aspect must match the display exactly.

## Troubleshooting

- **No effect in the editor viewport** — expected; the effect renders in
  game/PIE. Enable **Apply In Editor** to render in the level editor.
- **Material previews look unaffected** — expected; preview scenes and
  thumbnails are always excluded.
- **Preset choice didn't survive a restart** — `r.Rain64.Preset` is
  runtime-only. Persist settings via Project Settings.
- **Output is darker/brighter than vanilla** — the neutral tonemapper removes
  UE's filmic curve. Compensate with `r.Rain64.Gamma`, `Saturation`, and
  `Contrast`, or set `r.Rain64.NeutralTonemapper 0`.

## Support

- Discord: https://discord.gg/RsA8K8uAvc
- Or open an issue at https://github.com/yearsdev/Rain64-Manual.

## Version history

See `CHANGELOG.md`, included with the plugin.

---

© YEARS Dev Ltd. Distributed under the Fab EULA.
