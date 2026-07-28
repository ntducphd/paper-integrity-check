# Getting started — a step-by-step guide for running this yourself

This guide assumes no prior Python or command-line experience. If you've never opened
a terminal before, that's fine — every command you need is written out below; you copy
it, paste it, press Enter.

## What this actually does, in plain terms

You give it a folder of manuscripts (yours or a student's) and one target manuscript to
check. It tells you three things about that target:

1. **Does it share long passages of text with any other manuscript in the same folder?**
   (a real risk when someone is writing several related papers at once, and accidentally
   reuses a paragraph of methods or introduction text across two of them)
2. **Does it quote a cited source too closely**, instead of paraphrasing — checked against
   the actual PDF of that source, if you have a local copy in a `literature/` folder?
3. **Does the writing show common AI-generated-text patterns** — stock phrases, overused
   sentence constructions, that kind of thing?

**What it is not**: a replacement for Turnitin, iThenticate, or your institution's real
plagiarism-checking service. Those work by comparing against a licensed database of
billions of web pages and journal articles that this tool has no access to and never
will. This tool only ever compares files that are already sitting on your own computer.
Think of it as a fast, free, first-pass check for a specific and real risk — reusing your
own text across parallel manuscripts — not a substitute for your institution's official
screening.

## Step 1 — Check whether Python is already installed

Open a terminal:
- **Windows**: press the Windows key, type `PowerShell`, press Enter.
- **Mac**: open Spotlight (Cmd+Space), type `Terminal`, press Enter.

Type this and press Enter:

```
python3 --version
```

- If you see something like `Python 3.11.4` (any version **3.9 or higher**), you're set —
  skip to Step 2.
- If you see an error ("command not found" or similar), you need to install Python
  first: go to [python.org/downloads](https://www.python.org/downloads/), download the
  installer for your operating system, run it. **On Windows, tick the box that says "Add
  Python to PATH"** during install — this is the single most common thing people miss.
  Close and reopen the terminal after installing, then re-run the command above to
  confirm.

## Step 2 — Get the tool onto your computer

You don't need to know git for this. On the GitHub page for this tool:

1. Click the green **`Code`** button.
2. Click **`Download ZIP`**.
3. Find the downloaded file (usually in your Downloads folder) and unzip it — right-click
   it and choose "Extract All" (Windows) or double-click it (Mac).
4. You now have a folder called `paper-integrity-check-main` (or similar). You can rename
   it to just `paper-integrity-check` and move it somewhere convenient, like your Desktop.

*(If you're already comfortable with git: `git clone https://github.com/ntducphd/paper-integrity-check.git` does the same thing in one step.)*

## Step 3 — Install the two things this tool needs

In your terminal, navigate into the folder you just unzipped. If you put it on your
Desktop, that's:

```
cd Desktop/paper-integrity-check
```

(On Windows, if that doesn't work, try `cd Desktop\paper-integrity-check`.)

Then install the two required packages:

```
pip install pymupdf python-docx
```

Wait for it to finish (a few seconds to a minute). You should see a line ending in
"Successfully installed..." with no red error text above it. If `pip` isn't recognized,
try `pip3 install pymupdf python-docx` instead.

## Step 4 — Run it

Two commands, always in this order.

**4a. Point it at the folder holding the manuscripts you want to compare against each
other** (this can be one student's project folder, your whole lab's shared drive, or
anything in between — the bigger this folder, the more thorough the self-plagiarism
check, but the longer the first run takes):

```
python3 integrity_check.py build-cache --root "/path/to/that/folder"
```

Replace `/path/to/that/folder` with the real path — on Windows this often looks like
`"C:\Users\YourName\Documents\Lab Papers"` (keep the quotes if the path has spaces in
it). This step reads every `.docx`, `.pdf`, and `.md` file it finds and builds a local
index. The first run can take a few minutes for a large folder; every run after that is
much faster, since it only re-reads files that changed.

**4b. Check one specific manuscript**:

```
python3 integrity_check.py check "/path/to/the/manuscript.docx" --root "/path/to/that/folder"
```

This writes three versions of the same report to the `reports/` folder inside the tool,
all named `<manuscript-name>_integrity_report_<today's date>` with a different ending:

- **`.html`** — open this one first. Double-click it (or drag it into a browser tab) and
  you'll see a dashboard, a table of matched sources, and your full manuscript with the
  flagged passages highlighted right where they occur — closest to what a Turnitin report
  looks like.
- `.md` — the same information as plain text, if you prefer that or want to paste it
  somewhere.
- `.json` — only relevant if the report gets handed to a Claude Code session for the
  optional Stage 2 semantic review; you can ignore this one.

### A concrete worked example

Say you keep all your manuscripts in `Documents/My Papers`, and you want to check
`Documents/My Papers/Paper3/draft.docx` specifically:

```
python3 integrity_check.py build-cache --root "Documents/My Papers"
python3 integrity_check.py check "Documents/My Papers/Paper3/draft.docx" --root "Documents/My Papers"
```

That's the whole workflow. Run `build-cache` again (no `--rebuild` needed) any time
you've added or edited manuscripts, before running `check` on something new.

## Step 5 — Reading the report

The report has three sections, matching the three checks described above. For each
flagged passage, read the actual quoted text:

- **If it reads like a sentence of argument or description** (a real idea, in prose)
  — that's worth a closer look. Is it a legitimate, disclosed self-citation of your own
  earlier method description? Or genuine accidental reuse that should be reworded?
- **If it reads like a string of author names, a year, a journal name, and a DOI** — that's
  a shared reference-list entry. Two papers citing the same source will always share that
  text; it is expected and not a concern. The tool tries to auto-detect and collapse these
  into a separate, de-emphasized section, but it doesn't catch every case — if something
  in the "real prose overlap" section is obviously just a citation string, that's a known,
  documented limitation of the automatic detection, not a real finding.

The AI-writing-tell numbers at the bottom are a rough signal, not a verdict — a
technical manuscript with many tables will naturally show a higher raw count than a
narrative one, since table cells get counted too.

## Optional Step 6 — a manual open-web spot-check

Everything above only compares files on your own computer. If you want a quick sanity
check against the open web (the same thing as copying a sentence and pasting it into
Google, just done for you for the sentences most worth checking), run:

```
python3 integrity_check.py candidates "/path/to/the/manuscript.docx" --top 20
```

This prints a numbered list of the manuscript's longest, most distinctive sentences —
no internet connection needed for this step itself. `--top 20` caps it to a manageable
number to check by hand; drop that flag to list every qualifying sentence in the whole
manuscript instead, if you want to be thorough. Pick a few and paste each one (with
quotation marks around it) into Google. This part is optional and manual; there's a more
automated version of this same idea (`WEB_CHECK.md`) that searches and reports on the
entire list automatically, but it needs Claude Code, not just
Python.

## If something goes wrong

- **"ModuleNotFoundError: No module named 'fitz'" or similar** — Step 3 didn't complete.
  Re-run `pip install pymupdf python-docx` and check for errors.
- **The `check` command says a folder or file "not found"** — double-check the path is
  typed exactly right, including capitalization, and that you've kept the quotes around
  any path containing spaces.
- **It's taking a very long time on the first `build-cache` run** — normal for a large
  folder, especially if any of the files live on a cloud-synced drive (Google Drive,
  OneDrive) rather than your local disk, since each file may need to be downloaded first.
  Subsequent runs are much faster.
- **Anything else** — the main `README.md` in this same folder has more technical detail,
  including the tuning options (`--min-words`, `--max-doc-freq`) and what each one does.

## Questions

This tool was built and is maintained by Nguyen Trung Duc. If anything in this guide
doesn't match what you see on your screen, or a report result looks wrong, reach out
directly rather than guessing.
