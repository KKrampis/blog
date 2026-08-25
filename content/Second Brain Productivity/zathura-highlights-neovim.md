---
title: "A Workflow for Turning PDF Reading Into Linked Notes"
date: 2026-08-25
draft: false
description: "How to capture Zathura reading highlights into an Obsidian-style vault via a shell script, with wiki-links resolved by markdown-oxide inside Neovim."
tags: ["zathura", "neovim", "obsidian", "markdown-oxide", "zettelkasten", "pdf", "workflow"]
---

# A Workflow for Turning PDF Reading Into Linked Notes

Reading in [Zathura](https://pwmt.org/projects/zathura/) has a particular
appeal: it's fast, keyboard-driven, and deliberately minimal. It renders a
document and gets out of the way. That minimalism is also its limitation —
Zathura has no way to create a highlight. It can display annotations that
already exist in a file, but it cannot write new ones. There's no keybinding
for it, no menu, no hidden flag. A long-open feature request on Zathura's
issue tracker confirms this isn't an oversight so much as a design choice
that's never been resolved: a branch for annotation support has sat unmerged
since 2018.

That limitation shapes the workflow described here. Rather than waiting for
Zathura to become an annotator, the approach treats it as what it already is
— a reading surface — and builds a small bridge from that reading surface
into a note vault, using tools that are actually built for writing.

## Reading and note-taking as separate jobs

The core idea is to stop treating "read the PDF" and "save what mattered"
as one action. They become two: Zathura handles the first, and a short
script handles the second, triggered by a single keypress while still
inside the document.

Zathura's configuration format supports binding a key to an external
command, and it passes along useful context automatically — which file is
open, and which page is currently visible:

```
# ~/.config/zathura/zathurarc
map <C-n> exec bash -c "~/.local/bin/pdf-capture.sh '$FILE' '$PAGE'"
```

`$FILE` and `$PAGE` are filled in by Zathura itself at the moment the key is
pressed. Combined with Zathura's built-in text-selection mode (a toggle key,
then a click-drag to select), this is enough to capture *what* was being
read and *where*, without ever leaving the reader. Pressing the bound key
sends the selected text and page number to a script sitting outside Zathura
entirely.

That script's job is simple: take the selected text, wrap it in a
consistent format — a quote block, a page reference, a timestamp — and
append it to a note that already exists for that source document:

```bash
TEXT=$(wl-paste 2>/dev/null || xclip -o -selection clipboard)
NOTE="$HOME/vaults/main/Sources/$(basename "${FILE%.pdf}").md"

cat >> "$NOTE" <<EOF

> [!quote] p. $PAGE
> $TEXT
> — [[$(basename "${FILE%.pdf}")]] p.$PAGE
EOF
```

Nothing more elaborate than that is needed for the connection to work.

## Where the linking actually happens

The interesting part of this workflow isn't the capture script — it's what
happens afterward, and it happens almost by accident. The note being
appended to lives inside an Obsidian-style vault, edited in Neovim through
[obsidian.nvim](https://github.com/obsidian-nvim/obsidian.nvim) for vault
mechanics and [markdown-oxide](https://github.com/Feel-ix-343/markdown-oxide)
for link intelligence. The line the capture script appended a moment ago —

```
> — [[Deep Work]] p.42
```

— is a plain wiki-link, exactly the syntax Obsidian itself uses. The moment
markdown-oxide's language server touches that file, `[[Deep Work]]` becomes
a real, navigable reference: `gd` jumps to the *Deep Work* note, and that
note's own backlink references now include this quote. Nothing had to be
configured for this specific case — the LSP is set up once, generally, for
the whole vault:

```lua
require("lspconfig").markdown_oxide.setup({
  capabilities = capabilities, -- with dynamicRegistration enabled
})
```

and from then on every wiki-link written into the vault, by hand or by
script, is picked up the same way. No integration step is needed
between the PDF-reading half of this workflow and the note-taking half —
they never communicate directly. They simply agree on a shared folder and a
shared syntax. Zathura's capture script writes plain markdown; Neovim's
plugins read plain markdown. That's the entire contract between them, and
it's also why the workflow is easy to keep working over time — there's no
API version to track, no plugin dependency chain, nothing to break when one
tool updates independently of the other.

## When a real highlight is worth the extra step

The captured quote-and-link approach covers most reading, but sometimes a
persistent, visible highlight inside the PDF itself is worth having — for a
document that will be reopened later, or shared with someone else who
should see exactly what stood out. Zathura still can't produce that, but
Python libraries built specifically for PDF manipulation can. A small
addition to the same capture script can write an actual highlight
annotation into the file at the moment the text is selected, so the next
time the document opens — in Zathura or anywhere else — the highlight is
already there. Zathura is content to *display* annotations it didn't create
itself; it only refuses to make them.

## Handling PDFs that arrive already annotated

Not every document enters this workflow through the capture script. Some
PDFs already carry annotations — made by a collaborator, exported from a
reference manager, or produced during an earlier pass through a different
tool entirely. For those, a separate extraction step reads the PDF's
existing annotation layer and converts it directly into markdown, page
numbers and all, which then drops straight into the vault the same way a
live capture would. This turns the workflow into something that works
retroactively, not just for documents read start-to-finish inside Zathura.

## A heavier version for serious reading

For casual reference-checking, the Zathura-based workflow above is enough.
For denser reading — a real literature review, a stack of papers — the
better tool for the *annotation* step turns out not to be Zathura at all,
but a reference manager with a proper PDF reader built in. Its highlighting
is genuinely keyboard-driven, and unlike a PDF's own annotation layer, its
highlights sit inside a real, queryable database rather than being scattered
across individual files. A Neovim plugin bridges that database directly into
notes, offering citation-key completion while typing and a command that
pulls a session's highlights in with page-accurate references attached.
Zathura still has a place in that setup — for the fast lookups that don't
need any of this — but the actual highlighting work moves elsewhere.

## Why this shape works

None of this requires Zathura to become something it was never designed to
be. Its appeal is precisely that it does one job — rendering a document,
quickly, with vim-style keys — without trying to also be an annotation
suite, a reference manager, and a note database. The fix isn't making the
reader smarter; it's accepting that reading, annotating, and linking are
three distinct jobs, and connecting them with something boring enough not
to need maintenance: a shell script and a shared folder of plain text.

Small, separate pieces, loosely joined by a common format — that's the
whole workflow.

---

*Tools referenced: [Zathura](https://pwmt.org/projects/zathura/) ·
[obsidian.nvim](https://github.com/obsidian-nvim/obsidian.nvim) ·
[markdown-oxide](https://github.com/Feel-ix-343/markdown-oxide) ·
[PyMuPDF](https://pymupdf.readthedocs.io/en/latest/recipes-annotations.html) ·
[pdfannots](https://github.com/0xabu/pdfannots) ·
[zotcite](https://github.com/jalvesaq/zotcite)*
