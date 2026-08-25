---
title: "Zettelkasten, Obsidian, and Synced Annotations"
date: 2026-08-25
draft: false
description: "The ideas behind the Zettelkasten method, how Obsidian made linked notes mainstream, and where Zotero fits as the annotation layer underneath."
tags: ["zettelkasten", "obsidian", "zotero", "neovim", "obsidian-nvim", "markdown-oxide", "second-brain", "workflow"]
---

# Zettelkasten, Obsidian, and Synced Annotations

Behind any workflow for turning reading into linked notes sits a fairly old
idea about how notes should be structured, a piece of software that made
that idea mainstream, and a reference manager that solved a narrower but
related problem: keeping annotations attached to the reading they came from.
Understanding those three pieces separately makes it much clearer why such
a workflow ends up built the way it does.

## The Zettelkasten method

The term *Zettelkasten* — German for "slip box" — describes a note-taking
system built around one core constraint: every note should hold a single
idea, written in the note-taker's own words, and every note should be linked
to other notes rather than filed under a single category. The method is most
associated with the sociologist [Niklas
Luhmann](https://en.wikipedia.org/wiki/Niklas_Luhmann), who maintained a
physical box of roughly ninety thousand index cards over several decades and
credited it with much of his academic output. Each card carried an
identifier, a short piece of writing, and references to other cards — a
paper analogue of what a hyperlink does digitally.

What made the slip box unusual for its time wasn't the note-taking itself;
scholars had kept notes for centuries. It was the emphasis on *connection
over hierarchy*. A traditional filing system asks where a note belongs — 
which folder, which subject heading. A Zettelkasten asks instead what a note
relates to, and lets a web of links stand in for the folder structure
entirely. New ideas often surface not from writing a note but from noticing,
later, that two previously unrelated notes now sit near each other in the
link graph.

## Obsidian and the linked-note category

Obsidian, released in 2020, is the tool that most directly built a product
around the Zettelkasten principle of connection over hierarchy. Its notes
are plain markdown files stored in a folder — called a *vault* — with no
proprietary format underneath. The feature that defines the whole category
is the wiki-link: typing a note's name inside double square brackets,
`[[like this]]`, creates a reference to that note, whether or not the note
exists yet. Obsidian then does two things with every link it finds across
the vault: it lets a person jump directly to the linked note, and it builds
a *backlinks* panel on every note showing everything that links to it.

That second part is the piece that actually reproduces the slip-box effect.
A note doesn't just point outward to what it references — it also
accumulates, automatically, a record of what pointed back to it. Read
enough sources on a topic and a note can end up with backlinks from a dozen
different documents, none of which had to be filed together, sorted into
the same folder, or tagged with foresight. The connections show up because
the links were made in the moment of reading, not because a taxonomy was
planned in advance.

Because a vault is just a folder of markdown files, nothing about that
structure requires the Obsidian application itself. This is exactly the gap
that tools like [obsidian.nvim](https://github.com/obsidian-nvim/obsidian.nvim)
and [markdown-oxide](https://github.com/Feel-ix-343/markdown-oxide) fill —
they read and write the same `[[wiki-link]]` syntax, build the same
backlink relationships, inside a text editor instead of the dedicated app.
The vault format is the actual standard being interoperated with; the
Obsidian application is one client among several that can open it.

## Where Zotero fits — and where it doesn't overlap

Obsidian-style linking solves the problem of connecting notes to each
other. It has nothing to say, on its own, about where those notes come
from. For anyone whose notes originate mostly from reading — papers,
books, long PDFs — a second, older problem sits underneath: keeping a
highlight attached to the document it was made in, and to the citation
details of that document, without retyping either by hand.

Zotero is a reference manager built to solve that problem. Its core job is
bibliographic — storing authors, titles, publication details, and
generating citations in whatever format is needed — but its PDF reader also
supports real annotation: highlighting, sticky notes, ink markup, all
stored not as edits scattered across individual PDF files but as structured
records in Zotero's own database, tied back to the specific reference each
annotation belongs to. Open a paper, highlight a paragraph, and that
highlight is now a queryable object with a page number, a color, and a
parent citation — independent of whether the PDF file itself ever gets
touched.

That database is what makes annotation *syncing* meaningful. Because
highlights aren't just marks on a file but structured entries linked to a
citation key, they can be pulled out programmatically and turned into
markdown, with the citation information already attached. This is precisely
what [zotcite](https://github.com/jalvesaq/zotcite) does for Neovim: it
reads Zotero's database directly, offers citation-key completion while
writing, and can pull a document's highlights into a note as ready-made
quote blocks, each one already carrying a page-accurate reference:

```
[@Ahrens2017SmartNotes, p. 34]
```

The citation key follows the same principle as the wiki-link — a short,
stable identifier that both a person and a tool can resolve back to the
full source. It just resolves through Zotero's bibliographic database
instead of through a folder of markdown files.

## How the three pieces sit together

None of these three systems depend on each other, which is part of why they
combine cleanly. The Zettelkasten method is a set of principles about how
notes should relate to one another. Obsidian's vault format is one
concrete, portable implementation of those principles, built on plain text
and a link syntax simple enough for other tools to read and write. Zotero
solves an adjacent problem entirely — not how notes connect to each other,
but how a highlight stays connected to the document and citation it came
from.

A workflow for turning PDF reading into linked notes is really just a seam
stitched between the second and third of these: a highlight captured while
reading, formatted the way Obsidian's link syntax expects, dropped into a
vault where the linking layer picks it up automatically. Neither side needs
to know the other exists. The wiki-link doesn't care whether it was typed by
hand or generated by a citation database; the citation database doesn't
care whether the resulting note ever gets opened in Obsidian, Neovim, or
anything else. Each piece does one job, and the plain-text format is the
only thing holding them together.

## Where a text editor fits into a vault built for an app

Obsidian the application is not, strictly speaking, required by any of
this. A vault is a folder of markdown files following a link convention —
the application is one reader and writer of that convention, not the
convention itself. That distinction is what makes it possible to edit the
same vault from a terminal-based editor like Neovim, using a plugin like
[obsidian.nvim](https://github.com/obsidian-nvim/obsidian.nvim), and have
the result behave identically to editing it in the app.

obsidian.nvim's job is to reproduce the vault mechanics that make Obsidian
feel like more than a folder of text files: completing a `[[link]]` as it's
typed, scoped to notes that actually exist in the vault; creating daily
notes and populating them from templates; searching by tag; pasting an
image and having it land in the vault's attachments folder with a link
already in place. None of this changes the underlying files in a way the
Obsidian app would object to — a note created by obsidian.nvim on a laptop
opens normally in Obsidian on a phone, and the reverse holds just as well.
The vault doesn't know which editor last touched it.

That's what actually makes the PDF capture step, described in the workflow
posts, land somewhere useful rather than in an isolated file. The note a
highlight gets appended to is a normal vault note, opened and completed by
obsidian.nvim the same way any other note would be, and its backlinks show
up the same way in Obsidian's own graph view if the vault is ever opened
there. The capture script doesn't write *into obsidian.nvim* — it writes
into the vault, and obsidian.nvim is simply the thing making that vault
comfortable to work in from a terminal, alongside whichever tool actually
resolves the finer link intelligence, such as backlink-aware navigation
through an LSP like markdown-oxide.
