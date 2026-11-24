# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This is not an application — it is the **distribution channel** for Bible commentary
packages consumed by Theodia.app (a React Native/Expo app in a sibling directory). There
is no package manager, no test suite, and no CI. A change here means: add or replace a
`.commentary.zip` at the repo root, register it in `index.json`, and push to `main`.

Consumers read from the published `main` branch:
- `https://raw.githubusercontent.com/ecodiallc/theodia.commentaries/main/index.json` — the catalog
- `https://api.github.com/repos/ecodiallc/theodia.commentaries/contents/` — the file listing
- `raw.githubusercontent.com/.../main/<path>` — the packages themselves

So a package is only "shipped" once it is **both** committed at the root **and** listed in
`index.json`. A zip without a catalog entry is unreachable by the app.

`origin` is `ecodiallc/theodia.commentaries`; `upstream` is `otseng/UniqueBible_Commentaries`,
the original corpus many of these packages came from.

## Package format

Each zip contains **exactly one** SQLite file. The app unzips, finds the entry ending in
`.commentary`, copies it to `commentaries/<storageName>.commentary`, and validates that a
`Commentary` table exists. The name **inside** the archive is discarded — `storageName`
from `index.json` decides the on-disk filename. (The archive entry is still conventionally
named after the package, e.g. `cABC.commentary`; note `BSummary-*.zip` ships its entry as
`cbSummary.commentary`, which installs fine because of this.)

`Commentary` comes in three schema variants, and the app sniffs columns rather than
consulting the catalog:

| Level | Schema | Query used |
| --- | --- | --- |
| book | `Commentary(Book INT, Scripture TEXT)` | `WHERE Book = ?` |
| chapter | `Commentary(Book INT, Chapter INT, Scripture TEXT)` | `WHERE Book = ? AND Chapter = ?` |
| verse | `Commentary(Book INT, Chapter INT, Verse INT, Content TEXT)` | `WHERE Book AND Chapter AND Verse` |

A `Details` row (Title, Abbreviation, Information, Version, OldTestament, NewTestament,
Apocrypha, Strongs) is conventional but not required by the installer.

## The name prefix is load-bearing

The first character of `code`/`storageName` selects the level: `b` = book, `c` = chapter,
`v` = verse. The app calls `getCommentaryTypeFromName`, which inspects only that first
letter, and routes to the book- or chapter-level query accordingly. A book-level database
named `cX` will be queried with a `Chapter` column it does not have. Currently 2 packages
are `b`, 70 are `c`, 1 is `v`.

## index.json conventions

Each entry needs: `id` (lowercase dashed slug), `type: "commentary"`, `path` (the zip
filename), `name`/`description`/`fullName`, `version`, and `code`/`shortName`/`storageName`.

- `shortName` is always `code` minus its prefix (`cABC` → `ABC`), and `name` equals `fullName`.
- `storageName` is the on-disk identity and registry key. It must stay **stable across
  renames**; commit `be7401d` normalized it to `toStorageName(code)`, which lowercases and
  folds `-`/`+`/spaces to `_`. One outlier remains: `cOSHB2+` → `coshb2_` (the function would
  produce `coshb2`). Treat it as a bug, but changing it breaks existing installs.
- The `packages` array is ordered by `code` (case-sensitive). Keep it that way.
- `updatedAt` is edited by hand, not generated.

`README.md`'s table is the human-facing catalog, keyed by `shortName` (the two `b*` rows use
`code`). It is currently complete — one row per package — so keep it that way: correcting a
label here is a real fix, not cosmetics. `Sermon Bible` was once listed as a row label
matching no package at all, which made `cSermon` look covered when it was not.

## Adding or replacing a package

1. Zip the SQLite file as `<Code>-<Sentence-Case-Dash-Title>.commentary.zip` (bare
   `<Code>.commentary.zip` when there is no title). Every root filename starts with the
   code and is Title-Case-Dash.
2. Add the catalog entry in `code` order and bump `updatedAt`.
3. Add the README row.
4. Verify before committing — the installer rejects a package whose extracted DB lacks a
   `Commentary` table, and the app has no way to recover from a bad publish:

```bash
unzip -Z1 PACKAGE.commentary.zip                          # expect one .commentary entry
unzip -p PACKAGE.commentary.zip ENTRY.commentary > /tmp/x.db && sqlite3 /tmp/x.db \
  "PRAGMA integrity_check; SELECT COUNT(*) FROM Commentary; SELECT * FROM Details;"
sqlite3 /tmp/x.db "SELECT sql FROM sqlite_master WHERE name='Commentary';"  # confirm level
```

5. Commit both the zip and `index.json`. Push to `main` only — the app reads `main`.

## Never publish these

`cHenry` and `bAIC` ship **inside the app** and are copied on first install. The installer
refuses to install a package over a built-in, so publishing them would create entries that
can never be installed. Commit `081637d` removed them from this repo for exactly that
reason.

## Synthesis pipeline (untracked)

`/build/` and `scripts/` are gitignored — they are the authoring workspace for the
AI-synthesized `bSummary` and `bTheodia` packages, not distributed code. `scripts/` walks
`extract_book_intros.py` → `split_clusters.py` → (cluster synthesis) → `harvest_v2.py` →
`build_bsummary.py`, with `verify_bsummary.py` asserting 66 rows, no HTML markers, and all
12 required H2 sections per book.

`build_bsummary.py` and `verify_bsummary.py` both resolve their output as
`<repo-root>/BSummary-Bible-Book-Summary.commentary.zip`.

Two things about `bTheodia` differ from the rest of the catalog:

- **It has no build script.** It was packaged by zipping its SQLite source directly, so
  regenerating means repackaging the DB, not running `scripts/`.
- **Its source DB is not kept in the repo.** The `bTheodia.commentary` you are handed for an
  update is transient — package it and delete it. Before deleting, confirm the copy inside
  the published zip is byte-identical (`shasum -a 256` both), since the zip is then the only
  copy.

When republishing an update, bump the entry's `version` in `index.json`. The installer
refuses to overwrite an existing file, so the version is the only signal the download screen
has; it renders "installed vX · version Y" when the two differ.

## Repo hygiene

Files here pass through macOS and carry AppleDouble siblings (`._foo`); `.gitignore` covers
`._*` and `.DS_Store`, and they must never be committed. Two published zips
(`CDakes-…`, `CLange-…`) contain stray `__MACOSX/._*` entries — harmless, since the
installer only looks for the `.commentary` suffix, but do not reintroduce them.
