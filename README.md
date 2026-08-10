# recur_web

Browser-based live video effects and generative shader tool. WebGL2, real-time,
built for VJ use.

Feed it a video file, a camera, or a screen capture, stack up to four generative
shaders and four effects over it, and drive any parameter from audio, an LFO, or
MIDI. Record the output or cast it to a second screen.

**[dirgemedia.com/recur](https://dirgemedia.com/recur)** · no install, no account.

---

## Run it

Open `index.html`. That's it — one self-contained file, no build step, no
dependencies, works from `file://`.

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

- **GEN** — 14 generative shaders, composited onto each other with per-slot blend
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
| `BKSP` | switch param layer, gen ↔ fx |
| `P` `I` | perf meter · system info |
| `?` | help |

Everything is also on the control panel — tap the tab at the bottom of the screen.

## Save and share

**SAVE / LOAD** keep presets in browser storage. **SHARE** encodes the entire
patch into a URL — every parameter, chain order, blend mode and modulation
binding, a few hundred bytes, no server. The video file itself is not included.

Share URLs are forward-compatible: an older build will still open a link made by
a newer one.

## Performance

Two independent controls in **ZOOM & CHAINS**:

- **RES** (½ / ¾ / 1×) — a *fraction* of your display resolution. First thing to
  lower if a deep chain drops frames.
- **RENDER CAP** (off / 1080 / 4K, default 4K) — an *absolute* pixel ceiling,
  independent of screen size. Bounds GPU memory on large displays; does nothing
  below the ceiling. Saved per device, not part of presets.

Press `P` for a live frame-time meter, `I` for a GPU and platform report with a
copy button. If something runs slower on one machine than another, those two
answer it — the `mediump` line in particular, which differs between platforms and
is not otherwise visible.

## Development

The single-file structure is deliberate. Keep it.

For shader work there is an opt-in hot-reload path that leaves the shipped file
untouched:

```
node tools/extract-shaders.js     # index.html -> shaders/
python3 serve.py                  # then open /?dev
node tools/inline-shaders.js      # fold edits back into index.html
```

Without `?dev` that code path returns immediately and issues no requests.

Validate before committing: extract the `<script>` and run `node --check`, and
check element balance on any markup change.

## Licence

GPL-3.0
