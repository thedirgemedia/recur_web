# recur_web

Browser-based live video effects and generative shader tool. WebGL2, real-time,
built for VJ use.

Feed it a video file, a camera, or a screen capture, stack up to four generative
shaders and four effects over it, and drive any parameter from audio, an LFO, or
MIDI. Record the output or cast it to a second screen.

**[dirgemedia.com/recurweb](https://dirgemedia.com/recurweb)** · no install, no account.

---

## Run it

Open `index.html`. One self-contained file, no build step, no dependencies,
works from `file://`.

Requires a browser with **WebGL2**: Chrome/Edge 56+, Firefox 51+, Safari 15+.
Share links additionally need `DecompressionStream` (Safari 16.4+).

## Use it

**Three modes**, cycled with `ENTER`:

| | |
|---|---|
| **SAMPLER** | a loaded video file is the source |
| **SHADER** | purely generative; optionally blend a source underneath |
| **LIVE** | the camera is the source |

SAMPLER and LIVE arrive with the shader stack bypassed so you see the source
first. Tap the blend toggle to bring the stack back.

**Two chains**, up to four slots each, shown in the flow strip:

- **GEN** — 15 generative shaders, composited onto each other with per-slot blend
  mode and amount.
- **FX** — 21 effects, applied in series, each with its own wet/dry and blend mode.

Drag blocks in the strip to reorder, drag one in from the grid to insert,
double-click to bypass, drag off the strip to remove. Duplicates are allowed and
each gets its own seed.

**Modulation** — every parameter can be driven by an audio band (bass/mid/treble/
volume), one of three LFOs, or a MIDI CC. LFOs run free or locked to a tap-tempo
BPM clock.

### Keys

| key | |
|---|---|
| `ENTER` | cycle mode |
| `SPACE` | play / pause |
| `R` | play backwards |
| `O` | open a video file |
| `L` | camera on / off |
| `F` | fullscreen |
| `.` | feedback trail |
| `0` | set in → out → clear (SAMPLER) · cycle gen (SHADER) |
| `4`–`9` | toggle gen shaders 1–6 |
| `−` `+` | previous / next FX |
| `/` | clear the FX chain |
| `1` `2` `3` | next param · decrease · increase |
| `[` `]` | previous / next preset |
| `BKSP` | switch param layer, gen ↔ fx |
| `P` `I` | perf meter · system info |
| `?` | help |

Everything is also on the control panel — tap the tab at the bottom of the screen.

## Save and share

**SAVE / LOAD** keep presets in browser storage. Saved presets appear as blocks
on the **BANK** row of the flow strip — click one to jump straight to it, or step
through with `[` and `]`. A jump restores everything the preset captured,
including the mode, so a preset saved in LIVE will start the camera.

A preset saved in SAMPLER also records **which clip it was built on** — filename,
size and length — plus its in/out points. The clip itself is not stored. Load a
preset whose clip isn't open and a bar names the missing file with a LOCATE
button. The shaders load either way, and locating a file once covers every preset
using it for the rest of the session.

Browser storage belongs to the browser profile, not to `index.html`, so presets
do **not** travel with the file. **EXPORT BANK** writes them all to one `.json`
you can carry alongside it; **IMPORT BANK** reads one back, or just drag the
`.json` onto the window. Importing merges — it never overwrites what you already
have.

**SHARE** encodes the whole patch into a URL — every parameter, chain order,
blend mode and modulation binding. A few hundred bytes, no server. Video files
are not included in presets or share links.

Share URLs are forward-compatible: an older build will still open a link made by
a newer one. That covers the *format*, not the meaning of every slider — if a
control's range changes between builds, an old link still loads, but that shader
may be framed differently than it was when the link was made.

## Performance

Two independent controls in **ZOOM & CHAINS**:

- **RES** (½ / ¾ / 1×) — a *fraction* of your display resolution. First thing to
  lower if a deep chain drops frames.
- **RENDER CAP** (off / 1080 / 4K, default 4K) — an *absolute* pixel ceiling,
  independent of screen size. Bounds GPU memory on large displays; does nothing
  below the ceiling. Saved per device, not part of presets.

`maze-flight`, `quaternion` and `mandelbox` are raymarched and cost far more
than the rest of GEN. Each has a **detail** slider that buys surface detail with
frame rate — drop that before you drop RES. Zooming inside the two fractal
solids is the most expensive thing the tool does.

`P` shows a live frame-time meter; `I` shows a GPU and platform report with a
copy button. The report's `mediump` line is the usual explanation for the same
patch running at different speeds on similar hardware: 10 bits means the GPU runs
it as fp16, 23 means fp32.

## Development

The single-file structure is deliberate. Everything — CSS, JS, GLSL — is inline
in `index.html`, and there is no build step. Keep it that way.

Validate before committing: extract the `<script>` and run `node --check`, and
check element balance on any markup change.

## Licence

GPL-3.0
