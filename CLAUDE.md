# web-selfhosting

Personal blog (Hugo static site, theme `hugo-theme-stack`) about the IT-related stuff that doesn't fit on other sites in the family. Part of a family of sites under too-many-machines.com (see `menu.main` in `hugo.toml`); this one is `random.too-many-machines.com`.

## Content structure

- Posts are Hugo page bundles: `content/<section>/<slug>/index.md`, with any images co-located in the same directory (referenced by bare filename, e.g. `![alt](photo.png)`).
- Front matter fields, in this order:
  ```yaml
  ---
  title: "..."
  date: 2026-07-07T00:00:00
  draft: true
  tags: ["tag-one"]
  ---
  ```
  - **New posts must be created with `draft: true`.** The user flips it to `false` themselves when ready to publish - never set `draft: false` when scaffolding a new post.
  - `tags` uses the inline array style shown above, not YAML block-list style.
  - `date` should be a safely past timestamp on the day it's written - Hugo excludes future-dated content from the build by default, which silently drops the page.
- First (by alphabetical order) image in the page bundle becomes the banner/featured image, no need for explicit `image: filename.png` and skip linking to it in the article.

## Writing style

- British English spelling throughout (colour, organise, centimetre, etc.).
- Em dashes are not used - replace with a spaced hyphen ( ` - ` ).
- ALL CAPS words are intentional emphasis - never "fix" the casing.
- Preserve the author's voice/sentence structure/fragments when editing; only fix genuine typos, spelling, missing words, and repeated words automatically. Style, restructuring, and questionable factual claims are suggestions, not automatic edits.

## Build

- `hugo` from the repo root builds the site into `public/`; drafts and future-dated posts are excluded by default (no special flags needed for a "no drafts" build).
- For a clean rebuild: `rm -rf public && hugo`.
- This is one of several sibling Hugo/Pelican sites under `/home/big/code/` (`web-advent`, `web-diy`, `web-gallery`, `web-personal` (Pelican), `web-selfhosting`) - conventions here don't necessarily apply to those.
