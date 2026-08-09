# recur-web — Project Context

Context document for the **recur-recur-browser** project (aka **recur_web**).
Give this to Claude at the start of a session.

*Reconciled against `index.html` on 2026-08-09 — 8929 lines / ~422 KB /
sha256 `f253cbdf5a3c…`. `BUILD` constant currently reads
`2026-08-09d · randomise` and **needs bumping before the next deploy**.*

> **No open bug mid-investigation.** The reverse/ping-pong static-frame bug that
> dominated the previous three sessions is **fixed and confirmed on device**. See
> *Solved: reverse showed a static frame* for the cause, because the shape of it
> generalises to anything that touches video-to-texture.

---

## What it is

- Browser-based live video effect and generation tool — the WebGL2 sibling of
  the Raspberry Pi **recur** project (an mpv-based video sampler on Pi 5).
- Repo: `https://github.com/thedirgemedia/recur-recur-browser` (public, GPL-3.0).
- Deployed at `dirgemedia.com/recur…`; Dirge pushes and deploys manually.
- Owner: **Dirge** — electrical/building-services trade background, NZ, very
  comfortable with low-level graphics/DSP work. Tests on desktop **and** an
  Android phone; reports issues from the phone often. **The phone is 120Hz** —
  measured, and worth remembering for anything frame-rate dependent.

## Non-negotiable conventions

- **Single self-contained `index.html`.** No build step, no modules. All CSS, JS
  and GLSL inline. Keep it that way.
- **Replace the file directly; no git patches.** Dirge commits and pushes.
- **Nothing is ever GPU-verified here.** Always say so when shipping shader work.
- **"0 is neutral" for every appended param.** Prove it numerically. See
  *Preset and share-URL safety* — the rule is real but the share format used to
  violate it, and the details matter.
- **Bump `BUILD` on every handover.** It shows dim in the panel header and logs
  to console. Added because Dirge spent a round reporting a bug against a stale
  deploy — if the string on screen doesn't match, they're not running your file.
- **One entry point per user action.** `togglePlay`, `toggleReverse`, `setMode`
  each exist because the same logic had been copy-pasted across a keyboard
  handler, a panel button and a legacy button, and the copies had drifted. Every
  drift was a real bug. Do not add a fifth copy — call the function.

---

## Solved: reverse showed a static frame

Kept because the diagnosis took four sessions and three of them were wasted.

**Symptom.** In reverse (and ping-pong) the picture froze, while the in/out
playhead swept backwards normally. Pausing snapped the picture to the correct
frame.

**Cause.** `uploadVideoFrame()` bailed on `readyState < 2`. A seek in flight
drops `readyState` to `HAVE_METADATA` (1), and the reverse stepper issues a seek
every frame, so essentially every upload was skipped and the texture held the
last frame from before reverse started.

**What made it hard.** Three separate things each looked like the cause and
weren't:

- `vid.currentTime` **returns the seek target the moment you assign it**, so a
  sweeping playhead proves only that the assignment ran — not that the seek
  retired, and not that a frame was decoded. Every "the playhead is moving so
  seeking works" inference was unfounded.
- Backward seeking measured healthy in isolation (`seek-check.html`: 20/20 seeks,
  6ms mean, 17/20 frames changed). True, and irrelevant — the failure was
  downstream of seeking entirely.
- Three rounds of static analysis each found a *real* bug in the stepper and
  fixed none of the symptom.

**What actually settled it** was one second of on-device counters: `up0 bail120`
against `vfc116` with `rs1`. 116 frames presented per second, none uploaded.
Instrument before theorising when the failure is in a pipeline.

**The fix** is in two parts, both still in the file:

1. `uploadVideoFrame()` accepts `readyState >= 2` **or** "a frame has been
   presented and `videoWidth > 0`". Whether the element has a frame to read is
   the right question; `readyState` is not.
2. The upload also runs inside `requestVideoFrameCallback`, at the instant a
   frame is presented — the one moment it is guaranteed valid. `render()` skips
   the repeat via `_vidTexFresh`.

---

## Architecture

- **WebGL2**, `#version 300 es`, `precision mediump float`.
- **Modes:** `MODES = ['SAMPLER','SHADER','LIVE']`.
- **`GEN[]` 12, `FX[]` 21 — 33 entries, of which 32 carry GLSL.** Entry:
  `{name, params, def, [src], [usesPrev], [snapVals]}`. **Append new shaders at
  the end.** `sample-hold` is the only entry with **no `src:` at all** — it is a
  JS-side effect, and any tool walking the tables must tolerate that rather than
  assuming every entry is a shader.
- **Field formatting is not uniform.** `levels` is written `src: H + \`` with
  spaces around the `+`; every other entry uses `src: H+\``. `feature-dots` puts
  `usesPrev: true` between `name:` and `params:`. Match tolerantly or you will
  skip entries and never know.
- **Chains** max 4 each, duplicates allowed. Parallel arrays: `*SlotParams`,
  `*SlotAudioMod`, `*SlotLfoMod`, `*SlotMidiMod`, `genChainMode`/`genChainAmt`,
  `fxSlotAmt`/`fxSlotMode`, `genChainSeeds`. **Any mutation must update all of
  them plus the `*Disabled` position sets.**
- **`genIdx`/`fxIdx` are chain POSITIONS**, not shader indices.
- **Blend:** GEN and FX slots both carry a mode. FX gate is
  `slotAmt < 0.999 || slotMode !== 0`.
- **`#hud` is `display:none` and nothing ever turns it on.** `updateHUD()` still
  runs full DOM work on those invisible elements every frame. Not fixed — noted
  so nobody debugs a HUD that cannot appear, and as an easy perf win.

### Modes and the gen bypass — `setMode()` (~l.4906)

Every interactive mode change goes through `setMode(m)`: the ENTER key, the MODE
button, the three set-mode buttons, and `toggleCamera()`.

- **SAMPLER and LIVE bypass the whole gen chain on arrival.** Both modes are
  about the source; landing in one with the shader stack painted over the top
  hides the thing you just switched to look at. The bypass is positional and the
  chain is left intact, so the blend button restores it in one tap.
- **SHADER clears an all-off bypass on arrival**, because SAMPLER and LIVE both
  leave exactly that state behind and the ENTER cycle runs SAMPLER → SHADER.
  Without this you land in the mode whose entire purpose is the gen stack and see
  the bare source, or black with no source. **A partial per-slot bypass set from
  the flow strip is deliberately preserved** — only the all-off state is cleared.
- **`applyState()` deliberately does NOT use `setMode()`.** A preset carries its
  own `genDisabled` and must be restored exactly as saved, not overwritten by
  this policy.
- Still bypassing `setMode()` and arguably inconsistent: **`loadVideo()` and
  `captureScreen()` both set SAMPLER directly** without the bypass. Left alone on
  purpose — dropping a video is not the same gesture as switching mode, and
  silently wiping a look in progress seemed worse. Dirge knows; it is a one-line
  change each if wanted.

## Sampler playback

- `loop()` (~l.4272) has three branches: ping-pong, reverse, forward-native.
- `stepPlayhead()` (~l.3679) advances a **virtual playhead `_stepT`** at the true
  rate and pushes it to the element. Reading `vid.currentTime` and subtracting a
  frame does not work: during an in-flight seek `currentTime` has not moved, so
  every frame re-targets from a stale position and the browser discards the
  intermediates.
- **Do not add a divergence-resync guard.** One was tried
  (`if (|_stepT − currentTime| > 0.75) resync`) and deadlocked: on slow seeks the
  virtual head legitimately runs ahead, got snapped back every frame, and the
  picture never moved. Anything that genuinely relocates the clip sets
  `_stepT = null` instead.
- **Seek pacing.** `_seekPending` allows one seek in flight at a time, cleared
  when a frame is actually *presented* (rVFC), not on `seeked` — `seeked` fires
  before the picture is composited, so clearing on it lets the seek storm back
  in. `SEEK_WATCHDOG_MS = 150` reopens the gate if a frame never arrives, and
  **deliberately overrides `vid.seeking` too**: gating the watchdog behind
  `!vid.seeking` made it unable to fire in the one case it exists for, since a
  seek that never retires leaves `seeking` stuck true forever.
  - Honest status: pacing was **not** the cause of the static-frame bug — `vfc116`
    showed frames presenting fine without it. Kept because measured `w116/vfc116`
    is one seek per presented frame with no waste, and software-only decode on
    the Pi 5 is exactly where queueing unretirable seeks would hurt.

### Transport — `togglePlay()` (~l.3716), `toggleReverse()` (~l.3742)

Both directions are real transport buttons and are mutually exclusive:

- **play** — run forward, or pause if already running forward.
- **reverse** — run backward, or pause if already running backward. The play
  button is how you return to forward.
- Each snaps the playhead off the boundary it is heading towards before starting,
  or it runs for `PLAY_EPS`, hits the end and stops again, which reads as dead.
- The icon shows the **action**, not the state, and the caption follows the icon.
  Reverse originally had one glyph and only changed colour, which read as the
  button not working at all.
- Bug fixed alongside: play was a blind `paused = !paused`, so pressing play
  *while reversing* set `paused = true` and stopped the clip instead of handing
  back to forward.

---

## Preset and share-URL safety

**Read this before appending a param to any shader.** The "0 is neutral" rule is
correct, but there are two separate padding paths and they used to disagree.

- **GL uniforms** pad with 0. `setUniforms` writes every declared param uniform
  every time, because GL uniforms are per-PROGRAM state and the program is shared
  by every chain slot — a slot that did not write uP5–uP9 inherited them from
  whatever ran last.
- **The V5 share format** always writes `N_P = 12` param bytes per slot,
  regardless of how many the shader declares. It **used to pad with 0.5**, which
  silently broke the rule: a URL made when plasma had four params fed 0.5 into
  `style`, which snaps to index 4 and reopened as *marble* instead of *classic*,
  with detail/contrast/fold/drift all half-on. Now fixed to pad with 0. Byte
  layout is unchanged, so older builds still read new URLs.
- **Share URLs made before that fix cannot be repaired.** A 0.5 written as
  padding is indistinguishable from a 0.5 chosen deliberately. Do not try to
  detect it heuristically.
- **`_padSlots()` (~l.7617) normalises every slot array on load** to the shader's
  currently-declared param count. It cuts both ways: JSON presets arrive **short**
  (a preset saved before a param was appended), V5 URLs arrive **long** with 12
  entries and trailing junk. Missing entries fill with **0, not `def`** — `def`
  is the fresh-instance value and is not neutral (plasma's `detail` defaults to
  0.45); neutral is what makes an old preset render as it did when saved.
  - The short case was a **hard crash**, not just wrong values: the panel reads
    `shader.params.length` entries, so `pvals[i].toFixed(2)` hit `undefined` and
    threw inside `updateSPI`/`updateHUD`, which run every frame. Loading such a
    preset killed the whole control panel.

### V5 share format — trailing blocks
Length-guarded, appended after the original body, **fixed write order, append
only, never reorder**: 1. `fxSlotMode` (fxLen bytes) 2. LFO `skew` (3) 3. LFO
`curve` (3). Reserved-bit additions: byte 12 bypass masks, byte 20 bits 3–4
`blendSrc`/`vidMirrorH`. `blendAmount` (byte 11) is **dead** — written, never
read. Dirge chose to leave it.

---

## Coordinate anchoring — centre, not corner

`vU = aP*.5+.5`, so UV origin is **bottom-left**. Any lattice built as
`floor(vU * N)` is pinned to that corner, and animating the size param slides the
whole pattern diagonally out of it instead of scaling in place.

All ten generative GEN shaders were already centred (`(vU-.5)*asp*2.`). Four FX
lattices were re-anchored to the frame centre (2026-08-09), measured by mean
pattern motion per LFO step in the middle 18% of the frame:

| shader | improvement |
|---|---|
| ascii | 4.4× less motion |
| feature-dots | 3.6× |
| bitcrush | 3.3× |
| halftone (size) | 2.1× |
| halftone (angle) | 1.3× |

**halftone was the big one**: `rot2()` rotates about the origin, so the *angle*
params were swinging the entire screen around the bottom-left corner, not just
the dot lattice.

**Deliberately left corner-anchored — do not "fix" these:**

- **gamma-ray** — measured at exactly 1.0×, no gain. Its cell value is
  re-randomised every frame, so where the lattice sits is invisible.
- **grain** — per-pixel white noise with no spatial scale param. Re-anchoring
  changes which random value each pixel gets and nothing else.
- **oscilloscope** — already centred; waveform drawn about `uv.y = .5`, radial
  style about `(uv-.5)`. Its only corner lattice is a hardcoded 64 bars nothing
  animates.
- **scatter** — its cell count is an integer that tiles the frame exactly, so it
  re-tiles rather than drifting, and re-anchoring would break the permutation
  that keeps `swap` a true involution.

The general transform: replace `vU` with `(vU - 0.5)` in the lattice, add the
centre back when converting to a texture coordinate. This changes existing
presets by under half a cell statically — the dramatic difference is only under
animation, which is why it was done outright rather than behind a new param.

Note the shader is **feature-dots**, not `sample-hold`. They are adjacent in
`FX[]` and a bad extractor conflated them; see the `name:` trap below.

---

## Randomise (RND)

Far left of the zoom row, `randomiseAll()` — 4 distinct GEN, 4 distinct FX, all
params, blend modes and amounts, fresh per-slot seeds, all three LFO oscillators
re-rolled, and 2–4 LFO assignments onto random params. Switches to SHADER,
because the GEN chain is bypassed in SAMPLER and LIVE and the result would be
invisible from either.

Three things that are load-bearing, not incidental:

- **Blend modes come from a weighted bag (`RND_MODES`), never uniform.**
  `multiply`, `darken` and `burn` each remove light, so a uniform pick over ten
  modes puts one into a four-deep chain most of the time and three in a row leave
  a black frame. Measured over 4000 rolls: normal 44%, screen 16%, add 16%,
  lighten/overlay/difference ~8% each, light-removing modes 0%.
- **Slot 0 is forced to `normal` at amount 1.0.** It composites onto black, where
  any subtractive mode is an immediate blackout regardless of what follows.
- **Snapped params must land exactly on a position** — the value that round-trips
  to index k is `k/(len-1)`. Anything else renders between two selector
  positions, so plasma's `style` would come out as a blend of two styles.

`lfoUseBpm` is deliberately not touched — running to a clock is the user's
choice. Both `period` and `lfoBeatIdx` are set so it lands correctly either way.
Audio and MIDI bindings are left alone: random audio bindings do nothing in
silence, and random MIDI would fight a controller mapping.

---

## Validation tooling

- **JS:** extract `<script>`, `node --check`.
- **HTML nesting: run this on EVERY markup change.** A `<div>` balance count plus
  `html.parser`. Skipping it let an unclosed `#spi-source-section` ship, which
  would have nested the whole flow strip inside a `display:none` block.
- **GLSL:** `npm i @shaderfrog/glsl-parser`; AST walk for undeclared identifiers,
  skipping `field_selection.selection` or every swizzle false-positives. Rebuild
  the `${...}` template constants (`ASCII_*`) or the ascii shader will not parse.
  **Expect 33 entries and 32 parsed** (only `sample-hold` has no GLSL). 32
  entries means the extractor swallowed `feature-dots`; 31 parsed means it
  dropped `levels` over the spaces in `src: H + \``.
- **Check the count your tooling reports, every time.** Both of the above were
  found by the count being off by one, not by anything failing — a silently
  skipped entry looks exactly like a passing run. Two shaders went unchecked
  through several "ALL PASS" runs this session because nobody read the total.
- **Extract the real function and drive it.** Pulling `togglePlay`/`toggleReverse`
  /`setMode`/`_padSlots` out of the file verbatim and running state machines over
  them caught four genuine bugs this session, three of them in code written the
  same hour. Mock the DOM/GL surface; `var`, because `const` in `eval()` does not
  leak.
- **Numpy porting is the highest-value check for shader work** — renders catch
  visual bugs a compiler can't, and give exact preset-safety proof
  (`max|old−new| == 0`).
- **Pick a metric that matches the complaint.** Mean pixel difference said "no
  improvement" for an ascii fix that was plainly better, because it saturates as
  soon as a glyph changes; glyph-index distance was the right measure. Same trap
  hit again with centring: measuring drift at a single point showed a misleading
  1.5×, while mean pattern motion in the frame centre — which is what the eye
  actually tracks — showed 3–4×.
- **Model the mechanism, not the outcome.** A seek-pacing simulation that treated
  "seek complete" and "frame presented" as the same instant showed old and new as
  identical and proved nothing. Splitting them into two phases reproduced the bug
  exactly.
- `/tmp` gets wiped mid-session — test files and `node_modules` disappear without
  warning. Rebuild rather than trusting a stale "ALL PASS". This bit again on
  2026-08-09.

### Regression tests worth recreating
`regress.js` (LFO skew/curve neutrality + V5 trailing blocks), `rate_test.js`
(speed log-map + detents), `play_test.js`, `scrub_test.js`, `step_test.js`,
`collapse_test.js`, `params_fold_test.js`, `stale_test.js`, plus this session's:
transport state machine (exclusivity over 3000 random presses), `setMode` mode
cycling, `_padSlots` short/long/idempotency, upload-gate readyState matrix.

## Traps

- **Never regex-delete across `</div>` boundaries.** Bit twice: `.*?</div>\s*</div>`
  over-matched and once swallowed most of the in/out block (5 ids vanished).
  **Locate the region and rebuild it wholesale** from extracted parts instead.
- **Never resolve a shader field by scanning forward from `name:`.** Bit four
  times — `{ name: 'ascii', raw: … }` in the charset table, and
  `name:…snapVals:` spanning into a *later* shader.
  - **The old advice in this doc — anchor on `name: '(\w+)',\n\s*params:` — is
    WRONG and caused the fourth.** `feature-dots` has `usesPrev: true` between
    its `name:` and its `params:`, so that pattern skips the entry entirely and
    silently folds its source into the *previous* one. That made a 33-entry table
    look like 32, attributed feature-dots' shader body to `sample-hold` (which
    has no shader body at all), and meant feature-dots was never GLSL-checked.
  - **Use the entry-level line itself as the boundary:** `^    name: '([\w-]+)',$`
    at four-space indent, with each match running to the next. Nothing may be
    assumed about field order inside an entry.
- **Never put a backtick in a GLSL comment** — shader sources are template
  literals.
- **`vid.currentTime` returns the seek target immediately on assignment.** It is
  not evidence that a seek completed or that a frame was decoded.
- **`readyState` drops to 1 for the duration of a seek.** Never gate a texture
  upload on `readyState >= 2` in code that seeks continuously.
- **mediump is fp16 on mobile.** `float(uFrame)` stops resolving single frames
  past ~2048 (~34s), and adding a large `t` to a small spatial term quantises the
  spatial term away — that was the plasma "blocky over time" bug. All ten
  time-deriving shaders now declare `highp float t`, and function params that
  receive it must be `highp` too.
- **GL uniforms are per-PROGRAM and persist.** See *Preset and share-URL safety*.
- **`const` in `eval()` doesn't leak** to the enclosing scope — test harnesses
  need `var`.
- **TDZ:** `let` declared after an IIFE that uses it throws at load. Check
  declaration order when adding module-scope flags.
- **Unicode glyphs tofu on mobile.** `∿ △ ⊿ ⊓` and `▾ ▸` (U+25BE/25B8) have no
  glyph in Courier New on Android — the fold arrows silently didn't render. Draw
  icons as **inline SVG** and arrows as **CSS border triangles**.

---

## Shaders with style selectors

Same pattern: original behaviour exact at style 0 with new params at 0, snapping
`style` slider, extras neutral at 0. Preset safety verified numerically.

- **plasma** (GEN 0, 9p) — classic/fractal/warped/ridged/marble/ripples/julia/flow.
  julia needs a zoom window (too tight = smooth interior, too wide = moiré).
- **flowing-colours** (GEN 3, 11p) — **`steady`** matters: the original multiplies
  by `sin(t/10)` so it washes out to flat periodically.
- **zoom-clouds** (GEN 9, 11p) — `contrast` pulls the ramp *gain* back (the
  original blows ~47% of the frame to flat cream, so an S-curve had nothing to
  separate); `rough` renormalises; `relief` is normalised by `uRes` because raw
  `dFdx` scales with pixel size.
- **glitch** (FX 1, 9p) — run-length coherent macroblock damage from KinoGlitch /
  KinoDatamosh. The DCT ripple's amplitude must use a random **independent of the
  trigger**.
- **scatter** (FX 20, 7p) — grid shuffle: scatter/swap/rows/columns/spin. `swap`
  is a true involution so the frame stays a permutation.
- **feature-dots** (FX, 9p, `usesPrev`) — Shi-Tomasi cornerness evaluated densely
  over a jittered lattice. The entry carries `usesPrev: true` between `name:` and
  `params:`, which is what breaks naive extractors. Its lattice is centre-anchored.
- **ascii** (FX 15, 6p) — 5 charset blocks each density-sorted separately, 2D
  atlas 640×112 (`ASCII_COLS = 64`), `highp` atlas UV. `variation` jitters luma
  *before* quantising — the old `mix()` of two indices produced a fractional
  index that straddled two atlas cells in 80% of cells. Grouping is inherent:
  a flat region renders 1 distinct glyph at variation 0, 22 at the 0.45 default.

## LFO
3 oscillators `{shape, amp, offset, period, skew, curve}`. **skew** warps phase
(one warp covers every shape); **curve** bends the trajectory. Both identity at
0.5. Icons are inline SVG. Preview canvas fixed 76×18.

## SPI panel

`updateSPI()` throttled ~33ms, dirty-checks via `_spiPrev`, refs cached in
`_spiRefs` — **but `_spiRefs` is built before the shader-button grids exist**, so
`genBtns`/`fxBtns` are re-captured after the grids are built. Click delegation is
on `#spi-panel`.

`_spiRefs.reverse` resolves `querySelector('[data-action="reverse"]')`, which
matches the **source icon**, not the legacy `.sk` keypad button — so the
`sk-active` it writes lands on a `.src-ico` and is inert (that rule needs `.sk`).
Harmless, but it means the legacy keypad reverse lamp is dead.

**Nine foldable headers** (PARAMS, ZOOM & CHAINS, and 7 in `#spi-body`), state in
`localStorage` under `recur.spiCollapsed`. VIDEO/LFO/AUDIO INPUT/MIDI/MISC closed
by default. Content is found by walking siblings to the next header; **PARAMS and
ZOOM & CHAINS are special-cased** because a walk there would swallow the panel
body. Hiding uses its own `.spi-hidden` class so elements' own display logic
resumes.

### Source section (rebuilt repeatedly — current shape)
One button row + two slider rows + a blend row:
- `#src-main-row`: transport group (SAMPLER) · camera group (LIVE, or SHADER when
  `blendSrc==='live'`) · xform group (always) · blend toggle pinned right.
- `#src-speed-row` (SAMPLER): log-mapped 0.25–4× slider, 1× dead centre, soft
  detents within ~2% of the track, plus a 1× reset.
- `#src-inout-row` (SAMPLER): label above, full-width track. **`.inout-track` sets
  `max-width:220px` further down the stylesheet** — must be overridden per-context.
- `#src-blend-row` (SHADER only).

Rules learned: **dim** things unavailable on *this hardware* (front/rear on
desktop); **hide** whole groups belonging to another mode. Grouped items sit tight
(2px) with 12px between groups — spreading everything edge-to-edge is what read
as "scattered".

**In/out scrubbing** parks the playhead on the handle being dragged;
`inOutScrubbing` makes all three stepping branches and the `timeupdate` watcher
yield, or the seek is overwritten instantly.

---

## Open threads / known issues

- mediump folding incomplete: the glitch *slice* stage still feeds raw `t`.
- **zoom-clouds `rn()` is not magnitude-safe** (`sin(dot)*43758` → ~2e6, past
  fp16's ceiling). Left alone deliberately: changing the hash changes every
  existing preset's clouds. Fix if black/speckled output appears on the phone.
- Share URLs break on recipients when a live camera is in the stack.
- Share URLs created before the 2026-08-09 padding fix open with any
  since-appended param half-on. Unrepairable; see above.
- feature-dots `settle` default 0.16 is a guess, needs GPU tuning.
- `#hud` is dead markup that still costs DOM work every frame.
- `loadVideo()` / `captureScreen()` enter SAMPLER without the gen bypass.
- 13 shaders had no help entry; all 32 are now documented.

## Pi 5 assessment (done, see `pi-check.html`)

WebGL2 is hardware-accelerated. **The bottleneck is video, not the GPU:** BCM2712
dropped the H.264 decode block entirely and Chromium can't use the HEVC one, so
all decode is software; there is **no hardware encoder at all**, so MediaRecorder
export is CPU-bound. Measured pass counts: light 5, typical 9, heavy 19 — a
typical chain at 1× 1080p60 wants 1.12 Gpx/s. Plan on **½ render scale at 30fps**.
`pi-check.html` reports `getShaderPrecisionFormat` for mediump (23 bits = fp32
and the precision hazards stay latent; 10 = fp16 and they're real), plus limits,
codecs and a fill-rate benchmark.

## Files in the outputs folder

`index.html` (the app) · `recur-web-project.md` (this) · `seek-check.html`
(backward-seek diagnostic) · `pi-check.html` (Pi capability + fill rate) ·
reference renders: `plasma_styles`, `flowing_styles`, `clouds_styles`,
`scatter_modes`, `glitch_before_after`, `ascii_diagnosis`, `ascii_charset`,
`src_grouped_final`. Various `_tune_*` / `_glitch_*` / `_lfo_*` PNGs are scratch
and can be binned.

## When picking up work

1. Copy `index.html` from the mounted knowledge folder.
2. Preserve the single-file structure, parallel-array invariants, and "0 is
   neutral" for new params — checking *Preset and share-URL safety* first if the
   change appends a param.
3. Call the existing `togglePlay`/`toggleReverse`/`setMode` rather than adding
   another copy of their logic.
4. Validate: `node --check`, **HTML nesting on every markup change**, GLSL parse +
   AST audit, numpy renders for anything visual, and extract-and-drive for
   anything stateful.
5. Update the in-app help overlay for user-facing changes.
6. **Bump `BUILD`.** Say it needs a live GPU check. Surface the file with
   `present_files`.
