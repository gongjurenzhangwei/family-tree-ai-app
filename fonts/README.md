# Bundled CJK Font for Book PDF Export

This directory holds the bundled CJK serif font used by the A4 print-ready
Book PDF exporter (see `src/utils/bookPdf/bookPdfFont.ts`). The font is
loaded at runtime via `fetch('/fonts/NotoSerifSC-Regular-subset.otf')` and
registered with `jsPDF` through `addFileToVFS` + `addFont`, so the CJK
glyphs render correctly without depending on the host system's fonts.

> **Related spec tasks:** 9.3 `ensureCjkFont()` loader, 9.4 bundled asset,
> 9.6 missing-font fallback test.
> **Validates requirements:** 12.6, 17.4.

## Required file

```
public/fonts/NotoSerifSC-Regular-subset.otf
```

The file name is matched literally by `ensureCjkFont()`. Do not rename it
without updating that loader.

## Current state ⚠️

The `NotoSerifSC-Regular-subset.otf` currently committed here is a
**placeholder text stub**, not a real OpenType font. It exists so that:

- `GET /fonts/NotoSerifSC-Regular-subset.otf` returns `200` instead of
  `404` in `vite dev` and in the built bundle.
- Vite's static-copy step and electron-builder's `dist/**/*` glob pick it
  up automatically on fresh clones (no CI config changes required).

Because the stub is not a valid OTF, invoking the Book PDF exporter
against it in dev will either trigger the `ensureCjkFont()` serif
fallback (if the loader validates bytes) or throw inside
`jsPDF.addFont()`. Before shipping a release build, a maintainer **must**
replace the stub with a real subset produced via the `pyftsubset` command
below. CI should fail the release job if the file is smaller than
~500 KB or starts with the ASCII string `PLACEHOLDER`.

## Source

Either of the following upstreams is acceptable. Both are licensed under
the SIL Open Font License 1.1.

- **Noto Serif SC** — https://fonts.google.com/noto/specimen/Noto+Serif+SC
  (download the `Regular` weight as `NotoSerifSC-Regular.otf`).
- **Adobe Source Han Serif SC** — https://github.com/adobe-fonts/source-han-serif
  (use the `SourceHanSerifSC-Regular.otf` Region-specific subset).

A copy of the upstream `OFL.txt` license file should be kept next to the
raw font in your maintainer workspace. A copy of the license summary is
reproduced at the bottom of this README for convenience.

## Subsetting

The full Noto Serif SC font is ~20 MB, which is too large to bundle with
the app. Subset it to the Unicode ranges the app actually renders, which
brings the file to ~1–2 MB:

| Range              | Codepoints     | Purpose                       |
| ------------------ | -------------- | ----------------------------- |
| Latin-1 Basic      | `U+0020-007E`  | ASCII letters, digits, punct. |
| Latin-1 Supplement | `U+00A0-00FF`  | Accented Latin for names      |
| CJK Symbols & Punct| `U+3000-303F`  | Chinese punctuation           |
| CJK Unified Ideo.  | `U+4E00-9FFF`  | Common Chinese characters     |
| CJK Unified Ideo. A| `U+3400-4DBF`  | (optional) Rare ideographs    |

### `pyftsubset` command

`pyftsubset` ships with the `fonttools` Python package.

```bash
pip install fonttools brotli

pyftsubset NotoSerifSC-Regular.otf \
  --output-file=NotoSerifSC-Regular-subset.otf \
  --unicodes="U+0020-007E,U+00A0-00FF,U+3000-303F,U+4E00-9FFF" \
  --flavor=otf \
  --layout-features='*' \
  --no-hinting \
  --desubroutinize \
  --name-IDs='*' \
  --name-legacy \
  --name-languages='*'
```

Add `U+3400-4DBF` to `--unicodes` if your families contain rare
ideographs (Extension A). Expect the subset size to grow to ~2.5 MB in
that case.

### Verifying

After subsetting, verify:

1. The file size is between ~1 MB and ~2.5 MB.
2. `file NotoSerifSC-Regular-subset.otf` reports an OpenType font.
3. Running the app and triggering **"导出为 A4 打印版 PDF (族谱书)"** no
   longer prints `"Missing bundled CJK font; falling back to generic
   serif"` in the devtools console, and Chinese names render in the
   exported PDF.

## How the file reaches production builds

- **Vite** copies every file in `public/` to `dist/` as-is during
  `npm run build`. No `vite.config.ts` changes are required — the font
  will be served from `/fonts/NotoSerifSC-Regular-subset.otf` in both
  `vite dev` and production builds.
- **Electron** — `package.json` → `build.files` already includes
  `dist/**/*`, so the font is automatically bundled into the installer
  once Vite has copied it to `dist/fonts/`. No `extraResources` entry or
  `asarUnpack` addition is required because the renderer loads the font
  via the same `fetch('/fonts/...')` URL used in the browser, which
  resolves relative to the `dist/` root at runtime.

## Git tracking

A small ASCII **placeholder** is intentionally checked into the
repository so that `npm run build` produces a bundle that does not 404
on the font URL, and the Book PDF exporter's missing-font fallback
(`ensureCjkFont()` → `console.warn` → generic serif) can be exercised
cleanly in dev.

Before every release, replace the placeholder in-place with a real
subset (keep the filename stable), then commit the binary. Do not move
it to Git LFS without also updating the CI build scripts.

If you need to regenerate the subset after adding new characters to the
ranges above, replace the file in-place and commit it.

## License notice

The bundled font is redistributed under the SIL Open Font License,
Version 1.1. You are free to use, study, copy, modify, embed, merge,
and redistribute the font. The license forbids selling the font by
itself and requires that derivative fonts not use the Reserved Font
Names "Noto" or "Source Han" without permission. See the full license
text in the upstream repositories linked above.
