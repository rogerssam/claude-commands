---
description: Draft a NEWS.md entry for the current change, in this package's section order and house style.
argument-hint: "[branch | PR number | path] (defaults to the current branch vs the default branch)"
---

Draft the `NEWS.md` entry for `$ARGUMENTS`. If no target was given, use the
current branch against the repository's default branch.

## First, work out whether an entry is warranted

Only **user-facing** changes get an entry. Refactors, test-only changes,
internal renames, CI tweaks and formatting passes do not, however large the
diff. If the change is not user-facing, say so in one line and stop — do not
invent an entry to have something to show.

Read the diff, then read the existing `NEWS.md` before writing anything. The
file is the authority on this package's house style: how bullets are phrased,
function names should be backticked, whether entries are sentences or
fragments, how the in-development version is headed. Match what is there rather
than what is here.

## Section order

Within a release, sections appear in this order regardless of how many entries
each holds:

1. **Major Changes**
2. **Minor Changes**
3. **Bug Fixes**

Omit any section that would be empty — never leave an empty heading behind.
If the target section does not exist yet, create it in the correct position
relative to the sections already present.

## Writing the entry

Keep it to a short 1–2 sentence summary of what changed from the user's point
of view, not paragraphs of explanation and not a description of the
implementation. If you cannot say it in two sentences, that usually means the
diff holds more than one entry — write them separately.

Reference a GitHub issue by number in parentheses, e.g. `(#1)`, **only** where
you are certain the change relates to that issue — a number in a branch name or
a commit message counts, a guess from similar wording does not. When uncertain,
leave the reference out and say you did.

Pick the section by user impact, not by diff size:

- **Major** — breaking changes, removed or renamed user-facing arguments, new
  top-level functionality.
- **Minor** — new arguments with defaults, additional output, relaxed input
  validation, performance changes users would notice.
- **Bug Fixes** — behaviour that was wrong and is now right.

Where a change could sit in two sections, put it in the higher one and say why
in a sentence.

## Then

Show me the proposed bullet and the section it belongs in, and tell me which
existing entries you matched the style against. Apply it to `NEWS.md` once I
confirm — do not edit the file before that. If the diff warrants more than one
entry, list them all before applying any.
