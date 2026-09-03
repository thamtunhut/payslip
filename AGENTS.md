# AGENTS.md

Context for whoever (human or AI agent) works on this repo next.

## What this is

A payslip PDF generator for **Công ty Cổ phần Giải trí và Truyền thông Lam Phương**.
HR uploads the monthly payroll Excel file → the tool fills the numbers straight onto
the company's real payslip PDF template → downloads one PDF per employee (or a ZIP
of all of them).

Live at: https://payslip.vinhhp2.site (GitHub Pages, custom domain via `CNAME`).

## Files

| File | Purpose |
|---|---|
| `index.html` | The tool. Fully self-contained — open it directly (`file://`) or serve it, no build step, no server, works offline. |
| `calibrate.html` | A drag-and-drop companion tool for re-aligning where values get printed on the template, when the template PDF changes. See "Recalibrating" below. |
| `README.md` | User-facing usage instructions (Vietnamese). |
| `.gitignore` | Blocks real payroll `.xlsx`/`.csv` files and generated `Payslip_*.pdf` / `PhieuLuong_*.zip` output from ever being committed (they contain employee PII — CCCD, MST, salary). |
| `Payslip Template.pdf` | The company's official blank template (logo, labels, grid). **Not tracked in git** — it's embedded as base64 inside `index.html`/`calibrate.html` already, so the tool doesn't need this file on disk. Kept locally only as the source-of-truth to re-embed from if it's edited again. |
| `Lương NV HĐLĐ T7.26.xlsx` | A real payroll file left in the working directory by the user for testing. Never commit files like this — `.gitignore` covers the pattern but double-check before any `git add -A`. |

There is intentionally no `package.json`, build tool, or bundler. Everything (SheetJS,
pdf-lib, `@pdf-lib/fontkit`, JSZip, pdf.js, and two font files) is fetched once and
inlined as raw `<script>` blocks / base64 strings directly in the HTML files. This is
deliberate — the whole point is a single portable file that still works with no
internet connection and no install step. When bumping a library, re-download it and
re-embed it the same way (see "Rebuilding embedded assets" below); don't switch to a
CDN `<script src>` or a package manager.

## How `index.html` generates a PDF

`generatePdfBytes(emp)` (search for it in the file) does the real work:

1. Loads the template PDF bytes (`TEMPLATE_PDF_BASE64`) fresh into a new `pdf-lib`
   `PDFDocument` for every employee — the template is never mutated, just read.
2. Embeds three fonts into that document and draws plain `page.drawText(...)` calls
   at fixed coordinates. Nothing is rasterized/screenshotted; the output PDF has the
   template's real vector content (logo, borders, labels) plus new real text on top.
3. Coordinates come from `FIELD_POSITIONS`, a `{x, top, size}` map, one entry per
   field (`hoTen`, `cccd`, `mst`, `luongCung`, ... `title` for the month/year in the
   header). `top` is measured from the **top** of the 792×612pt page down to the
   text's **baseline** — i.e. it's already `pageHeight - y`, matching what
   `calibrate.html` shows and exports. If you ever hand-edit a coordinate, keep that
   convention or text will land above/below where you expect.

### The three embedded fonts, and why there are three

| Constant | Font | Used for | Why |
|---|---|---|---|
| `VIETNAMESE_FONT_BASE64` | Roboto (regular, variable-font instance) | All normal body text/values | Full Vietnamese diacritic coverage. Subsets fine with `{subset:true}`. |
| `ROBOTO_BOLD_FONT_BASE64` | Roboto Bold (static instance, built with `fonttools varLib.instancer` from the variable font at `wght=700`) | The employee's name (`hoTen`) | Needs to be bold *and* support Vietnamese *and* subset correctly — see the pdf-lib bug below. |
| `CALIBRI_BOLD_FONT_BASE64` | Carlito Bold, pre-subset down to `0123456789/ ` only | The `mm/yyyy` in the title | See below — this one is a workaround, not a general-purpose font. |

**Why not just use real Calibri?** Calibri is a Microsoft-licensed font; it can't be
redistributed by embedding it in a public repo/site. **Carlito** is the open-source
(SIL OFL) metrics-compatible clone of Calibri — same look, free to embed — so it's
the substitute used for anything that's meant to visually match "Calibri Bold".

**The pdf-lib/fontkit subsetting bug:** embedding the *full* Carlito-Bold.ttf with
`{subset:true}` silently drops most digit glyphs in the output PDF (some digits
render, most don't — verified across multiple independent PDF renderers, so it's a
real bug in how pdf-lib/fontkit subset this font's glyph table, not a rendering
artifact). Flattening composite glyphs and stripping hinting did **not** fix it.
The working fix: pre-subset the font down to only the characters actually needed
(`pyftsubset ... --text="0123456789/ "`) *before* handing it to pdf-lib, then embed
that tiny (~2.6KB) font with `{subset:false}` so pdf-lib never re-subsets it itself.
If you ever need Carlito Bold for a *different* string (not just digits/slash), you
must regenerate the pre-subset font with those characters included — see below — or
switch that field to the Roboto Bold instance instead (which subsets correctly for
arbitrary text, just isn't a Calibri lookalike).

If you need another bold-and-safe font: the pattern that reliably works is
"instantiate a static weight out of Roboto's variable font with `fonttools
varLib.instancer`" (composite-glyph-free, subsets cleanly). The pattern that's
risky: embedding some other TTF at full size with `subset:true` — test it the same
way (draw `'0123456789'` and actually render the output PDF) before trusting it.

## Recalibrating field positions (`calibrate.html`)

If the template PDF changes (a field moves, a row is added, etc.), don't hand-guess
new coordinates. Open `calibrate.html`:

1. It loads the same embedded template (or upload a newer version of the PDF via the
   file input at the top).
2. Drag each field's sample text directly on the rendered page, or type exact
   `x` / `top` / size values in the side panel.
3. "Xuất PDF thử" generates a real PDF through the same pdf-lib code path, so what
   you see is what you get.
4. "Sao chép cấu hình" copies a ready-to-paste `const FIELD_POSITIONS = {...}` block
   — paste it over the existing one in `index.html` (and optionally in
   `calibrate.html` itself, so its own default stays in sync).

`calibrate.html` renders the PDF background using `pdf.js`. It includes a small
`requestAnimationFrame` shim at the top of the file — some embedding contexts
(headless/automation browser panes) never mark the tab as visible, so the native
rAF (which browsers pause for hidden tabs) never fires and `page.render()` hangs
forever. The shim falls back to `setTimeout` when `document.visibilityState ===
'hidden'`. Don't remove it just because it looks unnecessary in a normal browser —
it's there for exactly the environment where the previous version silently hung.

## Rebuilding embedded assets

All embedded libraries/fonts/the template were fetched once and spliced into the
HTML as raw text (see conversation history / commit messages for the exact `perl`
one-liners used — search-and-replace a placeholder token or an exact known byte
string, since these files are too large to paste through a normal diff-based editor
safely). If you need to redo this:

- **Libraries** (SheetJS, pdf-lib, `@pdf-lib/fontkit`, JSZip, pdf.js): download the
  exact `.min.js` from cdnjs/jsdelivr, wrap as `<script>...</script>`, verify it
  contains no literal `</script` substring before inlining (a couple of these
  libraries' minified source does, in string literals — check first, it'll break
  the page silently otherwise).
- **pdf.js worker**: can't be a normal inline `<script>`, it must run as a Web
  Worker. It's embedded as `<script type="text/plain" id="pdfWorkerSrc">` (so the
  browser doesn't execute it as a script and it doesn't need base64/text-decoding
  gymnastics), then turned into a real worker at runtime via
  `new Blob([...textContent], {type:'application/javascript'})` +
  `URL.createObjectURL`.
- **Fonts / template PDF**: base64-encode (`base64 -w0 file > out.txt`) and splice
  into the matching `const ..._BASE64 = "...";` line.
- **Carlito Bold pre-subset**: regenerate with
  `pyftsubset Carlito-Bold.ttf --output-file=out.ttf --text="0123456789/ " --glyph-names --layout-features='' --no-hinting`
  if the character set that field needs ever changes.
- **Roboto Bold instance**: regenerate with
  `fonttools varLib.instancer Roboto[wdth,wght].ttf wght=700 wdth=100 -o out.ttf`.

## Testing without a real browser session

There's no test suite. What was actually used to verify changes during development:
render the generated PDF with `pdfplumber` (`page.to_image()`) from Python instead
of trusting a screenshot tool — it's an independent rendering engine from whatever
the browser/PDF-preview tool uses, so it catches font/glyph bugs a single renderer
might mask or, going the other way, might falsely flag. When in doubt, check the
output PDF with at least two different renderers before concluding something is
(or isn't) broken.

## Privacy / what never goes in this repo

The real payroll Excel file and any generated `Payslip_*.pdf` / `PhieuLuong_*.zip`
contain employee CCCD, MST, and salary data. This repo is **public**. `.gitignore`
already blocks the common patterns — don't override it with `git add -f`, and
double-check `git status` before any commit that wasn't scoped to just the tool
files.
