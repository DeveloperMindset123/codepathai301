# Evidence guide: where proof lives in a reproduction package

This is the map for every check in `rubric.md`. Each family names where to
look in an eval bundle and where to look live, then gives a condition someone
else could apply and get the same answer.

Three places hold almost everything:

- **The issue side.** In a bundle: the `## Issue` section (title, opener and
  their association, labels, body) and `## Thread highlights`. Live: the issue
  page, read with `gh issue view <N> -R <owner/repo>` or
  `gh api repos/<owner>/<repo>/issues/<N>` and `.../issues/<N>/comments`.
- **The repo's rules.** In a bundle: the `## Repo facts` block, whose
  `bug reports:` line is the template's asks and whose `contribution policy:`
  line is the AI and contribution policy. Live: the repo's CONTRIBUTING file
  (for Path Review, `docs/CONTRIBUTING.md`), any `AI_POLICY.md`, and
  `.github/ISSUE_TEMPLATE/bug_report.md`.
- **The candidate side.** In a bundle: `## Candidate claim comment` and
  `## Candidate repro report`. Live: the student's draft files, read as a
  stranger on the thread would read the posted comments. Only what the drafts
  contain or quote counts; other files in the working directory do not.

## Environment

**Where it lives.** The environment line or block in the repro report, usually
its first lines ("Environment: ..."), sometimes a table. The comparison target
is the version, OS, and build stated in the issue body, plus any thread comment
that narrows the affected build ("works in Debug, crashes in Release", "built
from git main", "confirmed on latest and main"). The template's version and OS
asks are in the repo-facts `bug reports:` line. Live, the issue body and thread
are the target; for Path Review, the code state is a commit of `main`, so the
record should name the commit hash and the Python version, since there are no
releases.

**What good looks like.** The record names the version actually run and the
OS or platform, and wherever the issue says a condition changes the behavior
(Windows only, a specific driver, a release build, a browser language order, a
shell), it states that condition for this run. The version run is the affected
one or newer, or the record says plainly that it differs. A difference stated
in the report ("filed against 13.0.0; tested 15.2.0") is a pass; the same
difference left unstated is a fail. An older release than the one the issue
targets, unstated, fails even when its output looks like the issue's, because
it is evidence about a different codebase.

## Steps

**Where it lives.** The repro report's numbered steps or command transcript
(lines starting with `$`, `>>>`, or a described UI action), plus any input
file shown with `cat` or `printf`. Inputs may also be the issue's own: a
script, input file, or playground link published in the issue body is
available to every stranger, so "ran the issue's script verbatim" is a
followable step.

**What good looks like.** Starting from the stated environment, a reader could
type or click the same things and reach the trigger. Check three things: every
command or action is given; every input is shown, or is published in the issue;
the triggering condition the issue names is present in the steps (for example,
the single custom header, the `--replace` flag, the non-English language
first, the driver flag). A step that runs against a private repository, an
unshared config, or internal tooling cannot be repeated and fails. A short
transcript that covers all three passes; length is not the measure.

## Behavior shown

**Where it lives.** The fenced output blocks in the repro report, the quoted
error text, exit codes (`echo $?`), log excerpts, produced CSS or JSON, and
described screen state ("the window remained open afterward"). The target is
the exact failure in the issue body: its error message, crash or panic text,
exit code, or wrong output, and the trigger that produces it.

**What good looks like.** Put the artifact next to the issue's failure and
compare the failure class, not the vibe:

- Same class passes: the issue's panic text appears, the wrong line numbers
  appear, the missing header is missing.
- A different error fails even though it is an error. A graceful argument or
  syntax error (exit 1) is not a capacity-overflow panic (exit 101). A compile
  error about an unbound variable is not a runtime "Invalid path expression".
  Pages of garbled escape text with the window still open is not a crash.
  A version banner and a session list show that the tool runs, not that it
  fails.
- Compare the steps' input against the issue's input. If the student changed
  the part that drives the trigger (a prefix range instead of offset-from-end,
  a colon instead of `=`, a different expression), the artifact answers a
  different question. Changes that keep the trigger (`--offline` instead of a
  live request, a minimal file that keeps the offending section) are fine.
- A control run that differs only in the trigger and does not fail is the
  strongest form of this evidence, but it is preferred, not required.
- An honest cannot-reproduce shows the real attempt's output, says the behavior
  did not appear, and names what differed from the issue's conditions (OS,
  shell, input distribution, limits). That is a complete outcome.

## Honesty

**Where it lives.** The verbs in the claim comment and the report:
"reproduced", "confirmed", "verified", "identified the root cause",
"guaranteed", "100%", "on the Store release too". Each one is a claim, and its
backing is whatever artifact in the report it points at.

**What good looks like.** For every such verb, an artifact in the package shows
it, and the stated outcome matches what the artifact shows. Signals of a
report that claims more than it holds: certainty words with no output block; a
root cause described in prose with no trace, log, or test that shows it; a
conclusion that contradicts its own artifact (a "crash" whose output shows the
program still running); repetition offered as proof ("ran it ten times", "two
machines") over the wrong artifact; generalizing to builds or versions that
were not run. A hypothesis that is labelled as one ("looks plausible", "may be
required", "I suspect") is honest. In a claim-only draft there is no report
yet, so the claim comment must promise a reproduction rather than assert one.

## Comms

**Where it lives.** The claim comment against the issue's title, body, and
thread; both comments against the repo-facts `contribution policy:` line (live:
`docs/CONTRIBUTING.md`, `AI_POLICY.md` if present).

**What good looks like.**

- Specific: the claim names something only this issue has (the symptom, the
  trigger, a file or function, a pointer from the thread) and says what the
  commenter will do next. A test: if the comment would read the same pasted
  onto a different issue, it is boilerplate.
- Honest intent: "I'd like to take this" or "I'd like to work on this" is the
  ordinary way to claim and is not a reservation request; a planned approach
  ("start from the draft patch", "make it warn") is a next step, not a
  promise. What fails is a guarantee or a date on a fix ("within 2 days
  guaranteed"), announcing self-assignment ("assigning myself"), or asking a
  maintainer to assign or hold the issue ("kindly assign it to me", "keep this
  reserved for me").
- Disclosure: read the policy line for what it actually requires. A policy
  that requires disclosing all AI usage, or AI use in issues and comments,
  needs an explicit disclosure in the comments, naming the tool or the extent
  of help; package comments are treated as AI-assisted work. A policy whose
  disclosure ask is limited to pull requests, a responsibility-only or
  quality-only policy, or no stated AI policy requires no disclosure in a
  comment. A policy requiring comments in the contributor's own words is
  satisfied by a first-person comment that is specific to the issue. Path
  Review's `docs/CONTRIBUTING.md` states no AI policy.
- Path Review house rules (live only, from `scope.md`): a classmate's claim
  does not block a new claim, and a repro comment must be the student's own
  proof in their own words, never "same as above, can confirm".
