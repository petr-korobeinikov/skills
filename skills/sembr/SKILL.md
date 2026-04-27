---
name: sembr
description: >-
  Reformats prose using Semantic Line Breaks (https://sembr.org/) — strictly,
  no compromises. Never alters wording or rendered output.
when_to_use: >-
  Use in two modes. (1) Explicit reformat — when the user asks to apply sembr
  to existing text (e.g. "оформи по sembr", "перепиши по sembr",
  "format with sembr", "apply semantic line breaks"). (2) Default-on-edit —
  when writing or editing prose in `.md`, `.markdown`, or `.rst` files:
  apply sembr to new prose, match the existing paragraph's style for
  in-place edits, never reformat untouched paragraphs.
---

# sembr

Reformat prose so its physical line structure mirrors its semantic structure,
per the Semantic Line Breaks specification at https://sembr.org/.
Rules below are mandatory —
never skip them silently and never relax a MUST rule for convenience.

## Modes

### Explicit reformat

The user asked to reformat existing text.
Apply MUST/SHOULD/MAY rules to the targeted prose
(snippet, file, or section).
Reformat fully within the target;
do not touch surrounding paragraphs.

### Default-on-edit

Triggered automatically when adding or modifying prose
in `.md`, `.markdown`, or `.rst` files —
without an explicit user request.
The guiding principle:
**only the part being edited gets sembr-shaped;
everything else stays byte-identical.**

- **New prose** (new file, new paragraph, freshly inserted block):
  write in sembr from the start.
- **Edits inside an existing paragraph:**
  match the paragraph's existing line-break style.
  If it already follows sembr
  (multiple short lines, breaks after sentence terminators or clauses) —
  keep sembr in your edit.
  If it is one long line, hard-wrapped at a fixed column,
  or follows another visible convention —
  match that convention.
  If the paragraph is too short to tell —
  default to sembr.
- **Untouched paragraphs:**
  never reformat.
  Editing one paragraph
  must not change line breaks, spacing, or wording in any neighbour.
  Diffs must stay localised to the edited region.
- **Opt-out:**
  if the user says
  «не применяй sembr здесь» / «don't apply sembr here»
  (or sets a project-level preference) —
  follow the file's existing convention instead.

### When NOT to use

Skip both modes for plain `.txt`,
GFM with `<br>` line-break behavior,
email,
chat (Slack, Telegram, Discord),
or any context where single newlines render as visible breaks.
Sembr there would change the rendered output and violate rule 2.

## Inputs

Identify the target before editing:
- A snippet pasted in the conversation → return the reformatted text.
- A file path or section in a file → edit the file in place.
- Markdown / HTML documents → preserve all rendering
  (headings, lists, code blocks, links, inline markup).

Sembr applies only to prose paragraphs
and prose-bearing list items and blockquotes.
**No-touch regions** —
never modify whitespace,
line breaks,
or any content inside:
- fenced code blocks (` ``` `) and indented code blocks;
- inline code spans (`` ` ``);
- HTML islands;
- Markdown tables;
- YAML / TOML frontmatter;
- reST line blocks (lines starting with `|`);
- reST directives (`.. name::` and their indented content);
- reST literal blocks (after a trailing `::`, until dedent);
- reST grid and simple tables;
- reST doctest blocks (lines starting with `>>>`).

If the target is ambiguous, ask before editing.

## Rules

### MUST (mandatory, no exceptions)

1. **Break after every sentence terminator** — `.`, `!`, `?`.
   A `.` is a terminator only when it ends a sentence —
   not inside abbreviations (`e.g.`, `Mr.`, `U.S.`, `etc.`),
   version or dotted identifiers (`1.22.4`, `node.js`, `co.uk`),
   or ellipses (`…`, `...`).
   When in doubt,
   check the next character:
   if it starts a new sentence
   (capital letter after a space, or end of paragraph),
   break;
   otherwise do not.
2. **Never alter the rendered output.**
   The visible result after rendering (Markdown, HTML, etc.)
   must be byte-identical to the input.
   The only allowed change is replacing an intra-paragraph space with
   a single newline
   (or vice versa).
   Forbidden:
   - adding, removing, or changing any non-whitespace character;
   - inserting or removing blank lines;
   - touching anything inside a no-touch region (see `Inputs`);
   - normalizing trailing whitespace, tabs, or blank-line counts.

   Verification:
   diff input vs. output —
   only space ↔ newline swaps within prose paragraphs may appear.
3. **Never break inside a hyphenated word**
   (e.g. `state-of-the-art` stays on one line).

### SHOULD (apply unless they would violate a MUST rule or change meaning)

4. **Break after each independent clause**
   punctuated by `,`, `;`, `:`, or `—` (em dash).
   Heuristic: an independent clause has a subject and a finite verb,
   and could stand alone as a sentence.
   «I went to the store, and I bought eggs» —
   both sides are independent;
   break after the comma.
   «I went to the store to buy eggs» —
   «to buy eggs» is dependent;
   no break.
   When the structure is genuinely ambiguous,
   leave the clause unbroken and surface the case in the report
   (see `Reporting`).
5. **Break before an enumerated or itemized list.**
6. **Keep lines under 80 characters.**
   When a line exceeds 80,
   break at the latest semantic boundary that satisfies the limit.

### MAY (use to improve readability or to satisfy MUST/SHOULD)

7. Break after a dependent clause for clarity or length.
8. Break between list items to group related entries.
9. Break before or after a hyperlink or inline markup.
10. Allow a line to exceed 80 characters
    when the overflow is an unbreakable unit
    (URL, code span, inline markup).

## Procedure

1. Read the input verbatim.
2. Confirm the rendering context.
   For `.md`, `.markdown`, `.rst`, `.html`,
   or contexts known to follow
   CommonMark / reST / standard HTML semantics,
   proceed.
   For plain text,
   GFM with `<br>` line-break behavior,
   email,
   chat (Slack, Telegram, Discord),
   or any context where single newlines render as visible breaks —
   in explicit-reformat mode, ask the user before continuing;
   in default-on-edit mode, skip sembr entirely.
   Sembr there would visibly change the output and violate rule 2.
3. Apply MUST rules first.
4. Apply SHOULD rules wherever they don't conflict with a MUST rule
   or distort meaning.
5. Use MAY rules to fit the 80-character soft limit and improve readability.
6. Never paraphrase, reword, add, or remove content.
   If the source wording is ambiguous and forces a structural choice,
   ask before editing.
7. Confirm the rendered output matches the input.

## Examples

Bad — multi-clause line, no semantic structure:

```
The build pipeline succeeded; however, the deploy step failed because the secret rotated overnight, and we did not refresh the cache — so production is now serving stale config.
```

Good — sembr-formatted:

```
The build pipeline succeeded;
however, the deploy step failed because the secret rotated overnight,
and we did not refresh the cache —
so production is now serving stale config.
```

Bad — break inside a hyphenated word (violates MUST rule 3):

```
This is a state-
of-the-art system.
```

Good:

```
This is a state-of-the-art system.
```

## Reporting

After editing,
state briefly which file or section was reformatted.
Do not quote the full reformatted text back to the user unless they
ask.

If any rule was relaxed or skipped during the pass —
ambiguous independent clause left unbroken,
non-standard rendering context,
unbreakable line over 80 characters,
or any other place where the strict rules could not be applied
confidently —
list each occurrence so the user knows where the result is best
effort rather than strict.
