<p align="center">
  <img src="./axiom.svg" alt="Axiom" width="320">
</p>

# Axiom

**Small independent software studio.**

Axiom is a small independent software studio. This repository is the source for [axiomwtf.github.io](https://axiomwtf.github.io), the studio website. Everything is on GitHub: <https://github.com/axiomwtf>.

> **Status:** Live.

## Projects

- **Jiggle** (`jglc`, spec `0.2.1`) is deprecated in favor of Poly. It takes a single `.jgl` source file containing Rust, JavaScript, Python and other languages and compiles it into one native executable.
- **vfsx** (spec `0.2.0`) packs a folder into one signed, content-addressed archive (Ed25519) that anyone can verify.
- **Poly** (spec `0.2.1`) is a build system and package manager for Rust, C/C++ and Go, and it will replace Jiggle.

## What's in here

- `index.html` is the home page.
- `jiggle.html` covers Jiggle.
- `vfsx.html` covers vfsx.
- `poly.html` covers Poly.
- `polyspec.md` is the Poly spec.

Each page is a single HTML file with its CSS and favicon inlined, so there is no build step or framework. The only thing loaded from outside is the fonts (Archivo and Spline Sans Mono) from Google Fonts. Pushing to `main` publishes the site through GitHub Pages.

The numbered grids use a fixed number of columns. If an item is added, keep the total a multiple of the column count so the last row is not left half empty.

## Running it locally

Open any page in a browser, or serve the folder:

```bash
python3 -m http.server
```

## Spec

The full Poly specification is in [`polyspec.md`](./polyspec.md).

## License

MPL-2.0
