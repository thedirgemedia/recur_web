# recur-web — Project Context

Context document for **recur-recur-browser** (aka **recur_web**). Give this to
Claude at the start of a session.

*Rebuilt from `index.html` on 2026-08-12 — 10,596 lines / 525,264 bytes /
sha256 `e6103e81 e37ba645 b0c49c3d fa18fd06`. `BUILD` reads
`2026-08-12c · header + bank name`; bump before deploy, and keep the name
**within 77 characters** — see* Panel header *below.*

**Every claim below is derived from the file itself** — its code, its constants
and its comments — not from a previous version of this document. Where the file
and this document could disagree, the file wins; re-derive rather than trust.
The one section that is not file-derived is *Carried forward*, at the end, and
it is marked as such.

Nothing here has been GPU-verified. There is no GPU in the authoring
environment, and the file says so itself in its own rules block.

---

## What it is

- Browser-based live video effect and generation tool, WebGL2. Feed it a video
  file, a camera or a screen capture; stack up to four generative shaders and
  four effects over it; drive any parameter from audio, an LFO or MIDI; record
  the output or cast it to a second window.
- Repo `github.com/thedirgemedia/recur-recur-browser`, public, GPL-3.0.
  Deployed at `dirgemedia.com/recur`. Dirge pushes and deploys manually.
- Sibling of the Raspberry Pi **recur** project; `starfield` is a direct port of
  `thedirgemedia/recur-recur shaders/starfield.glsl`.

## Rules

From the file's own preamble, verbatim in substance:

- **One entry point per user action:** `togglePlay`, `toggleReverse`, `setMode`.
- **Mutate chains only via** `chainInsert` / `chainRemove` / `chainMove`.
- **0 is the neutral value for every appended shader param.** Prove it.
- **Bump `BUILD` on every handover.**
- **Nothing here is ever GPU-verified.** Say so when shipping shader work.

And from the structure the file enforces rather than states:

- **Single self-contained `index.html`.** No build step, no modules. CSS, JS and
  GLSL inline. One `<script>`, one `<style>`.
- **Replace the file directly; no git patches.**
- **The wire format is frozen.** `getState` / `applyState` are, in the file's own
  words, "exactly two places" the flat preset/share shape meets slot objects.
  Older builds must read newer files.
- **Comments carry the reasoning.** The file is unusually well commented and the
  comments are load-bearing documentation — most of this document is a summary
  of them. Keep that standard: one line for a function or constant, a short
  block for a non-obvious mechanism, a half line to justify a magic number. No
  change history in comments; it belongs here.
- **This document cites function and constant names, never line numbers.**

---

## File layout

The preamble lists the top-to-bottom order, and the file holds to it:

| section | contents |
|---|---|
| GLSL sources | `VERT`, shared header `H`, shared snippets `PAL` / `HUESHIFT` / `HSV` |
| `GEN[]` / `FX[]` | the shader tables — one entry per shader, **append only** |
| WebGL init | context, program cache, framebuffers, pixel budget |
| App state | modes, chain slots, slot layout, playback, transport |
| Render | uniforms, draw, gen chain, FX chain, main loop |
| HUD / overlays | toast, perf meter (`P`), system info (`I`) |
| Keyboard | one handler, one entry point per action |
| Camera / audio | `getUserMedia`, FFT, GPU audio texture, LFOs, MIDI |
| File loading | clip loading, source assessment, source binding |
| SPI control panel | sections, flow strip, source section, param sliders |
| Export | MediaRecorder plus container-duration repair |
| State | `getState` / `applyState`, presets, bank, V5 share URLs |
| Cast | `BroadcastChannel` output window |

`MODES = ['SAMPLER','SHADER','LIVE']`.
`BLEND_MODES` has ten entries: `normal, add, screen, multiply, difference,
lighten, darken, overlay, dodge, burn`.
`PRESERVE_DRAWING_BUFFER` is `false`; the context is created with
`antialias:false, powerPreference:'high-performance'`.

---

## Shader tables

**GEN 15 entries, FX 21 — 36 total, 35 with GLSL.** The one entry without `src:`
is `sample-hold`; that is the durable invariant, and the counts should be
re-derived from the file rather than trusted here.

Entry shape: `{name, params, def, [src], [usesPrev], [snapVals]}`. Append at the
end. Formatting across entries is not uniform — `feature-dots` puts
`usesPrev: true` between `name:` and `params:`, and `src:` prefixes vary.

### GEN

| # | name | params | notes |
|---|---|---|---|
| 0 | plasma | 9 | snapped `style` selector |
| 1 | kaleidoscope | 4 | snapped |
| 2 | tunnel | 5 | |
| 3 | flowing-colours | 11 | snapped `style` selector |
| 4 | hypnotic-rings | 8 | |
| 5 | squarewaves | 12 | reads the audio texture |
| 6 | starfield | 11 | global `precision highp` |
| 7 | voronoi | 9 | |
| 8 | waves | 5 | |
| 9 | zoom-clouds | 11 | snapped `style` selector |
| 10 | gamma-ray | 4 | deliberately corner-anchored |
| 11 | oscilloscope | 9 | `style` selector; reads the audio history ring |
| 12 | maze-flight | 12 | global `precision highp` |
| 13 | quaternion | 12 | global `precision highp` |
| 14 | mandelbox | 12 | global `precision highp` |

### FX

| # | name | params | notes |
|---|---|---|---|
| 0 | vhs | 4 | |
| 1 | glitch | 9 | |
| 2 | feedback | 5 | |
| 3 | mirror | 4 | |
| 4 | posterize | 4 | |
| 5 | invert | 4 | |
| 6 | bitcrush | 7 | snapped; centre-anchored lattice |
| 7 | colorizer | 4 | |
| 8 | grain | 4 | |
| 9 | hsv-shift | 4 | |
| 10 | hue-cycle | 4 | |
| 11 | kaleido-warp | 4 | snapped |
| 12 | rotate-zoom | 4 | |
| 13 | wobble | 4 | |
| 14 | zoom-fx | 4 | |
| 15 | ascii | 6 | snapped charset; 2D atlas |
| 16 | halftone | 7 | centre-anchored lattice and pivot |
| 17 | levels | 5 | spline curve editor in the panel |
| 18 | sample-hold | 1 | **no `src:`** — the frozen-frame buffer is the effect |
| 19 | feature-dots | 9 | `usesPrev`; drives the RG16F material pair |
| 20 | scatter | 7 | snapped mode selector |

**The widest shader declares 12 params, and `N_P` — the V5 per-slot param byte
count — is also 12.** A thirteenth param on any shader overflows the share
format. Check this before adding one.

### `src:` prefixes

The prefix is **not** always `H`. Measured across the tables: `H+` 21 entries,
`H+PAL+` 6, `H+HUESHIFT+` 3, `H+HSV+` 4, `H+PAL+HUESHIFT+` 1. Any tool that
anchors on `` src: H+` `` silently skips a third of the table while still
reporting "no failures". Match `src:\s*((?:[A-Z][A-Z0-9_]*\s*\+\s*)+)` and
resolve each name.

### The shared header `H`

```
#version 300 es
precision mediump float;
uniform sampler2D uTex;
uniform sampler2D uPrev;
uniform vec2 uRes;
uniform int uFrame;
uniform float uTime,uP1..uP12,uSeed;
in vec2 vU; out vec4 FC;
```

`TAU` is **not** in `H`. The three shaders that use it (`maze-flight`,
`quaternion`, `mandelbox`) each `#define` it themselves. A GLSL parser that does
not run the preprocessor will report it as undeclared in exactly those three —
that is a false positive, not a bug.

---

## Chain slots

`genSlots` and `fxSlots` are arrays of up to `CHAIN_MAX = 4` slot objects. The
file's rationale: everything belonging to one link lives on the same object, so
insert, delete and reorder move all of it or none of it.

```
{ idx, params[], audioMod[], lfoMod[], midiMod[], seed, mode, amt, off, pos }
```

| field | |
|---|---|
| `idx` | index into `GEN[]` / `FX[]`. Duplicates allowed in a chain. |
| `params[]` | one entry per declared param |
| `audioMod[]` `lfoMod[]` `midiMod[]` | one entry per param |
| `seed` | GEN only. FX is fixed at 0 — the FX path passes no seed, and drawing from `Math.random` would consume entropy for a value nothing reads. |
| `mode` | blend onto the layer below, index into `BLEND_MODES` |
| `amt` | blend amount (GEN) or wet/dry (FX, 1 = full effect) |
| `off` | bypassed |
| `pos` | which of the `CHAIN_MAX` boxes it is **drawn** in — see *Slot layout* |

`genIdx` / `fxIdx` are chain POSITIONS, not shader indices. `genSlots` is never
empty. `fxSlots` can be, with `fxIdx === -1`.

`makeGenSlot` / `makeFxSlot` wrap `makeSlot(table, idx, opts)`. FX defaults `amt`
to 1.0, GEN to 0.5. `CHAIN_MAX` is enforced in both `chainInsert` and
`applyState`.

Parameter and modulation state lives per CHAIN SLOT, not per shader. The old
per-shader parallel arrays are gone; presets written by older builds are
migrated onto slots in `applyState`.

### The three mutators

```
chainInsert(slots, slot, pos)   // shifts the oldest out when full; returns where it landed
chainRemove(slots, pos)
chainMove(slots, fromPos, toPos)
```

**`chainMove` currently has zero call sites.** Drag reorder goes through
`layoutPlace` instead. The rule "mutate chains only via the three" still holds
for `chainInsert`/`chainRemove`; `chainMove` is retained but dead. Either wire
it back up or delete it, but do not assume it is exercised.

---

## Slot layout — "one ruler"

The current build's headline feature, and the reason `pos` exists.

A chain stays **dense and in signal order**. `pos` says which of the four boxes
a slot is **drawn** in, so a chain of one can sit in box 4 with three empty boxes
ahead of it. Holes cost nothing at render time — `[A,_,_,_]` and `[_,_,_,A]`
composite identically. That is the whole point: none of the render-path reads of
`genSlots`/`fxSlots` need to know positions exist, and the wire keeps writing a
dense chain that older builds draw identically.

**Invariant, enforced by `layoutNormalise` and assumed everywhere else:**
positions are integers in `0..CHAIN_MAX-1`, unique within a chain, and **ascend
with array order**. Drawing left to right is therefore drawing in signal order.

| function | |
|---|---|
| `layoutNormalise(slots)` | clamps each `pos` into `[next, CHAIN_MAX - (len - i)]`. Absorbs `undefined`, the `-1` from a full `layoutFreePos`, and garbage from a decoded URL. Only ever pushes a position **up**, never compacts one down — which is what preserves a deliberate hole. |
| `layoutAt(slots, pos)` | the slot drawn in a box, or `null` |
| `layoutDenseIndex(slots, pos)` | where a box falls in the dense array |
| `layoutPlace(slots, from, to)` | moves a block between boxes; an occupied target **swaps**, so nothing untouched moves. Re-sorts, so dense order stays equal to display order. |
| `layoutPlaceNew(slots, slot, pos)` | drops a new block into a box. Empty box fills, occupied box **replaces**. With fixed boxes there is no gap to insert into, and replace is the only outcome that puts the block where it was dropped. A full chain therefore always takes the replace branch, so `chainInsert`'s drop-the-oldest path is unreachable from here. |
| `layoutFreePos(slots)` | lowest empty box, or `-1` when full |

Every call site either uses a `layout*` helper or follows `chainInsert` /
`chainRemove` with `layoutNormalise`. Keep it that way — the invariant is not
defended anywhere else.

**Including the declaration.** `makeSlot` leaves `pos` undefined for callers that
do not care where a slot lands, and `layoutAt` matches *on* `pos` — so an
un-normalised chain draws no block at all. The boot chain
`let genSlots = layoutNormalise([makeGenSlot(0, { seed: 0 })])` must stay wrapped:
without it plasma renders correctly and every GEN box reads empty, which looks
like a render bug and is a layout one.

Dragging a block **off** the strip removes it and leaves the box empty. Nothing
else shifts, so a block can be pulled out, listened without, and dropped back
into the same box.

### `_armSlot`

Click an empty box to arm it, then pick from the shader grid and the block lands
there — building a chain in any order without dragging. `_armSlot` is
`{kind, pos}`, a box reference rather than a slot reference, so it cannot dangle.
Clicking the armed box again disarms; picking a filled box cancels a pending
place.

It is **not** cleared by `randomiseAll` or `applyState`. Since it only ever holds
a box index 0-3 and every box is always valid, the worst case is a leftover
highlight after loading a preset and the next grid pick replacing that box.
Cosmetic, but it is the one loose end in otherwise disciplined handling.

---

## Render

**One path serves every mode:** base layer → gen chain composited over it → FX
pipeline. Base is the video in SAMPLER, the camera in LIVE, and in SHADER
whichever source the blend picker names.

### Framebuffer inventory

| buffer | |
|---|---|
| `fbo` / `fbo2` | ping-pong accumulator, always resident |
| `fbo3` | scratch for each gen render; also transform scratch in `getBlendTex` |
| `fboFx` | FX wet/dry pre-effect save; also holds the base when it arrived in `fboTex2` |
| `fboPrev` | previous frame of the FX-chain *input*, for temporal-difference effects |
| `fboHist` | persistent trail history |
| `fboHold` | sample-and-hold frozen frame |
| `fboMat[2]` | RG16F material-coordinate pair for `feature-dots` |

`fbo`/`fbo2`/`fbo3` are needed by every path. The other four RGBA8 buffers and
the RG16F pair are allocated **on first use** — the file's note is that
allocating all of them costs 200 MB+ at Retina, enough for iOS Safari to drop
the tab. Callers null-check, so a failed allocation disables that effect, not
the frame.

`_makeAttached` checks `getError` and `checkFramebufferStatus` and returns `null`
on failure; `makeFBO` returns a boolean.

### Buffer aliasing

The recurring correctness risk. Three guards worth knowing:

- `renderGenChain` returns `null` rather than `fboTexFx` when it has nothing to
  composite, so the caller falls back to `baseTex`. Returning `fboTexFx` would
  hand the FX chain a texture it uses as its own scratch — a read-write loop on
  one attachment, **reachable by default in LIVE**.
- A base sitting in `fbo`/`fbo2` is copied aside before the accumulator
  ping-pongs over it.
- A second `sample-hold` downstream of a dry one arrives with `cur` already
  `=== fboTexHold`; capturing it into itself is a no-op in intent but a
  read-write loop in GL, so the copy is skipped.
- FX wet/dry renders into `fbo3` (free after the gen chain, and it can never
  alias `cur`) then mixes back over the intact input — **two passes, not three**.
- `getBlendTex` uses `fbo3`, never `fbo2`, since `renderGenChain` may have left
  its output in `fboTex2`.

FX blend gate: `slotAmt < 0.999 || slotMode !== 0`.

### Uniforms

`setUniforms` writes **every** declared param uniform every time, padding with 0.
This is required, not tidiness: GL uniforms are per-PROGRAM state and the program
is shared by every slot using that shader, so any uniform a slot skips inherits
the previous slot's value. 0 being neutral for appended params is what makes
appending a param preset-safe.

### `setMode`

Single entry point for every interactive mode change — ENTER, the MODE button,
the three set-mode buttons, `toggleCamera`. Do not add another copy.

- SAMPLER and LIVE arrive with the gen chain bypassed, chain intact.
- SHADER clears an **all-off** bypass set on arrival; a partial per-slot bypass
  is kept.
- `applyState` does **not** use `setMode` — a preset carries its own bypass set.

Tapping a gen button deliberately does **not** force SHADER mode: forcing the
switch dropped the camera/video base entirely, so tapping any gen while in LIVE
showed the shader alone with no way to blend it over the source.

---

## Sampler playback

`loop` has three branches: ping-pong, reverse, forward-native. Ping-pong drives
**both** directions manually so native playback never runs into EOF, where MP4
decoders stall and refuse to seek back. Everything else keeps native forward
playback with a manual reverse leg.

- `stepPlayhead` advances a virtual playhead `_stepT` at the true rate and pushes
  it to the element. **Do not derive it from `vid.currentTime`:** during an
  in-flight seek that value has not moved, so every frame retargets from a stale
  position and the browser discards the intermediates.
- **Do not add a divergence-resync guard.** On slow seeks the virtual head
  legitimately runs ahead and snapping it back deadlocks playback. Anything that
  relocates the clip sets `_stepT = null`.
- `_seekPending` allows one seek in flight, cleared when a frame is **presented**
  (rVFC), not on `seeked` — that fires before compositing and would let the seek
  storm back in. `SEEK_WATCHDOG_MS = 150` reopens the gate and deliberately
  overrides `vid.seeking`: a seek that never retires leaves it stuck true.
- `hookFramePresented` is load-bearing, not diagnostic — clearing `_seekPending`
  there is what paces the reverse stepper. Registered once from `loop`, which
  always runs, so it survives a clip loaded later. The no-rVFC fallback (older
  WebKit) paces on `seeked`: looser, but still one seek per completed seek
  rather than one per rAF.
- `togglePlay` / `toggleReverse` are exclusive; play forces `reversed = false`.
  The `R` key and the panel button both route through `toggleReverse` so they
  cannot drift. The icon shows the action, not the state.
- `PLAY_EPS = 0.08` keeps turnarounds a hair short of the true out/end, because
  EOF stalls MP4. The in/out drag preview uses the same back-off.
- `inOutScrubbing` makes all three stepping branches and the `timeupdate` watcher
  yield, or the preview seek is overwritten.
- In/out enforcement (`vidBoundaryForward`) covers **native forward playback
  only**; ping-pong never reaches it.

### The upload gate

`uploadVideoFrame` must **never gate on `readyState` alone**: an in-flight seek
drops it to 1, and the reverse stepper seeks every frame, so every upload would
be skipped and reverse would show a static frame. The right question is whether
a frame has been presented. Upload happens inside `requestVideoFrameCallback`;
`render()` skips the repeat via `_vidTexFresh`.

---

## Source texture and assessment

`uploadVideoFrame` uploads at the video's **native** resolution every frame.
RENDER CAP does not bound it: 1080p 8 MB, 4K 33 MB, 8K 132 MB.

- `texSubImage2D` once dimensions are known, `texImage2D` on a size change.
  `texImage2D` reallocates storage each call; `texSubImage2D` reuses it.
  `USE_TEXSUBIMAGE = true` reverts the whole thing in one line.
- `vidTex` is shared between the sampler clip and the camera, so the branch
  compares **dimensions**, not a first-time flag.
- `buildGLResources` resets `_vidTexW`/`_vidTexH`: it recreates `vidTex` at 1×1
  and also runs on context restore, and a stale size would send the next frame
  down the `texSubImage2D` path against a 1×1 texture. The declaration sits above
  `buildGLResources` to avoid TDZ.
- A source wider than `MAX_TEXTURE_SIZE` fails with `INVALID_VALUE` and the
  texture silently keeps its last contents. `getError` is checked and reported
  **once only** — it is a pipeline flush and this runs 60× a second.
- A lone vertical flip is free via `UNPACK_FLIP_Y_WEBGL` and `vidFlipV` defaults
  on. Rotation and mirror still need the shader pass: the transform rotates
  before it flips, and flipping at upload would reverse that order at 90°/270°.

`assessSource` runs on `loadedmetadata`: resolution, duration, live detection and
the `MAX_TEXTURE_SIZE` comparison. Over the cap it **warns and does not alter the
source** — silently resampling someone's footage is worse than telling them.
`canPlayType` is checked before the load; `vid` has an `error` listener covering
all four `MediaError` codes.

`startFpsProbe` estimates frame rate from `presentedFrames`/`mediaTime` over the
first second of rVFC, then stops. Both counters come from rVFC metadata; there
is no other route to a source frame rate in the browser. Two concurrent rVFC
chains are legal.

### Source binding

A preset names its clip but cannot carry it, so it stores a fingerprint and asks
for the file once per session. A SAMPLER preset stores
`{fileName, name, bytes, duration}` plus `inPt`/`outPt`.

- `srcFingerprint` is name + size + duration. **No hashing** — reading the file
  to hash it would defeat the point of not loading it. Name plus size is strong
  enough for footage; duration guards a re-encode keeping the name.
- `_srcCache` holds the **File**, not an object URL: `loadVideo` revokes the
  previous blob URL on every clip change.
- `requestSource` runs **last** in `applyState` and never blocks; a missing clip
  raises a dismissible bar with a LOCATE button. `_srcDeclined` is per-session.
  Locating a file once covers every preset using it for the rest of the session.
- `bindSourceFile` restores `modeIdx` after `loadVideo`, which forces SAMPLER.
- in/out are applied in `assessSource`, not `applyState`, and clamped to the real
  duration. `_pendingInOut` carries them across.

---

## Memory ceilings

### Pixel budget

`RES` (`renderScale`) is a **fraction** of display resolution — 0.5 / 0.75 / 1,
nothing above 1×. The chain is up to seven fullscreen passes, so on a Retina
panel at DPR 2 this is the single biggest perf lever available; 0.75 often looks
*better* for this aesthetic, softer and less aliased.

`RENDER CAP` (`PIXEL_CAPS` — `off` / `1080p` / `4k`, default 4K) is an
**absolute** ceiling, independent of display size. Below it nothing changes;
above it `capDims` scales both axes by `sqrt(budget / px)`, keeping aspect.
`MAX_TEXTURE_SIZE` is a hard driver limit and is always applied on top.

Persisted to `localStorage` under `recur.pixelCap`, per device. Deliberately
**not** in presets or share state, for the same reason `renderScale` is not:
what this machine can hold is none of a shared patch's business. The file's cited
figure: uncapped, a 5K display costs 169 MB resident and 506 MB peak.

Exposure is display size × `devicePixelRatio`; `MAX_DPR_TOUCH = 2` clamps coarse
pointers only, via `effectiveDpr`.

`onResize` halves and retries up to four times, resizing the canvas in lockstep
because the render path's viewport calls are written against
`cv.width`/`cv.height`. It stops at 64px. Window resize is debounced 120 ms
through `scheduleResize` — dragging a window edge fires continuously and each
event would otherwise tear down and rebuild six textures and six framebuffers.

### Reaping

`reapFBOs` frees lazy buffers after `REAP_FRAMES = 120` frames of disuse — two
seconds of grace, so a momentary bypass tap does not destroy feedback history.
It scans at most four slots, so it runs every frame and needs no dirty flag.

**Discrepancy to fix:** the comment above `reapFBOs` says it frees
`fboFx / fboHist / fboHold / fboPrev`. The body frees only `fboHist`, `fboPrev`
and `fboHold` — there is no `_reapFx` counter and `fboFx` is never dropped.
`fboFx`'s liveness condition is known to `render()`, not to `reapFBOs()`. Either
fix the code or fix the comment; right now they disagree.

### Share links

`b64ToState` is capped at both ends. `MAX_SHARE_B64 = 256 KB` bounds the encoded
string before `atob`; `_readCapped` reads the gzip stream incrementally and
cancels past `MAX_SHARE_BYTES = 64 KB`. The file's reasoning: the largest
legitimate V5 payload is **541 bytes** and gzip amplifies ~1000:1, so reading the
stream whole would let a 512 KB URL allocate ~386 MB before anything inspected
it.

The `buf[0] === 0x7b` branch `JSON.parse`s to `applyState`, which filters chains
for valid indices **and** caps length with `.slice(0, CHAIN_MAX)`.

A failed share load reports to the user rather than the console: there is no
console on a phone, and `DecompressionStream` needs Safari 16.4+, which is the
usual cause on older iOS.

---

## Wire format

`getState` / `applyState` are the only places the flat shape meets slot objects.
Both preset JSON and the V5 URL predate slot objects and are **frozen** — an
older build must keep reading a newer file.

### `getState` shape

Flat, shader-keyed-looking arrays built from slots:
`modeIdx, genIdx, genChain, genChainSeeds, genChainMode, genChainAmt, fxChain,
fxIdx, layer, selParam, genDisabled, fxDisabled, blendActive, blendAmount,
blendSrc, vidRot, vidFlipV, vidMirrorH, vidRate, vidLoopOn, vidPingPong,
audioHold, inPt, outPt, src, genSlot*, fxSlot*, genSlotPos, fxSlotPos, lfos,
globalBpm, lfoUseBpm, lfoBeatIdx`.

- `genChainMode` / `genChainAmt` are **padded out to 4**, because the V5 encoder
  writes four fixed nibble pairs (bytes 14-17) whatever the chain length. A
  short array here would change those bytes.
- `renderScale` and the pixel cap are deliberately absent.
- `inPt`/`outPt` are present only because presets now know which clip they belong
  to; they are restored only once the named clip is bound.
- `genSlotPos` / `fxSlotPos` are **additive** keys. The chain above is still
  dense and in signal order, so a build that ignores them renders the identical
  image and just packs the strip to the left.

### `applyState`

Flat in, slot objects out — JSON preset, V5 URL, or legacy shader-keyed state.
Nothing downstream sees the flat arrays. Anything the incoming state omits falls
back to what is on screen, so current values are captured **before** the slots
are replaced.

- `_slotsFrom` prefers slot-keyed data, falls back to legacy shader-keyed data by
  chain lookup, otherwise fills from defaults.
- `_padSlots` forces every slot array to the length its shader declares **now**.
  Old JSON presets arrive SHORT (undefined entries then throw in the panel every
  frame); V5 URLs arrive LONG with 12 entries. Missing entries fill with **0, not
  `def`** — 0 is the neutral value for an appended param, `def` is the
  fresh-instance value.
- Bypass is positional on the wire, so it is applied after the chain is known,
  assembled against the **filtered** chain so a stale-length array cannot leave a
  slot half-populated.
- `blendActive` was once hardcoded true here, which silently discarded the saved
  value every time a preset or share URL loaded. It is carried in JSON and in V5
  byte 10 bit 1.
- Source binding runs last.

### V5 binary

37-byte header, then variable blocks sized from `genLen`/`fxLen`, then
append-only trailing blocks. `N_P = 12` param bytes per slot regardless of
declared count, padded with **0** (padding with 0.5 broke the neutrality rule —
a URL made when a shader had fewer params fed 0.5 into a snapped selector).

Header highlights: `[0]` version, `[1]` modeIdx, `[2]` genChain[0],
`[3-5]` genChain[1-3]+1 (0 = absent), `[6-9]` fxChain[0-3]+1,
`[10]` flags (layer, blendActive, vidFlipV, selParam),
`[11]` blendAmount — **written and never read**, `[12]` blendMode,
`[13]` vidRot, `[14-17]` gen modes and amounts as nibble pairs,
`[18-19]` globalBpm, `[20-24]` LFO bpm/beat/shape bits,
`[25-36]` LFO amp / offset / period.

**Trailing blocks, fixed order, each length-guarded independently, append only:**

1. `fxSlotMode` — `fxLen` bytes
2. LFO `skew` — 3 bytes
3. LFO `curve` — 3 bytes
4. `genSlotPos` + `fxSlotPos` — `genLen + fxLen` bytes ← *new in this build*

An older URL decodes with each absent block at its neutral default: modes 0,
skew/curve 0.5, positions packed left by `layoutNormalise`.

**Appending a shader is wire-safe.** V5 is per-SLOT, not per-shader —
`genLen × N_P`, not `GEN.length × N_P` — so growing a table moves no bytes.
Shader indices are stored as whole bytes, so the tables have room to 254.
`applyState` filters `i < GEN.length`, which only ever widens.

**`_bufToStateV4` is not.** It sizes its param blocks from the *current*
`GEN.length` / `FX.length`. It is decode-only, but that means every shader ever
appended has shifted its read offsets: a V4 buffer written when `GEN` held fewer
entries now misaligns by `N_P` bytes and everything after the gen block decodes
as garbage. Pre-existing and cumulative, not caused by any one addition.

---

## Preset bank

`localStorage` key `recur-presets`. Belongs to the browser profile, not to
`index.html`, so presets do not travel with the file. On `file://` Chrome shares
one opaque origin across every local file and Safari often refuses outright. A
sidecar `presets.json` cannot be auto-loaded — `fetch()` on `file://` is CORS
blocked — so transport is a user-initiated bank export/import.

### Preset names

`defaultPresetName()` returns `hhmm-ddmon-preset` — `1432-12aug-preset`.

A `.fb-bank` block is 64px wide with 3px padding and a 1px border, and `.fb` is
8px Courier New at 0.5 letter-spacing: **ten characters visible** before the
ellipsis. Whatever leads the name is therefore the whole of what reads on the
strip. The old default, `preset-2026-08-12`, showed `preset-202` — identical on
every preset ever saved. `hhmm-ddmon` is exactly ten and tells every save in a
session apart at a glance.

`preset` trails behind, where the strip truncates it away but the LOAD list, the
block tooltip and an exported bank all still read it. Emptying the name field
falls back to the same stamp rather than the bare word, for the same reason.

Month names are a **fixed table**, not `toLocaleString` — the ten-character
budget only holds if the string is the same length in every locale.

**Export is in the SAVE dialog, import is in the LOAD dialog.** Writing the bank
out is a save; reading one in is a load.

The save dialog does two jobs, so it is split rather than interleaved: preset
name and CANCEL/SAVE above, then a `.sls-sub` rule, then the bank's own name
field and EXPORT BANK. That also keeps EXPORT clear of the SAVE button.
`#bank-export` does *not* close the modal — you can export, then still save. The
hidden `#bank-file` input stays in the LOAD modal with the IMPORT button that
triggers it.

**The bank filename is user-typed, so it is sanitised before it reaches
`a.download`.** `safeBankFile` strips a trailing `.json`, replaces anything
outside `[A-Za-z0-9._-]` with a dash — one rule that covers both path separators
and the Windows-reserved set, rather than a blocklist — collapses runs, trims
leading and trailing punctuation, caps at 64 characters, and falls back to
`defaultBankName()` if nothing survives, so it can never emit a bare `.json`.
`exportBank(rawName)` takes the name as an argument rather than reading the DOM,
which keeps it callable from anywhere. The toast names the file actually written,
since a sanitised name can differ from what was typed.

Both keep direct `addEventListener` wiring rather than `data-action` delegation:
they live in static modal markup, not in anything `buildFlowStrip` rebuilds.

Both keep direct `addEventListener` wiring rather than `data-action` delegation:
they live in static modal markup, not in anything `buildFlowStrip` rebuilds.

- `exportBank` writes `{format: 'recur-bank', version: 1, build, saved, presets}`.
  `importBank` accepts that and a bare array.
- Import **merges, never replaces**: an imported bank must not destroy the local
  set, and a merge is undoable by deleting. Identity is `name + date`, so
  re-importing the same file is a no-op. Malformed entries are counted and
  skipped.
- `savePresets` is wrapped and returns false rather than throwing. `localStorage`
  is ~5 MB and a preset carries per-slot params plus three mod tables for up to
  eight slots, so the quota is reachable; Safari private browsing throws on the
  first write.
- Drag-and-drop accepts `.json` as well as video.
- A jump applies full state exactly as saved, **including mode**, so a preset
  saved in LIVE will start the camera.
- `[` and `]` step the bank. Both keys were free and sit next to each other on
  every layout, which is what a live control needs.

---

## SPI panel

The panel is `width: min(432px, 100%)`, fixed — the resize handle sets
**height** only. `#spi-panel` is `overflow:hidden`, so any row that cannot fit is
**clipped, not scrolled**. Every row therefore has to be able to reflow.

### Panel header

Nothing here gets a line to itself:

```
#spi-header-top   SHADER          [SAVE] [LOAD] [SHARE] [CAST]
#spi-mode-hdr     MODE ─────────────── 2026-08-12c · header + bank name
                  [sampler] [shader] [live]
```

`#spi-mode` is `flex:1`, which pushes the buttons right. The build marker rides
on the **MODE section rule** — the word MODE is 30px of a 408px rule and the rest
was empty, so the marker costs no height at all there. It has been on both other
lines and neither worked: sharing the button row clipped CAST, and moving it onto
the mode line pushed the buttons down a row.

Nesting a `<span>` inside a `.spi-section` is safe **only because MODE is not
foldable**. `_spiFoldables()` matches `#spi-body .spi-section`, and this one
lives in `#spi-header`, so `_spiFoldKey`'s `textContent.trim()` never reads it.
Putting the marker in any other section header would make the build string part
of that section's `localStorage` key.

**Nothing in either row can shrink.** Every item is `white-space:nowrap`, and a
nowrap flex item has `min-width:auto`, so `flex:1` and the default
`flex-shrink:1` are both inert on them. The header rows therefore carry
`flex-wrap`, as `#spi-zoom-row` does — without somewhere to wrap to,
`#spi-panel`'s `overflow:hidden` **clips** the last item rather than reflowing,
which is how CAST used to vanish on any viewport under ~412px, i.e. most phones.

`#spi-build` is `flex-shrink:0`: a truncated build marker cannot answer the
question it exists to answer, so it wraps whole or not at all.

Both rows clear every realistic width — 277px and 189px against 296px of usable
panel even at a 320px viewport. The build name has **77 characters** before the
MODE rule wraps.

`updateSPI` is throttled ~33 ms, dirty-checked via `_spiPrev`, refs cached in
`_spiRefs`. `_spiRefs` is built before the shader grids exist, so `genBtns` /
`fxBtns` are **re-captured** after the grids are in the DOM — without that, the
highlight and chain-position badge loops silently do nothing. Click delegation
lives on `#spi-panel`.

`_setTxt` and friends exist because writing `textContent` unconditionally every
frame invalidates layout even when the string is identical.

### Collapsible sections

State in `localStorage` under `recur.spiCollapsed`. Content is found by walking
siblings to the next header. Two special cases:

- **MODE is excluded by omission** — a sibling walk there would swallow half the
  panel.
- **PARAMS owns one specific element** (`#spi-params`): `#spi-params` is followed
  by `#spi-body` and neither carries `.spi-section`, so a walk would fold the
  whole panel body away.

`SPI_DEFAULT_COLLAPSED = ['VIDEO', 'LFO', 'AUDIO INPUT', 'MIDI', 'MISC']` — the
two shader pickers are what you reach for constantly, the rest are
set-and-forget. Only applies on a fresh install; once anything is toggled the
user's layout wins.

Folding SHADERS AND FX CHAINS sets `_spiViewHidden`, which stops
`buildFlowStrip` rebuilding into something invisible; `_spiFlowDirty` forces one
rebuild on unfold so it never reappears stale.

### The column ruler

Two grid elements — `#bank-strip` and `#flow-strip` — share
`grid-template-columns: 46px 1fr` so they read as a single grid. Four rows down
the panel:

```
RND  │ ▁   SRC  ○CAM  ▁   BANK  OVR  CLR    #bank-strip
   ██p1 ██p2 ██p3 ██p4 ██p5 ██p6
GEN  │ ██   ██   ██   ██                     #flow-strip
FX   │ ██   ██   ██   ██
```

Two blank slots, both load-bearing: one after RND, because it replaces both
chains wholesale, and one between CAM and BANK, because what the source controls
do has nothing to do with what the bank controls do. Without the second the row
reads as a single run of five.

- **The top row is everything that is not a chain.** RND leads from the label
  gutter, one empty slot clear of the rest, because it replaces both chains
  wholesale and should not read as part of any group. SRC and CAM used to have a
  row of their own for two blocks; folding them up here cost nothing and saved a
  row of height.
- **The preset row spans the gutter** (`grid-column: 1 / -1`) and starts hard
  left. It is the one row in the section deliberately *off* the shared ruler: a
  preset is not a chain position, nothing above or below it lines up, and the
  46px buys most of another block per line.
- The label gutter is fixed at 46px, not `auto` — an auto column would shift the
  rows sideways when the BANK label gains a page digit.
- `#flow-strip` and `#bank-strip` are both `grid-template-rows: auto auto`.
- GEN and FX emit **identical geometry** — `CHAIN_MAX` boxes — whatever the
  chains hold. Emitting only the filled positions is what pulled the rows out of
  line: the insert count tracked chain length, so a 1-slot FX chain sat a whole
  block left of a 3-slot GEN chain. There are now no insert spacers and no flow
  arrows between boxes; every row is boxes on the shared 42px + 3px pitch, so
  column N lands at the same x on GEN and FX.

### Everything on the top row is rebuilt

`buildFlowStrip` replaces `#bank-strip`'s `innerHTML`, so **nothing on that row
may hold a direct listener** — every control resolves through `data-action`
delegation on `#spi-panel`. RND, SRC, CAM, BANK, OVR and CLR all qualify.

This is a deliberate reversal of an earlier decision. The camera button was
*moved out* of the strip precisely so it could survive the rebuild and be patched
in place before the dirty-key check. Now that it sits inside the rebuild, its
state is **emitted** instead, which means **`camOn` has to be in the dirty key**
or a camera toggle would not repaint. The trade is one strip rebuild per camera
toggle against SRC/CAM sharing the RND line. If anything on this row ever needs a
real listener, it has to move back out to static markup.

### Bank rows

`BANK` is a block on the top row, not a gutter label, and becomes the page
control (`BANK1`, `BANK2`…) only once there is a second page to reach.

`BANK_PAGE = 8`. The preset row wraps to at most two lines and pages beyond that.
8 is the *guaranteed* fit, not the typical one: worst case every name hits the
64px ellipsis cap, so a 356px row takes `floor((356+3)/(64+3)) = 5` blocks a
line — and the preset row is now the full 402px with the gutter reclaimed, so 8
fits with room. `_bankFollow` pages to whichever page holds the current preset.

**EXP is not on the strip.** Bank export is `EXPORT BANK` inside the SAVE dialog
and nowhere else — one route is enough for something reached between sets rather
than during one. See *Preset bank*.

Bank blocks carry **no `data-fb-type`**: the strip's `pointerdown` matches
`.fb[data-fb-type]` for dragging and a preset is not a chain slot. They use
`data-action="preset-jump"` through the panel delegation. The bank is not a drag
target at all — drag listeners bind to `#flow-strip` only.

`presetIdx` is declared with `_flowStrip`, not with the bank functions —
`buildFlowStrip` reads it and a later `let` is in TDZ. It is in the strip's dirty
key, as are `bankPage`, `camOn`, `_armSlot`, `_ovrArm`, `_clrArm` and
`_ovrTarget`.

### The two armed controls

**There is no `confirm()` anywhere in the file.** Both destructive bank
operations are guarded on the strip instead: a modal is the wrong thing to put in
front of someone mid-set, and lighting the block you are about to destroy says
more than a dialog does.

`OVR` and `CLR` both consume the next click and both act on the bank, so **only
one can be armed at a time**. `setOvrArm` and `setClrArm` each disarm the other,
and only in their `on` branch — disarming is the quiet branch, which is what
keeps the pair from recursing.

| | taps | |
|---|---|---|
| **OVR** | 3 | Arm OVR → tap the target preset, which lights `fb-ovr-target` and is named in the toast → tap that **same** block again to write. Tapping a different preset **re-aims** rather than writing, so a slip moves the target instead of destroying the wrong preset. While armed, a preset click never jumps. |
| **CLR** | 2 | Arm CLR → tap CLR again. Deletes **every** saved preset via `clearBank`. |

`_ovrTarget` is the armed target's index, or -1. `setOvrArm` clears it on **every**
transition, including disarm — leaving one behind would let the next arm commit
against a preset chosen several clicks ago.

Both commit paths check the write before claiming success. `clearBank` resets
`presetIdx` to -1 and `bankPage` to 0 only after `savePresets([])` returns true;
`overwritePresetAt` only toasts after `savePresets(list)` does. Claiming success
over the top of a quota warning is how you lose a preset you thought was saved.

`overwritePresetAt` mutates its `list` *before* the save — safe only because
`loadPresets` is `JSON.parse(localStorage…)` and hands back a fresh deep copy
every call. A test harness that returns the live array instead will report a
false failure here; model the parse.

Escape cancels either. Any other panel control cancels an armed CLR, via a single
guard at the top of the `#spi-panel` click handler rather than a `setClrArm(false)`
sprinkled through every case — a stale arm two clicks later is exactly how a bank
gets deleted by accident.

CLR is styled red rather than the bank controls' magenta, at three-class
specificity (`.fb-ctl.fb-clr.fb-armed`) so it wins over `.fb-ctl.fb-armed`
wherever the two sit in the sheet. `.fb-bank.fb-ovr-target` outranks the
loaded-preset highlight for the same reason: about-to-be-overwritten matters more
than currently-playing.

**`colorizer` abbreviates to `COLR`, not `CLR`.** Adding the bank's CLR button
put two different controls reading `CLR` in the same section — one an FX block,
one a bank wipe. The FX abbreviation moved.

### Source section

Shown whenever the camera is on, or in LIVE or SAMPLER.

- `#src-speed-row` — 0.25× to 4×, log-mapped, `t = 0.5` exactly 1×, every
  doubling the same distance along the track. `RATE_MIN = 0.25`, span 16.
- `#src-inout-row` — dual-handle range over the clip duration; handles drive
  `inPt`/`outPt`, which the boundary logic already honours. The cyan bar is the
  live playhead. Dragging parks the playhead on the handle being dragged.
- The base blend is governed by **gen slot 0's mode and amount**, not by the
  legacy `blendAmount` global, which nothing reads.
- Dim what is unavailable on this hardware; hide whole groups belonging to
  another mode. The camera group is about the camera being (or becoming) the
  source — front/rear are dimmed rather than hidden on desktop, which ignores
  `facingMode`, so the group keeps its shape.
- In LIVE the camera IS the source, so the camera button is lit but inert —
  `toggleCamera` would read a tap as "leave LIVE" and drop to SAMPLER. MODE is
  how you leave. The `L` key still calls `toggleCamera` directly.

Param drag requires clearer movement, and favours vertical, before committing to
a horizontal drag — a quick or slightly diagonal touch should scroll, not nudge
a parameter.

---

## LFO, audio, MIDI

Three oscillators `{shape, amp, offset, period, skew, curve}`.
`LFO_SHAPE_NAMES = ['sine','triangle','ramp','square','s&h']`.
`BPM_BEATS` has eight divisions, 1/8 to 16.

- **`skew`** warps phase *before* the shape is evaluated: a piecewise-linear
  remap sending `k` to the halfway point. Triangle peak slides, square duty
  becomes `k`, sine gains fast rise / slow fall. Identity at 0.5.
- **`curve`** bends the trajectory: below 0.5 accelerates toward the peak
  (spiky), above decelerates (plateaued). Applied to the unipolar output so it
  covers every shape including S&H. Linear at 0.5.

Audio: band scalars (bass/mid/treble/vol) are always computed, but the GPU
upload — ~1300 ops plus two `texSubImage2D` — is **gated on chain demand**, since
only `squarewaves` and `oscilloscope` read it. The history ring refills within
`AUDIO_HIST_H` frames.

- `AUDIO_TEX_W = 256`, 2 rows R8: row 0 spectrum (log-ish FFT downsample), row 1
  waveform. NEAREST so the rows do not bleed.
- History ring is `AUDIO_TEX_W × AUDIO_HIST_H = 256 × 64` R8. The newest waveform
  is written to `audioHistRow`, which advances and wraps; `uHistRow` tells the
  shader which row is newest so it can walk backwards through ages. That is how
  `oscilloscope` gets persistence trails with no frame-feedback machinery.

MIDI: the permission choice is remembered in `localStorage` under `recur.midi`,
so anyone actually using a controller is not asked every session.

Camera: `facingMode` is persisted per browser under `recur.camFacing` but
deliberately **not** carried in share URLs — it describes the device in your
hand, not the patch. Always plain `facingMode`, never `{ exact: ... }`: a device
reporting neither `user` nor `environment` (most desktop webcams) would return no
media at all under an exact constraint. `_camSwitching` guards a deliberate
switch, since stopping the old track fires `ended` and would otherwise have
`onCameraLost` reacquire the OLD facing mode and race the new request.

---

## Randomise

`randomiseAll` fills both chains: 4 distinct GEN, 4 distinct FX, all params,
blend modes and amounts, fresh seeds, LFO shape/period/amp re-rolled, a handful
of assignments across the three oscillators. Switches to SHADER. Every box is
taken, but `layoutNormalise` runs anyway so this is not a second place that has
to be kept correct.

- **Blend modes come from a weighted bag, `RND_MODES = [0,0,0,0,1,1,2,2,5,7,4]`**
  — normal ×4, add ×2, screen ×2, lighten, overlay, difference. `multiply`,
  `darken` and `burn` remove light, and a uniform pick blacks out a four-deep
  chain most of the time.
- **Slot 0 is forced to `normal` at 1.0** — it composites onto black, where any
  subtractive mode is an immediate blackout regardless of what follows.
- **Snapped params land exactly on a position:** `k/(len-1)`. Any other value
  renders between two selector positions, which for something like plasma's
  `style` means a half-applied style.
- `skew`/`curve` stay at 0.5 identity — warping an already-random shape makes the
  motion unreadable. `lfoUseBpm`, audio and MIDI bindings are not touched.
- Non-snapped params roll uniform in `[0.05, 0.95]` via `_rndParams`.

---

## Export

MediaRecorder at `videoBitsPerSecond: 8_000_000`, MP4 (`avc1`) where supported
and WebM otherwise. `EXPORT_LENGTHS` = 5, 10, 15, 20, 30, 45, 60, 90, 120
seconds, mapped linearly on the slider. Peak memory is roughly 2×: 8 Mbps × 120 s
≈ 120 MB of chunks, doubled transiently by `new Blob()`.

**Duration repair** is the substantial part. MediaRecorder writes container
metadata before it knows the recording length, so no duration is stored and
players report a few seconds regardless — the frames are all present, only the
header is wrong. `fixWebmDuration` writes a Duration element into WebM's Segment
Info (EBML walk via `readVint`/`readId`/`findBox`); `fixMp4Duration` patches
`mvhd`, `tkhd` and `mdhd`.

The duration written is **wall-clock elapsed, not the requested length**:
recording can start late or be stopped early, and writing a duration longer than
the frames produces a file that appears to hang at the end.

Blob URLs are revoked on replace and 10 s after export.

---

## Cast

`BroadcastChannel`. The output window renders from state pushed by the control
window. `applyState` on the receiver rebuilds every slot and touches the DOM, so
the sender **diffs first** — when nothing has changed it sends only the audio
levels.

---

## Diagnostics

- **`P`** — `perfSample` takes the rAF-to-rAF delta at the top of the frame.
  Rolling one-second window. A frame over 1.5× the 60 Hz budget is counted as
  late, rather than averaged away, because dropped frames are what a set actually
  looks like going wrong. `_bufMB` mirrors the real allocation.
- **`I`** — `diagReport`: renderer, vendor, backend, mediump and highp precision,
  texture limits, float render targets, dPR, screen and buffer size, cap and
  scale, source line, UA. COPY button.
- **The mediump line matters most:** 10 bits is true fp16 (Apple Silicon, ~2×
  throughput), 23 is fp32 (ANGLE on Windows) — a real cross-platform speed
  difference from identical shaders.

`#hud` is `display:none` and nothing turns it on. `_paintHud` is split out of
`updateHUD` behind a visibility check — **not** an early return, because
`updateHUD` also dispatches `updateSPI`. The check reads computed `display`
once, into `_hudLive`.

### `?dev` hot-reload

Opt-in. With `?dev` in the URL any `.glsl` in `shaders/manifest.json` overrides
its inlined counterpart and recompiles on change, substituting the `${ASCII_*}`
tokens and keeping the previous program if the new one does not build. Without
`?dev` it returns immediately and issues no requests, so the shipped build is
unaffected.

---

## Shader notes worth knowing

### Precision

The shared header is `precision mediump float`, which is **genuine fp16 on
mobile and Apple Silicon**. Precision is managed at two levels.

**Global `precision highp` — four shaders**, all in GEN. A global precision
statement applies to everything after it, so it promotes the whole shader
without touching `H` or any other entry:

| shader | reason given in the file |
|---|---|
| starfield | none stated |
| maze-flight | the camera's z **is** the clock — every value is position-and-time math on an unbounded coordinate |
| quaternion | the running derivative `md2` reaches ~1e10, five orders past fp16; at mediump the DE returns 0 everywhere and paints a solid block |
| mandelbox | `dr` compounds as `dr*|scale|+1` and reaches ~4e8 at 16 iterations, four orders past fp16's 65504 |

**Local `highp` declarations** — thirteen more shaders promote individual
values, and function params receiving one must be `highp` too:

- `plasma` and `flowing-colours` declare `highp float t`, because `uFrame` is
  unbounded: at mediump it stops resolving single frames past ~2048, then
  quantises small spatial terms away when added to a large `t`.
- `ascii` needs it for the atlas address specifically — a texel index rebuilt
  from a normalised coordinate, where mediump carries ~0.4 texels of error
  against a 640-texel row, enough to sample the neighbouring glyph's column.
- `feature-dots` needs it for the gradient products, which land at 1e-2 to 1e-4
  and band into visible steps at mediump.
- `scatter` needs it for `rate`, then folds the value because it lands beside a
  cell index in a hash where a large number swallows the index.

### Time phases

**Wrap every time phase at the period of what reads it.** `th = TAU*fract(...)`
is seamless for `cos(th)`/`sin(th)`, but *halving* it is not. Both `quaternion`
and `mandelbox` run the tumble at half the morph rate and therefore carry their
own half-rate phase — halving `th` instead put a 180° flip in one frame at every
wrap. Where a coordinate is unbounded rather than periodic (`maze-flight`), the
world is **rebased**: `gK` carries the displacement as an exact integer and the
march only ever touches a local `p` near the origin.

Relatedly, `maze-flight` uses **integer hashing**, not `fract(sin(x)*43758)`. On
unbounded coordinates the sine argument reaches the tens of thousands and fp32
sheds the mantissa the hash depends on. Integer ops are exact at any magnitude,
and cheaper.

### Coordinate anchoring

`vU = aP*.5+.5`, so the UV origin is bottom-left. A lattice built as
`floor(vU * N)` is pinned to that corner, and animating size slides the pattern
diagonally instead of scaling in place. `rot2()` rotates about the origin, so an
angle param swings the whole screen around the corner.

Centre-anchored: **ascii, halftone, bitcrush, feature-dots**. Deliberately left
corner-anchored: **gamma-ray** (measured at exactly 1.0× from re-anchoring — the
cell value is re-randomised every frame), **grain** (per-pixel noise, no spatial
scale), **oscilloscope** (already centred), **scatter** (its cell count tiles the
frame exactly, and re-anchoring would break the permutation that keeps `swap` an
involution).

Transform: replace `vU` with `(vU - 0.5)` in the lattice, add the centre back
when converting to a texture coordinate.

### Raymarched shaders

`maze-flight`, `quaternion` and `mandelbox` cost far more than the rest of GEN.
Each has a `detail` slider that buys surface detail with frame rate — drop that
before dropping RES.

**quaternion** (GEN 13, 12p) — port of "Gilded Quaternion" by ufffd (CC0-1.0),
a 4D Julia set with Inigo Quilez's analytic DE, orbit-trap colour and a hot rim
light. Ranges measured on a CPU port.

- `zoom` is **inverted against camera distance** so up means closer. Default
  `.592` is distance 2.45 (the ISF's 2.7 was framed for a square poster; on 16:9
  that leaves the object at 14%, and 2.45 gives ~19%).
- At the near end the frame is 78-100% object for four of the five form seeds,
  and the marcher is *cleaner* there — near-miss rescues fall from 1.5% to 0.2%.
- `form` walks a smoothstepped path through five seed quaternions; the object
  holds 18-40% of frame across the range.
- The 4th component is the **slice** — a 3D cross-section of a 4D object, so
  moving w cuts a different section rather than deforming one. The set is empty
  past |w| ~0.85, so the slider stops at 0.75.
- `detail` is a cost control, not a shape control: silhouette held at 26.4-26.9%
  from 5 to 14 iterations.
- Grazing rays leave black speckle along the silhouette (0.35% of frame, 1.4% at
  detail 0); accepting a near-miss inside the bounding sphere recovers about two
  thirds.
- Palette gains are the reciprocals of each trap channel's measured 5th-95th
  spread (0.157 / 0.343 / 0.947), so any selection sweeps a full turn.

**mandelbox** (GEN 14, 12p) — Tom Lowe's Mandelbox on the quaternion's camera
rig, orbit trap and shading block; only the map differs. Folded architecture
rather than a creature. Ranges measured on a CPU port.

- **Surface radius is `RK = 3.42 × fold` and does not depend on `scale`**
  (300k samples). The camera is normalised against it, which turns `fold` from a
  size knob into a pure shape knob: apparent size holds 19.9-22.7% of frame
  across the whole slider, so nothing needs re-framing after a fold move.
- `scale` is **warped** so equal slider steps give equal silhouette change: under
  a linear map the shape moves 3.7× faster near -3.2 than near -1.5; warped, the
  spread is 3.0×.
- `morph` clamps both knobs inside the measured range, so it cannot drive them
  anywhere that has not been checked for a blank frame.
- No transcendentals in the DE loop at all, which is why it costs about what the
  quaternion does despite three times the iterations. `dr` only ever grows, so it
  needs no divide-by-zero guard.
- Default `zoom .392` is camera distance 2× the object radius, framing at 22%.

**maze-flight** (GEN 12, 12p) — port of "Can't Find My Way Out" by ksin
(CC0-1.0). Corridors carved at three nested scales; corridors are keyed to the
**face**, not the cell, so adjacent cells agree at the seam and the field stays
continuous. A guaranteed corridor follows the camera path so the flythrough
never walls itself in — Chebyshev distance to a moving centre is not a true SDF,
so it is scaled to stay Lipschitz-safe (0.5 against the ISF's 0.2; measured zero
overshoot at 0.8 over 43,200 rays). `uSeed` is `mod()`ed, since it runs to 10000
and would otherwise start the flythrough tens of thousands of units downrange.

### Other shaders

- **plasma** (GEN 0, 9p) — classic/fractal/warped/ridged/marble/ripples/julia/
  flow. The julia zoom window is narrow on purpose: too tight is smooth interior,
  too wide aliases into moiré.
- **zoom-clouds** (GEN 9, 11p) — `contrast` pulls the ramp **gain** back while
  steepening the curve; the base ramp multiplies f by 2 and blows ~47% of a frame
  to flat cream. Exactly the base ramp at 0, short-circuited there rather than
  trusting the algebra, which differs by a float ULP. `rough` renormalises to the
  weight at g=0.5 (0.96875), not to 1, so 0.5 returns the original untouched.
  `relief` is normalised by `uRes` because raw `dFdx` scales with pixel size.
- **glitch** (FX 1, 9p) — run-length coherent macroblock damage in the manner of
  KinoGlitch / KinoDatamosh. One bad motion vector is reused by a **run** of
  consecutive macroblocks so damage streaks rather than scattering, capped at 8
  (mean run ~8 cells at a 0.12 break chance). Displacement is snapped to the
  lattice, as a decoder applies an integer motion vector — unsnapped offsets
  smear instead of tiling. Dropout uses real MPEG/JPEG failure modes (DC-only
  blocks, scrambled-AC blocks as a directional DCT ripple, chroma desync, channel
  roll), biased toward already-displaced blocks so both stages read as one fault.
- **ascii** (FX 15, 6p) — 5 charset blocks, each density-sorted **separately**
  (a sub-range of a globally sorted list is not itself a correct density ramp).
  `ASCII_N = 506` glyphs, `ASCII_COLS = 64`, so the atlas is 640×112 — a single
  row would be 5060px, past the 2048 every WebGL2 device guarantees. Glyphs are
  drawn smaller than the cell with the baseline placed inside it, because a
  monospace face needs ~1.21× its em for ascent+descent and full-size drawing
  clips descenders, which then ranks them too light in the density sort.
  `variation` jitters luma **before** quantising, keeping the glyph index
  integral — a fractional index straddles two atlas cells and draws glyph
  fragments. Measured: 1 distinct glyph at 0, 10 at 0.2, 22 at the 0.45 default.
  Charset 0 is `all`, so a preset written before the param existed loads as 0 and
  keeps the look it was saved with; new instances default to block 1.
- **feature-dots** (FX 19, 9p, `usesPrev`) — dense Shi-Tomasi cornerness, one
  candidate per cell so dots cannot clump. Flow is Lucas-Kanade, dense rather
  than sparse, reusing the tensor cornerness already needs. Solved once per
  fragment rather than per candidate (4×36 taps becomes 36). The lattice lives in
  **material space**, advected by the RG16F pair: each texel stores the uv the
  material now at this position started from, so a dot slides with what it
  marked instead of the grid blinking as features cross it. Relaxation is
  essential — without it the field shears without bound and the lattice tears
  itself apart within seconds. The ±2-cell scan window bounds jitter, which maxes
  at 2.0.
- **sample-hold** (FX 18, 1p) — the only entry with no `src:`; the frozen-frame
  buffer *is* the effect.

---

## Traps

- **Never regex-delete across `</div>` boundaries.** Locate the region and
  rebuild it wholesale.
- **Never resolve a shader field by scanning forward from `name:`.** Anchoring on
  `name: '(\w+)',\n\s*params:` is wrong — `feature-dots` has `usesPrev: true`
  between them. Use the entry line itself as the boundary:
  `^    name: '([\w-]+)',$` at four-space indent, each match running to the next.
- **Scan the template literal, do not regex it.** Greedy `` [\s\S]*` `` runs the
  last entry's source to the end of the file and swallows the ASCII block's
  backticks. Walk from the opening backtick honouring `${...}` nesting.
- **A brace-matching extractor must skip the parameter list.** `opts = {}` closes
  on itself and returns an empty body. Match the parens first.
- **Never put a backtick in a GLSL comment** — shader sources are template
  literals.
- **`vid.currentTime` returns the seek target on assignment.**
- **`readyState` drops to 1 during a seek.** Never gate a texture upload on it in
  code that seeks continuously.
- **GL uniforms are per-PROGRAM and persist.** See `setUniforms`.
- **mediump is fp16 on Apple Silicon and mobile**, not a hint.
- **`const` in `eval()` does not leak.** Test harnesses need `var`.
- **TDZ:** several declarations sit deliberately *above* the code that reads them
  — `_vidTexW`/`_vidTexH` above `buildGLResources`, `feedbackInit` with the
  framebuffer state, `presetIdx` with `_flowStrip`, `_spiViewHidden` before
  `initSpiCollapse`. Do not tidy them downward.
- **`innerHTML` destroys child listeners.** The camera button left `#flow-strip`
  to become static markup; it works via `data-action` delegation on `#spi-panel`.
- **Unicode glyphs tofu on mobile.** Icons are inline SVG and CSS border
  triangles.
- **A range input will not shrink below its UA intrinsic width** whatever `flex`
  says, because `min-width` defaults to `auto`. The same rule bit the panel
  header: a `white-space:nowrap` flex item has `min-width:auto` too, so `flex:1`
  and `flex-shrink:1` are both inert on it. A row of nowrap items cannot shrink
  at all — it must wrap, or the panel's `overflow:hidden` eats the last one.
- **`#spi-panel` clips; it never scrolls.** Any new row has to reflow on its own.
- **`_padSlots` fills with 0, not `def`.**
- **Adding a 13th param to any shader overflows `N_P`.**

---

## Validation

Everything below was run against this file in this environment and passed. It is
reproducible without a GPU.

| check | how | result |
|---|---|---|
| JS syntax | extract the single `<script>`, `node --check` | pass |
| HTML nesting | strip script/style, `html.parser` with an explicit tag stack | 0 errors, 0 unclosed |
| element balance | `div`/`span`/`button`/`svg`/`select` open vs close | 399/399, 451/451, 67/67, 33/33, 2/2 |
| CSS braces | count over the single `<style>` | 349/349 |
| shader table | walk `GEN`/`FX` with the entry-line boundary regex | 36 entries, 35 with `src:`, `sample-hold` the exception |
| GLSL parse | `@shaderfrog/glsl-parser`, prefixes resolved, `ASCII_*` **evaluated** in node rather than stubbed | 35/35 parse |
| GLSL scope audit | walk parser scopes for bindings without a declaration | clean; `TAU` ×3 is a `#define` the parser does not preprocess |
| layout invariant | extract the `layout*`/`chain*` functions verbatim, drive 20,000 trials × 40 random operations including hostile wire positions up to 255 | 800,000 operations, **0 violations** |
| arm state machine | extract `setOvrArm`/`setClrArm`/`clearBank`, mock `loadPresets`/`savePresets`/`toast`, fuzz 20,000 arm/disarm calls | never both armed, no recursion; `clearBank` leaves the bank and `presetIdx` untouched when the write fails |
| OVR three-tap flow | lift the `preset-jump` case body **verbatim** out of the switch and drive it, with `confirm` stubbed to throw | targets without writing, re-aims on a different block, writes only on the second tap of the same one, disarm drops the target, no false success on quota failure, `confirm` never reached |
| flow strip | drive the real `buildFlowStrip` against a mocked DOM | 3 grid children, RND in the gutter, preset row has no gutter cell, control order SRC→CAM→BANK→OVR→CLR, EXP absent, 8 presets a page, none on the control row, dirty key reacts to `camOn`, `_clrArm` and `_ovrTarget`, target block lights |
| boot layout | `eval` the real `let genSlots = …` declaration read out of the file | `pos === 0`, `layoutAt(genSlots, 0)` resolves |
| preset naming | extract `defaultPresetName` + `_MON` + `_p2`, stub `Date` at fixed instants, truncate to the measured 10-character block width | correct at midnight, single-digit months and days; the word `preset` is what gets cut, never the timestamp; 207 distinct visible names across a day |
| bank filename | extract `safeBankFile` + `defaultBankName`, drive 12 hostile literals plus 3,000 fuzzed strings of path separators and shell metacharacters | output always matches `^[A-Za-z0-9._-]+\.json$`, never leads with punctuation, never exceeds 69 chars, `../../etc/passwd` → `etc-passwd.json`, all-punctuation input falls back to the dated default |

Method notes that matter:

- **Evaluate `ASCII_*`, do not stub them.** Stubbing the computed ones as `1`
  makes the ascii source fail to parse and reports a false failure. Extract the
  constant block, stub `document.createElement` with a Proxy returning no-op
  canvas methods, and `eval` it — `ASCII_N` comes back 506, atlas 640×112.
- **Locate harness slices by content, never by line number.** Two of these
  harnesses were written against line offsets and broke on the first edit that
  shifted them. Anchor on `src.indexOf('const ASCII_CW')` and a regex for the
  declaration, so a passing run means the current file passed.
- **Lift the branch under test verbatim.** The OVR harness slices the real
  `case 'preset-jump':` body out of the switch and rewrites `break` to `return`
  rather than paraphrasing the tap sequencing — a paraphrase tests the paraphrase.
- **Model the storage boundary, not just the function.** `loadPresets` deep-copies
  through `JSON.parse`; a mock returning the live array made
  `overwritePresetAt` look like it corrupted the bank on a failed write. The code
  was right and the harness was wrong, which is the failure mode to watch for
  when a test contradicts a careful reading.
- **Extract the real function and drive it.** Mock the DOM/GL surface, use `var`.
- **Read the count your tooling reports.** "ALL PASS" over 34 of 36 entries is
  not a pass.
- **Differential testing** for any refactor: extract old and new verbatim, drive
  both over the same seeded random operation sequence, diff by differing **field
  path** rather than by a boolean, and patch the old model with exactly the fixes
  claimed before re-running.
- **Numpy porting** for shader work gives exact preset-safety proof
  (`max|old−new| == 0`).
- **Match the metric to the complaint.** Mean pixel difference saturates as soon
  as a glyph changes; glyph-index distance was the correct metric for ascii.
- `/tmp` is wiped mid-session; rebuild rather than trusting a stale pass.

---

## Open threads

Each re-verified against this file, this session.

- **`_bufToStateV4` read offsets drift with `GEN.length`/`FX.length`.** Confirmed
  present — `const NG = GEN.length, NF = FX.length` inside the decoder. Every
  appended shader has shifted the offsets of every legacy V4 URL.
- **`fboFx` is never reaped, and the comment says it is.** `reapFBOs` tracks only
  `_reapHist`, `_reapPrev`, `_reapHold`. Code/comment disagreement — fix one.
- **`randomiseAll` rolls `zoom` uniformly.** `_rndParams` gives `[0.05, 0.95]` for
  any non-snapped param, so a random quaternion or mandelbox patch starts from
  inside the object a meaningful fraction of the time. Not weighted.
- **`chainMove` is dead code** — zero call sites.
- **`_armSlot` survives `applyState` and `randomiseAll`.** Cosmetic; see *Slot
  layout*. `_clrArm` does not have this problem — any other panel control
  disarms it.
- **`CLR` has no undo.** `EXPORT BANK` in the SAVE dialog is the only backup and
  the bank does not travel with the file. The arm step is the whole of the
  protection.
- **Every control on the top row is delegation-only, by necessity.** Nothing
  enforces it — adding one with an `addEventListener` would work until the next
  rebuild and then silently stop. See *Everything on the top row is rebuilt*.
- **Nothing warns before a long export take at full resolution.**
- **The perf overlay measures wall-clock frame time** and cannot separate GPU
  from main thread. `disjoint_timer_query` would.
- **Legacy `_bufToState` (pre-V4) is not length-checked** the way V5 is.
- **`captureScreen` does not run `assessSource`,** and both `loadVideo` and
  `captureScreen` set SAMPLER directly without the gen bypass that `setMode`
  applies.
- **Not GPU-verified** — which is to say all of it, but specifically: the
  `onResize` halve-and-retry path, `reapFBOs` timing, the render cap,
  `texSubImage2D` on the video upload, the quaternion and mandelbox camera rigs,
  and every measured range in this document that is attributed to a CPU port.

---

## Carried forward

*Not derived from this file. Retained from the previous context document because
it cannot be re-derived from source, and flagged so it is not mistaken for
verified fact.*

- Share URLs written before the V5 padding fix open with any since-appended param
  half-on. A padded 0.5 is indistinguishable from a chosen 0.5, so they are
  unrepairable.
- Reversing the quaternion `zoom` sense reframed every preset and share link
  written before build `2026-08-11o`. The bytes are still valid; the framing is
  not. Links from other people cannot be corrected.
- Share URLs are reported to break on recipients when a live camera is in the
  stack.
- The quaternion `slice` stop at ±0.75 may not hold across all `form` values —
  a CPU port rendered 0% coverage at both ends at `form` 0.5. Re-measure across
  `form` before trusting it.
- `zoom-clouds`'s `rn()` is not magnitude-safe. Changing the hash changes every
  existing preset's clouds, so fix only if black or speckled output appears.
- Pi 5: WebGL2 is hardware-accelerated, but BCM2712 dropped the H.264 decode
  block and Chromium cannot use the HEVC one, so decode is software and there is
  no hardware encoder. Plan on ½ render scale at 30 fps with RENDER CAP at 1080.

---

## When picking up work

1. Copy `index.html` from the mounted knowledge folder.
2. Preserve the single-file structure, the slot model, the layout invariant, and
   0-is-neutral.
3. Call `chainInsert`/`chainRemove` plus a `layout*` helper rather than splicing a
   chain by hand; call `togglePlay`/`toggleReverse`/`setMode` rather than
   duplicating them.
4. Re-derive the shader-table counts from the file. This document cites function
   names, never line numbers.
5. Validate: `node --check`, HTML nesting on every markup change, GLSL parse and
   scope audit with `ASCII_*` evaluated, the layout fuzz for anything touching
   `pos`, extract-and-drive for anything stateful, a differential harness for a
   refactor, numpy renders for anything visual.
6. Update the in-app help overlay for user-facing changes.
7. Bump `BUILD`. State that it needs a live GPU check. Surface the file.

## Files

`index.html` · `recur-web-project.md` · `README.md`
