<p align="center">
  <img src="./axiom.svg" alt="Axiom" width="320">
</p>

# Axiom

Source for [axiomwtf.github.io](https://axiomwtf.github.io),the website of us, a
independent software studio making small, plain, dependable tools.

## Pages

- `index.html` - studio home: principles, current work, contact.
- `jiggle.html` - Jiggle: many languages, one program.
- `vfsx.html` - vfsx: signed folder archives.

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


## Contact

Everything lives on GitHub: <https://github.com/axiomwtf>
