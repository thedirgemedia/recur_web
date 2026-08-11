# recur-web — Project Context

Context document for **recur-recur-browser** (aka **recur_web**). Give this to
Claude at the start of a session.

*Reconciled against `index.html` on 2026-08-11 — 10123 lines / ~482 KB /
sha256 `561f5a52743b7a6d…`. `BUILD` reads `2026-08-11p · mandelbox`; bump before
deploy.*

Not GPU-verified: the `onResize` retry path, `reapFBOs` timing, the render cap,
and `texSubImage2D` on the video upload. `USE_TEXSUBIMAGE` reverts the last in
one line. Added to that list this session — the quaternion tumble phase, its
reversed zoom range, and the whole of `mandelbox`.

---

## What it is

- Browser-based live video effect and generation tool. WebGL2 sibling of the
  Raspberry Pi **recur** project (mpv-based video sampler on Pi 5).
- Repo `github.com/thedirgemedia/recur-recur-browser`, public, GPL-3.0.
- Deployed at `dirgemedia.com/recur…`. Dirge pushes and deploys manually.

## Rules

- **Single self-contained `index.html`.** No build step, no modules. CSS, JS and
  GLSL inline.
- **Replace the file directly; no git patches.**
- **Never claim GPU verification.** It is not available here.
- **0 is neutral for every appended shader param.** Prove it numerically.
- **Bump `BUILD` on every handover.** Shown in the panel header, logged to
  console.
- **One entry point per user action:** `togglePlay`, `toggleReverse`, `setMode`.
- **Mutate chains only via** `chainInsert` / `chainRemove` / `chainMove`.
- **The wire format is frozen.** `getState`/`applyState` are the only places the
  flat preset/share shape meets slot objects. Older builds must read newer files.
- **Comments:** one line for a function or constant, two to four for a section
  banner or non-obvious mechanism, half a line to justify a magic number. Never
  longer than the code it describes. No change history — it belongs here.

---

## Chain slots

`genSlots` and `fxSlots` are arrays of up to `CHAIN_MAX = 4` slot objects:

```
{ idx, params[], audioMod[], lfoMod[], midiMod[], seed, mode, amt, off }
```

| field | |
|---|---|
| `idx` | index into `GEN[]` / `FX[]`. Duplicates allowed in a chain. |
| `params[]` | one entry per declared param |
| `audioMod[]` `lfoMod[]` `midiMod[]` | one entry per param |
| `seed` | GEN only; FX is fixed at 0, the FX path passes no seed |
| `mode` | blend onto the layer below, index into `BLEND_MODES` |
| `amt` | blend amount (GEN) or wet/dry (FX, 1 = full effect) |
| `off` | bypassed |

`genIdx` / `fxIdx` are chain POSITIONS, not shader indices. `genSlots` is never
empty. `fxSlots` can be, with `fxIdx === -1`.

`makeGenSlot` / `makeFxSlot` wrap `makeSlot(table, idx, opts)`. FX defaults `amt`
to 1.0, GEN to 0.5.

`CHAIN_MAX` is enforced in **both** `chainInsert` and `applyState`.

### Replaced 13 parallel arrays

Formerly `genChain`, `genChainSeeds`, `genChainMode`, `genChainAmt`,
`genSlotParams`, `genSlotAudioMod`, `genSlotLfoMod`, `genSlotMidiMod`, the fx
equivalents, and `genDisabled`/`fxDisabled` as Sets of bypassed positions. Ten
sites wrote the lockstep mutation by hand; three had drifted, all live bugs:

1. `commitReorder` gen branch moved six arrays, left `genChainMode` and
   `genChainAmt` behind — a dragged GEN slot dropped its blend.
2. `commitReorder` deleted the bypass of the moved slot in both chains
   (`_adjustDisabled(fromPos, -1)` is the remove remap, not a relocate). Since
   `genChainToggle` branches on bypass, the next tap then removed the shader.
3. `commitInsert` gen branch never spliced `genChainMode`/`genChainAmt`, so a
   mid-chain insert shifted every downstream slot's blend onto its neighbour.

Every read site was correct. `_adjustGenDisabled` / `_adjustFxDisabled` are gone.

### Differential testing

The method that settled it, reusable for any refactor:

- Extract old and new functions **verbatim** from both files, drive them over the
  same random operation sequence, diff the resulting state. Use `var` — `const`
  in `eval()` does not leak.
- Seed a PRNG identically in both sandboxes. First run diverged only on `seed`,
  because `makeFxSlot` consumed a `Math.random()` the old FX path did not; that
  is why FX seeds are fixed at 0.
- **Patch the OLD model with exactly the fixes claimed, then re-run.** Unpatched:
  1,003 of 4,000 trials diverged. Patched with the three fixes above: 20,000
  trials × 40 operations, zero divergence.
- Classify divergences by differing **field path**, not by a boolean. The
  histogram separated bugs 2 and 3 and showed the whole-slot divergences were
  downstream of the bypass bug rather than independent.

Wire results: `getState()` shape identical to the old build 5,000/5,000;
`applyState(getState())` a fixed point 5,000/5,000; V5 bytes identical
5,000/5,000. One intended difference — `genDisabled`/`fxDisabled` emit in
ascending order rather than Set-insertion order. The encoder folds them into a
bitmask, so order is not part of the format.

---

## Memory ceilings

### Share links

`b64ToState` read the whole gzip stream with `Response.arrayBuffer()`. Measured
amplification 1028:1 — a 512 KB URL decompressed to ~386 MB. Largest legitimate
V5 payload is **541 bytes**.

- `_readCapped` reads incrementally, cancels past `MAX_SHARE_BYTES = 64 KB`.
  `MAX_SHARE_B64 = 256 KB` caps the encoded string before `atob`.
- The `buf[0] === 0x7b` branch `JSON.parse`s to `applyState`, which filtered
  chains for valid indices but did not cap length. A 200,000-entry `genChain`
  built 200,000 slot objects that the render loop then walked every frame. Fixed
  by `.slice(0, CHAIN_MAX)`; 200,000 entries now resolve to 4 in 14 ms.
- Tested: 541-byte payload accepted; 1 MB, 64 MB and 386 MB bombs rejected after
  ≤64 KB.

### Render buffers

RES offers 0.5 / 0.75 / 1 — there is no setting above 1×. Exposure is display
size × `devicePixelRatio`; `MAX_DPR_TOUCH = 2` clamps coarse pointers only.

| display | canvas | resident | worst case |
|---|---|---|---|
| 1080p | 1920×1080 | 24 MB | 71 MB |
| 4K @ dpr 2 | 3840×2160 | 95 MB | 285 MB |
| Studio Display 5K | 5120×2880 | 169 MB | 506 MB |
| Pro Display XDR | 6016×3384 | 233 MB | 699 MB |

Three RGBA8 always resident, four lazily (`fx`/`hist`/`hold`/`prev`), plus the
RG16F material pair.

- `capDims` applies `PIXEL_CAPS` — `off` / `1080p` / `4k`, default 4K, persisted
  to `localStorage` under `recur.pixelCap`. Both axes scale by
  `sqrt(budget / px)`. `MAX_TEXTURE_SIZE` is always applied. On 5K this takes
  worst case 506 MB → ~285 MB.
- Not in presets or share state, as with `renderScale`.
- `_makeAttached` checks `getError` and `checkFramebufferStatus` and returns
  `null` on failure; `makeFBO` returns a boolean; `onResize` halves and retries
  up to four times, resizing the canvas in lockstep because the render path's
  viewport calls use `cv.width`/`cv.height`.
- `reapFBOs` frees the four lazy buffers after `REAP_FRAMES = 120` frames of
  disuse. The delay stops a momentary bypass tap destroying feedback history.
  Scans ≤4 slots, so it runs every frame without a dirty flag.
- `exportChunks` is bounded by `exportLenSeconds` (120 s max), but peak is ~2×:
  8 Mbps × 120 s ≈ 120 MB of chunks, doubled transiently by `new Blob()`.

### Source texture

`uploadVideoFrame` uploads at the video's native resolution every frame. RENDER
CAP does not bound it: 1080p 8 MB, 4K 33 MB, 8K 132 MB.

- `texSubImage2D` once dimensions are known, `texImage2D` on a size change.
  `vidTex` is shared between clip and camera, so the branch compares dimensions.
  `USE_TEXSUBIMAGE` reverts.
- `buildGLResources` must reset `_vidTexW`/`_vidTexH` — it recreates `vidTex` at
  1×1 and also runs on context restore. The declaration sits above
  `buildGLResources` to avoid TDZ.
- A source wider than `MAX_TEXTURE_SIZE` fails with `INVALID_VALUE` and the
  texture keeps its last contents. `getError` is checked, reported once.

### Video length

`URL.createObjectURL(file)` streams; the media cache holds a moving window. A
4-hour file and a 4-minute file cost about the same resident memory.

Reverse playback scales badly with length, in CPU not memory: a backward seek
decodes forward from the preceding keyframe, up to 250 frames per displayed frame
on a typical GOP. Reverse itself is stable and repeatable — confirmed on device.

---

## Source assessment and binding

`assessSource` runs on `loadedmetadata`: resolution, duration, live detection,
and the `MAX_TEXTURE_SIZE` comparison. Over 4K it warns and does not alter the
source. `canPlayType` is checked before the load. `vid` has an `error` listener
covering all four `MediaError` codes. `startFpsProbe` estimates frame rate from
`presentedFrames`/`mediaTime` over the first second of rVFC, then stops — the
only route to fps from a `<video>` element. Two concurrent rVFC chains are legal.

A SAMPLER preset stores `{fileName, name, bytes, duration}` plus `inPt`/`outPt`.

- `srcFingerprint` is name + size + duration. No hashing.
- `_srcCache` holds the **File**, not an object URL: `loadVideo` revokes the
  previous blob URL on every clip change.
- `requestSource` runs last in `applyState` and never blocks; a missing clip
  raises a dismissible bar.
- `_srcDeclined` is per-session.
- `bindSourceFile` restores `modeIdx` after `loadVideo`, which forces SAMPLER.
- in/out are applied in `assessSource`, not `applyState`, and clamped to the real
  duration. `_pendingInOut` carries them across.

`inPt`/`outPt`/`src` are additive keys: older builds ignore unknown JSON keys,
V5 carries no descriptor, and a pre-change preset applies with `src` absent.
Verified — pre-existing keys byte-identical 5,000/5,000, new keys present in new
and absent in old 5,000/5,000, stripped legacy and V5-decoded states both apply.

---

## Preset bank

`localStorage` key `recur-presets`. Per-browser-profile and per-origin; not part
of `index.html`. On `file://` Chrome shares one opaque origin across every local
file and Safari often refuses. `fetch()` on `file://` is CORS-blocked, so a
sidecar `presets.json` cannot be auto-loaded — transport is a user-initiated bank
file.

- `exportBank` writes `{format, version, build, saved, presets}`. `importBank`
  accepts that and a bare array.
- Import **merges**. Identity is `name + date`, so re-importing is a no-op.
  Malformed entries are counted and skipped. Quota failure propagates from
  `savePresets`, which returns false rather than throwing.
- Drag-and-drop accepts `.json` as well as video.
- `savePresets` is wrapped: `localStorage` is ~5 MB and Safari private browsing
  throws on the first write.

### Bank row

The flow strip is three rows — GEN, FX, BANK — hence
`grid-template-rows: auto auto auto`. SRC sits above it in `#spi-src-cap-row`.

- Bank blocks carry **no `data-fb-type`**: the strip's `pointerdown` matches
  `.fb[data-fb-type]` for dragging. They use `data-action="preset-jump"`.
- `presetIdx` is declared beside `_flowStrip`, not with the bank functions —
  `buildFlowStrip` reads it and a later `let` is in TDZ.
- `presetIdx` is in the strip's dirty key.
- Delete and save both shift indices; both call sites adjust `presetIdx`.
- `[` and `]` step the bank.
- A jump applies full state including mode, so a LIVE preset starts the camera.

---

## Diagnostic overlays

- **`P`** — `perfSample` takes the rAF-to-rAF delta at the top of the frame.
  Rolling one-second window, repainted twice a second. Shows fps, mean, peak, and
  late (frames over 25 ms). Amber over 16.8 ms, red over 33.3 ms. `_bufMB`
  mirrors the real allocation.
- **`I`** — `diagReport`: renderer, vendor, backend (native / ANGLE-D3D11 /
  ANGLE-Metal / ANGLE-Vulkan), mediump and highp precision, texture limits, float
  render targets, dPR (flagged when fractional), screen and buffer size, cap and
  scale, source line, UA. COPY button.
- **mediump precision:** 10 bits = true fp16 (~2× throughput, Apple Silicon),
  23 bits = promoted to fp32 (ANGLE on Windows). Same shaders, different speed.

Cross-platform bisect, in order: RES ½ — if fps roughly quadruples it is
fill-rate bound, if flat it is not GPU-bound. Then an empty FX chain with one
cheap GEN — still slow means upload or compositing, not shaders. Then
`chrome://gpu` for the active adapter and hardware acceleration status.

Other measured cross-platform factors: per-frame texture reallocation (fixed),
`UNPACK_FLIP_Y_WEBGL` defeating the fast upload path when `vidFlipV` is on, CSS
`transform:scale()` on the canvas at zoom ≠ 1, and fractional `devicePixelRatio`
forcing compositor resampling.

---

## Leak audit (2026-08-11)

No leak found. These are correct; do not re-audit:

- `webglcontextlost`/`restored` handled with `preventDefault()`, program cache
  reset, audio textures nulled.
- `_dropFBO` deletes framebuffer and texture; `makeFBO` frees before allocating.
- `hookFramePresented` is idempotent via `_framePresentHooked`.
- `stopCamera` / `audioStop` stop tracks, disconnect nodes, close the context.
- Blob URLs revoked on replace and 10 s after export. `toast` reuses one element.
  `_tapTimes` trimmed. Audio history is a fixed ring. `?dev` poller is gated.

Bounded and deliberate: shaders are not `deleteShader`'d after link (≤2 per live
program); `audioTex`/`audioHistTex` are nulled only on context restore.

`#hud` is `display:none` and nothing turns it on. `_paintHud` is split out of
`updateHUD` behind a visibility check — **not** an early return, because
`updateHUD` also dispatches `updateSPI`. The check reads computed `display`, not
`offsetParent`, which is null for `position:fixed` even when visible.

---

## Architecture

- WebGL2, `#version 300 es`, `precision mediump float` in the shared header `H`.
- `MODES = ['SAMPLER','SHADER','LIVE']`.
- **`GEN[]` 15, `FX[]` 21 — 36 entries, 35 with GLSL.** Entry:
  `{name, params, def, [src], [usesPrev], [snapVals]}`. Append at the end.
  `sample-hold` is the only entry with no `src:`.
- Field formatting is not uniform: `levels` uses `src: H + \`` with spaces;
  others use `src: H+\``. `feature-dots` puts `usesPrev: true` between `name:`
  and `params:`.
- FX blend gate: `slotAmt < 0.999 || slotMode !== 0`.

### `setMode`

Every interactive mode change routes here: ENTER, the MODE button, the three
set-mode buttons, `toggleCamera`.

- SAMPLER and LIVE bypass the gen chain on arrival; the chain is left intact.
- SHADER clears an all-off bypass on arrival. A partial per-slot bypass is kept.
- `applyState` does **not** use `setMode` — a preset carries its own bypass set.
- `loadVideo` and `captureScreen` set SAMPLER directly without the bypass.

## Sampler playback

- `loop` has three branches: ping-pong, reverse, forward-native.
- `stepPlayhead` advances a virtual playhead `_stepT` at the true rate and pushes
  it to the element. Do not derive it from `vid.currentTime`: during an in-flight
  seek that value has not moved, so every frame retargets from a stale position.
- **Do not add a divergence-resync guard.** One deadlocked: on slow seeks the
  virtual head legitimately runs ahead. Anything that relocates the clip sets
  `_stepT = null`.
- `_seekPending` allows one seek in flight, cleared when a frame is **presented**
  (rVFC), not on `seeked` — that fires before compositing.
  `SEEK_WATCHDOG_MS = 150` reopens the gate, and deliberately overrides
  `vid.seeking`: a seek that never retires leaves `seeking` stuck true.
- `togglePlay` / `toggleReverse` are exclusive; each snaps the playhead off the
  boundary it heads towards. The icon shows the action, not the state.

### Solved: reverse showed a static frame

`uploadVideoFrame` bailed on `readyState < 2`. A seek in flight drops
`readyState` to 1 and the reverse stepper seeks every frame, so every upload was
skipped. Diagnosed by on-device counters: `up0 bail120` against `vfc116`.

Fix: accept `readyState >= 2` **or** "a frame has been presented and
`videoWidth > 0`", and upload inside `requestVideoFrameCallback`. `render()`
skips the repeat via `_vidTexFresh`.

Three things looked like the cause and were not: `vid.currentTime` returns the
seek target on assignment; backward seeking measured healthy in isolation
(`seek-check.html`, 20/20 seeks, 6 ms mean); three rounds of static analysis each
found a real stepper bug that fixed no symptom.

---

## Preset and share-URL safety

- **GL uniforms pad with 0.** `setUniforms` writes every declared param uniform
  every time: uniforms are per-PROGRAM state and the program is shared by every
  slot using that shader.
- **V5 writes `N_P = 12` param bytes per slot** regardless of declared count. It
  padded with 0.5, which broke the neutrality rule — a URL made when plasma had
  four params fed 0.5 into `style`, which snaps to index 4. Now pads with 0.
  Byte layout unchanged.
- **URLs written before that fix cannot be repaired.** A padded 0.5 is
  indistinguishable from a chosen 0.5.
- `getState` pads `genChainMode`/`genChainAmt` back out to 4: the encoder writes
  four fixed nibble pairs (bytes 14-17) whatever the chain length.
- `_padSlots` normalises every slot array to the shader's current param count.
  JSON presets arrive SHORT (undefined entries throw in the panel every frame),
  V5 URLs arrive LONG with 12 entries. Missing entries fill with **0, not `def`**
  — `def` is the fresh-instance value (plasma's `detail` is 0.45).

### Appending a shader is wire-safe; V4 decode is not

**V5 is per-SLOT, not per-shader** — `genLen × N_P`, not `GEN.length × N_P` — so
growing a table moves no bytes. Shader indices are stored as whole bytes
(`buf[2]`, `buf[3-5]` as `+1`), so the table has room to 254. Verified by
extracting the real `_stateToBufV5` and driving it under both table sizes:
5,000/5,000 payloads byte-identical between `GEN.length` 14 and 15, and
`genChain` round-trips 5,000/5,000 with the new index in play. `applyState`
filters `i < GEN.length`, which only ever widens.

**`_bufToStateV4` sizes its param blocks from the current `GEN.length`.** It is
decode-only, but that means every shader ever appended has shifted its read
offsets — a V4 buffer written when `GEN` held 14 entries now misaligns by
`N_P` bytes and everything after the gen block decodes as garbage. Pre-existing
and cumulative, not caused by any one addition.

### V5 trailing blocks

Length-guarded, appended after the original body, fixed order, append only:
1. `fxSlotMode` (fxLen bytes) 2. LFO `skew` (3) 3. LFO `curve` (3).
Reserved-bit additions: byte 12 bypass masks, byte 20 bits 3-4
`blendSrc`/`vidMirrorH`. `blendAmount` (byte 11) is written and never read.

---

## Coordinate anchoring

`vU = aP*.5+.5`, so UV origin is bottom-left. A lattice built as
`floor(vU * N)` is pinned to that corner, and animating size slides the pattern
diagonally instead of scaling in place.

All GEN shaders were already centred. Four FX lattices were re-anchored
2026-08-09, measured by mean pattern motion per LFO step in the middle 18%:

| shader | improvement |
|---|---|
| ascii | 4.4× |
| feature-dots | 3.6× |
| bitcrush | 3.3× |
| halftone (size) | 2.1× |
| halftone (angle) | 1.3× |

`rot2()` rotates about the origin, so halftone's angle params were swinging the
whole screen around the corner.

Left corner-anchored — do not change:

- **gamma-ray** — measured exactly 1.0×; the cell value is re-randomised every
  frame.
- **grain** — per-pixel noise, no spatial scale param.
- **oscilloscope** — already centred; its only corner lattice is 64 static bars.
- **scatter** — its cell count tiles the frame exactly, and re-anchoring would
  break the permutation that keeps `swap` an involution.

Transform: replace `vU` with `(vU - 0.5)` in the lattice, add the centre back
when converting to a texture coordinate.

---

## Randomise

`randomiseAll` — 4 distinct GEN, 4 distinct FX, all params, blend modes and
amounts, fresh seeds, three LFO oscillators re-rolled, 2-4 LFO assignments.
Switches to SHADER.

- **Blend modes come from a weighted bag (`RND_MODES`).** `multiply`, `darken`
  and `burn` remove light; a uniform pick blacks out a four-deep chain most of
  the time. Measured over 4000 rolls: normal 44%, screen 16%, add 16%,
  lighten/overlay/difference ~8% each, light-removing 0%.
- **Slot 0 is forced to `normal` at 1.0** — it composites onto black.
- **Snapped params must land exactly on a position:** `k/(len-1)`.
- `lfoUseBpm`, audio and MIDI bindings are not touched.

---

## Validation

- **JS:** extract `<script>`, `node --check`.
- **HTML: run on every markup change.** Element balance for `div`, `span`,
  `button` at minimum, plus `html.parser`. Add a CSS brace count for stylesheet
  changes.
- **GLSL:** `npm i @shaderfrog/glsl-parser`; AST walk for undeclared identifiers,
  skipping `field_selection.selection`. Rebuild the `${ASCII_*}` constants —
  stub them as **int literals**, because the ascii source interpolates them as
  `${X}.0` and a float stub fails to parse.
  Expect **36 entries, 35 parsed**. Re-derive that count from the file — it was
  stale at 33/32 for two sessions. The durable invariant is: exactly one entry
  without GLSL, and it is `sample-hold`.
- **The `src:` prefix is not always `H`.** `PAL`, `HUESHIFT` and `HSV` are
  prepended too, and a matcher anchored on `` src: H+` `` silently skips a third
  of the table while still reporting "no failures". Match
  `src:\s*((?:[A-Z][A-Z0-9_]*\s*\+\s*)+)` and resolve each name.
- **Scan the template literal, do not regex it.** Greedy `` [\s\S]*` `` runs the
  last entry's source to the end of the file and swallows the ASCII block's
  backticks. Walk from the opening backtick honouring `${...}` nesting.
- **Read the count your tooling reports.** Two shaders once passed several "ALL
  PASS" runs unchecked.
- **Extract the real function and drive it.** Seven genuine bugs across two
  sessions. Mock the DOM/GL surface; use `var`.
- **Differential testing** against a deliberately-patched old version — see
  *Chain slots*.
- **Numpy porting** for shader work; gives exact preset-safety proof
  (`max|old−new| == 0`).
- **Match the metric to the complaint.** Mean pixel difference saturates as soon
  as a glyph changes and reported no improvement for a real ascii fix;
  glyph-index distance was correct. Single-point drift showed 1.5× for centring
  where mean pattern motion showed 3-4×.
- **Model the mechanism.** A seek-pacing simulation treating "seek complete" and
  "frame presented" as one instant proved nothing; splitting them reproduced the
  bug.
- **Layout is checkable without a browser.** Courier New advance is 0.6em; sum
  row widths against 432px minus ~20px body padding and ~12px scrollbar gutter.
- `/tmp` is wiped mid-session; rebuild rather than trusting a stale pass.

### Regression tests worth recreating

`regress.js` (LFO skew/curve neutrality, V5 trailing blocks), `rate_test.js`,
`play_test.js`, `scrub_test.js`, `step_test.js`, `collapse_test.js`,
`params_fold_test.js`, `stale_test.js`, transport state machine, `setMode`
cycling, `_padSlots` short/long/idempotency, upload-gate readyState matrix,
chain differential harness, wire round-trip (shape, fixed point, V5 bytes,
new-keys-additive), `capDims`, gzip bomb rejection, overlay logic, bank logic,
source binding, legacy/V5 compatibility, **shader-table append safety** (V5
bytes identical across `GEN.length`, driven on the extracted encoder), and
**time-phase continuity** (worst single-frame step of every time-derived
quantity against that trial's own 99th-percentile step; smooth is ~1.0).

---

## Traps

- **Never regex-delete across `</div>` boundaries.** `.*?</div>\s*</div>`
  over-matched twice; once swallowed the in/out block (5 ids lost). Locate the
  region and rebuild it wholesale.
- **Never resolve a shader field by scanning forward from `name:`.** Bit four
  times. Anchoring on `name: '(\w+)',\n\s*params:` is wrong — `feature-dots` has
  `usesPrev: true` between them. Use the entry line itself as the boundary:
  `^    name: '([\w-]+)',$` at four-space indent, each match running to the next.
- **A brace-matching extractor must skip the parameter list.** `opts = {}` closes
  on itself and returns an empty body. Match the parens first.
- **Never put a backtick in a GLSL comment** — shader sources are template
  literals.
- **`vid.currentTime` returns the seek target on assignment.**
- **`readyState` drops to 1 during a seek.** Never gate a texture upload on it in
  code that seeks continuously.
- **Wrap every time phase at the period of what reads it.** `th = TAU*fract(...)`
  is seamless for `cos(th)`/`sin(th)`, but *halving* it is not: half of a
  TAU-periodic phase is π-periodic, so `rot2(th*.5)` flipped the quaternion 180°
  in one frame at every wrap. Period 16 s, first cut at a seed-dependent point,
  so it reads as "a jump after about 8 seconds". A sub-rate reader needs its own
  phase at its own rate: `fract(t*rate/(2*PERIOD) + off*.5)`, identical angular
  velocity, wrap lands on a full turn. Diagnosed by measuring the worst
  single-frame step of the whole time state against that trial's own 99th
  percentile step — 1841× before, 1.004× after, over 400 random trials.
- **mediump is fp16 on mobile.** `float(uFrame)` stops resolving single frames
  past ~2048 (~34 s). Time-deriving shaders declare `highp float t`, and function
  params receiving it must be `highp` too.
- **GL uniforms are per-PROGRAM and persist.**
- **`const` in `eval()` does not leak.** Test harnesses need `var`.
- **TDZ:** a `let` declared after code that reads it throws at load.
- **Unicode glyphs tofu on mobile.** `∿ △ ⊿ ⊓` and `▾ ▸` (U+25BE/25B8) have no
  glyph in Courier New on Android. Use inline SVG and CSS border triangles.
- **A range input will not shrink below its UA intrinsic width** (~129px in
  Chrome) whatever `flex` says, because `min-width` defaults to `auto`.
- **`innerHTML` destroys child listeners.** The camera button left `#flow-strip`
  to become static markup; it works via `data-action` delegation on `#spi-panel`.

---

## Shaders with style selectors

Pattern: original behaviour exact at style 0 with new params at 0, snapping
`style` slider, extras neutral at 0.

- **plasma** (GEN 0, 9p) — classic/fractal/warped/ridged/marble/ripples/julia/
  flow. julia needs a zoom window: too tight is smooth interior, too wide moirés.
- **flowing-colours** (GEN 3, 11p) — `steady` matters: at 0 the frame pulses via
  `sin(t/10)` and washes out flat.
- **zoom-clouds** (GEN 9, 11p) — `contrast` pulls the ramp gain back (the base
  ramp blows ~47% of a frame to flat cream); `rough` renormalises; `relief` is
  normalised by `uRes` because raw `dFdx` scales with pixel size.
- **glitch** (FX 1, 9p) — run-length coherent macroblock damage from KinoGlitch /
  KinoDatamosh. The DCT ripple amplitude must use a random independent of the
  trigger.
- **scatter** (FX 20, 7p) — scatter/swap/rows/columns/spin. `swap` is a true
  involution.
- **feature-dots** (FX, 9p, `usesPrev`) — dense Shi-Tomasi cornerness over a
  jittered lattice. Centre-anchored. `usesPrev: true` sits between `name:` and
  `params:`.
- **ascii** (FX 15, 6p) — 5 charset blocks density-sorted separately, 2D atlas
  640×112 (`ASCII_COLS = 64`), `highp` atlas UV. `variation` jitters luma before
  quantising; measured 1 distinct glyph at 0, 10 at 0.2, 22 at the 0.45 default.
- **maze-flight** — port of "Can't Find My Way Out" by ksin (CC0-1.0). Global
  `precision highp` — the camera's z is the clock. Integer hashing, world
  rebasing via `gK`.
- **quaternion** (GEN 13, 12p) — port of "Gilded Quaternion" by ufffd (CC0-1.0).
  Global `precision highp` — the running derivative reaches ~1e10, past fp16.
  `zoom` is **inverted against camera distance**, `cd = mix(5.5, .35, uP4)`, so
  up is closer; def `.592` reproduces the original framing at 2.45. Above zoom
  0.718 the eye is inside the 1.8 bounding sphere. At the top of the travel the
  frame is 78-100% object for four of the five form seeds; **form ≈ 0.5 puts the
  eye inside the solid** and washes out, which is the one dead spot on that
  slider. Marching is *cleaner* close in than at the default framing — near-miss
  rescues 0.2% against 1.5%.
- **mandelbox** (GEN 14, 12p) — Tom Lowe's Mandelbox on the quaternion's camera
  rig, orbit trap colour and shading block; only the map differs. Folded
  architecture rather than a creature. Global `precision highp` — `dr` compounds
  as `dr*|scale|+1` and reaches ~4e8 at 16 iterations. No transcendentals in the
  loop, so it costs about what the quaternion does at three times the iteration
  count. See *Mandelbox ranges* below.

### Mandelbox ranges

All measured on a CPU port; none of it is GPU-verified.

- **Surface radius is `3.42 × fold` and does not depend on `scale`** (300k
  samples per setting). The camera is normalised against it, which is what turns
  `fold` from a size knob into a pure shape knob: apparent size holds 19.9-22.7%
  of frame across the whole fold slider, `scale` holds 19.0-24.6%. Nothing needs
  re-framing after a fold move.
- **Only negative scales are compact.** At `scale` +1.2 to +3.0 the object
  sprawls to radius 6.8-12+; every negative scale measured sits at ~3.3 (at
  fold 1). The slider is negative-only, -1.5 to -3.3.
- **`scale` is warped**, `1.57u − 0.57u²`. Linear, the silhouette moves 3.7×
  faster near -3.2 than near -1.5 and half the slider does nothing. Warped, the
  spread is 3.0×. Measured as silhouette IoU at eight equal slider steps:
  0.955 → 0.866 warped, against 0.978 → 0.756 linear.
- **`minRadius` is not worth a param.** Swept 0.05-0.85 it holds IoU 0.92-0.97 —
  a weaker knob than anything else here. Fixed at 0.25 (`m = 4` in the core).
- **The step budget rises with zoom**, `80 + detail·71 + zoom²·150`. The interior
  needs about twice the marching the silhouette does; at a flat 130 steps the
  close end lost 4.2% of the frame to starved rays and 7.6% to near-miss
  rescues. Lowering the march factor below 0.85 makes it *worse*, not better —
  it is step starvation, not a DE bound problem. With the ramp, near-miss stays
  ≤0.16% and starved ≤0.05% everywhere on the zoom range.
- Default (`zoom .392`, `scale .5`, `fold .21` → cd 2R, scale -2.59, fold 0.85)
  frames at 22.1% with **zero** near-miss and **zero** starved, 19.9 steps/px.
- Palette gains are the reciprocals of each trap channel's measured 5th-95th
  spread — 12.08 / 1.07 / 3.18 / 3.18, so gains .08 / .94 / .31 / .31.
- `morph` drifts `scale` ±0.30 and `fold` ±0.07 on a circle. Both clamp inside
  the measured range so it cannot reach an unchecked corner. The worst corner it
  *can* reach (scale -3.3, fold 0.62) still fills 6.1% of frame.

## LFO

Three oscillators `{shape, amp, offset, period, skew, curve}`. `skew` warps phase
before the shape is evaluated; `curve` bends the trajectory. Both identity at 0.5.
Icons are inline SVG. Preview canvas fixed 76×18.

## SPI panel

`updateSPI` throttled ~33 ms, dirty-checked via `_spiPrev`, refs cached in
`_spiRefs` — built before the shader grids exist, so `genBtns`/`fxBtns` are
re-captured afterwards. Click delegation on `#spi-panel`.

`_spiRefs.reverse` resolves to the source icon, not the legacy `.sk` keypad
button, so the `sk-active` it writes is inert. The legacy keypad reverse lamp is
dead.

Nine foldable headers, state in `localStorage` under `recur.spiCollapsed`.
VIDEO/LFO/AUDIO INPUT/MIDI/MISC closed by default. Content is found by walking
siblings to the next header; PARAMS and ZOOM & CHAINS are special-cased.

### ZOOM & CHAINS

1. `#spi-zoom-row` — RND · ZOOM · 1:1 · RES.
2. `#spi-src-cap-row` — SRC · CAM · RENDER CAP · resolution readout, right-aligned
   via `margin-left:auto`.
3. `#flow-strip` — GEN, FX, BANK.

Budgets at Courier New 9px against ~400px usable: zoom row 321px, SRC row 317px.
Both carry `flex-wrap` as a backstop.

`buildFlowStrip` replaces the strip's `innerHTML`, so the cam button is static
markup updated in place **before** the key check. `camOn` is not in the strip key.

### Source section

`#src-main-row` (transport / camera / xform groups, blend toggle right),
`#src-speed-row` (log-mapped 0.25-4×, 1× centre, soft detents),
`#src-inout-row` (`.inout-track` sets `max-width:220px` further down the
stylesheet and must be overridden per context), `#src-blend-row` (SHADER only).

Dim what is unavailable on this hardware; hide whole groups belonging to another
mode. 2px within a group, 12px between groups.

`inOutScrubbing` makes all three stepping branches and the `timeupdate` watcher
yield, or the seek is overwritten.

---

## Open threads

- mediump folding incomplete: the glitch slice stage still feeds raw `t`.
- `zoom-clouds` `rn()` is not magnitude-safe (`sin(dot)*43758` → ~2e6). Changing
  the hash changes every existing preset's clouds. Fix if black or speckled
  output appears on the phone.
- Share URLs break on recipients when a live camera is in the stack.
- Share URLs from before the 2026-08-09 padding fix open with any since-appended
  param half-on. Unrepairable.
- `feature-dots` `settle` default 0.16 is a guess, needs GPU tuning.
- `loadVideo` / `captureScreen` enter SAMPLER without the gen bypass.
- Unverified on a GPU: `onResize` halve-and-retry, `reapFBOs` timing, the render
  cap, `texSubImage2D` upload, the quaternion tumble phase and reversed zoom,
  and all of `mandelbox`.
- **The quaternion `slice` stop may not hold.** The claim above is that ±0.75
  cannot reach a blank frame because the set is empty past |w| ~0.85. On the CPU
  port at `form` 0.5, both ends of the slider render **0% coverage** at every
  camera distance tried. Either the original margin was measured at one form
  only, or the port is wrong. Re-measure across form before trusting the stop.
- Quaternion `form` ≈ 0.5 at maximum zoom puts the eye inside the solid, so the
  frame washes to near-flat colour. The only dead spot on that slider.
- Reversing quaternion `zoom` reframed every preset and share link written
  before `2026-08-11o`. The bytes are still valid; the framing is not. Share
  links from other people cannot be corrected.
- `randomiseAll` rolls `zoom` uniformly, so ~28% of random quaternion and
  mandelbox patches now start from inside the object. Not weighted.
- `_bufToStateV4` read offsets drift with `GEN.length` — see above.
- `fboFx` is not reaped — its condition is known to `render()`, not `reapFBOs()`.
- Legacy `_bufToState` (pre-V4) is not length-checked the way V5 is.
- `captureScreen` does not run `assessSource`.
- Nothing warns before a 4K export take.
- The perf overlay measures wall-clock frame time and cannot separate GPU from
  main thread. `disjoint_timer_query` would.

## Pi 5

WebGL2 is hardware-accelerated. The bottleneck is video: BCM2712 dropped the
H.264 decode block and Chromium cannot use the HEVC one, so decode is software;
there is no hardware encoder, so MediaRecorder export is CPU-bound. Measured pass
counts: light 5, typical 9, heavy 19. A typical chain at 1× 1080p60 wants
1.12 Gpx/s. Plan on ½ render scale at 30 fps with RENDER CAP at 1080.
`pi-check.html` reports `getShaderPrecisionFormat` for mediump (23 bits = fp32,
10 = fp16), limits, codecs and a fill-rate benchmark.

## Files

`index.html` · `recur-web-project.md` · `README.md` · `seek-check.html` ·
`pi-check.html` · reference renders `plasma_styles`, `flowing_styles`,
`clouds_styles`, `scatter_modes`, `glitch_before_after`, `ascii_diagnosis`,
`ascii_charset`, `src_grouped_final`.

## When picking up work

1. Copy `index.html` from the mounted knowledge folder.
2. Preserve the single-file structure, the slot model, and 0-is-neutral.
3. Call `chainInsert`/`chainRemove`/`chainMove` and
   `togglePlay`/`toggleReverse`/`setMode` rather than duplicating them.
4. Re-derive the shader-table count from the file. This document cites function
   names, never line numbers — line numbers drifted three times in one session.
5. Validate: `node --check`, HTML nesting on every markup change, GLSL parse and
   AST audit, numpy renders for anything visual, extract-and-drive for anything
   stateful, differential harness for a refactor.
6. Update the in-app help overlay for user-facing changes.
7. Bump `BUILD`. State that it needs a live GPU check. Surface the file with
   `present_files`.
