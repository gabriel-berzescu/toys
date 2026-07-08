---
name: verify
description: How to build/launch/drive this repo's toys for runtime verification. Static HTML5 canvas toys; drive them in the pre-installed Chromium via playwright-core.
---

# Verifying the toys

Static site, no build step. Every toy is a self-contained HTML file (`ink/ink.html`, `alchemy/alchemy.html`) linked from `index.html`.

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

WebGL (ink) and Canvas2D (alchemy) both work under SwiftShader.

## Drive

- Collect `pageerror` + console errors on every page — the toys log nothing in normal operation, so any output is a finding.
- Draw strokes with `page.mouse.down()/move(...,{steps})/up()`; a click without movement triggers the radial burst.
- Top-level `const`s in the toys are global lexical bindings, reachable from `page.evaluate`: `params`, `parts`, `rings`, `SYMS`, `spriteCache`, `fpsEMA` (alchemy); `params` (ink). Use them to assert spawn counts, glyph filtering, and adaptive budget.
- Settings: `#gear` opens `#panel`; any canvas pointerdown closes it (by design — reopen before touching sliders). Sliders need a manual `input` event after `fill`.
- Fusion probe (alchemy): scribble tight circles in one spot for a few seconds, then watch for `rings.some(r => r.col === '190,140,255')`.

## Gotchas

- Headless here is software-rendered: fps numbers are ~10× below real desktops. Don't judge performance by absolute fps; check the adaptive budget (`partCap()` shrinks when `fpsEMA < 24`) engages and recovers instead.
- `favicon.ico` 404 on index is pre-existing and harmless.
- Alchemical block glyphs (U+1F70x) render in this container; on fontless systems the toy's tofu filter drops them — assert `SYMS.length > 0` rather than a fixed count.
