---
description: Maintainer's design review of a branch or PR in an R package - layering, altitude, duplication, conventions. Not a bug hunt.
argument-hint: "[branch | PR number | path] (defaults to current branch vs the default branch)"
---

Review `$ARGUMENTS` as a maintainer of this R package, not as a bug hunter. If no
target was given, review the current branch against the repository's default
branch.

Assume the code works. Do not spend the review on correctness - that is what
`/code-review` and the test suite are for. I want the review a careful colleague
who already knows this package gives about design and fit with what is already
there.

## Before you start

Size the change first: how many files, how many lines, and what kind - feature,
bugfix, refactor, docs. State that in one line along with the depth it warrants,
so the calibration is visible rather than implied.

Four conventions are mine and hold in every package I maintain - they are listed
under **Convention drift** below and need no evidence from the codebase before
you apply them. Everything else is this package's, so work out what those
actually are rather than assuming the last one's. Read whichever of these exist:
`CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`, `DESCRIPTION`, and the formatter or
linter config (`air.toml`, `.lintr`, `.editorconfig`). Those are the rules
someone bothered to write down, and if one of them names review criteria
specific to this package, treat those as additions to the lenses below.
Everything else is unwritten, and you establish it the way a new contributor
would: by reading the neighbours - sibling files in `R/`, the existing
implementations of whatever the diff adds another of, and the last handful of
commits that touched this area.

Then read the files the diff touches: in full when the diff is small enough to
allow it, otherwise the changed functions and their callers, plus whichever
siblings set the relevant convention. Several findings will only be visible from
what the surrounding code already does. A convention is a fact about this
package, not about general R style, so establish it by looking before you call
something drift. Where the codebase is genuinely inconsistent, that is not a
convention - say so rather than picking a side and calling the other side drift.

Then find out what has already been settled, so the review spends itself on new
ground. Two records, both read up front, before the findings exist - not
searched once per finding:

- **Filed to fix later** - `gh issue list --state open --limit 200 --json
  number,title,labels`. Read the body of only the handful whose titles look
  close to something you are likely to report. If `gh` is missing or
  unauthenticated, or the remote is not GitHub, say so in one line and carry on.
- **Settled without a change** - `.github/DESIGN-DECISIONS.md`, if the repo has
  one. These are questions with an answer, recorded precisely because the code
  still looks wrong to a fresh reader. Read it in full; it is short.

## What to look for

Not every lens applies to every change. Skip the ones that do not rather than
forcing a finding out of them.

- **Duplication across layers** - logic checked or handled in two places when the
  package already has one conventional home for it, such as argument validation
  added inline to an exported function when the package has a validation helper
  file the other entry points already go through.
- **Altitude** - top-level functions that have grown inline blocks which should
  be helpers, so the entry point stays an orchestrator.
- **Over-engineering** - code handling cases that cannot arise, or taking five
  lines to do what one line does.
- **Comment noise** - comments that restate the code, or defend a decision nobody
  would question. Say so; I will not volunteer this otherwise.
- **Convention drift** - two kinds, and both count.

  Mine, which hold in every package I maintain regardless of what the code
  around them does: British spelling in user-facing names (function and argument
  names, messages, documentation); roxygen2 as the documentation source of
  truth, with `man/*.Rd` and `NAMESPACE` regenerated via `devtools::document()`
  rather than hand-edited; `NEWS.md` sections ordered **Major Changes**, **Minor
  Changes**, **Bug Fixes**, omitting any that is empty; and an explicit
  `return()` as the final line of every function, including functions defined
  inside other functions.

  Then the package's own, established above: naming and casing, file naming and
  placement, argument order and defaults, condition and message style, and any
  other deliberate terminology choices in user-facing names.

  Where a package's written instructions contradict one of mine, say so once and
  follow the package.
- **Control flow that keeps working after it is known to be pointless** - loops
  that cannot make progress, retries that cannot succeed.
- **Test redundancy** - tests that overlap, or that could collapse into one.
- **API surface** - is this interface worth supporting at all? Flag anything where
  you would say "let's discuss" rather than deciding for me.

## How to report

Anchor every finding at `file:line` using clickable markdown links. Group them:

1. **Change these** - you are confident and would make the change.
2. **Preference** - defensible either way; say which you would pick and move on.
3. **Open questions** - needs my product context, not yours. Ask, do not resolve.

Omit any bucket that comes back empty.

Check every finding against both records before you report it, and then say
nothing about either record in the review. No tally of what you suppressed, no
"already tracked in #123" annotations, no closing note on how much of the diff
was known. Both records are filters, not content.

Where a finding is **already filed** as an issue, or **already settled** in
`DESIGN-DECISIONS.md`, drop it. Do not report it, do not report that you dropped
it, and do not re-argue it because this instance looks different.

The single exception is a genuinely new finding that a filed issue *brushes
against without covering* - near enough that I would want to know it lands
beside existing work, but not fixed by it. Report it in the bucket it belongs
in, name the issue, and be specific about the gap: what the issue covers, and
what this finding adds that it does not. If you cannot state the gap in a
clause, it is either a duplicate - drop it - or unrelated - do not mention the
issue. A bare "related to #123" is not worth the line.

An issue this branch is closing, referenced in `NEWS.md`, a commit message or
the PR body, is not a match at all: that is the work itself.

If a settled decision is genuinely wrong, or the code has moved out from under
it, say *that*, once, and name the entry. That is a finding about the decision,
which I want; it is not licence to re-run the original argument, which I do not.

Then close the loop: end the review by listing the findings I waved off in
discussion, drafted as `DESIGN-DECISIONS.md` entries - what was proposed, what
was decided, why - and offer to append them. A decision that stays in the
conversation is one I pay for again next review. Only ones I actually settled;
do not invent a verdict I did not give.

Judge yourself on coverage, not on count: apply every lens that applies, and
report what each one turned up, which may be nothing. The failure mode to avoid
is stopping at the two or three findings you are most confident about while
sitting on a dozen smaller ones - a low-stakes observation belongs in bucket 2,
not in the bin. But an invented one belongs nowhere. Findings scale with the
diff, so a multi-file feature branch will usually turn up ten or more where a
thirty-line bugfix turns up two, or none. "Nothing here worth changing" is a
legitimate result; padding to look thorough is not.

Where a finding is a claim about behaviour, verify it - run the code or write a
throwaway script in the scratchpad rather than reasoning about it. Use whichever
invocation this package actually uses to load and test itself, per its own
instructions, rather than assuming a bare `R` or `Rscript` on `PATH` is the
right entry point. Say plainly which findings you verified and which you did
not.

Do not apply any fixes. This is a review; I will decide what to act on.
