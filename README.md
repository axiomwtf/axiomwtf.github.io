# Axiom

Source for [axiomwtf.github.io](https://axiomwtf.github.io) — the website of Axiom, an
independent software studio making small, plain, dependable tools.

## Pages

- `index.html` — studio home: principles, current work, contact.
- `jiggle.html` — Jiggle: many languages, one program.
- `vfsx.html` — vfsx: signed folder archives.

## Conventions

- One self-contained HTML file per page. Markup, styles and icon are inlined —
  no build step, no framework, no sidecars.
- Every page shares the same stylesheet, typefaces (Archivo, Spline Sans Mono)
  and palette, inlined in `<head>`.
- Grid lists are numbered in Roman numerals (I, II, III, …) and use fixed column
  counts, so every row is always full — no half-empty grids.
- Source stays readable: whitespace between sections and grid cells is kept.
- Pushes to `main` publish automatically via GitHub Pages.

## View locally

Open any page in a browser, or serve the folder:

    python3 -m http.server

## Tools

- **Jiggle** (spec 0.2.1) — compile a single `.jgl` source file containing Rust,
  JavaScript, Python and more into one native executable.
- **vfsx** (spec 0.2.0) — pack a folder into one signed, content-addressed
  archive (Ed25519) anyone can verify.

## Contact

Everything lives on GitHub: <https://github.com/axiomwtf>
