---
name: verify
description: How to build/launch/drive this repo's toys for runtime verification. Static HTML5 canvas toys; drive them in the pre-installed Chromium via playwright-core.
---

# Verifying the toys

Static site, no build step. Every toy is a self-contained HTML file (`ink/ink.html`, `murmuration/murmuration.html`, `accretion/accretion.html`, `poise/poise.html`) linked from `index.html`. Murmuration, accretion and poise each vendor Three.js next to themselves (`three.module.min.js` + `three.core.min.js` — the min build imports the core file; both must be present); poise additionally vendors the Rapier physics engine (`rapier.es.js`, wasm inlined as base64, Apache-2.0 text in `rapier-LICENSE.txt`).

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
  args: ['--use-gl=angle', '--use-angle=swiftshader', '--enable-unsafe-swiftshader']
})
```

WebGL (ink) and Three.js/WebGL2 (murmuration, accretion, poise — the latter also runs Rapier's wasm) all work under SwiftShader.

## Drive

- Collect `pageerror` + console errors on every page — the toys log nothing in normal operation, so any output is a finding.
- Draw strokes with `page.mouse.down()/move(...,{steps})/up()`; a click without movement triggers the radial burst.
- Top-level `const`s in ink are global lexical bindings, reachable from `page.evaluate`: `params`.
- Settings: `#gear` opens `#panel`; any canvas pointerdown closes it (by design — reopen before touching sliders). Sliders need a manual `input` event after `fill`.
- Murmuration is a module script, so nothing is a global lexical binding; its hatch is `window.murm` (`params`, `pos`/`vel` typed arrays, `centroid`, `pred`, getters `fpsEMA`/`budget`/`active`/`strike`/`predW`, `reseed()`). Wait for `window.murm && murm.active > 0`, assert motion by sampling `pos` twice, click to see `strike` jump to 1 then decay, and `predW` rise after `pointermove`.
- Poise (module script) is a Rapier joint-physics mobile; its hatch is `window.poise` (`seed`, `world`, `recs` (all rigid bodies, interpolation records), `leaves`, getters `fpsEMA`/`ambient`, `maxVel()` (max linear+angular speed across bodies), `kick(s)` (flick a leaf), `screenPoint(i)` (leaf i in CSS px, for aiming drags), `rehang(seed?)`). `?seed=N` pins the sculpture; wait for `window.poise && poise.ready`. Assert: under `emulateMedia({reducedMotion:'reduce'})` `poise.ambient === 0`, it spawns at rest and `maxVel()` stays ≤ ~0.05 (the computed torque balance is a true equilibrium); `kick()` then expect `maxVel()` back under 0.04 in 20–40 s; drag via `screenPoint(i)` + `mouse.down/move` displaces that leaf ~a metre and the release ring-down decays; a ≥ 2 s `mouse.down` hold on empty space raises `maxVel()` (the breath). `#rehang` swaps in a new seed with console staying clean.
- Accretion (also a module script) renders everything in one fragment shader; its hatch is `window.hole` (`params`, `view` `{theta, phi, dist}`, `uniforms`, getters `fpsEMA`/`resScale`/`frames`/`swirlT`, and `setScale(s)` which pins the internal render scale and disables auto-adaptation until reload). Wait for `window.hole && hole.frames > 3`. Assert: drag changes `view.theta/phi`, wheel shrinks `view.dist` (clamped to [2.05, 40]), sliders drive `params` and uniforms follow on the next frame. A solid-black canvas plus console shader warnings means the fragment shader failed to compile.

## Gotchas

- Headless here is software-rendered: fps numbers are ~10× below real desktops. Don't judge performance by absolute fps; check the adaptive budget (`partCap()` shrinks when `fpsEMA < 24`) engages and recovers instead. Accretion's equivalent is `resScale` sinking toward 0.22 and fps recovering; for native-res screenshots call `hole.setScale(1)` and expect ~0.5 fps headless (wait on `hole.frames` deltas, not wall time).
- `favicon.ico` 404 on index is pre-existing and harmless.
