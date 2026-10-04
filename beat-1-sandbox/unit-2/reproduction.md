# Unit 2 — Claim and Reproduce

Path: `beat-1-sandbox/unit-2/reproduction.md`

Record of your claim and reproduction on the issue you chose in Unit 1, and of the
evaluation runs that produced `eval-run.txt`. This file is graded at the path above; a copy
kept anywhere else in the repository is not read.

Complete every labelled field below. Each is graded on its own; content placed under the wrong
label is not graded.

---

## Your identity upstream

**GitHub username**

DeveloperMindset123

---

## Posted upstream

**Claim comment**

https://github.com/codepath/pathreview-ai301-fa26-s1/issues/6#issuecomment-5985247170

Text as posted:

````markdown
I'd like to take this one. Reading `rag/retriever/hybrid.py` on `main` (f89c06f), `HybridRetriever.retrieve` assigns `all_chunks = self._get_all_chunks(collection_name)` and never passes it to `keyword_searcher.index()`, which lines up with the first half of the report. The second half, each side being divided by its own batch's top score, is the part I want to understand properly, because it decides what a blended score of 0.7 actually means.

I have not run anything yet. My next step is a small reproduction against a throwaway ChromaDB collection: call `retrieve()` with a fresh `KeywordSearcher` and record `keyword_score` for every result, then repeat with the searcher indexed beforehand to see what per-batch normalization does to a weak keyword match. I'll post that report here, with my environment and commit, before proposing any change. What the normalization should be instead is a question I'd like to raise once I have the numbers in front of me.

Disclosure: I'm using Claude Code (an AI assistant) to help with this work, including drafting this comment.
````

**Reproduction comment**

https://github.com/codepath/pathreview-ai301-fa26-s1/issues/6#issuecomment-5985320679

Text as posted:

````markdown
Reproduction report, following up on my claim above. Both halves of the issue reproduce on `main`.

**Environment**

- macOS 15.5 (arm64)
- Python 3.11.9 (the version CI pins), venv created per `docs/SETUP.md`
- Code state: my fork of `main` at commit `f89c06f`, no local changes
- Installed by `pip install -e ".[dev]"`: chromadb 1.5.9, rank-bm25 0.2.2, structlog 26.1.0
- Baseline before the repro: `pytest tests/unit -m unit` gives 375 passed, 53 xfailed

No Docker, Postgres, Redis, or API key is needed: `VectorStore` is a local `chromadb.PersistentClient`, and the script points it at a temp directory. Nothing on `main` constructs `HybridRetriever` (I searched `api/`, `core/`, `ingestion/`, `rag/`, `agent/`, `safety/`, `scripts/` and `tests/`), so the script calls it directly.

**Steps**

```
git clone https://github.com/codepath/pathreview-ai301-fa26-s1.git
cd pathreview-ai301-fa26-s1
git checkout f89c06f   # upstream main at the time of this run; my fork is identical
python3.11 -m venv .venv
.venv/bin/pip install -e ".[dev]"
# save the script below as repro_issue6.py in the repo root, then:
.venv/bin/python repro_issue6.py
```

`repro_issue6.py` (five toy chunks with 3-d embeddings; `VectorStore.add_chunks` reads `.id`, `.source_id`, `.chunk_index` and `.section`, which `ingestion.chunking.base.Chunk` does not carry, so the script passes `SimpleNamespace` objects with those fields):

```python
import tempfile
from types import SimpleNamespace

from rag.retriever.hybrid import HybridRetriever
from rag.retriever.keyword_search import KeywordSearcher
from rag.retriever.vector_store import VectorStore

TEXTS = {
    "c1": "built a fastapi service with postgresql and redis caching",
    "c2": "trained a convolutional network in pytorch for image classification",
    "c3": "led a team of four students to ship a react dashboard",
    "c4": "wrote unit tests with pytest and set up github actions ci",
    "c5": "capstone notes covering design reviews sprint planning retrospectives demos "
    "documentation onboarding pairing sessions and one passing mention of kubernetes",
}
# Toy 3-d embeddings. The query embedding is nearly orthogonal to all of them,
# so every vector match is weak.
EMB = {"c1": [1, 0, 0], "c2": [0, 1, 0], "c3": [0, 0, 1], "c4": [1, 1, 0], "c5": [0, 1, 1]}
QUERY_EMB = [0.2, -1.0, -1.0]

store = VectorStore(persist_dir=tempfile.mkdtemp())
store.add_chunks(
    [
        (SimpleNamespace(id=k, text=t, source_id="s1", chunk_index=i, section=""), EMB[k])
        for i, (k, t) in enumerate(TEXTS.items())
    ],
    "profile_demo",
)


def show(label, results):
    print(f"\n{label}")
    for r in sorted(results, key=lambda r: r["id"]):
        print(
            f"  {r['id']}  score={r['score']:.3f}  "
            f"vector={r['vector_score']:.3f}  keyword={r['keyword_score']:.3f}"
        )


# Run 1: a fresh KeywordSearcher, which is all retrieve() ever gets.
fresh = HybridRetriever(store, KeywordSearcher())
show(
    "run 1: fresh KeywordSearcher, query 'pytest github actions'",
    fresh.retrieve("pytest github actions", "demo", QUERY_EMB, max_chunks=5, min_score=0.0),
)

# Run 2 (control): same call, but the searcher is indexed with the collection first.
searcher = KeywordSearcher()
searcher.index(fresh._get_all_chunks("profile_demo"))
indexed = HybridRetriever(store, searcher)
show(
    "run 2: searcher indexed first, same query",
    indexed.retrieve("pytest github actions", "demo", QUERY_EMB, max_chunks=5, min_score=0.0),
)

# Run 3: per-batch normalization. A 3-of-3 term match and a 1-of-3 term match
# in a long chunk both come out at keyword_score 1.0; same for the vector side.
print("\nrun 3: raw score vs normalized score for the best match on each side")
for q in ["pytest github actions", "kubernetes helm terraform"]:
    raw = searcher.search(q, top_k=1)[0]
    out = indexed.retrieve(q, "demo", QUERY_EMB, max_chunks=5, min_score=0.0)
    top = max(out, key=lambda r: r["keyword_score"])
    print(
        f"  query {q!r}: best raw bm25={raw['bm25_score']:.3f} ({raw['id']})"
        f" -> keyword_score={top['keyword_score']:.3f} ({top['id']})"
    )
raw_vec = store.query(QUERY_EMB, "profile_demo", n_results=1)[0]
top_vec = max(out, key=lambda r: r["vector_score"])
print(
    f"  best raw vector similarity={raw_vec['score']:.3f} ({raw_vec['id']})"
    f" -> vector_score={top_vec['vector_score']:.3f} ({top_vec['id']})"
)
```

**Output**

Run 1 is shown in full. For runs 2 and 3 I removed the structlog `info` lines (collection lookups and query-complete counts) and kept everything the script prints.

```
2026-10-04 18:42:52 [info     ] created_collection             collection_name=profile_demo
2026-10-04 18:42:52 [info     ] added_chunks                   collection=profile_demo count=5
2026-10-04 18:42:52 [info     ] retrieved_collection           collection_name=profile_demo
2026-10-04 18:42:52 [info     ] vector_query_complete          collection=profile_demo results_count=5
2026-10-04 18:42:52 [info     ] retrieved_collection           collection_name=profile_demo
2026-10-04 18:42:52 [warning  ] keyword_search_empty_index
2026-10-04 18:42:52 [info     ] hybrid_retrieval_complete      blended_count=5 filtered_count=5 final_count=5 keyword_results=0 query_len=21 vector_results=5

run 1: fresh KeywordSearcher, query 'pytest github actions'
  c1  score=0.700  vector=1.000  keyword=0.000
  c2  score=0.482  vector=0.689  keyword=0.000
  c3  score=0.482  vector=0.689  keyword=0.000
  c4  score=0.543  vector=0.776  keyword=0.000
  c5  score=0.435  vector=0.622  keyword=0.000

run 2: searcher indexed first, same query
  c1  score=0.700  vector=1.000  keyword=0.000
  c2  score=0.482  vector=0.689  keyword=0.000
  c3  score=0.482  vector=0.689  keyword=0.000
  c4  score=0.843  vector=0.776  keyword=1.000
  c5  score=0.435  vector=0.622  keyword=0.000

run 3: raw score vs normalized score for the best match on each side
  query 'pytest github actions': best raw bm25=3.400 (c4) -> keyword_score=1.000 (c4)
  query 'kubernetes helm terraform': best raw bm25=0.862 (c5) -> keyword_score=1.000 (c5)
  best raw vector similarity=0.538 (c1) -> vector_score=1.000 (c1)
```

**What this shows**

1. Keyword indexing is skipped. In run 1, `c4` is the only chunk containing all three query words, yet every `keyword_score` is 0.000, the log reports `keyword_search_empty_index` and `keyword_results=0`, and `c1` (no query words at all) ranks first at 0.700. The chunks `retrieve()` fetches with `_get_all_chunks()` would have fixed this: run 2 indexes exactly that list before the same call, and `c4` moves to keyword 1.000 and a blended 0.843, ahead of `c1`.
2. Each side is normalized by its own batch maximum. In run 3, a query matching all three terms in a short chunk (raw BM25 3.400) and a query matching one of three terms once in a long chunk (raw BM25 0.862) both produce `keyword_score=1.000`. The vector side does the same: `c1`'s raw similarity is 0.538, and it becomes `vector_score=1.000`. So the top result on each side always receives that side's full weight (0.7 or 0.3), however weak the match is, which is why `c1` scores 0.700 in run 1 without sharing a single word with the query.

Expected, per the issue: the keyword side should be built from the collection's chunks, and a weak best match should not be scaled up to look like a perfect one. What the normalization should be instead (a fixed scale, min-max over the indexed corpus, or rank-based fusion) is not specified in the issue, and I have not tested any alternative. I'd like a maintainer's view on the intended scoring before I propose a change.

Disclosure: I used Claude Code (an AI assistant) to help write the script, run it, and draft this comment; the output above is pasted from the actual run.
````

## Eval iterations

Answer all four sections. Quote source text directly; paraphrase does not satisfy these
fields.

**Run history**

0. Warm-up, no credit spent: I graded `calib-02` by hand against the first draft
   of my rubric. It rejects on six of the seven required checks (no environment,
   no steps, no artifact, "It is obviously the null result thing" asserted with
   nothing behind it, and a "+1!! ... claiming this one" claim with no next step).
   Only `ai-disclosure` passes, because Joplin states no AI policy.
1. 20/20, full run (`--out results-run1.json`), with all five categories
   matched: clear-accept 8/8, disclosure 1/1, no-evidence 4/4,
   unfollowable-comms 3/3, wrong-target 4/4.
2. 5/5 scored, plus both calibration packages matching, on a targeted canary
   re-run after I loosened `claim-specific`:
   `--only pkg-08,pkg-13,pkg-17,pkg-19,pkg-20,calib-01,calib-02 --include-calibration`
   (partial, so no bar verdict).
3. **20/20, full run, saved to `eval-run.txt`.** The agreement line reads
   `agreement: 20/20 scored items  (bar: 18/20: PASS)`.

**Package analysis**

`pkg-16`, pandas-dev/pandas#66656, "Passing a tuple at creation for 1-d index in
df is fine but rename_axis with tuple fails".

**My rubric's decision: `reject`. Gold label: `reject`.** They agreed in both
full runs.

This is the package I find most instructive, because on its face it is a good
report. The steps are the issue's own three lines, and the artifact is the
right error from the right trigger. My `shows-issue-behavior` check passed it
in both full runs, with the evidence "Artifact shows 'ValueError: Length of new
names must be 1, got 3' from the exact rename_axis trigger the issue
describes". `steps-rerunnable` and `env-recorded` passed too. A rubric that
only asked whether the artifact matches the issue would accept it.

What is wrong is the version line: "Environment: pandas 1.5.3 (pip), Python
3.10.12, Ubuntu 22.04 (x86_64)." The issue's repo facts give the latest release
as v3.0.5, the bug template "asks reporters to confirm the bug exists on the
latest version and on the main branch", the reporter confirmed it on a
main-branch build, and the one thread comment "confirms on pandas 2.3.3 and
current main". The report runs a release older than all of them and never says
so. My `faithful-target` check is what caught it. In run 3 its evidence was
"Report runs pandas 1.5.3; issue confirmed on latest (v3.0.5) and main, thread
confirms on 2.3.3 and main; the version gap is unstated".

I wrote `faithful-target` to read this as a fail even though the error text is
identical, because an artifact from 1.5.3 is evidence about a different
codebase. It cannot tell a maintainer whether the `Index.set_names` path the
thread points at is still broken on main, and the same message can come out of
a different code path in a release two major versions behind. That is why the check's
pass condition says an older release fails "even if the output looks like the
issue's". The fix the report needed was small: one sentence saying "I ran
1.5.3, not latest", which my check would pass, and which would also tell the
reader exactly how much weight the artifact can carry.

The two runs also show where the verdict's weight sits. In run 1, `pkg-16`
failed two required checks. `claims-backed` also failed, with "'The crash the
issue describes is confirmed' generalizes confirmation to the issue's targeted
versions (latest/main) when the run was only on pandas 1.5.3". In run 3 the
grader passed `claims-backed` on the same text, so the reject rested on
`faithful-target` alone. The verdict was stable because the version check is
mechanical: it compares two version strings and looks for a sentence
acknowledging the gap. The honesty reading is a judgment about how far
"confirmed" reaches, and it moved between runs.

**Check rationale**

The check I am writing about is `claim-specific`, quoted from
`tools/repro-check/rubric.md` as uploaded:

```
| claim-specific | The claim comment, read against the issue's title, body, and thread | The claim comment names something specific to this issue (its symptom, trigger, file, function, or a pointer from the thread) and states a concrete next step the commenter will take. Saying they would like to take, work on, or look into the issue is the normal way to claim and passes, and so does describing a planned approach or where they will start reading. Fails on a +1 or me-too with no stated intent; on boilerplate that would read the same pasted onto any other issue; on a guarantee or a deadline attached to a fix ("guaranteed", "within 2 days", "by Friday"); on announcing self-assignment ("assigning myself"); and on asking a maintainer to assign the issue or hold it for them ("kindly assign it to me", "keep this reserved for me") | required |
```

My first version scored 20/20 and I revised it anyway, because reading the
per-check evidence showed the grader applying it inconsistently. The original
middle sentence was "It promises only investigation and reporting back", and it
failed "any guaranteed fix, delivery date or deadline, self-assignment, or
request that the issue be assigned to or reserved for the commenter". In run 1
the grader failed `pkg-17` with "'would like to take the issue' is a
self-assignment request, which the rubric explicitly fails", and `pkg-08` with
"'I would like to take this...as my first jq contribution' reserves the issue;
'I will follow the draft patch...as my starting point' promises a fix, not
investigation and reporting back". In the same run it passed the same phrase on
`pkg-20` ("I'd like to take this one") and `pkg-03` ("I'd like to take a run at
this one"). The score hid the problem because `pkg-08` and `pkg-17` both fail
other required checks. But the phrase is how nearly every real claim starts,
and my own claim comment opens with "I'd like to take this one". On a package
with nothing else wrong, that reading would have been a false reject.

The fix had two parts. First, I named the ordinary claim phrasing and a stated
plan as passes, so "would like to take" and "start from the draft patch" stop
being read as a reservation or a fix promise. Second, I replaced the abstract
nouns ("self-assignment", "request that the issue be ... reserved") with the
literal forms they stand for, quoted. The grader anchors on quoted text, so
quoting "assigning myself" and "kindly assign it to me" gives it a thing to
match instead of a category to interpret.

What I rejected: deleting the assignment and reservation clauses outright, or
moving the no-promises rule into my voice guide only. Either would have flipped
`pkg-19`. Its repro report is fine and `claim-specific` is the only required
check it fails ("Kindly assign it to me, I will fix it within 2 days
guaranteed", "keep this issue reserved for me"), and eval mode never reads the
voice guide.

**Trade-offs**

The revision loosened `claim-specific`, so before spending a full run I re-ran
canaries with
`--only pkg-08,pkg-13,pkg-17,pkg-19,pkg-20,calib-01,calib-02 --include-calibration`.
Each one was there for a reason:

- `pkg-19` is the package whose reject rests on this check alone. It still
  failed, with "Comment asks maintainer to assign ('Kindly assign it to me') and
  to hold the issue ('keep this issue reserved for me'), and attaches a fix
  guarantee with a deadline ('within 2 days guaranteed')".
- `pkg-13` tests the self-assignment form. It still failed `claim-specific`
  with "Comment says 'Assigning myself to this'".
- `calib-02` tests the +1 form and still failed: "Opens with '+1!!' (explicit
  fail condition), names no concrete next step".
- `pkg-20` is the single disclosure package, the category the README names for
  canaries. Its claim also says "I'd like to take this one", so it is exposed to
  the change. It still rejected on `ai-disclosure` alone.
- `pkg-08` and `pkg-17` were the two misreads. Both now pass `claim-specific`
  and still reject on `shows-issue-behavior` and `claims-backed`, which is
  where their real problems are.
- `calib-01` is the case the old wording endangered: "Plan: find where the
  stash-name prompt decides to appear and make the untracked-only case either
  warn or stash with `--include-untracked`". It accepted.

The confirming full run then came back 20/20 with every category matched, so
nothing flipped elsewhere.

The case I accept the check will miss is a soft promise in words my examples
do not quote. `pkg-15`'s claim says "I should be able to have a fix approach to
discuss soon". That is a vague timeline with no date in it, and my check passes
it. Something like "I'll have this wrapped up shortly" would likely pass too. I
am taking that trade deliberately: a check that tries to catch every hint of
confidence starts failing "I'd like to take this", which is the worse error,
because it lands on honest claims. The cost is bounded because the rest of the
rubric holds the packages that make such promises. `pkg-15` still rejects on
`steps-rerunnable`, `shows-issue-behavior`, and `claims-backed`, because its
real problem is a root cause with no evidence behind it, not its phrasing.

---

Related paths: `eval-run.txt` in this directory; your skill's files in
`tools/repro-check/`.
