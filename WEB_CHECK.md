# Stage 3: open-web / academic-database cross-check

Stage 1 (`integrity_check.py`) only ever compares files already sitting on your own
machine. That is its most important limitation, documented throughout this repo: it cannot
tell you whether a sentence overlaps with a paper you never cited and do not have a local
copy of. If a passage really were copied from someone else's published work, the fastest
real-world way to catch it is exactly what you already do by hand: copy the sentence, paste
it into Google (or a database search box), see what comes back. Stage 3 formalizes that
instinct and runs it exhaustively — by default, every sentence in the manuscript long enough
to be worth searching, not a hand-picked sample of whichever ones you happen to get
suspicious about.

Like Stage 2 (`AGENT_REVIEW.md`), this is a **documented procedure for a Claude Code
session**, not a second standalone script — a bare Python process has no legal, free way to
query Google at scale (no ToS-compliant free API for that), but a Claude Code session already
has a `WebSearch` tool for exactly this kind of query.

## What this does and does not cover

- **Covers**: the open web — indexed copies of other authors' papers, preprints, theses,
  course notes, blog posts, anywhere a sentence could have been lifted from or into.
- **Does not cover**: Scopus or Web of Science. Both gate their search APIs behind a paid
  institutional subscription (Elsevier / Clarivate) that this tool does not assume you have.
  If VNUA's library later provides you with API credentials for either, that would plug in
  as a fourth data source alongside the two below — ask if you want that built once you have
  the key; it is not safe to ship untested code written against an API neither of us can
  actually call.
- **Two free, no-key, ToS-compliant sources already wired in as an optional automatic
  pre-filter** (see "Optional: automatic pre-filter" below): Semantic Scholar and CrossRef
  search paper titles/abstracts, not full body text — weaker than a real web search, but
  free, scriptable, and worth running before spending WebSearch calls.

## Privacy note

Every method here sends fragments of your manuscript's actual sentences to a third-party
service over the network (Google, via the WebSearch tool; Semantic Scholar; CrossRef). That
is a different privacy posture than Stage 1, which is 100% local. It is not a new category of
exposure — a real Turnitin/iThenticate submission uploads the *entire* manuscript to a
third party — but worth knowing before running this on something under embargo or pre-any-
disclosure. Only the sentences you choose to search leave your machine, not the whole file.

## Procedure

1. Generate candidate sentences — deterministic, offline, no corpus needed:
   ```
   python integrity_check.py candidates <manuscript>
   ```
   With no `--top`, this returns **every** qualifying sentence (12-40 words, not a
   citation string) — a full-manuscript scan, not a sample. A typical ~6000-word review
   produces on the order of 100-160 candidates. Writes
   `reports/<stem>_candidates_<date>.json` plus a printed numbered list. Pass `--top N` only
   if you deliberately want a quick partial check instead.

2. For each candidate sentence, in a Claude Code session, run `WebSearch` with the exact
   sentence text (optionally trimmed to its most distinctive clause if it's long) as the
   query, quoted for an exact-phrase search where the search backend supports it. Note:

   - **A hit that traces back to the manuscript's own already-cited source** is expected and
     fine — that's why the sentence exists.
   - **A hit against a paper NOT in the manuscript's reference list, with near-identical
     wording**, is the actual finding this stage exists to catch.
   - **No hits** for a distinctive, specific sentence is a positive signal (though not proof
     of originality — a true negative here just means nothing indexed by that search backend
     matches, which is still narrower than Turnitin's licensed database).

   **At full-scan scale (100+ candidates), don't search them one at a time in the main
   session** — that's slow and floods the session's own context with ~100+ raw search-result
   blocks. Instead: split the candidate JSON into batches of ~25, write each batch to its own
   file, and dispatch one background subagent per batch (the `Agent` tool, `general-purpose`
   subagent type) with instructions to read its batch file, WebSearch every sentence in it,
   and report back a compact `N | CLEAN` / `N | HIT: ...` line per sentence — full detail only
   for genuine hits. Running batches in parallel keeps wall-clock time reasonable and keeps
   the main session's context to just the compact per-batch summaries, not ~100+ raw results.

3. Compile the batch summaries into a single verdict list and render it visually:
   ```
   python integrity_check.py stage3-report <results.json>
   ```
   where `results.json` is `{"target": "<manuscript path>", "pipeline_note": "<optional
   free-text note, e.g. about a false positive investigated and resolved>", "results":
   [{"sentence": str, "n_words": int, "verdict": "clean"|"hit", "note": str}, ...]}` — one
   entry per searched sentence, assembled by hand or by a small script from the batch
   summaries. Writes `reports/<stem>_stage3_web_check_<date>.html`: same dashboard design as
   Stage 1's report (scanned/clean/flagged counts, a flagged-for-review table, and the full
   per-sentence list) so the whole scan is visually auditable, not just asserted in prose.
   Also write a plain `reports/<stem>_stage3_web_check_<date>.md` alongside it if a text
   version is useful (gitignored, same as Stage 1/2 outputs — see `.gitignore`).

## Optional: automatic pre-filter (Semantic Scholar + CrossRef)

Both are free, public, require no API key, and are safe to call from a plain script — they
search paper **titles and abstracts**, not full body text, so they will only catch a
candidate sentence if it (or something very close to it) also appears in some paper's
title/abstract. That is strictly weaker coverage than a real web search, but it costs nothing
and can be run before spending WebSearch calls on candidates that already show a clear hit.
This is intentionally left as a documented manual step rather than a `check --web-check`
flag: wiring live network calls into the main script would change its "runs anywhere, fully
offline, deterministic" character by default, which is worth keeping opt-in rather than
silently on.

## Validated 2026-07-28, at full-manuscript scale

Ran this procedure for real, exhaustively, against `2026_paper_4_Ms733_Beyond_Sweetness_VJAS_eng`
(the "Beyond Sweetness" sweet-corn review, under review at VJAS), whose actual iThenticate
Similarity Report (28% overall similarity, 0 Integrity Flags, no single source over 2%) was
already available locally for comparison — a genuine known-ground-truth case, not a blind
first run. All 146 candidate sentences generated at the time were dispatched across 6
parallel background subagents (25/batch); every batch reported back. Result: 145/146 clean;
the 1 flagged "HIT" was investigated and turned out to be a false positive from this tool's
own candidate-extraction pipeline (a reference-list entry's title, split apart from its
author/year header, offered up as if it were manuscript prose) — not a real integrity issue.
Fixed (see `select_candidates()` in `integrity_check.py`: bibliography-likeness is now
checked at the whole-paragraph level before sentence-splitting), which also dropped the
candidate count from 146 to 82 by correctly excluding reference-list-derived fragments that
were never real prose to begin with. True result: **82/82 genuine body-prose candidates
clean** — full detail in `reports/733-+Article+text_stage3_web_check_2026-07-28.md` (not
committed, quotes real manuscript text — see `.gitignore`). This is the same "validate
against a known-good baseline, and don't trust a first run at face value" discipline as
Stage 1/2 (see the main README), applied here at full scale rather than a small sample —
and it genuinely caught a real bug in this exact run, which is the point of validating this
way instead of just documenting the procedure and assuming it works.

## Limitation

Like Stage 2, this requires a Claude Code session (or an equivalent agent runtime with a web
search tool) — it will not run from a plain terminal with just Python installed. The
`candidates` command is the exception: fully standalone, no AI or network involved, so it's
useful even without a Claude Code session (e.g., to prepare a list you paste into Google
yourself).
