# recur_web

A live video effect and generation tool that runs in a browser. Twelve
generative shaders, twenty-one effects, four-slot chains with per-slot blend
modes, LFO / MIDI / audio modulation, and video export — in **one HTML file with
no build step and no dependencies**.

It's the WebGL2 sibling of [recur](https://github.com/thedirgemedia/recur), an
mpv-based video sampler for the Raspberry Pi 5.

---

## Quick start

Download `index.html` and open it or drag it onto a web browser



**Requirements:** any browser with WebGL2 — Chrome, Edge, Firefox, Safari 15+.
Share links additionally need `CompressionStream` (Safari 16.4+). No install, npm, server-side
Everything runs locally

Tap the **◀** at the bottom of the screen to open the control panel.

---

## The three modes

| Mode | Source | Use |
|---|---|---|
| **SAMPLER** | a video file you load | sampling and mangling footage |
| **SHADER** | purely generative | visuals from scratch, optionally composited over footage |
| **LIVE** | the device camera | live camera processing |

## Signal flow

```
  source  ──▶  GEN chain (max 4, each composited with its own blend mode)
                  │
                  ▼
               FX chain (max 4, serial, each with wet/dry + blend mode)
                  │
                  ▼
                screen  ──▶  optional CAST window / video export
```

The **flow strip** in the panel shows both chains. Drag a shader button onto it
to insert at a position, drag blocks to reorder, double-click to bypass.

## Shaders

**Generative (12)** — plasma, kaleidoscope, tunnel, flowing-colours,
hypnotic-rings, squarewaves, starfield, voronoi, waves, zoom-clouds, gamma-ray,
oscilloscope.

Four of these carry a **style selector** with eight variants each — plasma,
flowing-colours and zoom-clouds cover classic / fractal / domain-warped / ridged
/ marble / contour / spiral / flow families, and plasma includes a real
escape-time Julia set. Style 0 is always the original behaviour.

**Effects (21)** — vhs, glitch, feedback, mirror, posterize, invert, bitcrush,
colorizer, grain, hsv-shift, hue-cycle, kaleido-warp, rotate-zoom, wobble,
zoom-fx, ascii, halftone, levels, sample-hold, feature-dots, scatter.

- **glitch** — macroblock corruption modelled on how codecs fail: runs
  of consecutive blocks share a wrong motion vector, DC-only blocks, scrambled
  DCT coefficients, chroma desync.
- **ascii** — five selectable character sets (full printable ASCII, katakana,
  CJK), each ranked by ink coverage so the tonal ramp is correct.
- **feature-dots** — Shi-Tomasi corner detection with Lucas-Kanade optical flow;
  dots track structure
- **scatter** — cuts the frame into a grid and rearranges it; the swap modeso nothing is duplicated or lost.
- **levels** — a five-point tone curve with a live editable canvas.

Every slot has up to 12 parameters, and every parameter can be driven by audio,
an LFO, or a MIDI CC — these stack.

## Modulation

- **Audio** — mic, tab audio or the loaded video's track, split into bass / mid /
  treble / volume with a hold-decay envelope. Two shaders also read the raw
  spectrum and waveform on the GPU.
- **LFO** — three oscillators, each with shape, amplitude, offset, **skew**
  (warps the waveform's timing) and **curve** (bends the trajectory spiky or
  plateaued). Period in seconds or in musical divisions of a tap-tempo BPM.
- **MIDI** — opt-in Web MIDI. Tap a parameter's MIDI badge and move a knob to
  bind it. Bindings save with presets and share links.

## Saving and sharing

- **SAVE / LOAD** store named presets in the browser.
- **SHARE** encodes the entire patch — chains, every per-slot parameter, blend
  modes, LFO config, all modulation bindings — into a compact gzipped URL.
  Video files are not included.
- **CAST** opens a second window showing only the canvas, for a projector or
  second screen. It syncs live over `BroadcastChannel` (same browser, same
  machine).

Share links are forward-compatible: newer fields are appended as length-guarded
trailing blocks, so a link made in an older build still opens.

## Recording

Set a duration (5–120s), name the file, hit record. Saves as MP4 where the
browser supports it, WebM otherwise — both can be reloaded as SAMPLER sources.
Includes a hand-rolled WebM duration repair, because `MediaRecorder` writes
unseekable files.

Full documentation for every shader and control is in the **in-app help
overlay** (`?`), which is kept in sync with the code.

---

## Development

**The single-file structure is deliberate.** 
one file you can drop on a USB stick, email, or serve from anything. All CSS,
JavaScript and GLSL live inline in `index.html`. Please keep it that way.



## Known issues

- **Reverse and ping-pong playback can show a static frame.** Under
  investigation. Backward playback is seek-driven and depends heavily on the
  file's keyframe interval; `seek-check.html` will tell you whether your footage
  is a factor.
- **Share links can fail on a recipient's machine when a live camera is in the
  patch** — suspected camera-permission mismatch, mostly on mobile.
- Heavy chains need the render-scale control (`½` / `¾` / `1×` in the panel) on
  phones and single-board computers. A typical 9-pass chain at 1080p60 asks for
  around 1.1 Gpx/s.
- **On a Raspberry Pi 5**, WebGL2 is hardware-accelerated but *video* is not:
  the H.264 decode block was removed from the silicon, Chromium can't drive the
  HEVC one, and there is no hardware encoder at all. Generative work is fine;
  expect software decode for SAMPLER and slow exports.

## Licence

GPL-3.0. See [LICENSE](LICENSE).
