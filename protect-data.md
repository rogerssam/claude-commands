---
description: Lock a repo's client/collaborator data out of Claude's reach — deny rules across every read surface, OS-level isolation where the platform supports it, then verify both actually hold.
argument-hint: "[path(s) holding the confidential data] | --verify (terse re-check) | --verify --verbose (full report)"
---

Set up hard protection for confidential client or collaborator data in this
repository, covering `$ARGUMENTS`. The data stays where it is — the point is to
make it unreadable by me, not to move it.

**Treat this as a safety task, not a config chore.** The deliverable is not a
settings file; it is a *demonstrated* guarantee. A rule that looks right but
does not fire is worse than no rule, because it buys false confidence.

## The line

Reading file **names** is permitted. Reading file **contents** is not. Writing
to or deleting the data is not. Everything below follows from that.

## What this creates

Everything lands inside the repo. **No user-level settings are changed.**

| File | Purpose |
|---|---|
| `.claude/settings.json` | deny rules, plus the sandbox block where supported |
| `.ignore` | keeps the paths out of search and file suggestions |
| `.gitignore` | one line, so the probe never gets committed |
| `CLAUDE.md` | a note saying what is protected and why |
| `<path>/.claude-guard/probe.txt` | sentinel used to verify, never real data |
| `check-data-guard.R` | interpreter probe (sandboxed platforms only) |

## 1. Establish scope

If no paths were given, ask — do not guess from directory names. Confirm which
paths hold confidential data (directories, not file globs, wherever possible)
and whether the repo is shared with collaborators.

Do **not** read, open, grep, or sample the data while setting this up. Listing
filenames is fine — `git ls-files` or `Glob` is often how you confirm a path is
the right one. What must not happen is opening anything.

## 2. Check whether the data is tracked in git — warn loudly if it is

Run `git ls-files -- <path>` for each protected path. If anything comes back,
**stop and warn before writing any config.** Do not refuse — there are
legitimate cases, such as a git server inside a network firewall — but do not
proceed quietly either, and do not let the setup imply the data is covered.

Say this plainly:

> The data is committed to git. **Neither layer below protects it.**
> `git log -p`, `git show` and `git diff <rev>` print file contents while
> reading from `.git/objects`, not from the protected path — so the deny rules
> never see a matching path, and `sandbox.filesystem.denyRead` on the data
> directory does not apply either, because the bytes are not coming from there.
> Denying `.git` instead would break every git operation, so it is not an
> option. Committed data is outside what this command can secure.

Add the specifics that make it concrete:

- History is **retroactive**. Gitignoring the data now leaves every earlier
  commit intact and readable.
- The user-level allow rules include `Bash(git log:*)` and `Bash(git diff:*)`,
  so these reads are **auto-approved** — no prompt would appear.
- Ask where the remote points. A firewall-internal server is a materially
  different risk from a hosted one, and it is the user's call, not yours.

Offer partial mitigation, described honestly as partial: repo-level denies on
the content-dumping commands, e.g. `Bash(git log -p*)`, `Bash(git show:*)`,
`Bash(git format-patch*)`. That narrows the surface but is whack-a-mole — other
plumbing (`git cat-file`, `git archive`) reaches the same objects, and blanket
denies would break ordinary review work.

The only complete fixes are to stop tracking the data and purge history, or to
accept the exposure deliberately. Get an explicit decision, then carry it into
the final report rather than burying it.

## 3. Check that `Glob` and `Grep` exist in this session

Before promising anything, confirm both tools are actually available — check
the tool list and `ToolSearch`. They have been absent in real sessions, and
their absence changes what this command can deliver:

- **No `Glob`** → there is no filename channel at all. `ls` and `find` are
  blocked by the Bash rules, and the sandbox mask hides the directory from
  subprocesses, so nothing can list the path. The "filenames permitted" half of
  the policy is undeliverable. Say so at setup rather than letting it surface
  as a verification failure later.
- **No `Grep`** → `Grep(<path>/**)` can never be tested, so its status stays
  unknown rather than passing.

Neither is caused by these rules and neither is fixable from here. Report the
absence, set up everything else, and note in the final report which guarantees
are reduced.

**Suggest the terminal.** Tool availability has been observed to differ between
the terminal CLI and an IDE integration on the same machine — Glob present in
one, absent in the other, same user and same repo. If either tool is missing,
say that re-running from a terminal session may restore it, rather than
treating the absence as a property of the machine. Grep additionally needs
ripgrep on `PATH` (`which rg`); if that is empty, installing it is a separate
fix worth naming.

## 4. Check whether OS-level isolation is available

Sandboxing runs on **macOS, Linux, and WSL2 only — not native Windows**, where
it is simply absent. Determine which case applies before writing anything, and
carry that through to the report. Do not present a Windows setup as equivalent.

On a sandbox-capable platform, confirm `bwrap` actually works before relying on
it — Ubuntu-derived distros restrict unprivileged user namespaces, which
bubblewrap needs:

```sh
bwrap --ro-bind / / --dev /dev true && echo "bwrap works"
```

If that fails, the sandbox layer cannot start. Say so and set up the permission
layer alone rather than writing a sandbox block that will not engage. Do not
suggest disabling `kernel.apparmor_restrict_unprivileged_userns` — turning off
a system-wide hardening measure to enable a sandbox is a bad trade.

## 5. Write `.claude/settings.json`

Merge into any existing file; never overwrite one. Shared repo → commit it, so
collaborators are covered too. Solo repo → `.claude/settings.local.json`,
confirmed gitignored.

**Permission rules — always, every platform:**

- `Read(<path>/**)`
- `Grep(<path>/**)` — denied **wholesale**, even though filenames are
  permitted: Grep can return matching lines, and no permission rule
  distinguishes its filename-only mode from its content mode.
- `Edit(<path>/**)` and `Write(<path>/**)` — destruction, not just disclosure.
  These write files directly without touching a shell, so Bash rules never
  covered them; an Edit can truncate a data file as effectively as a delete.
- `Bash(* <path>/*)` and `Bash(*<path>/*)` — shell reads (`cat`, `type`,
  `Get-Content`, `sls`, `findstr`, `head`, `awk`, …). These also cover `rm`,
  `mv` and `del` against the path, since the pattern matches the path anywhere
  in the command.
- `Bash(cd <path>*)` — **required, and easy to miss.** The Bash tool's working
  directory persists between calls, so `cd data` in one call followed by
  `cat trial_yields.csv` in the next defeats every pattern above: neither
  command contains the protected path. Add the absolute form too.
- Write each rule in both **relative and absolute** form. A rule matching
  `data/**` does not match `/home/you/repo/data/...`.

Treat this list as a floor, not a ceiling. Pattern matching on command strings
is inherently leaky — if you spot another shape that reaches the path without
naming it, add a rule and say you did. The read mask is what actually closes
this class; the Bash rules are defence in depth.

Put them in `permissions.deny`, never `ask` — `ask` is a prompt I can talk you
into, `deny` is a wall. Deny beats allow, so they compose safely with existing
allow rules.

**Do not deny `Glob`.** It returns paths only, and it is the sanctioned channel
for seeing what data exists so analysis code can reference the right files
without opening them.

**But do not promise that no capability is lost — it may be.** `Glob` runs
in-process and is the *only* mechanism that can return these filenames, because
both alternatives are closed:

- The `Bash` patterns block `ls` and `find` on the path as collateral. That
  breadth is deliberate — it is what catches `cat`, `head`, `awk`, `rm` and
  `mv` in one rule — so do **not** carve out `ls` to restore listing. That
  trades real safety for convenience.
- On a sandboxed platform, `filesystem.denyRead` masks the directory's contents
  from every subprocess anyway, so a shell listing would return empty even if
  permitted.

So if `Glob` is unavailable in a session, **there is no filename channel at
all**, and the stated policy is not achievable. Say so plainly rather than
asserting the fallback works. In that state, analysis code has to use filenames
the user supplies directly.

Include the backslash form `Bash(* <path>\*)` only on Windows; on Linux and
macOS it is dead weight.

**Sandbox block — sandbox-capable platforms only.** Note that per-repo scope
buys filesystem isolation and **nothing else**: `network.strictAllowlist` is
honoured only from user/managed settings, and without it the `allowedDomains`
list is inert — a non-allowlisted domain has been reached with no prompt and no
failure. Include the list anyway as forward-compatible documentation of intent,
but never describe it to the user as egress control. The filesystem mask is the
whole of what this delivers, and it is what was wanted.

```json
"sandbox": {
  "enabled": true,
  "filesystem": {
    "denyRead":   ["<path>", ...],
    "allowWrite": ["/tmp", "<R library path>"]
  },
  "network": { "allowedDomains": ["packagemanager.posit.co"] }
}
```

Get the library path from `Rscript -e '.libPaths()'` rather than guessing; miss
it and `install.packages()` fails in a way that reads like a network problem.

**Check the actual CRAN mirror before writing the domain list** —
`Rscript -e 'getOption("repos")'`. A `.Rprofile` frequently sets something
other than the Posit mirror, and allowlisting only `packagemanager.posit.co`
would silently break `install.packages()`. Add whatever the mirror really is.

Keep this **per-repo**. Enabling the sandbox in user settings would impose
network allowlisting on every project on the machine, breaking ordinary R work
for no benefit. The cost of per-repo scope is that `network.strictAllowlist`
and `filesystem.disabled` are honoured only from user/managed settings, so
non-allowlisted domains prompt rather than fail closed — that is egress
control, not the filesystem guarantee, so it does not matter here.

## 6. Supporting files

- Add the protected paths to `.ignore` at the repo root. Always respected, and
  keeps the data out of file suggestions and directory sweeps. Defence in
  depth, not the guarantee — an explicitly named path can still be searched.
- Add `<path>/.claude-guard/` to `.gitignore`.
- Add a short `CLAUDE.md` note naming the protected paths and the reason — not
  as the protection, but so the wall is understood rather than treated as a bug
  to route around, and so I ask you instead of improvising when I hit it.

## 7. Plant the sentinel probe

**Verification must never touch a real file.** A probe only tells you anything
when it *fails*, and a failed probe against real client data would read that
data and send it to the API — exactly the harm this command prevents.

Create `<path>/.claude-guard/probe.txt` containing one generated sentinel
string and nothing else — no real values, no filenames copied from the
directory, no schema. Something like `CLAUDE-GUARD-PROBE-<random>`. Write it
with the Write tool; never copy or sample an existing file to make it.

It goes *inside* the protected path deliberately: it matches the same
`<path>/**` prefix as the real files, so a rule blocking the probe blocks them
too, while a leak costs nothing. Leave it in place afterwards as the fixture
for re-verifying later.

**Record that you created it.** Once written, read it back — the rules are not
active yet, so you can — and write a manifest at the repo root:

```
.claude-guard-manifest.txt      (gitignored)
probe: <path>/.claude-guard/probe.txt
sentinel-prefix: CLAUDE-GUARD-PROBE-<first 8 chars>
created: <ISO date>
```

This exists because of a real ambiguity in the verification: under a read mask
a hidden file and a missing file are **indistinguishable from inside the
sandbox** — both give `No such file or directory`. Without the manifest, a
probe that was never created looks exactly like a probe that is correctly
masked, and the check silently proves nothing.

On sandbox-capable platforms also write `check-data-guard.R` at the repo root
(gitignored):

```r
# Must FAIL. The path is inside this file, never on the command line,
# so no permission rule can see it — only OS-level isolation can stop it.
cat(readLines("<path>/.claude-guard/probe.txt"), sep = "\n")
```

## 8. Verify — `--verify`

Deny rules load when settings are read, so rules written in this session are
probably **not active yet**, and an unprotected probe looks identical whether
the rule is broken or merely unloaded. So:

- If the rules were not loaded this session, **do not probe at all.** Say the
  setup is complete but unverified, and tell me to restart and run
  `/protect-data --verify`. Do not guess, and do not imply a pass.
- If invoked with `--verify`, skip steps 1–7 and run the checks below.

Aim every content probe **at the probe file only**, never at a real file. Each
of these must be **refused**:

1. `Read` on `<path>/.claude-guard/probe.txt`
2. `Grep` for the sentinel, scoped to `<path>/.claude-guard/`
3. `Bash` with `cat` on `<path>/.claude-guard/probe.txt`
4. `Edit` on the probe file, changing the sentinel — the destruction check

Target the probe by explicit path: that is the *hardest* case to block, since
explicit paths bypass `.ignore`, so a refusal there implies a sweep is refused.

Then one **positive** check:

5. `Glob` for `<path>/**` must **succeed** and return filenames. Filenames are
   permitted, so a refusal here is a misconfiguration too — the deny patterns
   are broader than intended and will obstruct ordinary work. Report it as a
   fault, not as extra safety.

Use the **`Grep` tool** for check 2, not `grep` through Bash. Running it via
the shell tests `Bash(* <path>/*)` and leaves `Grep(<path>/**)` completely
unexercised — and whether `Grep` accepts path rules at all is the least certain
assumption in this command. Report which rule produced the refusal; if the
`Grep` tool is unavailable in the session, say the rule is **untested** rather
than counting the Bash refusal as a pass.

And on sandbox-capable platforms, two checks for the sandbox itself. Do the
write-confinement one first — it is the stronger evidence:

6. **Interpreter confinement.** `Rscript check-data-guard.R` must **not print
   the sentinel**. Expect a permissions error or `No such file or directory` —
   a read mask presents as absence, so both are consistent with the guard
   holding.

   This is the check that matters: the path appears only *inside* the script,
   where no permission rule can see it, so a non-result can only have come from
   OS-level isolation.

   Read `.claude-guard-manifest.txt` first. If it records the probe as created
   and readable at creation time, absence now means masking, and this check is
   **conclusive**. If the manifest is missing, report **inconclusive** — an
   uncreated probe returns exactly the same error. Never present an ambiguous
   result as a pass.

   **Do not add a second confinement probe.** Two were tried and both failed as
   designs, for instructive reasons: a write test has no valid target, since
   the effective `allowWrite` includes the repo and aiming outside trips the
   auto-mode classifier before the sandbox is reached; and a network test
   proves nothing, because per-repo scope gives no egress control at all — a
   non-allowlisted domain was reached with no prompt and no failure. The
   manifest is what made a second probe unnecessary.

If any check fails, report exactly which surface leaked, fix it, and re-verify.
Never report success on a partial pass.

## 9. Report

**`--verify` is meant to be re-run routinely**, not once at setup. Settings
drift, Claude Code updates, and tool availability changes between sessions — a
guard that held last month can stop holding without anything visibly breaking.
So the default output is terse, and detail is reserved for when something is
actually wrong.

**Routine re-check (`--verify`), everything passing — one line, nothing more:**

```
data/ — guard holds. 6/6, sandbox active, probe verified.
```

**Anything failed, inconclusive or untested — report only those**, each with
what it means and what to do. Stay silent about the checks that passed beyond
the count. A partial pass is never summarised as "holds":

```
data/ — 4/6. FAILED: filename listing (Glob unavailable this session; no
filename channel exists — analysis code needs filenames supplied directly).
UNTESTED: Grep(data/**) — Grep tool unavailable, Bash refusal does not count.
```

**Full detail only on first setup, or with `--verbose`.** That is when the
detail earns its place: you are deciding whether to trust the thing. After
that, one line is the useful output and everything else is noise that gets
skimmed — which is how a real failure goes unnoticed.

On first setup or `--verbose`, tell me:

- which paths are protected, and which file holds the rules
- whether OS-level isolation is **active, unavailable, or failed to start**
- **whether any protected path is tracked in git**, and what was decided about
  it — this belongs in the summary, not only in the earlier warning, because it
  is the one exposure neither layer covers
- which surfaces verified as refused, and which are pending a restart
- where the probe was planted

**On native Windows, say plainly that the interpreter gap is open.** Permission
rules key on tool names and command strings, so `Bash(* <path>/*)` catches
`python -c "open('data/x.csv')"` but not a script *file* holding the path
internally — `Rscript analyse.R` never mentions it. Nothing in the rule set
sees that, and without a sandbox nothing closes it; the standing rule in the
global `CLAUDE.md` never to route around a denial is a behavioural commitment,
not enforcement. State that as a limitation of the setup, not a footnote.

Never author or run a script that reads the protected paths, in any language,
including while running this command.
