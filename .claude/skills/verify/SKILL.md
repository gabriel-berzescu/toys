---
name: verify
description: How to build/launch/drive this repo's toys for runtime verification. Static HTML5 canvas toys; drive them in the pre-installed Chromium via playwright-core.
---

# Verifying the toys

Static site, no build step. Every toy is a self-contained HTML file (`ink/ink.html`, `alchemy/alchemy.html`, `murmuration/murmuration.html`, `accretion/accretion.html`, `poise/poise.html`, `nightrider/nightrider.html`) linked from `index.html`. Murmuration, accretion, poise and nightrider each vendor Three.js next to themselves (`three.module.min.js` + `three.core.min.js` — the min build imports the core file; both must be present); poise additionally vendors the Rapier physics engine (`rapier.es.js`, wasm inlined as base64, Apache-2.0 text in `rapier-LICENSE.txt`).

## Launch

```bash
python3 -m http.server 8907 --bind 127.0.0.1 &          # serve repo root
npm i playwright-core                                    # in scratchpad, no browser download
```

Launch Chromium via playwright-core (there is no `gh`/`playwright` CLI here):

```js
chromium.launch({
  executablePath: '/opt/pw-browsers/chromium',
  headless: true,
  args: ['--use-gl=angle', '--use-angle=swiftshader', '--enable-unsafe-swiftshader',
         '--autoplay-policy=no-user-gesture-required']   // nightrider's AudioContext
})
```

WebGL (ink), Canvas2D (alchemy), and Three.js/WebGL2 (murmuration, accretion, poise — the latter also runs Rapier's wasm) all work under SwiftShader.

## Drive

- Collect `pageerror` + console errors on every page — the toys log nothing in normal operation, so any output is a finding.
- Draw strokes with `page.mouse.down()/move(...,{steps})/up()`; a click without movement triggers the radial burst.
- Top-level `const`s in the toys are global lexical bindings, reachable from `page.evaluate`: `params`, `parts`, `rings`, `SYMS`, `spriteCache`, `fpsEMA` (alchemy); `params` (ink). Use them to assert spawn counts, glyph filtering, and adaptive budget.
- Settings: `#gear` opens `#panel`; any canvas pointerdown closes it (by design — reopen before touching sliders). Sliders need a manual `input` event after `fill`.
- Fusion probe (alchemy): scribble tight circles in one spot for a few seconds, then watch for `rings.some(r => r.col === '190,140,255')`.
- Murmuration is a module script, so nothing is a global lexical binding; its hatch is `window.murm` (`params`, `pos`/`vel` typed arrays, `centroid`, `pred`, getters `fpsEMA`/`budget`/`active`/`strike`/`predW`, `reseed()`). Wait for `window.murm && murm.active > 0`, assert motion by sampling `pos` twice, click to see `strike` jump to 1 then decay, and `predW` rise after `pointermove`.
- Poise (module script) is a Rapier joint-physics mobile; its hatch is `window.poise` (`seed`, `world`, `recs` (all rigid bodies, interpolation records), `leaves`, getters `fpsEMA`/`ambient`, `maxVel()` (max linear+angular speed across bodies), `kick(s)` (flick a leaf), `screenPoint(i)` (leaf i in CSS px, for aiming drags), `rehang(seed?)`). `?seed=N` pins the sculpture; wait for `window.poise && poise.ready`. Assert: under `emulateMedia({reducedMotion:'reduce'})` `poise.ambient === 0`, it spawns at rest and `maxVel()` stays ≤ ~0.05 (the computed torque balance is a true equilibrium); `kick()` then expect `maxVel()` back under 0.04 in 20–40 s; drag via `screenPoint(i)` + `mouse.down/move` displaces that leaf ~a metre and the release ring-down decays; a ≥ 2 s `mouse.down` hold on empty space raises `maxVel()` (the breath). `#rehang` swaps in a new seed with console staying clean.
- Nightrider (module script) is a two-lane synthwave freeway with a live Web Audio synth; its hatch is `window.nr` (`S` state: `lane`, `px`, `I` intensity 0..1, `streak`, `passes`, `crashes`, `taps`, `layers`, `prog`; `cars` (scheduled traffic, each with `z`, `midi`, `arrT`, `dare`, `passed`, `crashed`); getters `T` song-time, `frames`, `fpsEMA`, `resScale`, `ctx`, `eng`, `nextStep`; `start()`, `steer(±1)`, `tapAt(cssX, cssY)`, `setProg(i)`, and the pure factories `makeEngine(ctx)` / `makeSequencer(eng, api)`). Wait for `window.nr && nr.frames > 3`; a `keyboard.press` (or `touchscreen.tap` on a `hasTouch` context) dismisses the title card and starts audio — assert `nr.S.started` and `nr.ctx.state === 'running'`. Cars only ever occupy lane 1 (`LANE_X[0]`, the right lane of a divided highway: our two lanes, a median, then two oncoming lanes whose outer lane `ONC_X` carries headlight-only traffic in `nr.oncoming`, each pass a doppler whoosh, never a collision); wait for `nr.cars.some(c => c.z > -16 && !c.passed)`, press `ArrowLeft`, and expect `passes` to increment with a `dare` recorded on that car; press `ArrowRight` and wait for `crashes` to increment (`streak` resets, `I` drops to ≤ 0.15, `S.glitch` spikes). Swipes are pointer events with `pointerType:'touch'` and |dx| > 28 on `#c`; a short pointerup without movement is a tap note (`taps++`). Toggle layers with `button[data-layer=…]`, cycle progressions with `#prog` (pending cars re-pitch). `?seed=N` pins the traffic, oncoming, palm placement and arp patterns. Pause with `p`/Space/Escape or `#pause` (`nr.setPaused(bool)`): `nr.S.paused` flips, `nr.ctx.state` becomes `suspended`, `nr.T` stops advancing, and a canvas click/tap resumes without playing a note.
- Nightrider audio can be measured without ears: build an `OfflineAudioContext`, `nr.makeEngine(off)` + `nr.makeSequencer(eng, {chordAt: nr.chordAt, intensity: () => I, layers, rng: Math.random, drumMutedUntil: () => 0, tOff: () => 0})`, call `scheduleStep(s, s * STEP)` for a few bars, `startRendering()`, and check peak < 1, RMS around −14 dBFS, and that the first-difference/RMS ratio rises with `I` (the master lowpass opening). Each voice (`kick`, `snare`, `bass`, `leadNote`, `tapNote`, `crash`, `whoosh`) can be rendered alone the same way.
- Accretion (also a module script) renders everything in one fragment shader; its hatch is `window.hole` (`params`, `view` `{theta, phi, dist}`, `uniforms`, getters `fpsEMA`/`resScale`/`frames`/`swirlT`, and `setScale(s)` which pins the internal render scale and disables auto-adaptation until reload). Wait for `window.hole && hole.frames > 3`. Assert: drag changes `view.theta/phi`, wheel shrinks `view.dist` (clamped to [2.05, 40]), sliders drive `params` and uniforms follow on the next frame. A solid-black canvas plus console shader warnings means the fragment shader failed to compile.

## Gotchas

- Headless here is software-rendered: fps numbers are ~10× below real desktops. Don't judge performance by absolute fps; check the adaptive budget (`partCap()` shrinks when `fpsEMA < 24`) engages and recovers instead. Accretion's equivalent is `resScale` sinking toward 0.22 and fps recovering; for native-res screenshots call `hole.setScale(1)` and expect ~0.5 fps headless (wait on `hole.frames` deltas, not wall time).
- `favicon.ico` 404 on index is pre-existing and harmless.
- Nightrider under SwiftShader sinks `resScale` to 0.5 within a few seconds; that is the adaptive budget working, not a bug. Traffic positions come off the audio clock, so a slow renderer never desyncs the cars from the beat.
- Alchemical block glyphs (U+1F70x) render in this container; on fontless systems the toy's tofu filter drops them — assert `SYMS.length > 0` rather than a fixed count.
