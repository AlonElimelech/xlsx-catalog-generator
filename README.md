# xlsx-catalog-generator

Client-side HTML form that generates a formatted Hebrew RTL product-catalog XLSX,
with barcode scanning from a photo.

## Usage

Open `multi-product-catalog.html` in a browser. No build step, no backend, no install.
(Some browsers block camera/file access on `file://` — if so, serve the folder:
`python -m http.server 8743`.)

Fill in the supplier/franchisee details and the product rows, upload a product image,
and the page produces an `.xlsx` matching the layout of the reference template it was
built against — yellow entity headers, thin borders, RTL sheet, Arial fonts, image
embedded in the sheet. The exact layout is documented in `CLAUDE.md`.

Barcodes can be typed manually or scanned: upload/photograph a barcode and the digits
are decoded into the barcode field. The scan image is discarded — only the digits persist.

## How it works

- [`xlsx-js-style`](https://github.com/gitbrent/xlsx-js-style) writes the sheet *with*
  cell styles (SheetJS Community Edition can't).
- SheetJS can't write images at all, so JSZip re-opens the generated workbook and injects
  `xl/media/`, the drawing XML, and the relationship entries by hand.
- [`@zxing/browser`](https://github.com/zxing-js/browser) decodes barcodes from images.

All three load from CDN — nothing is vendored.
