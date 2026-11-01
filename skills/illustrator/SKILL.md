---
name: illustrator
description: Guide for controlling Adobe Illustrator via MCP tools and osascript.
user-invocable: true
---

# Adobe Illustrator via MCP

Note that this is an early draft of a skill. You should not rely on it alone; expect to dig deep into the MCP tool docs and code to learn how to solve problems. Feel free to suggest improvements to the skill (additions, subtractions, organization) as you use it.

The intention of these tools is to structure interactions with Adobe Illustrator, and is oriented towards scientific figure production.

## How the MCP works (ie3jp/illustrator-mcp-server)

The MCP provides ~30 tools that talk to Illustrator via osascript. Tools fall into four groups:

- **Read** — `get_document_info`, `get_document_structure`, `get_layers`, `list_text_frames`, `get_colors`, `get_path_items`, `find_objects`, `get_selection`, etc. Read tools tag every object with a string ID (stored in the object's `note` field) and return it. Use these IDs to reference objects in modify/export calls.
- **Create** — `create_rectangle`, `create_ellipse`, `create_line`, `create_text_frame`, `create_path`, `place_image`, `create_document`, `close_document`. Each returns an ID for the new object.
- **Modify** — `modify_object` takes an object ID and a properties bag (position, size, fill, stroke, opacity, rotation, contents, font_name, font_size). `convert_to_outlines` and `apply_color_profile` operate on the whole document.
- **Export** — `export` (SVG/PNG/JPG by artboard, selection, or object ID) and `export_pdf` (print-ready with optional marks and bleed).

### Gotchas

- **Coordinate system**: Default `artboard-web` = origin at artboard top-left, Y-down (like CSS). Use `coordinate_system: "document"` for Illustrator-native Y-up.
- **Colors** are always `{"type": "rgb", "r": 0, "g": 0, "b": 255}`, `{"type": "cmyk", ...}`, or `{"type": "none"}`.
- **Font names** must be PostScript names (e.g., "ArialMT" not "Arial"). If wrong, the tool returns candidate matches.
- **Limitations**: No undo, no delete, no boolean ops, no gradients, no grouping, no effects, no layer reordering. Illustrator steals window focus on every modify/export call.

## osascript fallback

For anything the MCP can't do, you can try running ExtendScript directly:

```bash
# Inline
osascript -e 'tell application "Adobe Illustrator" to do javascript "app.activeDocument.name"'

# From file (preferred for multi-line)
osascript -e 'tell application "Adobe Illustrator" to do javascript file "/tmp/script.jsx"'
```

ExtendScript is ES3 — `var` only, no arrow functions, no template literals, no native `JSON`. Return values come back on stdout. This gives full access to Illustrator's DOM for anything the MCP tools don't cover (delete objects, boolean ops, gradients, effects, grouping, etc.).

## Opening files

```bash
osascript -e 'tell application "Adobe Illustrator" to open POSIX file "/path/to/file.ai"'
```

Works for `.ai`, `.svg`, `.pdf`, `.eps`. Illustrator refuses to open a file that is
already open (it just activates the tab); close the stale copy first. A document's
`saved` property is `false` once the user has edited it, so only close documents
with `saved == true` when refreshing re-rendered figures:

```javascript
for (var i = app.documents.length - 1; i >= 0; i--) {
    if (app.documents[i].saved) app.documents[i].close(SaveOptions.DONOTSAVECHANGES);
}
```

### matplotlib figures: open the PDF, not the SVG

Both carry editable text when saved with `svg.fonttype: none` and `pdf.fonttype: 42`,
but the SVG import raises the dialog "Clipping will be lost on roundtrip to Tiny"
(matplotlib's clip paths) on every open, while the PDF imports silently with the
clipping masks intact. Treat the PDF as the Illustrator master.

### Missing-font dialog from matplotlib math text

matplotlib sets math text (`r$^2$`, `$10^6$`) in DejaVu Sans regardless of
`font.family`, so Illustrator reports DejaVu Sans missing even in an all-Arial figure.
Add to the rcParams baseline (see the figure-formatting skill):

```python
"mathtext.fontset": "custom", "mathtext.rm": "Arial", "mathtext.it": "Arial:italic",
"mathtext.bf": "Arial:bold", "mathtext.default": "regular",
```

Verify with `pdffonts figure.pdf` (poppler) or by grepping the SVG for `font-family`.

## Exporting PDFs for LaTeX embedding

**Preferred: use the MCP `export_pdf` tool.** It clips to the artboard, doesn't change the active document path, and produces XeLaTeX-safe output with default settings:

```
export_pdf(output_path="/path/to/figure.pdf", options={trim_marks: false, registration_marks: false, color_bars: false, page_information: false, bleed: false})
```

After export, trim whitespace and verify XeLaTeX safety:

```bash
# 1. Trim to content bounds (2bp margin)
pdfcrop --margins 2 figure.pdf figure.pdf

# 2. Verify XeLaTeX-safe (no ICC stream reference)
python3 -c "
import pikepdf
pdf = pikepdf.open('figure.pdf')
for page in pdf.pages:
    if '/Group' in page and '/CS' in page['/Group']:
        cs = page['/Group']['/CS']
        assert not isinstance(cs, pikepdf.Array), f'ICC ref found — run latex-pdf-preprocess skill'
print('OK')
"
```

**Fallback: JSX script via osascript.** Use this if you need finer control over PDF settings. Note that `saveAs` changes the active document path as a side effect.

```javascript
var opts = new PDFSaveOptions();
opts.compatibility = PDFCompatibility.ACROBAT7;
opts.preserveEditability = false;
opts.fontSubsetPercentage = 100.0;
opts.artboardRange = "1";
opts.trimMarks = false;
opts.registrationMarks = false;
opts.colorBars = false;
opts.pageInformation = false;
opts.bleedOffsetRect = [0, 0, 0, 0];
```

The key settings are `ACROBAT7` + `preserveEditability = false` — this writes `/DeviceRGB` in the page transparency group instead of an ICC stream reference, avoiding the xdvipdfmx crash. Do **not** use named presets like `[PDF/X-4:2008]` or `[Smallest File Size]` — those embed ICC profiles that break xdvipdfmx. If a PDF was exported with a preset, use the `latex-pdf-preprocess` skill to fix it after the fact.

## Setup

See [install.md](./install.md) for installation instructions.
