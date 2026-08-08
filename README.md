# recur_web

A live video effect and generation tool that runs in a browser. Twelve
generative shaders, twenty-one effects, four-slot chains with per-slot blend
modes, LFO / MIDI / audio modulation, and video export — in **one HTML file with
no build step and no dependencies**.

It's the WebGL2 sibling of [recur](https://github.com/thedirgemedia/recur), an
mpv-based video sampler for the Raspberry Pi 5.

---

## Quick start

Download `index.html` and open it. That's the whole install.

```
git clone https://github.com/thedirgemedia/recur-recur-browser
cd recur-recur-browser
open index.html          # or double-click it
```

To serve it instead (needed if you want the camera on a device other than
localhost, since `getUserMedia` requires a secure context):

```
python3 -m http.server 8000
# then visit http://localhost:8000
```

**Requirements:** any browser with WebGL2 — Chrome, Edge, Firefox, Safari 15+.
Share links additionally need `CompressionStream` (Safari 16.4+). No install,
no npm, no server-side anything. Everything runs locally; no media ever leaves
your machine.

Tap the **◀ tab** at the bottom of the screen to open the control panel.

---

## The three modes

| Mode | Source | Use |
|---|---|---|
| **SAMPLER** | a video file you load | sampling and mangling footage |
| **SHADER** | nothing — purely generative | visuals from scratch, optionally composited over footage |
| **LIVE** | the device camera | live camera processing |

`ENTER` cycles between them. In SAMPLER and LIVE the generative chain is
bypassed by default so the shaders don't hide your source — the **blend** toggle
in the SRC row layers them back over it.

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

Some worth calling out:

- **glitch** — macroblock corruption modelled on how codecs actually fail: runs
  of consecutive blocks share a wrong motion vector, DC-only blocks, scrambled
  DCT coefficients, chroma desync.
- **ascii** — five selectable character sets (full printable ASCII, katakana,
  CJK), each ranked by ink coverage so the tonal ramp is correct.
- **feature-dots** — Shi-Tomasi corner detection with Lucas-Kanade optical flow;
  dots track the structure they mark.
- **scatter** — cuts the frame into a grid and rearranges it; the swap mode is a
  true permutation, so nothing is duplicated or lost.
- **levels** — a five-point tone curve with a live editable canvas.

Every slot has up to 12 parameters, and every parameter can be driven by audio,
an LFO, or a MIDI CC — they stack.

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

## Keyboard

| Key | Action |
|---|---|
| `ENTER` | cycle SAMPLER → SHADER → LIVE |
| `SPACE` | play / pause (SAMPLER) |
| `R` | reverse playback (SAMPLER) |
| `O` | open a video file |
| `L` | toggle the camera |
| `F` | fullscreen |
| `0` | SAMPLER/LIVE: in → out → clear · SHADER: cycle selected gen |
| `4`–`9` | toggle gen shader slots 1–6 |
| `−` / `+` | previous / next FX slot |
| `*` | next FX slot |
| `/` | clear the FX chain |
| `.` | toggle the feedback trail |
| `1` / `2` / `3` | next param · decrease · increase |
| `BKSP` | switch param layer, gen ↔ fx |
| `?` | help overlay |

The layout mirrors a numpad — it's designed to be played from one.

Full documentation for every shader and control is in the **in-app help
overlay** (`?`), which is kept in sync with the code.

---

## Development

**The single-file structure is deliberate.** Portability beats structure here:
one file you can drop on a USB stick, email, or serve from anything. All CSS,
JavaScript and GLSL live inline in `index.html`. Please keep it that way.

### Shader hot-reload

Editing shaders inside a 400 KB HTML file is miserable, so there's an opt-in dev
path:

```
node tools/extract-shaders.js index.html      # explode shaders into shaders/
python3 serve.py                              # then open /?dev
node tools/inline-shaders.js                  # fold edits back into index.html
```

With `?dev` the page polls `shaders/` and recompiles on change. Without it, no
requests are made at all — the shipped build is unaffected.

### Adding a shader

Append to the end of `GEN[]` or `FX[]` — never insert in the middle, or you
shift every saved preset's indices.

```js
{
  name: 'my-shader',
  params: ['speed', 'scale'],      // up to 12
  def:    [.5, .5],
  snapVals: [null, [0,1,2,3]],     // optional: discrete selectors snap
  src: H + `void main(){ FC = vec4(vU, 0., 1.); }`
}
```

Two rules that matter:

- **0 must be neutral for any parameter you append to an existing shader.**
  Presets saved before it existed load it as 0, so 0 has to mean "off".
- **Declare time as `highp`.** `uFrame` grows without bound and `mediump` is
  fp16 on mobile, where it stops resolving single frames after about 34 seconds.

### Version marker

`BUILD` near the top of the script shows dim in the panel header and logs to the
console. Bump it when you deploy — it's the fastest way to tell whether a device
is running a cached copy.

### Diagnostics

Two standalone pages live alongside the app:

- **`pi-check.html`** — GPU capabilities, float precision, texture limits, codec
  support and a fill-rate benchmark. Written for the Raspberry Pi 5 but useful on
  any unfamiliar device.
- **`seek-check.html`** — drop a video on it to measure whether backward seeking
  actually works, which is what reverse and ping-pong playback depend on.

---

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
