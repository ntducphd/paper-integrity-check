# paper-integrity-check

A local pre-check for a researcher's own manuscript portfolio: self-plagiarism / text
recycling across your own papers, over-close paraphrase of a source you've cited, and a
formalized AI-writing-tell scan.

**New to this tool, or new to the command line?** Start with
[`GETTING_STARTED.md`](GETTING_STARTED.md) instead — a step-by-step walkthrough with no
assumed prior experience, useful whether you're checking your own manuscript, a
collaborator's, or a student's. This README is the fuller technical reference.

**This is not a Turnitin/iThenticate replacement.** Those tools work because their owners
pay publishers for access to a proprietary database of billions of web pages and journal
articles — nothing here has access to that, and it never will without that kind of
licensing deal. What this script *does* check is fully derivable from files already on
your machine, and is specifically valuable if you run several parallel manuscripts on
related topics at once (a very real risk: it's easy to accidentally reuse a paragraph of
your own methods/intro text across two papers written months apart):

1. **Self-plagiarism / text reuse** — does a manuscript share long passages with any other
   manuscript, or already-published paper, anywhere under `--root` (optionally also any
   extra archive you point at with `--extra-dir`)?
2. **Citation-paraphrase** — does a manuscript share long passages with the full text of a
   source it cites, when that source's PDF sits locally in a project's `literature/`
   folder (i.e., quoting a cited paper too closely instead of paraphrasing)?
3. **AI-writing tells** — known AI-writing lexical/structural patterns (stock phrases like
   "delve"/"leverage"/"underscore", the `, not X` construction, em-dash/colon clustering).

Stage 1 by itself does **not** check against the open web or any publisher's database of
other authors' work you have not cited and do not have a local copy of — Stage 3 below
narrows that gap with an exhaustive scan of every sufficiently distinctive sentence in the
manuscript, but neither replaces your institution's real Turnitin/iThenticate access. Run
this alongside it, not instead of it.

## Three-tier design: lexical scan, agent semantic review, open-web cross-check

This repo has three layers, deliberately kept separate:

- **Stage 1** (`integrity_check.py`, this file's main subject) — word-8-gram lexical
  matching. Fast, deterministic, no AI involved, runs anywhere Python is installed. It can
  tell you *that* two passages share words; it cannot judge whether that overlap is a real
  problem or benign (an established phrase, a disclosed self-citation, shared reference-list
  text).
- **Stage 2** (`AGENT_REVIEW.md`) — a documented procedure for a Claude Code session to
  dispatch Stage 1's flagged passages to `feynman-review` (judges each flagged passage as a
  peer reviewer would: genuine concern or benign reuse) and `nature-ref-verifier` (checks
  citation metadata accuracy against CrossRef/PubMed — a different integrity dimension Stage
  1 can't check at all). This only works inside Claude Code — there's no API for a bare
  script to invoke these agents — so it's a runbook, not a second script. See
  `AGENT_REVIEW.md` for the full procedure and why these two agents specifically.
- **Stage 3** (`WEB_CHECK.md`) — the actual open-web check: `integrity_check.py candidates
  <manuscript>` picks EVERY sufficiently distinctive, "googleable" sentence in the manuscript
  by default (fully offline, no AI or network involved — not a hand-picked sample); a Claude
  Code session then searches each one with its `WebSearch` tool, typically via parallel
  background subagents at full-manuscript scale — literally automating "copy the sentence,
  paste it into Google," exhaustively rather than leaving it to chance. Scopus and Web of
  Science are explicitly out
  of scope (both require a paid institutional API key neither of us has); see `WEB_CHECK.md`
  for what this does and doesn't cover, including a privacy note about sentences leaving your
  machine.

`check` always writes both a `.md` report and a machine-readable `.json` sibling — the JSON
is Stage 2's input format (`self_plagiarism.prose_matches` / `citation_paraphrase.prose_matches`,
with bibliography-entry matches already excluded since there's nothing semantic to ask an
agent about a shared reference-list line).

## Install

Requires Python 3.9+ and two packages:

```
pip install pymupdf python-docx
```

## Usage

```
# Point at the folder that holds your manuscripts. Builds/refreshes a local cache
# (extracted text + word-8-gram shingles) so repeat runs are fast.
python integrity_check.py build-cache --root /path/to/your/papers

# Optionally also scan a separate published-papers archive, tagged as your own prior
# published work rather than a "different manuscript." Off by default -- if it's on a
# cloud-synced drive (Google Drive, OneDrive, ...), the FIRST scan can be very slow
# (each file may trigger a network fetch); cached after that.
python integrity_check.py build-cache --root /path/to/your/papers \
    --extra-dir /path/to/your/published-archive

# Exclude any extra folder name from scanning (repeatable) -- useful for e.g. a personal
# website repo sitting inside the same tree.
python integrity_check.py build-cache --root /path/to/your/papers --exclude my-website-repo

# Check one manuscript. Rebuilds/refreshes the cache first using the same --root/--extra-dir.
python integrity_check.py check /path/to/manuscript.docx --root /path/to/your/papers

# Tuning knobs (defaults shown):
python integrity_check.py check <path> --root <root> --min-words 20 --max-doc-freq 3

# Stage 3: pick EVERY sufficiently distinctive sentence in the manuscript for an open-web
# search (add --top N only if you want a capped subset instead) -- no corpus/cache needed,
# fully offline, works even without a Claude Code session (you can paste the output into
# Google yourself). See WEB_CHECK.md for the full procedure.
python integrity_check.py candidates /path/to/manuscript.docx

# Stage 3, once a Claude Code session has compiled WebSearch verdicts into a results JSON
# (see WEB_CHECK.md): render it as a visual HTML dashboard, same design as `check`'s report.
python integrity_check.py stage3-report /path/to/results.json
```

Output is written to `reports/<stem>_integrity_report_<date>.md` (human-readable),
`reports/<stem>_integrity_report_<date>.json` (Stage 2 input, see above), and
`reports/<stem>_integrity_report_<date>.html` (a single self-contained file — dashboard,
ranked source table, and the full document with matches highlighted in place; open it in
any browser, light/dark theme aware).

## How this compares to Turnitin/iThenticate

Ranking this against Turnitin only makes sense dimension by dimension — one tool doesn't
strictly dominate the other:

| | This tool | Turnitin / iThenticate |
|---|---|---|
| Detection scope | Only files under `--root`/`--extra-dir` on your machine | Licensed database of billions of web pages + journal articles |
| Report transparency | Open source — every match rule is a readable `if` statement | Proprietary similarity algorithm |
| Cost / speed | Free, runs locally, seconds per manuscript | Institutional license, queued web submission |
| Bibliography-entry noise | Auto-detected and demoted to a collapsed section | Also flags shared reference-list text as "similarity" |
| Self-plagiarism across your own unpublished drafts | Its actual design target — compares against your other in-progress manuscripts, which are never in Turnitin's database until you submit them | Cannot see your own unsubmitted drafts at all |

**The gap that matters most**: this tool cannot tell you whether your wording overlaps
with a paper you never cited and don't have a local copy of — that requires the licensed
database neither of us has access to. Stage 3 (`WEB_CHECK.md`) narrows this by actually
searching the open web for every sufficiently distinctive sentence in the manuscript (not a
hand-picked sample), but it is still bounded by one search backend's index, not the same
exhaustive licensed-database scan a real submission runs. Run this *before* a real
Turnitin/iThenticate submission, as a fast, free, zero-setup pass for the specific risks
it's built for (text reuse across your own parallel manuscripts, over-close paraphrase of a
cited source, and now a spot-check against the open web), not as a replacement for it.

**Where the HTML report earns its keep**: `.md` output is a list of matched snippets;
Turnitin's PDF report shows your actual paper with color-coded highlights so you can judge
each match in its real sentence, in seconds. The HTML report here does the same — real
casing and punctuation, not a lowercased token dump — plus states the "Local Overlap
Index" is explicitly *not* a Turnitin similarity score in the report itself, so it can't be
misquoted out of context to a committee or editor. `stage3-report` gives the open-web
check the same treatment: a dashboard of scanned/clean/flagged counts plus every searched
sentence listed with its verdict, so "was the whole manuscript actually covered" is
visually checkable, not just claimed in prose.

## How it works

- Extracts plain text from `.docx` (python-docx, including table cells), `.pdf` (pymupdf),
  and `.md` files under `--root` (and any `--extra-dir`).
- Builds word-level 8-gram "shingles" and looks for **contiguous runs** of shingles shared
  between the target manuscript and any other corpus document — not just isolated matching
  phrases.
- A shingle shared with more than `--max-doc-freq` (default 3) other documents is treated
  as boilerplate/common phrasing and not flagged — this is what keeps standard methods
  language ("analysis of variance was performed using...") from drowning out real matches.
- A matched passage shorter than `--min-words` (default 20) is not flagged as noise.
- The target manuscript's own same-folder `.docx`/`.pdf`/`.md`/`README.md` siblings (its
  own other export formats, its own project documentation) are excluded automatically —
  comparing a docx against its own PDF export is not a plagiarism finding.
- Matches that look like a shared bibliography/citation-string entry (contain a DOI, a
  URL, or 2+ four-digit years close together) are collapsed into a separate,
  de-emphasized section — two papers citing the same source will naturally share that
  reference-list text, and that's not a prose-reuse concern.
- Exact-duplicate files (e.g. the same manuscript copied into a `submission_package/`
  folder) are collapsed to one canonical entry so the report doesn't repeat the same
  match 3 times.

## Validated against a known-good baseline

First real run, against a manuscript from an active multi-paper portfolio, surfaced two
bugs in the tool itself before it could be trusted:

- **False positive**: the top "self-plagiarism" hit was the manuscript's own `.pdf`/`.md`
  export in the same folder — the same document, not a different one. Fixed via the
  same-folder-sibling exclusion described above.
- **Noise**: nearly every "citation-paraphrase" match was a shared bibliography entry
  (author/year/journal/DOI text), not real prose. Fixed via the bibliography-entry
  heuristic, which now auto-collapses most of these; the handful it doesn't catch are
  still visible in the report but easy to recognize by eye.

After both fixes, a full portfolio scan found **zero genuine copied original prose** for
that manuscript — matching a prior independent manual audit of the same paper. That match
against an already-known result is what makes the tool trustworthy, not just that it runs
without crashing; don't trust a new detection tool's first report at face value — validate
it against a case where the ground truth is already known.

**Stage 2 validated the same way, and improved Stage 1 in the process.** Running the
`AGENT_REVIEW.md` procedure for real against the same manuscript's 15 then-flagged prose
passages, `feynman-review` judged all 15 BENIGN or BOILERPLATE — but also diagnosed *why* 9
of them slipped past the Stage-1 bibliography heuristic: they were citation strings sitting
in an informal context (a markdown literature-notes file, a PDF-cache extraction) with only
one year and no DOI/URL in the matched window, so the original heuristic missed them. That
diagnostic was concrete enough to act on immediately: added an author-initial-list detector
(a run of single-character tokens — "t", "v", "g", "p" — from tokenized "T.", "V.", "G.",
"P." initials) to `looks_like_bibliography()`. Re-running Stage 1 dropped the flagged-prose
count from 15 to 9; the 9 remaining are exactly the cases that genuinely need semantic
judgment (self-references to the manuscript's own README, one mandatory Data Availability
sentence, one bare paper-title fragment with no lexical citation markers at all) — a lexical
heuristic alone cannot resolve those, and isn't meant to. Separately, `nature-ref-verifier`
checked the 3 newest (single-sourced, unverified) references against CrossRef/PubMed and
found all 3 clean, plus two minor informational notes (a reference missing volume/pages; a
cited paper with a published Author Correction) — a citation-accuracy dimension Stage 1
cannot check at all, by design.

**Stage 3 validated at full-manuscript scale against a real iThenticate report already in
hand, and it caught a real bug doing so.** Ran `WEB_CHECK.md`'s procedure — exhaustively,
not a sample — against a manuscript under review at a journal that had already returned an
iThenticate Similarity Report (28% overall similarity, 0 Integrity Flags, no single source
over 2%). All 146 candidate sentences generated at the time were searched (6 parallel
subagents, 25/batch); 145 came back clean and 1 flagged "HIT" turned out, on inspection, to
be this tool's own bug — a reference-list entry's title, separated from its author/year
header by the sentence-splitter, offered up as if it were manuscript prose. Fixed (see
`WEB_CHECK.md`), which also dropped the candidate count from 146 to 82 by correctly
excluding reference-list fragments. True result: 82/82 genuine body-prose candidates clean
— agreeing with iThenticate's own "0 Integrity Flags" verdict for the same file, from an
independent method. This is the same "validate against a known-good baseline, don't trust a
first run at face value" discipline as Stage 1/2 above, now proven at the scale it needs to
work at.

## Known limitations

- No web/publisher-database check in Stage 1 itself — see the scope note above. Stage 3
  narrows this with an exhaustive scan of every sufficiently distinctive sentence (see
  `WEB_CHECK.md`), but it is still bounded by one search backend's index, not the same
  licensed-database scan Turnitin runs; this remains the fundamental gap relative to real
  Turnitin/iThenticate, not a bug.
- Stage 3 has no Scopus/Web of Science integration — both require a paid institutional API
  key this tool does not assume you have (see `WEB_CHECK.md`).
- Stage 2 requires a Claude Code session with the `feynman-review`/`nature-ref-verifier`
  subagents configured — it will not run from a plain terminal with just Python installed.
  Stage 1 alone remains fully standalone.
- The bibliography-entry heuristic (DOI/URL/2+ years/author-initial-list pattern) doesn't
  catch every citation-string match — some still appear in the "real prose overlap"
  section even though they're clearly reference-list text on inspection (see the Stage 2
  validation above for a concrete example). Read the actual snippet, or run Stage 2, before
  assuming everything in that section is a genuine concern.
- AI-tell colon-clustering counts table cells too (docx table extraction concatenates
  cell text), so a table-heavy manuscript will show a higher raw count than a prose-only
  one — read this as a rough signal, not a precise metric.
- A single global cache (`cache/corpus_cache.json`) is used regardless of `--root` — if
  you switch between unrelated portfolios, `--rebuild` on the first run of each is safest.

## Files

- `integrity_check.py` — Stage 1, plus the `candidates` and `stage3-report` commands that
  feed and render Stage 3, the standalone tool (single file, `pymupdf` + `python-docx` the
  only deps).
- `AGENT_REVIEW.md` — Stage 2, the agent-based semantic review procedure (Claude Code only).
- `WEB_CHECK.md` — Stage 3, the open-web cross-check procedure (Claude Code only for the
  search step; `candidates` itself is standalone).
- `cache/` — the extracted-text + shingle cache. Gitignored; safe to delete, will rebuild.
- `reports/` — generated `.md`/`.json`/`.html` reports (Stage 1), `candidates` output, and
  any Stage 2/3 write-ups, one per run, dated. Gitignored — these quote passages from your
  own (possibly unpublished) manuscripts and should not be committed to a shared or public
  repo.
