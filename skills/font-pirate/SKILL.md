---
name: font-pirate
description: "Recreate a font from a specimen image using bitmap tracing. Analyzes the specimen, identifies a matching OFL base font from Google Fonts, renders glyphs at high resolution, traces with potrace into clean vector outlines, and builds an OTF font file. Invoke when the user shows you a font they like and wants their own version of it."
---

# Font Pirate — Bitmap Tracing Pipeline

You are a type designer. Given an image or example showing a font in use, you will identify its visual DNA, find a matching open-source base font, and produce a pixel-perfect traced version with a new name.

## Critical Lessons (Read First)

- **DO NOT hand-code bezier curves.** Hand-coding glyph outlines produces crude, amateur results. Always use the bitmap tracing pipeline below.
- **DO NOT block on reversible decisions.** If the user doesn't specify a font name, just use whatever name makes sense. Don't ask.
- **The bitmap tracing pipeline works.** Render → threshold → potrace → parse SVG → build font. This produces professional-quality results.
- **Contour direction matters critically.** Potrace outputs CCW outer / CW inner contours. TrueType needs CW outer / CCW inner. Use `Cu2QuPen(reverse_direction=True)` — never write your own contour reversal function.

## Requirements

Before starting, verify these tools are available:

```bash
# Required
potrace --version          # Bitmap tracer (install: brew install potrace)
python3 -c "from PIL import Image; print('OK')"         # Pillow for rendering
python3 -c "from fontTools.ttLib import TTFont; print('OK')"  # fonttools

# fonttools submodules needed
python3 -c "from fontTools.pens.ttGlyphPen import TTGlyphPen; print('OK')"
python3 -c "from fontTools.pens.cu2quPen import Cu2QuPen; print('OK')"
python3 -c "from fontTools.varLib.instancer import instantiateVariableFont; print('OK')"
```

If potrace is missing: `brew install potrace`

## Phase 1: Specimen Analysis

Analyze the provided image and document characteristics. Be agentic — don't ask the user questions you can answer yourself.

### Classify the Font
- **Category**: Serif, Sans-serif, Slab, Mono, Display, Script, Blackletter
- **Sub-style**: Humanist, Geometric, Grotesque, Neo-grotesque, Transitional, Old-style, Didone, etc.
- **Weight**: Estimate CSS weight 100-900
- **Contrast**: Monolinear → High contrast
- **Key features**: Serifs type, terminal style, a/g story, distinctive letters

### Choose a Base Font

Based on the analysis, select the closest matching Google Font (OFL-licensed). Common mappings:

| Specimen Style | Good Base Fonts |
|---|---|
| Geometric sans | Poppins, Montserrat, Inter, Outfit |
| Humanist sans | Nunito, Source Sans, Open Sans |
| Neo-grotesque sans | Roboto, Work Sans |
| High-contrast serif | Playfair Display, Cormorant |
| Didone/editorial serif | Libre Bodoni, Bodoni Moda |
| Bold display serif | DM Serif Display, Zilla Slab |
| Script/handwriting | Sacramento, Lobster, Dancing Script |
| Slab serif | Roboto Slab, Zilla Slab |
| Monospace | JetBrains Mono, Fira Code |

**Variable fonts** (Montserrat, Playfair Display, Nunito, etc.) can be instantiated at any weight. **Static fonts** (Poppins-SemiBold, DM Serif Display, Sacramento) come at a fixed weight.

### Download the Base Font

Download from Google Fonts GitHub (raw URLs work best):

```bash
# Variable fonts
curl -sL "https://github.com/google/fonts/raw/main/ofl/montserrat/Montserrat%5Bwght%5D.ttf" -o /tmp/font-bases/Montserrat.ttf

# Static fonts
curl -skL "https://github.com/google/fonts/raw/main/ofl/poppins/Poppins-SemiBold.ttf" -o /tmp/font-bases/Poppins-SemiBold.ttf

# Use -k flag if SSL certificate errors occur
```

Store base fonts in `/tmp/font-bases/`.

## Phase 2: The Tracing Pipeline

This is the core of font-pirate. For each font:

### Step 1: Load and Instantiate

```python
from fontTools.ttLib import TTFont
from fontTools.varLib.instancer import instantiateVariableFont

font = TTFont(base_path)

# For variable fonts, pin to target weight
if target_weight is not None and "fvar" in font:
    instantiateVariableFont(font, {"wght": target_weight})
```

### Step 2: Save Instantiated Font for Pillow

**Critical:** Pillow can't set variable font axes. For variable fonts, save the instantiated version to a temp file BEFORE loading in Pillow. Otherwise Pillow renders at the default weight while the font is instantiated at the target weight — causing a mismatch.

```python
if is_variable:
    temp_path = os.path.join(tmpdir, "instantiated.ttf")
    font.save(temp_path)
    pil_font = ImageFont.truetype(temp_path, render_size)
else:
    pil_font = ImageFont.truetype(base_path, render_size)
```

### Step 3: Render Each Glyph at High Resolution

Render at **2000px per em** (2x the UPM grid for 1000-UPM fonts). This gives potrace enough detail for smooth curves.

```python
RENDER_PX_PER_EM = 2000

canvas_w = int(render_size * 2.5)
canvas_h = int(render_size * 2.0)
x_offset = int(render_size * 0.3)
baseline_y_from_top = int(render_size * 1.3)

img = Image.new('L', (canvas_w, canvas_h), 255)  # White background
draw = ImageDraw.Draw(img)
draw.text((x_offset, baseline_y_from_top), char, font=pil_font, fill=0, anchor='ls')

# Threshold FIRST, then get bbox (avoids antialiasing expanding bounds)
binary = img.point(lambda p: 0 if p < 128 else 255, '1')
bbox = binary.getbbox()

# Crop to content + padding
binary_cropped = binary.crop((left, top, right, bottom))
binary_cropped.save('glyph.pbm')
```

**Important:** Threshold to binary BEFORE calling `getbbox()`. If you getbbox on the grayscale image, faint antialiasing pixels expand the bounding box unnecessarily.

### Step 4: Trace with Potrace

```bash
potrace -s --turdsize 2 --alphamax 1.0 --opttolerance 0.2 -o glyph.svg glyph.pbm
```

Potrace outputs SVG with coordinates at **10x pixel resolution**, Y-axis pointing UP.

The SVG contains a `<g transform="translate(0,H) scale(0.1,-0.1)">` — ignore this transform. Read the raw path `d` attributes directly; they're in potrace's native Y-up coordinate system.

### Step 5: Parse SVG Path Data

Parse the `d` attribute from `<path>` elements. Potrace uses:
- `M`/`m` — moveTo (absolute/relative)
- `C`/`c` — cubic bezier (absolute/relative)
- `L`/`l` — lineTo (absolute/relative)
- `z` — closePath

Convert all relative commands to absolute. Handle implicit command repeats (e.g., `c` followed by 78 numbers = 13 implicit cubic beziers).

### Step 6: Coordinate Transform

Convert potrace's 10x-pixel Y-up coordinates to font UPM coordinates:

```python
POTRACE_SCALE = 10
scale = upm / render_size  # e.g., 1000/2000 = 0.5

def transform(raw_x, raw_y):
    font_x = (raw_x / POTRACE_SCALE - x_offset) * scale
    font_y = (raw_y / POTRACE_SCALE - baseline_y_up) * scale
    return (round(font_x), round(font_y))
```

Where `baseline_y_up = crop_height - baseline_y_from_top_of_crop` (distance from bottom of cropped image to the baseline).

### Step 7: Build TrueType Glyphs

**CRITICAL: Contour direction reversal.**

Potrace outputs:
- Outer contours: **counter-clockwise** (CCW)
- Inner contours (holes): **clockwise** (CW)

TrueType requires:
- Outer contours: **clockwise** (CW)
- Inner contours: **counter-clockwise** (CCW)

**Use Cu2QuPen's built-in reversal.** Do NOT write your own `reverse_contour` function — it's extremely easy to get wrong (incorrect endpoint tracking when reversing cubic beziers causes catastrophic spiky artifacts).

```python
from fontTools.pens.ttGlyphPen import TTGlyphPen
from fontTools.pens.cu2quPen import Cu2QuPen

ttpen = TTGlyphPen(None)
cu2qu_pen = Cu2QuPen(ttpen, max_err=0.5, reverse_direction=True)

# Draw parsed SVG commands directly into cu2qu_pen
for cmd, pts in parsed_commands:
    if cmd == 'M':
        cu2qu_pen.moveTo(transform(*pts[0]))
    elif cmd == 'L':
        cu2qu_pen.lineTo(transform(*pts[0]))
    elif cmd == 'C':
        cu2qu_pen.curveTo(transform(*pts[0]), transform(*pts[1]), transform(*pts[2]))
    elif cmd == 'Z':
        cu2qu_pen.closePath()

new_glyph = ttpen.glyph()
```

### Step 8: Replace Glyph Outlines

Replace the traced characters in the base font's `glyf` table. Non-traced glyphs keep the original outlines.

```python
glyf_table[glyph_name] = new_glyph
hmtx[glyph_name] = (advance_width, new_glyph.xMin)  # Keep original advance width
```

### Step 9: Rename and Save

Update the name table entries (IDs 1, 2, 3, 4, 6, 9, 16, 17) to the new font name.

```python
font.save("FontName-Style.otf")
```

## Phase 3: Test Page

Generate `font-test.html` with:
- Font loaded via `@font-face`
- Size cascade: 96, 72, 48, 36, 24, 18, 14, 12px
- Pangram at each major size
- Full character set grid (A-Z, a-z, 0-9, punctuation)
- Distinctive pairs section
- Body text paragraph at 18px

## Character Set

Trace these 82 characters (the rest keep base font outlines):

```
A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
a b c d e f g h i j k l m n o p q r s t u v w x y z
0 1 2 3 4 5 6 7 8 9
. , ; : ! ? ' " - ( ) / @ & # $ % + = *
```

## Reference Implementation

The working pipeline is at: `trace-fonts.py` in the fashion-design project. Key functions:

- `parse_svg_path(d)` — SVG path parser with relative→absolute conversion
- `render_glyph(...)` — High-res Pillow rendering with proper baseline anchoring
- `trace_glyph(...)` — Potrace execution and SVG extraction
- `traced_paths_to_pen(...)` — Coordinate transform and pen drawing
- `process_font(...)` — Full pipeline for one font
- `rename_font(...)` — Name table updates

## Batch Processing

Multiple fonts can be processed in a single run. Define a config dict:

```python
FONTS = {
    "FontName": {
        "base": "BaseFont.ttf",       # Filename in /tmp/font-bases/
        "weight": 600,                 # Target weight (None for static fonts)
        "style": "SemiBold",           # Style name for output
        "desc": "Description",         # For test page
        "out_dir": "fontname-font",    # Output directory
    },
}
```

## Known Limitations

1. **82 of N glyphs are traced** — accented characters keep original outlines (subtle inconsistency possible but rarely noticeable)
2. **Slight stroke thinning** — binary threshold erodes ~0.5px per edge; negligible at 2000px/em for most fonts, but can affect hairline serifs
3. **No hinting** — traced glyphs lose TrueType hints (doesn't matter on macOS; may affect Windows small-size rendering)
4. **Base font matching** — quality depends on choosing the right OFL base font. If the result doesn't match the specimen, try a different base font.
