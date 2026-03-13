# Technical Spec: wood-jp.github.io

## Overview

Hugo static site using the [Blowfish theme](https://blowfish.page).
Deployed to GitHub Pages. This spec covers technical work only — content (posts) is out of scope.

## Current State

- Hugo with Blowfish installed as a Hugo module
- Config in `config/_default/` (split directory pattern, TOML format)
- Deployed via GitHub Pages

## Goals

Technical improvements to the site's Hugo/Blowfish setup. Specific tasks TBD via planning,
but areas of interest include:

- **Config correctness**: Ensure `config/_default/` files are complete and well-organized;
  remove any placeholder or default values that should be customized
- **Theme customization**: Custom layouts, partials, or shortcodes that extend or override
  Blowfish defaults appropriately
- **Asset pipeline**: Ensure custom CSS/SCSS (if any) is correctly processed via Hugo Pipes
- **Build hygiene**: `hugo --minify` produces zero warnings; no unused config keys
- **Performance**: Verify built output is optimized (minified HTML/CSS/JS, fingerprinted assets)

## Build Commands

```bash
hugo --minify              # must complete with no errors or warnings
hugo server --buildDrafts  # local verification
```

## Acceptance Criteria for Any Change

- `hugo --minify` passes with no errors or warnings
- The affected pages render correctly under `hugo server --buildDrafts`
- No Blowfish theme files are modified directly — only overrides in `layouts/` or `assets/`

## Notes

- Do not write or modify content (posts, pages) — technical structure only
- Do not modify the Blowfish theme module directly; use Hugo's override mechanism
