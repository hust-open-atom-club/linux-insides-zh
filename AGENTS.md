# AGENTS.md

This file provides guidance for coding agents working in this repository.

## Project overview

This repository is the Simplified Chinese translation of
[0xAX/linux-insides](https://github.com/0xAX/linux-insides), a book about Linux
kernel internals. The Chinese translation is maintained by @mudongliang and
@xinqiu under the `hust-open-atom-club` organization. The project is licensed
under CC BY-NC-SA 4.0.

Most changes are documentation changes. Preserve the technical meaning of the
upstream text, the translation history, and any unrelated work already present
in the working tree.

## Repository layout

- `SUMMARY.md` is the mdBook table of contents and defines which pages are
  included in the rendered book.
- `book.toml` sets the source root to the repository root and configures HTML,
  PDF, and EPUB output.
- `TRANSLATION_STATUS.md` records each article's translation state, translator,
  and, where available, the upstream commit to which it was synchronized.
- `TRANSLATION_NOTES.md` contains the authoritative translation and typography
  conventions.
- `CONTRIBUTING.md` describes the contribution workflow.
- `CONTRIBUTORS.md` lists translation contributors.
- `Scripts/validate_markdown_links.py` extracts Markdown links and checks live
  HTTP(S) targets. It expects a directory path and requires Python 3 plus the
  `markdown` package.
- Chapter content is stored in `Booting/`, `Initialization/`, `Interrupts/`,
  `SysCall/`, `Timers/`, `SyncPrim/`, `MM/`, `Cgroups/`, `Concepts/`,
  `DataStructures/`, `Theory/`, `Misc/`, and `KernelStructures/`.
- Chapter-specific images belong in that chapter's `images/` directory. Shared
  project assets belong in `Assets/`.
- `book/` is generated output and is ignored by Git; do not edit or commit it.

## Build and validation

Install all renderers needed for a complete local build:

```bash
cargo install mdbook mdbook-pdf mdbook-epub --locked
```

Build all configured outputs:

```bash
mdbook build
```

The generated files are written to:

- HTML: `book/html/`
- PDF: `book/pdf/output.pdf`
- EPUB: `book/epub/Linux 内核揭秘.epub`

Preview the book locally with:

```bash
mdbook serve --open
```

For Markdown link changes, install the script dependency if necessary and run
the validator on the relevant directory (or `.` for the entire repository):

```bash
python3 -m pip install markdown
python3 Scripts/validate_markdown_links.py .
```

The link validator performs network requests, so distinguish unavailable or
rate-limited services from malformed links. Also run:

```bash
git diff --check
```

After validation, confirm that generated files under `book/` have not entered
the Git diff and that unrelated working-tree changes remain untouched.

## CI and deployment

`.github/workflows/mdbook.yml` runs on pushes to `master` and by manual
dispatch. It installs `mdbook`, builds the project, uploads `book/html`, and
deploys that HTML output to GitHub Pages. PDF and EPUB are configured as
optional mdBook outputs and are not deployed by this workflow.

## Translation workflow

Before translating or updating an article:

1. Read `TRANSLATION_NOTES.md`, `TRANSLATION_STATUS.md`, and
   `CONTRIBUTING.md`.
2. Check that the article is not already claimed. Translation claims use an
   issue plus a PR that marks the article as in progress.
3. Synchronize the latest relevant upstream source before translating it.
4. Prefer one translated file per PR so review remains focused.
5. After the translation PR is merged, close the claim issue, record the first
   12 characters of the verified upstream commit in `TRANSLATION_STATUS.md`,
   and add the translator to `CONTRIBUTORS.md` when needed.

When adding, removing, moving, or renaming an article, keep `SUMMARY.md` and
`TRANSLATION_STATUS.md` consistent. Do not change an upstream commit hash or an
article's completion state without verifying it against the corresponding
source and local translation.

Report problems in the English source to `0xAX/linux-insides`; report Chinese
translation problems in this repository.

## Translation style

Follow `TRANSLATION_NOTES.md` as the source of truth. In particular:

- Translate for accurate meaning and natural Chinese; do not submit raw machine
  translation output.
- Keep recurring technical terms consistent across chapters.
- Use full-width Chinese punctuation except for parentheses, ellipses, and
  dashes as specified by the project conventions. Remove the half-width space
  that follows English punctuation when converting it to Chinese punctuation.
- Normally insert one half-width space between Chinese and English text,
  Chinese and Arabic numerals, and English text and Arabic numerals. Do not add
  a space next to punctuation.
- Keep English personal names untranslated.
- Chapter titles must contain Chinese text only. Introduce an original English
  term at its first occurrence in the body when useful, rather than repeating
  it in titles.
- Translate English "you" as “你”. If repetition harms readability, use “我们”
  or omit the subject when the context stays clear.
- Omit the upstream author's recurring statement that English is not his native
  language.

Preserve code blocks, identifiers, commands, URLs, image paths, and technical
values unless the source update specifically requires changing them. Avoid
unrelated reformatting or line-ending normalization in translation-only
changes.
