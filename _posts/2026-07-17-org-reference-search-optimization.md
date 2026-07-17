---
layout: post
title: "The 20-Second Stall That Wasn't a Search Problem"
date: 2026-07-17
description: "Loading 52 Org files took over 20 seconds, so I researched full-text search: inverted indexes, BM25, SQLite FTS5. Then I measured, and found that reading the files took 0.03s and opening them took 4.5s. A story about find-file-noselect, and why 0.66MB never needs an index."
---

I keep my study notes as Org files in a `references/` directory — 52 documents, about 90,000 words. [K-Agenda](https://github.com/rukmaldias/K-Agenda), the web dashboard I [wrote about earlier](/2026/07/11/k-agenda.html), has a References tab that lists them in a tree and renders whichever one you click.

It took over 20 seconds to show that list. Not to render a document — just to show the file *names*.

So I did what you're supposed to do: I researched the problem. What are the proven, widely-used approaches to optimising full-text search over a corpus of text files? The answer that came back was a good one. Inverted indexes are the foundation of every serious search engine. BM25 is the de facto ranking standard. SQLite's FTS5 gives you both in about twenty lines with no dependencies. Trigram indexes power substring search at scale. Build the index once, persist it, query it many times.

All of that is correct. None of it was my problem.

---

## The diagnosis buried in the research

Reading back through the research, one line mattered more than the whole list of algorithms:

> 89,674 words is roughly 500–600 KB of raw text. Reading that from disk should take milliseconds, not 20+ seconds — so the bottleneck is almost certainly not I/O.

That was right, and it should have stopped me. If reading the data is milliseconds and I'm waiting 20 seconds, then whatever I'm doing wrong happens *around* the data, not to it. An index makes queries faster. I didn't have a query problem — I had a "showing the list at all" problem, and the list is just filenames and headings. There was nothing to index.

The lesson I'd take from this: research answers the question you ask. Ask "how do I make text search fast" and you'll get an excellent answer about text search. Whether text search is your bottleneck is a different question, and only measurement answers it.

## Measuring instead of guessing

So I measured, in stages, against my real files:

| Step | Time |
|---|---|
| Reading all 52 files off disk | **0.032s** |
| Parsing 1,016 headings out of them | 0.38s |
| `find-file-noselect` on those 52 files, buffer setup alone | **2.57s** |
| Full tree build | 2.65s |

The parsing was nearly free. The *reading* was free. Ninety-seven percent of the time was one function: `find-file-noselect`.

Here's the code that caused it. It looks completely reasonable:

```elisp
(defun k-agenda-model--with-file-visited (file thunk)
  (let* ((already-open (get-file-buffer file))
         (buffer (find-file-noselect file)))
    (unwind-protect
        (with-current-buffer buffer (funcall thunk))
      (unless already-open (kill-buffer buffer)))))
```

Open the file, parse it, close it again if it wasn't already open. Tidy. It even cleans up after itself — the comment explained that leaving 90+ reference files in the buffer list would be rude.

That cleanup is what made it pathological. Because the buffers were killed every time, *every* tree build paid the full cost again. Nothing was ever warm.

## What `find-file-noselect` actually does

`find-file-noselect` is not "read a file." It's "do everything Emacs does when a human opens a file." That means every `find-file-hook`, and in my configuration that list is:

```
undo-tree-load-history-from-hook   ; reads and decompresses an undo history file
projectile-find-file-hook-function
recentf-track-opened-file
auto-revert-find-file-function
url-handlers-set-buffer-mode
epa-file-find-file-hook
vc-refresh-state                   ; spawns a git subprocess
```

Per file. Fifty-two times. `vc-refresh-state` alone forks git 52 times to ask which files are tracked — a question I never asked. `undo-tree` reads 52 undo-history files off disk so I can undo edits in buffers I'm about to destroy. Then `org-mode-hook` runs (`org-clock-load`), and in a GUI session font-lock fontifies all 52 buffers so they'll look nice in windows nobody will ever open.

All of it for buffers that get parsed and thrown away microseconds later.

This is why my batch measurements said 4.5 seconds while the real thing took over 20: batch mode doesn't fontify. The gap between those two numbers is Emacs prettying up text no human would see.

## Fix one: stop visiting files

The replacement reads the file into a temp buffer and turns on `org-mode` with the hooks suppressed:

```elisp
(defun k-agenda-model--with-file-parsed (file thunk)
  (with-temp-buffer
    (insert-file-contents file)
    (let ((buffer-file-name file)
          (org-inhibit-startup t))
      (delay-mode-hooks (org-mode))
      (funcall thunk))))
```

That's the whole fix. Org's parser doesn't care that the buffer is a throwaway — `org-map-entries`, `org-get-tags`, `org-entry-get` all work exactly as before, because `org-mode` still runs `org-set-regexps-and-options` and so still honours in-buffer `#+TODO:` settings. What's skipped is the machinery that only matters for a buffer a person is going to look at.

`buffer-file-name` is bound rather than set, because the parse genuinely needs it (heading IDs hash file+position) but the buffer must stay unvisited — no lock file, no save prompt, no entry in `buffer-list`.

**4.5s → 0.3s, byte-identical output.** I verified that last part by keeping a copy of the old implementation around and diffing the trees: 52 roots, 1,016 headings, every id, title, level and tag identical.

## Fix two: cache, but let the filesystem invalidate it

Reading is cheap, but doing it on every request when nothing changed is still waste. So each file gets cached with its `(mtime . size)`:

```elisp
(defun k-agenda-model--reference-entry (file)
  (let* ((stamp (k-agenda-model--reference-file-stamp file))
         (cached (gethash file k-agenda-model--reference-cache)))
    (if (and cached
             (k-agenda-model--reference-stamp-equal-p stamp (plist-get cached :stamp)))
        cached
      (let ((entry (append (list :stamp stamp)
                           (k-agenda-model--reference-parse file))))
        (puthash file entry k-agenda-model--reference-cache)
        entry))))
```

The part I like: **there is no invalidation call anywhere in the codebase.** No hook that clears the cache, no "remember to expire this" comment. The stamp *is* the invalidation. Save a file and its mtime changes, so it re-parses; the other 51 don't. And because the check is against the filesystem rather than against something Emacs was told, it stays correct when files change behind Emacs' back — a `git pull`, an edit from another editor.

That's the difference between a cache you trust and a cache you fear. Invalidation bugs come from caches that need to be *told* things.

**Result: 0.24s cold, 0.0008s warm.** Editing one file re-parses one file.

## The search, and why it has no index

Now the actual feature. With the corpus cached in memory, I measured search before building anything:

| | Time per query |
|---|---|
| Brute-force scan, all 52 files, from disk | 11ms |
| Brute-force scan, case-insensitive, in memory | **~5ms** |

Five milliseconds. Across everything. There is no index worth building here — the corpus is 0.66MB, and a linear scan finishes faster than the browser can paint the result. FTS5 would have added a schema, a persistence story, a rebuild path, and a whole class of "the index is stale" bugs to save four milliseconds I don't have.

Indexes earn their place at a scale I'm nowhere near. At 100MB, absolutely. At 0.66MB, the machinery to avoid the work costs more than the work.

The requirement was: match filenames and content, filenames ranked first. That's not a relevance model like BM25 — it's a two-tier sort. Files matching on name or `#+TITLE:` go in one bucket, files matching only on content go in the other, and the buckets are concatenated.

Matching sections come back nested in their real outline position. If a hit is buried under a heading that doesn't itself match, that ancestor is kept as scaffolding and flagged as *not* a match, so the result reads as a pruned outline rather than a flat pile of hits:

```
2D Cartoon Phase 1 Study Guide
  * The Central Concept: Stage vs Composer
  * The Interface Map
    Task Group 1 — Character Types          <- ancestor, not a hit
    * Open the G3 Quadruped template
  * Task Group 2 — Timeline & Keyframes
```

One more thing worth doing: queries go through `regexp-quote`. People type prose, and prose contains `*` and `c++` and `a.c`. A query treated as a regexp either errors or silently matches the wrong thing.

## The 39ms that should have been 5

My first working version took 39ms per query, not 5. The scan was 5ms, so where did the other 34 go? Profiling the internals:

| | Time |
|---|---|
| Scanning the corpus text | 5.5ms |
| Scanning each of 1,016 sections | 10.5ms |
| **Pruning trees to matches** | **24.9ms** |

The tree pruner was walking every node of every file — including the ~85% of files with zero matches, where it had nothing to do but was doing it anyway.

The fix came from an observation about the data rather than the code. Every section's title and body is a *contiguous substring* of its file's text. So if the whole-file scan doesn't match, no section in that file can possibly match — no scan of its sections, no walk of its tree. One cheap check licenses skipping all the expensive work:

```elisp
(text-hit (string-match-p needle (plist-get entry :text)))
...
(when (or name-hit text-hit)
  ;; only now look at sections
  ...)
```

**39ms → 6-14ms.** The lesson repeats: I'd assumed the scan was the expensive part, because scanning is the part that sounds expensive. It wasn't. Profile the thing, not your intuition about the thing.

## A bug fell out of the tests

Writing a test for section-level results, I asserted that a parent heading shouldn't be flagged as a match when only its *child* contained the query. It failed. The parent was matching.

The cause was in `k-agenda-model--entry-body`, code I hadn't touched and which had been shipping for months:

```elisp
(org-back-to-heading t)
(org-end-of-meta-data t)
(let* ((start (point))
       (end (save-excursion (or (outline-next-heading) (point-max)))))
  ...)
```

When a heading has no body of its own, `org-end-of-meta-data` leaves point sitting *on* the next heading. Then `outline-next-heading` — which always moves at least one heading forward — skips over it to the one after. So the "body" measured from the child heading to the child's sibling: a bodyless parent returned its entire first child, heading line and all, as its own text. The docstring said "stopping before the first child heading." The code said otherwise.

This had been live the whole time. Clicking any project anchor in my dashboard showed its first child's raw Org source. I'd looked straight at it and never registered it as wrong. The fix is three lines:

```elisp
(if (org-at-heading-p)
    ""
  ...)
```

Search didn't cause this bug; search made it *visible*, by asking a question about the data precise enough that a wrong answer was obvious. That's a decent argument for building features that inspect your data.

## Where reading from disk is wrong

The same `find-file-noselect` cost hit the main dashboard — a shared snapshot every page waits on. Same disease, smaller dose: 7 agenda files, 0.58s.

But the same fix would have been a bug. Reference docs are read-only mirrors; agenda files are the ones I actually *edit*. K-Agenda broadcasts on `org-after-todo-state-change-hook`, which can fire while an edit is still unsaved. Parse from disk there and the browser shows you the state you just changed away from.

The resolution isn't a compromise between speed and correctness — it's noticing that the two cases don't overlap:

```elisp
(let ((buffer (get-file-buffer file)))
  (if buffer
      ;; Open: may hold unsaved edits. Read the buffer.
      (with-current-buffer buffer
        (save-excursion (save-restriction (widen) (funcall thunk))))
    ;; Not open: unsaved edits cannot exist. Disk IS the truth.
    (k-agenda-model--with-file-parsed file thunk)))
```

If a file has no buffer, it *cannot* have unsaved changes, so disk is authoritative by definition. If it has one, use it. Both branches are exactly right rather than approximately right.

**0.58s → 0.16s**, and a side benefit: the old code silently left all 7 agenda files open in my buffer list every time I loaded the web UI. Now it opens none.

One function deliberately kept the old behaviour — the one that writes. It calls `save-buffer`, and `save-buffer` in a temp buffer with `buffer-file-name` bound would cheerfully write parse scratch over the real file.

## Where it landed

| | Before | After |
|---|---|---|
| References tree, cold | 4.5s (>20s in GUI) | **0.24s** |
| References tree, warm | 4.5s | **0.0008s** |
| Search query | — | **6-14ms** |
| Dashboard snapshot, cold | 0.58s | **0.16s** |
| Agenda buffers silently opened | 7 | **0** |

No index. No SQLite. No BM25. About 400 lines of Elisp, most of it comments explaining why the fast path is safe.

## What I'd tell myself at the start

**The bottleneck is rarely where the interesting problem is.** I wanted a search problem, because search has a rich literature and satisfying algorithms. I had a "you're calling the wrong function" problem. Those feel very different to work on, and only one of them was real.

**Measure in layers.** "Loading is slow" is useless. Reading is 0.03s, parsing is 0.38s, opening is 2.57s — that's an answer. Each layer you separate either exonerates itself or confesses.

**Convenience functions do convenient things.** `find-file-noselect` isn't slow; it's thorough. It's the right call when a human will look at the buffer, and the wrong one when a parser will. The API doesn't distinguish, so you have to.

**Small data doesn't need big machinery.** Every instinct trained on "scale" says index it. At 0.66MB, `string-match-p` in a loop wins, and it wins on maintenance too.

**And research answers what you ask.** The material I found on inverted indexes was accurate and well-sourced, and following it would have left me with a correctly-implemented BM25 index sitting on top of a function that still took 20 seconds to open 52 files. The one sentence in that research that mattered wasn't an algorithm — it was the observation that my numbers didn't add up.

The code is on [GitHub](https://github.com/rukmaldias/K-Agenda) if you want to read it.
