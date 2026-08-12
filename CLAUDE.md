# CLAUDE.md

## What this is
A single self-contained HTML app, [multi-product-catalog.html](multi-product-catalog.html),
that generates an XLSX product-catalog file entirely client-side. The output matches the
structure and formatting of the Hebrew RTL template `גלילי נייר למצלמה מדפיסה.xlsx`
(the format reference — deliberately **not** committed; it's in `.gitignore` because it's a
1.3 MB real supplier file. The layout below is the authoritative spec).

There is no build step, no backend, no package manager. Open the HTML file in a browser
(or serve the folder) and it works.

## Layout of the generated sheet (must stay exact)
A1:G(4+N) where N = number of items. The template's fixed header block is rows 1–4;
each product gets one row from row 5 down.

| Row | Contents |
|-----|----------|
| 1 | Entity headers — `שם ספק`/`שם זכיין`, `מספר ...` — **yellow fill `FFFF00`**, bold Arial 11, thin borders |
| 2 | Entity values (name, number) — Arial 11 |
| 3 | *empty separator row* |
| 4 | Product headers: תמונת מוצר, תיאור המוצר, ברקוד, משקל (ק"ג), מחיר עלות כולל מע"מ, מחיר מומלץ לצרכן, מלאי (יחידות) — bold Arial 12, wrapText |
| 5.. | One row per item; the item's image is anchored over cell `A(5+i)`, description Arial 14 wrapText, the rest Arial 14 |

All cells: thin borders, center/center alignment. Column widths mirror the template; every
product row has height 87. Sheet is RTL. Sheet name = the product description when there is
exactly one item, otherwise the entity name (sanitized, ≤31 chars).

Filename: `<entity>_<barcode>.xlsx` for a single item, `<entity>_<N>_פריטים.xlsx` for several.

## The multi-item editor
One product editor is reused for every item. `addItemBtn` commits the editor into the
`products` array and clears it; committed items render in the items list and the live preview.

- **Editing pulls the item out of the array** (`editItem` splices it into the editor,
  `commitEditorItem` re-inserts it at the same index). That's why only one item can be in
  the editor at a time and `editItem` refuses when the editor isn't empty.
- **Blob-URL ownership moves with the item.** `clearProductEditor` deliberately does *not*
  revoke `previewUrl` — the committed item now owns it. Revoking happens in `deleteItem`
  and in reset. Don't "fix" that missing revoke.
- **Submit auto-commits a filled editor** so a user who typed one item and hit generate
  without pressing "add" still gets it. An empty editor plus ≥1 committed item is also valid.
- Product-field errors stay hidden while the editor is `'empty'`, so an untouched editor
  above a committed list doesn't light up red.

## Non-obvious decisions — do NOT "simplify" these back
- **`xlsx-js-style`, not vanilla SheetJS.** SheetJS Community Edition cannot *write*
  cell styles (fonts/fills/borders). The fork is loaded from CDN and exposes the same
  global `XLSX`. Swapping it for `xlsx` will silently drop all formatting.
- **JSZip post-processing injects the images.** SheetJS CE can't write images at all, so
  after `XLSX.write(..., {type:'array'})` the code re-opens the zip and adds
  `xl/media/imageN.png` per item, one `xl/drawings/drawing1.xml` holding a `oneCellAnchor`
  per product row, the drawing rels, the sheet→drawing rel, and the `[Content_Types].xml`
  entries by hand. The original template embedded its photo via Excel's newer
  "image in cell" rich-value feature (that's why A5 shows `#VALUE!` when read as data) —
  we deliberately use standard floating pictures instead, which are far more portable.
- Each image is scaled to fit a ~132×112 px box (col width 20ch × row height 87pt) with
  aspect ratio preserved, so it sits inside its cell.
- **RTL is set two ways** — `wb.Workbook.Views[0].RTL = true` plus a safety-net patch of
  `rightToLeft="1"` into the sheet XML during the JSZip pass.
- **Barcode is written as text, not a number** (the one intentional deviation from the
  template's numeric cell) so leading zeros survive. Weight/prices/stock are real numbers.
- The image cell itself is written as a blank styled cell (`t:'z'`) — it only carries the
  border/alignment; the picture floats over it.
- Images are downscaled to ≤1000px via canvas and re-encoded to PNG before embedding.
- **Barcode scan-from-image uses `@zxing/browser`** (CDN, global `ZXingBrowser`). The user can
  upload/photograph a barcode; `decodeBarcodeFromFile()` runs
  `BrowserMultiFormatReader.decodeFromImageUrl()`, strips to digits, drops them into the same
  `#barcode` field, and calls `validateAll(false)` — so the manual-entry path and all existing
  validation still apply. The scan image is **decode-and-discard** (never embedded in the XLSX;
  only the digits persist). Default multi-format reader, no format hints — EAN-13's check digit
  makes false decodes unlikely. On decode failure it shows a "type it manually" message; manual
  entry is always available.

## Verifying a change
There is no direct file-download hook in this environment, so verify by:
1. Serve the folder (`.claude/launch.json` runs `python -m http.server 8743`) and drive
   the form with the preview tools.
2. Trigger generation, capture the blob's base64 in-page (override the download anchor's
   `click`), decode it locally, and inspect with `openpyxl` + strict XML parsing of the
   drawing/relationship chain. LibreOffice is **not** installed here, so this XML-level
   check is the strongest available correctness signal.
3. For multi-item changes, add at least two items and assert there are N `oneCellAnchor`
   elements, N `xl/media/imageN.png` parts, and N matching rels — an off-by-one in the
   rId/anchor loop is the easiest way to corrupt the file.

To test the barcode scanner: generate a known EAN-13 PNG with Python `python-barcode`+`Pillow`
(e.g. `barcode.get('ean13','693606543164',writer=ImageWriter())` → `6936065431647`), copy it
into the served folder, fetch it in-page as a `File`, inject into `#barcodeImageInput`, fire
`change`, and assert `#barcode` auto-fills. Delete the temp PNG from the project folder after.

Gotchas when scripting verification on this Windows box:
- PowerShell mangles Hebrew in console stdout — write results to a JSON/txt file and read
  that back instead of printing.
- Don't name a scratch Python file `inspect.py` (shadows the stdlib `inspect` module).
- `preview_screenshot` is flaky in this environment; verify UI state via `preview_eval`
  (read DOM / getComputedStyle) and the accessibility snapshot instead.
- To trigger generation from `preview_eval`, use `form.requestSubmit()` — a synthetic
  `generateBtn.click()` doesn't reliably submit.
