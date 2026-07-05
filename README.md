MAKE USE OF VERSION-4
# KiDocGen

**KiDocGen** is a KiCad PCB Editor plugin that turns a finished board design
into a professional, ready-to-share engineering report at the click of a
single toolbar button. It reads your live board and schematic directly
through KiCad's own APIs, builds an accurate Bill of Materials by
cross-referencing both, measures the physical board, renders real PCB and
schematic images, and assembles everything into a polished PDF — cover
page, project info, board statistics, images, BOM, and a datasheet index.
No manual screenshotting, no exporting BOMs by hand, no formatting a
report in a separate app. One click inside KiCad, one PDF out.

## Setup

**1. Install the plugin**
Download or clone this repo, then copy the whole `KiDocGen` folder into your KiCad scripting plugins directory:
- **Windows:** `C:\Users\<you>\Documents\KiCad\<version>\scripting\plugins\`
- **Linux:** `~/.local/share/kicad/<version>/scripting/plugins/`
- **macOS:** `~/Documents/KiCad/<version>/scripting/plugins/`

**2. Install Python dependencies into KiCad's own Python**
KiCad ships its own bundled Python, separate from any Python already on your system — dependencies must go into that one specifically.
```
<kicad-python> -m pip install reportlab svglib rlPyCairo
```
(To find `<kicad-python>`, open KiCad's **Tools → Scripting Console** and run `import sys; sys.executable`.)

**3. Confirm `kicad-cli` is available**
Ships with KiCad 7+, normally in the same folder as `kicad.exe`. KiDocGen finds it automatically even if it's not on your system PATH.

**4. Refresh and run**
In the PCB Editor: **Tools → External Plugins → Refresh Plugins**. A new **"KiDocGen - Generate Report"** button appears in the toolbar. Open a board with a matching schematic in the same project folder, click the button, fill in the dialog, and generate.

## What's implemented now
- `core/pcb_reader.py` — reads live board data via `pcbnew` (footprints, tracks, vias, pads, nets, layers)
- `core/sch_reader.py` — reads `.kicad_sch` file(s) directly (own dependency-free S-expression parser), follows hierarchical sheets
- `core/dimensions.py` — measures board width/height/thickness/area from the Edge.Cuts bounding box
- `core/bom.py` — merges schematic + PCB data into a grouped BOM (e.g. "R1-R4" with qty), categorizes parts by reference prefix
- `core/datasheets.py` — collects unique datasheet links from the BOM
- `core/cli_utils.py` — locates `kicad-cli` even when it's not on PATH (checks next to KiCad's Python)
- `core/screenshots.py` — exports **real PNG images**: PCB top/bottom via `kicad-cli pcb render` (raytraced), schematic via `kicad-cli sch export png` where available, falling back to `sch export svg` + pure-Python conversion (svglib + reportlab) on KiCad builds that don't have direct PNG export yet
- `core/report.py` — assembles everything into the final PDF with ReportLab (cover, project info, stats, images, BOM, datasheet index) — embeds the PNGs directly, not placeholders
- `gui/input_dialog.py` + `gui/input_form.html` — the HTML input dialog (unchanged from Phase 1)

## Images: what changed
Earlier drafts exported PCB/schematic images as SVG, which don't embed
cleanly into a PDF. This version exports real PNGs:
- **PCB**: `kicad-cli pcb render --side top/bottom` — a clean raytraced image, no conversion needed
- **Schematic**: tries `kicad-cli sch export png` directly; if your kicad-cli build doesn't support that subcommand yet (true for some KiCad 10.0 releases), it automatically falls back to SVG export + converting that SVG to PNG in pure Python

That fallback conversion needs two extra pip packages in KiCad's Python
(see Requirements below) — without them, the plugin still works, it
just leaves the schematic as an unembedded SVG reference in the PDF
instead of a real image.

## What's tested vs. what needs KiCad to test
`pcbnew` only exists inside KiCad's own Python interpreter, so
`pcb_reader.py`, `dimensions.py`, and the PCB half of `screenshots.py`
**cannot be run or tested outside KiCad** — they're written carefully
against the documented API but need a real run inside KiCad to confirm.

Everything else — `sch_reader.py`, `bom.py`, `datasheets.py`, and
`report.py` — has **zero pcbnew dependency** and was tested end-to-end
against a synthetic two-sheet schematic before being shipped here
(hierarchical sheet walking, DNP/no-BOM filtering, ref-range compression
like "R1-R4", datasheet dedup, and full PDF rendering all verified).

## Requirements
- KiCad 7+ (bundled Python + wxPython + wx.html2)
- `kicad-cli` findable either on PATH or in the same folder as KiCad's Python (this plugin checks both automatically)
- `reportlab`, `svglib`, `rlPyCairo` installed into KiCad's own Python (see Setup above)
  - `reportlab` — builds the PDF
  - `svglib` + `rlPyCairo` — only needed for the schematic-image fallback path (converting SVG to PNG) on KiCad builds where `kicad-cli sch export png` doesn't exist yet. If you skip these, the plugin still runs fine, the schematic image just won't embed.

## Known limitations to fix in later passes
- If a schematic symbol is instanced multiple times via hierarchical sheet re-use, it's currently only counted once (no per-instance reference disambiguation yet)
- No design-review/scoring page yet (that was Phase 6 in the original plan)
- No custom icon yet — the toolbar button uses KiCad's default plugin icon

## Roadmap
- Design review / scoring page (clearance checks, silkscreen overlap, trace width warnings)
- Multi-instance hierarchical sheet support for BOM accuracy
- Custom toolbar icon
- Packaging for KiCad's Plugin and Content Manager (PCM)
