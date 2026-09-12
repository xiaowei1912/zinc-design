---
name: zinc-docs-translation
description: Translate or update Zinc user-manual and proposal documents. Use when adding a language under user-manual/, syncing translations with user-manual/chinese/, reviewing translation PRs, or drafting bilingual design proposals.
---

# Zinc docs translation

Chinese in `user-manual/chinese/` is the source of truth. Read `AGENTS.md` first.

## Workflow

1. Identify the source chapter by numeric prefix (`0_` … `9_`, `99_`).
2. Create or update `user-manual/<language>/<english-filename>.md` using the filename table in `AGENTS.md`.
3. Copy structure exactly: headings, lists, tables, details/summary blocks, images, and code fences.
4. Translate prose only. Leave Zinc/Rust identifiers, APIs, and example typos unchanged.
5. Translate comments inside code **only if** the Chinese source comment is Chinese. Keep comments that are already English in the source.
6. Translate ASCII-diagram annotations; do not rename diagram field identifiers.
7. Add or refresh `user-manual/<language>/README.md` stating that Chinese is canonical and listing every chapter.

## Checks before finishing

- [ ] Same chapters as `user-manual/chinese/` (11 chapters + README)
- [ ] Same number of ` ``` ` fences and `TODO` markers
- [ ] Glossary terms match `AGENTS.md` (dictionary passing, ARC = Automatic, component, `.zno`)
- [ ] No drive-by edits to other languages or to `proposals/` unless requested
- [ ] Root `README.md` is left alone unless this PR is specifically documenting repo layout

## Review output

When reviewing an existing translation, report:

1. Completeness (missing chapters / extra files)
2. Faithfulness (meaning drift vs Chinese)
3. Code fidelity (changed samples)
4. Terminology consistency
5. Merge blockers vs follow-up nits
