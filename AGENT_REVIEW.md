# Stage 2: agent-based semantic review

`integrity_check.py` (Stage 1) is a standalone script: word-level n-gram matching, runnable
anywhere Python + `pymupdf`/`python-docx` are installed, no AI involved. That's deliberate — it's
fast, deterministic, and works outside any AI tool. But it is a *lexical* check: it can only find
overlap in the literal words used, and it cannot judge whether a flagged passage is actually a
problem (a genuine self-plagiarism risk) or benign (an established phrase, a disclosed
self-citation, shared field terminology).

Stage 2 closes part of that gap using Claude Code's specialized subagents — but only inside a
Claude Code session. There is no public API that lets a bare Python script invoke
`feynman-review` or `nature-ref-verifier` directly; these agents exist only as part of the Claude
Code agent runtime. So Stage 2 is a **documented procedure for a Claude Code session to follow**,
not a second script — this file is a runbook, not code.

## Why these two agents specifically

Of the available `feynman-*`/`nature-*` agents, two are actually purpose-matched to what Stage 1
flags — the others (`feynman-lit`, `nature-writing`, `feynman-draft`, etc.) are for
literature-search or drafting tasks, not for judging text-reuse severity:

- **`nature-ref-verifier`** — "multi-source cross-verification of reference lists (author,
  title, year, volume, pages, DOI) with structured conflict reports." This is a genuinely
  different integrity dimension from Stage 1's text-overlap check: Stage 1 can't tell you if a
  citation's metadata (year, volume, DOI) is *correct* — only that the manuscript and a source
  share text. Run this against the target manuscript's own reference list, independent of
  whichever passages Stage 1 flagged.
- **`feynman-review`** — "simulate an AI research peer review with likely objections, severity,
  and a concrete revision plan." Repurposed here as the semantic judge for Stage 1's *prose*
  matches (not the auto-collapsed bibliography-entry matches, which don't need semantic judgment):
  for each flagged passage, does a reviewer-level read conclude this is a real self-plagiarism/
  over-close-paraphrase concern, or legitimate reuse?

## Procedure

1. Run Stage 1 with `--json` output (already the default alongside the `.md` report):
   ```
   python integrity_check.py check <manuscript> --root <root> [--extra-dir ...] [--exclude ...]
   ```
   This writes `reports/<stem>_integrity_report_<date>.json` with `self_plagiarism.prose_matches`
   and `citation_paraphrase.prose_matches` (bibliography-entry matches are excluded already —
   nothing semantic to ask an agent about a shared reference-list line).

2. If either list is non-empty, dispatch to `feynman-review` with a prompt built from the JSON,
   e.g.:

   > "Here are N passages an automated text-overlap scan flagged in `<manuscript path>` as
   > shared with `<other document path>` (self-plagiarism candidate) / with a cited source's own
   > text (over-close-paraphrase candidate): `<snippets>`. For each, judge as a peer reviewer
   > would: is this a genuine academic-integrity concern, or legitimate reuse (established
   > terminology, a disclosed self-citation, standard methods phrasing)? Give a verdict and one-
   > sentence rationale per passage."

3. Separately, run `nature-ref-verifier` against the target manuscript's reference list to check
   citation metadata accuracy — independent of what Stage 1 flagged, since this catches a
   different error class (wrong year/volume/DOI, not text overlap).

4. Merge: append an "## Agent semantic review (Stage 2)" section to the Stage-1 Markdown report
   (or write a new file) summarizing both agents' findings, clearly labeled as a semantic
   judgment layer distinct from Stage 1's mechanical count.

## Validated 2026-07-25

Ran this procedure for real against the same sweetpotato-review manuscript used to validate
Stage 1 (see the main README's "Validated against a known-good baseline" section). Result and
any tool-level findings from that run are in this repo's git history / the corresponding
session's notes — this file documents the *procedure*, not a one-time result, so it stays valid
as new manuscripts are checked.

## Limitation

This step requires a Claude Code session (or equivalent agent runtime with these specific
subagents configured) — it will not run from a plain terminal with just Python installed. Stage 1
alone remains fully standalone and is the right choice if you just need a fast, no-AI-involved
pre-check.
