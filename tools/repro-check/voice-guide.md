# Voice guide: how I talk upstream

## Who I am in threads

I am a student contributor in AI301, working on my first contributions to an
established codebase. I am comfortable in Python and numerical code but new to
this repo's conventions. When I comment, readers can expect a specific claim
about one issue, commands they can rerun, and output I actually saw, with what
I have not checked yet stated as plainly as what I have.

## Rules I write by

### Rule: Promise investigation, never a fix or a date

Before I have read the code path, the only honest promise is to investigate and
report back. A fix, a timeline, or a guarantee is a promise I cannot cost yet.

- Wrong: "I'll have a PR up fixing this by the weekend."
- Right: "Next I will trace how `HybridRetriever.retrieve` builds the keyword
  side and post a reproduction here before proposing any change."

### Rule: Claim only what I have run

The tense has to match the evidence. If nothing has run yet, the comment says
what I will run; if something ran, the comment shows its output.

- Wrong: "I've reproduced this and the BM25 scores are definitely broken."
- Right: "I have not reproduced this yet. I will run `retrieve` against a small
  seeded collection and post the scores it returns."

### Rule: Name this issue's specifics

Every comment has to name something only this issue has: the function, the
file, the symptom, or the thread's pointer. If the sentence would fit any
issue in the tracker, it says nothing.

- Wrong: "Hi, I'd like to work on this issue as my first contribution."
- Right: "I'd like to take this: the keyword half of the hybrid blend never
  gets indexed, and each side is normalized by its own batch maximum."

### Rule: A guess is labelled as a guess

When I think I know the cause but have not shown it, I say "I suspect" and
point at the line I would check, rather than stating a diagnosis.

- Wrong: "The root cause is the per-batch normalization in `_normalize`."
- Right: "I suspect the per-batch max normalization is involved, but I have
  not traced it yet; that is the next thing I will check."

### Rule: Environment and code state go first

A stranger has to be able to place my run before reading what it showed. The
repro comment opens with the OS, the Python version, and the commit I ran on.

- Wrong: "Ran the tests locally and saw the failure."
- Right: "Environment: macOS 15.5 (arm64), Python 3.12, fork of `main` at
  commit `abc1234`."

## Things I never post

- A deadline, an estimate, or "guaranteed" about a fix.
- "Please assign this to me" or "keep this reserved for me".
- "Same as above, can confirm" or any repro that is not my own run.
- A diagnosis stated as fact that I have not shown with output.
- Filler praise ("great project, love it") in place of content.
- Output I trimmed or edited without saying so.
