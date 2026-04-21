<!-- auto:start — rewritten by scripts/sync-readme.sh from plugin.json; do not hand-edit between these markers -->
<h1 align="center">font-pirate</h1>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="License"></a>
  <a href=".claude-plugin/plugin.json"><img src="https://img.shields.io/github/package-json/v/codyhxyz/font-pirate?filename=.claude-plugin%2Fplugin.json&label=version" alt="Version"></a>
  <a href="https://claude.com/product/claude-code"><img src="https://img.shields.io/badge/built_for-Claude%20Code-d97706" alt="Built for Claude Code"></a>
</p>

<p align="center"><b>All fonts are free now.</b></p>

<p align="center">
  <img src="docs/hero.gif" alt="font-pirate demo" width="900">
</p>
<!-- auto:end -->

Drop an image of a typeface into the chat and you end up with a renamed OTF that matches its silhouette. font-pirate identifies the closest OFL base font on Google Fonts, renders each glyph at 2000px/em, runs potrace over it, and stitches the result back into a TrueType `glyf` table with contour directions flipped the way TrueType expects. No hand-coded beziers, no blocking questions, no Glyphs license.

## Before → After

> **Before:** "I found a font I love in a screenshot and I just want my own version of it."
> **After:** 82 glyphs traced at 2000px/em, base font instantiated at the right weight, name tables rewritten, `FontName-Style.otf` on disk, and an HTML specimen page with the full size cascade open in your browser.

## What you walk away with

- A working `.otf` you can `@font-face` into a site or drop into Figma today
- A matching OFL base font chosen for you — no Adobe/Linotype licensing trap
- A specimen page with A–Z/a–z/0–9/punct + a size cascade, so you know what the result looks like before you ship it
- A pipeline that doesn't hand-code bezier curves — the one thing that makes every "LLM made a font" experiment look amateur

## Install

### Option 1 — install from the codyhxyz-plugins marketplace *(recommended)*

```
/plugin marketplace add codyhxyz/claude-plugins
/plugin install font-pirate@codyhxyz-plugins
```

### Option 2 — install directly from this repo

```
/plugin marketplace add codyhxyz/font-pirate
/plugin install font-pirate@font-pirate
```

### Option 3 — local smoke test

```bash
git clone https://github.com/codyhxyz/font-pirate
claude --plugin-dir ./font-pirate
```

### Prereqs

```bash
brew install potrace
pip install pillow fonttools
```

## Try it — paste any of these

> "Here's a screenshot of a font I like — make me my own version called 'Halliford'."

> "Trace this specimen as a geometric sans at weight 600 and call it 'Moirai Display'."

> "Build me an OTF that matches this headline — pick whichever Google Font gets closest as the base."

## How it works

1. **Classify** the specimen — serif vs. sans, sub-style, weight, contrast, distinguishing letters.
2. **Pick an OFL base** — maps the classification onto Google Fonts (Montserrat, Playfair, Sacramento, etc.), prefers variable fonts so weight is a parameter not a download.
3. **Render** each of 82 target glyphs at 2000px/em into a Pillow canvas, threshold-binarize, crop.
4. **Trace** each bitmap with `potrace -s --turdsize 2 --alphamax 1.0 --opttolerance 0.2`.
5. **Rebuild** — parse the SVG path, transform into UPM coordinates, feed through `Cu2QuPen(reverse_direction=True)` so contour directions flip correctly, replace entries in the `glyf` table.
6. **Rename** name-table IDs 1/2/3/4/6/9/16/17, save as OTF, emit an HTML test page.

## Where it stops

- Only 82 glyphs are traced — accented characters keep the base font's outlines. Latin-only by design.
- No hinting pass. Fine on macOS; Windows small-size rendering can look soft.
- No glyph editing UI — if the trace misses a serif you care about, swap the base font and re-run, don't hand-correct beziers.
- Doesn't download commercial fonts. OFL Google Fonts only.

## Examples

<details>
<summary><b>Scenario 1</b> — "Make me an editorial serif from this headline screenshot"</summary>

<br>

You drop in a magazine cover crop. Claude classifies it as *high-contrast didone, weight ~700, sharp terminals*, picks `Playfair Display` as the variable base, instantiates at 700, traces the 82 glyphs, and saves `Halliford-Bold.otf` plus `font-test.html`. The specimen page shows the pangram at 96/72/48/36/24/18/14/12px so you can tell whether the hairline serifs survived the threshold — if they didn't, you ask for a rerun against `Libre Bodoni` and you're done in another ~90 seconds.

</details>

<details>
<summary><b>Scenario 2</b> — "I need a geometric sans at 600 for a logotype"</summary>

<br>

No specimen image, just a vibe description. Claude pulls `Montserrat[wght].ttf`, instantiates at 600, runs the pipeline anyway (rendering the base to itself gives you a clean, renamed OTF with no licensing questions), and hands you `Moirai-SemiBold.otf`. You rename, tweak, ship.

</details>

## Contributing

Issues and PRs welcome. If a trace comes out spiky or a base-font mapping is wrong for some style, file it with the specimen image + the glyph that broke.

## License

[MIT](LICENSE) © 2026 Cody Hergenroeder. Generated fonts inherit the base font's license — Google Fonts bases are OFL, which permits rename + redistribution.
