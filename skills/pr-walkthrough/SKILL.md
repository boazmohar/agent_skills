---
name: pr-walkthrough
description: How to describe pull requests for a reviewer -- a PDF walkthrough of several open PRs (background, the issue or feature, decisions, open questions, figures), and the GitHub PR description itself. Use when asked to write or rewrite PR descriptions, or to make a report that walks a reviewer through PRs.
user-invocable: true
---

# PR Walkthrough

A reviewer reads a PR to decide something. Every description, and every section of a
walkthrough report, exists to put that decision in front of them with just enough context
to make it. Follow `deslop` for the prose and `figure-formatting` for the figures.

## The unit: one PR, five parts

Write each PR in the same order, short (a walkthrough section fits on one page):

1. **Meta line** -- base branch, files and +/- lines, link. If a large part of the diff is
   data or generated files, say how much ("74,217 lines; 61,579 of them measurements").
2. **Background** -- two or three sentences: what exists today and why it is not enough.
   Use the reader's terms; define any new one once, on the background page (below).
3. **The issue or the feature** -- what the PR changes, in plain words. Show the new
   public API as a short code block if there is one. A table when the PR does the same
   thing several times (one row per language, per format).
4. **Decisions** -- the choices the PR made that someone could have made differently.
   Mark each one: decided by the owner, or "my choice, not confirmed".
5. **What we want from the review** -- the ask, as numbered decisions and open questions.
   "Approve the API" is a decision; "should hot functions get a direct path?" is a
   question. Put this in a highlighted box so it is findable at a glance.

Leave history out: "merged X in", "rebased", "split out of", old hashes, dated update notes.
At most one line at the end ("The diff includes #35. Review #35 first.").

## A GitHub PR description

Same order, shorter, in Markdown:

```markdown
<What the PR changes and why: 1-3 sentences.>

**To decide**
1. **<Decision>:** <the choice made and the alternative>.
2. ...

**What it holds**
- **<Part>:** <what it does; one measured number if it matters>.

**Checks:** <what ran, on which machines, what passed; what did not run and why>.
```

When a PR changes, edit its description in place to the new state. Do not prepend an
update paragraph. Aim for the shortest body that lets the reviewer decide.

## A walkthrough report of several PRs

### Gather

- `gh pr list`, then for each PR `gh pr view N --json title,body,baseRefName,headRefName,additions,deletions,changedFiles,commits`.
- Read the diff of the files a reviewer approves (API documents, decision logs, specs), not
  every line. Read the repository's decision log for what is already decided.
- Find which PR sits on which: a PR's base is another PR's branch in a stack. Note PRs that
  are not for merge (experiments, tests) and say so wherever they appear.
- Collect every number a figure will show into `data.json`, each with its source (a
  measurement file, a log, `gh`). Never plot a number you did not read from a source.

### Layout

1. **Page 1: title, summary, table.** Group the PRs the way they will merge (by base
   branch). The summary is one short paragraph per group. The table has one row per PR:
   number (linked), what it does, what we want from the review.
2. **How the PRs stack** -- a map: one box per PR, an arrow from each PR to its base,
   colour by kind (fix, documents, module, tests, not-for-merge) with a legend. Give it
   its own page; on page 1 it pushes the table onto a second page.
3. **Background in one page** -- the system in a paragraph, then a short glossary of every
   term the PR sections use.
4. **One section per PR** (the five parts above), each starting a new page, in review
   order: what merges first comes first. A figure where the PR has numbers (a speed-up,
   a comparison, a coverage count) or a mechanism worth drawing (what one call goes
   through). Number figures through the report; captions say what is measured, where,
   and how many repeats.
5. **Opinion** -- last, labelled as judgement: merge order, risks, what the numbers do and
   do not mean. Facts stay in the sections; opinion stays here.
6. **Sources** -- one line: where the descriptions, measurements and figures came from.

### Build

- Folder `development/<date>_<topic>/`: `data.json`, `figures.py` (reads only
  `data.json`), `figures/` (svg, png, pdf at 600 dpi), `report.html`, the PDF.
- Write the report as HTML with print CSS (`@page { size: Letter; margin: 0.6in 0.75in; }`,
  Arial, about 9.5 pt; `h2.pr { page-break-before: always; }`;
  `figure { page-break-inside: avoid; }`). Embed figures as SVG and size them with
  `style="width:NN%"`. A two-column `display:flex` block puts a figure beside the text.
- Render with headless Chrome:
  `"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --disable-gpu --no-pdf-header-footer --print-to-pdf="$PWD/<name>.pdf" "file://$PWD/report.html"`
  Wrap it in `timeout 90` and delete the old PDF first, so a stale Chrome cannot leave an
  old file that looks new.
- When the HTML is long, keep the parts in `parts/*.html` and join them; edit by anchors
  that exist (`assert anchor in text`). A replace with an empty search string inserts the
  replacement between every character.

### Check, every render

- Page count, and the first line of each page: `pymupdf` (`uv run --with pymupdf`),
  `page.get_text()`. An unexpected count means overflow or a broken edit.
- Look at every page as an image (`page.get_pixmap(dpi=60)`): no table split across
  pages, no figure pushed onto a page of its own by accident, no text over a figure.
- Look at every figure on its own: no clipped or overlapping labels, text 8 pt or larger.
- Every number in the text matches `data.json`; every PR number and link is right.

### Merging two reports into one

Take every PR from both; where both cover one PR, keep the newer facts. Copy the figures
with a prefix per source (`a_fig2...`, `b_fig2...`) so names do not collide, renumber the
figures through the merged report, redraw the stack map for all PRs, and rewrite the
summary and table for the whole set. Delete the old PDFs only when asked; keep their
sources.

## Deliver

Open the PDF for the user (`open <file>` on macOS) and give its path. In the message, say
what the report covers and anything it could not check (a machine not run, a number
without a source).
