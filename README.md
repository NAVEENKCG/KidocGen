# KiDocGen (Phases 1-5)

A KiCad PCB Editor plugin that reads your board and schematic, builds a
real BOM, exports images, and generates a professional PDF report -
all triggered from a single toolbar button, HTML input dialog included.

## What's implemented now
- `core/pcb_reader.py` — reads live board data via `pcbnew` (footprints, tracks, vias, pads, nets, layers)
- `core/sch_reader.py` — reads `.kicad_sch` file(s) directly (own dependency-free S-expression parser), follows hierarchical sheets
- `core/dimensions.py` — measures board width/height/thickness/area from the Edge.Cuts bounding box
- `core/bom.py` — merges schematic + PCB data into a grouped BOM (e.g. "R1-R4" with qty), categorizes parts by reference prefix
- `core/datasheets.py` — collects unique datasheet links from the BOM
- `core/screenshots.py` — exports PCB top/bottom SVGs via `pcbnew.PLOT_CONTROLLER`, and schematic SVG via `kicad-cli sch export svg`
- `core/report.py` — assembles everything into the final PDF with ReportLab (cover, project info, stats, images, BOM, datasheet index)
- `gui/input_dialog.py` + `gui/input_form.html` — the HTML input dialog (unchanged from Phase 1)

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

## Installation
1. Copy the whole `KiDocGen` folder into your KiCad scripting plugins directory:
   - **Windows:** `C:\Users\<you>\Documents\KiCad\<version>\scripting\plugins\`
   - **Linux:** `~/.local/share/kicad/<version>/scripting/plugins/`
   - **macOS:** `~/Documents/KiCad/<version>/scripting/plugins/`
2. Restart KiCad, or **Tools → External Plugins → Refresh Plugins**.
3. Open a PCB with a matching schematic in the same project folder, click "KiDocGen - Generate Report" in the toolbar.
4. Fill in the HTML dialog, click Generate. The PDF is written to your chosen output folder.

## Requirements
- KiCad 7+ (bundled Python + wxPython + wx.html2)
- `kicad-cli` on PATH for schematic image export (ships with KiCad 7+ by default; if missing, the report still generates, just without the schematic image)
- No pip installs needed — ReportLab is only required in this dev/test sandbox to prove the PDF layout; **inside KiCad you'll need `pip install reportlab --target=<kicad's site-packages>` once**, since KiCad's bundled Python doesn't ship it by default

## Known limitations to fix in later passes
- If a schematic symbol is instanced multiple times via hierarchical sheet re-use, it's currently only counted once (no per-instance reference disambiguation yet)
- PCB image export currently produces SVG, not PNG — embedding in the PDF works best with PNG, so a conversion step (e.g. via `cairosvg`, or asking KiCad to render PNG directly with `kicad-cli pcb export svg` + rasterize) is the next thing to add
- No design-review/scoring page yet (that was Phase 6 in the original plan)

## Next
Once you confirm this generates a real PDF from one of your actual boards
(even if the images come out empty or SVG-only at first), tell me what
broke and we'll fix the PNG export + review scoring next.

