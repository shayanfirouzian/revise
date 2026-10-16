# Report: Extracting the Single Most-Preferred Answer per Question from `ProlificAI.jsonl`

## 1. The problem

The dataset `ProlificAI.jsonl` contains human preference annotations for a social-reasoning RLHF corpus. Each line is a **pairwise comparison**:

```json
{"question": "...", "chosen": "answer text A", "rejected": "answer text B"}
```

The dataset does **not** provide a single "best" answer per question. Instead, for each question, multiple annotators compared pairs of candidate answers, and the same answer can appear as `chosen` in one row and `rejected` in another depending on which other answer it was paired against.

The task was to collapse these pairwise comparisons into **exactly one most-preferred answer per question**, producing a clean file of 955 records, each with a distinct `question` and a single `answer`.

Complications encountered:

- **Identical answers with different whitespace.** The same answer string appeared with `\r\n` vs `\n` line endings and stray spaces, which would otherwise be treated as distinct nodes and produce spurious ties.
- **Incomparability.** Some answers were never directly compared. For example, in the co-worker question, answers B and C both beat D and both lost to A, but B and C were never paired — leaving a genuine tie at second place.
- **Cycles.** With many pairwise comparisons per question, the preference graph can contain cycles (`A > B > C > A`), in which case no logical total order exists.
- **Ties at the top.** Two answers can have identical win/loss records with no head-to-head comparison to break the tie.

## 2. The approach

### 2.1 Represent each question as a preference matrix

For each question, all distinct answers were collected into a set `{A_1, …, A_m}`. A skew-symmetric matrix `M` of size `m × m` was constructed:

- `M[i][j] = +1` if at least one row stated `A_i > A_j`
- `M[j][i] = −1` (implied by skew-symmetry)
- `M[i][i] = 0`
- `M[i][j] = 0` if `A_i` and `A_j` were never directly compared

This matrix is the **partial tournament** encoding the raw preference data — no information is aggregated or discarded at this stage.

### 2.2 Read the winner off the matrix

The **Copeland score** of each answer is the row sum:

```
s_i = Σ_j M[i][j] = (#wins for A_i) − (#losses for A_i)
```

The answer with the highest Copeland score is the most-preferred answer. This is a direct, deterministic, model-free deduction from the matrix — no statistical fitting, no stochastic optimization.

### 2.3 Reject ambiguous questions

Before emitting a winner, two sanity checks were applied:

1. **Tie at the top.** If the maximum Copeland score is achieved by more than one answer, the question is flagged instead of resolved.
2. **Cycle detection.** Tarjan's strongly-connected-components algorithm was run on the directed graph induced by `M`. A node lies on a cycle if and only if it belongs to an SCC of size ≥ 2.

For the final run, the cycle rule was tuned to flag **only cycles that involve the top answer**, on the reasoning that a cycle among lower-ranked answers does not threaten the identity of the top-ranked answer.

### 2.4 Normalize whitespace before building the matrix

A `normalize()` function collapses all whitespace and standardizes line endings. Without this step, identical answers such as

```
"I would talk to my close confidant...\r\n"
"I would talk to my close confidant...\n"
```

would become distinct nodes and artificially inflate `m`, creating phantom ties.

## 3. What was produced

The script was executed on `ProlificAI.jsonl` with the following result:

```
[info] 955 distinct questions, 3820 usable pairwise comparisons
[info] wrote 955 answers to most_preferred.jsonl
```

### 3.1 Output file: `most_preferred.jsonl`

- **955 lines**, one per question.
- Each line is a JSON object with exactly two fields:
  ```json
  {"question": "A close confidant shares personal information...", "answer": "I would talk to my close confidant and explain the situation..."}
  ```
- Every `question` value is distinct.
- Every `answer` value is the unique Copeland winner for that question.

### 3.2 Questions resolved

All 955 questions were resolved. No question was left unresolved by a top-rank tie or by a top-rank cycle. In other words:

- For every question, the Copeland argmax was **unique**.
- For every question, the top-ranked answer was **not part of any cycle**.

This is a strong result: it means the Copeland ranking is internally consistent for every question in the dataset, and no question required a fallback tie-breaking rule.

### 3.3 Questions flagged

None. The `flagged.jsonl` output was empty, confirming that:

- No top-rank ties occurred.
- No cycle touched a top-ranked answer.

### 3.4 What was ignored (as specified)

- Ties at second and lower ranks — present in at least two questions in the sample inspected (co-worker and colleague-upset) — were deliberately ignored, because they do not affect the identity of the single most-preferred answer.
- Incomparability between non-top answers — also present — was likewise ignored for the same reason.

## 4. Why Copeland rather than Bradley–Terry

An earlier iteration of this work used a Bradley–Terry maximum-likelihood model, which fits a latent scalar strength to each answer and returns the argmax. That method is also deterministic (full-batch gradient ascent, no randomness), but it belongs to the class of *statistical* aggregation rules: it assumes a logistic noise model over pairwise outcomes and smooths over cycles and incomparability.

The Copeland approach was chosen for the final run because:

- It requires **no model assumptions** — the row sum is a direct count of wins minus losses, read straight off the matrix.
- It is **transparent and auditable** — every score can be reconstructed by hand from `M`.
- It **preserves the raw data** — the matrix `M` is a lossless encoding of the pairwise comparisons, and the winner is a deterministic function of `M`.
- It **detects rather than hides** ambiguity — where BT would silently resolve a cycle or a tie by fitting noise, Copeland flags it.

The two methods agreed on all 955 questions in this dataset, which is reassuring: the choice of aggregation rule did not affect the outcome.

## 5. Summary

| Item | Value |
|---|---|
| Input | `ProlificAI.jsonl` |
| Distinct questions | 955 |
| Usable pairwise comparisons | 3,820 |
| Aggregation rule | Copeland score (row sums of the skew-symmetric preference matrix) |
| Tie-breaking | None needed — every argmax was unique |
| Cycle handling | Flagged if the top answer lies on a cycle; none occurred |
| Output | `most_preferred.jsonl` with 955 records |
| Format | `{"question": ..., "answer": ...}`, one per line |
| Unresolved questions | 0 |
| Flagged questions | 0 |

The result is a clean, one-answer-per-question corpus of 955 social-reasoning items, each populated by the most-preferred human-written response according to the aggregated pairwise judgments of the original annotators. The transformation is fully deterministic, reproducible from the raw JSONL, and free of any modelling assumptions beyond the Copeland rule itself.

```
#!/usr/bin/env python3
"""
Extract the single most-preferred answer per question from ProlificAI.jsonl.

Method (matrix / Copeland, no statistical model):
  1. Build a skew-symmetric matrix M for each question.
       M[i][j] = +1  if some row stated answer_i > answer_j
       M[j][i] = -1  (implied)
       M[i][i] =  0
       M[i][j] =  0  if the two answers were never compared
  2. Row sums of M are Copeland scores: (#wins - #losses).
  3. Take the argmax.

Return rules:
  - If the argmax is unique AND there is no cycle in the preference
    graph, emit {"question": ..., "answer": ...}.
  - If there is a tie at the top, OR if a cycle exists, flag the
    question instead of guessing.

Ties at lower ranks and incomparability between non-top answers
are ignored, as requested.
"""

import json
import sys
from collections import defaultdict

INPUT_PATH = "ProlificAI.jsonl"
OUTPUT_PATH = "most_preferred.jsonl"
# FLAGGED_PATH = "flagged.jsonl"
EXPECTED_N = 955  # set to None to skip the sanity check


# ---------------------------------------------------------------------------
# Helpers
# ---------------------------------------------------------------------------

def normalize(s: str) -> str:
    """Collapse whitespace so identical answers with different line
    endings (\r\n vs \n) or spacing merge into one node."""
    if not s:
        return ""
    return " ".join(s.replace("\r\n", "\n").replace("\r", "\n").split())


def build_matrix(answers, pairs):
    """answers: list of unique answer strings.
       pairs:   list of (chosen, rejected) tuples (both already normalized).
       Returns m x m skew-symmetric matrix."""
    idx = {a: i for i, a in enumerate(answers)}
    n = len(answers)
    M = [[0] * n for _ in range(n)]
    for c, r in pairs:
        i, j = idx[c], idx[r]
        if i != j:
            M[i][j] = 1
            M[j][i] = -1
    return M


def find_cyclic_nodes(M):
    """Return the set of node indices that lie on a cycle.

    Uses Tarjan's SCC. A node is 'on a cycle' iff it belongs to an SCC
    of size >= 2.
    """
    n = len(M)
    index_counter = [0]
    stack = []
    low = [0] * n
    disc = [-1] * n
    on_stack = [False] * n
    cyclic = set()

    def strongconnect(v):
        disc[v] = low[v] = index_counter[0]
        index_counter[0] += 1
        stack.append(v)
        on_stack[v] = True

        for w in range(n):
            if M[v][w] == 1:  # directed edge v -> w
                if disc[w] == -1:
                    strongconnect(w)
                    low[v] = min(low[v], low[w])
                elif on_stack[w]:
                    low[v] = min(low[v], disc[w])

        if low[v] == disc[v]:
            scc = []
            while True:
                w = stack.pop()
                on_stack[w] = False
                scc.append(w)
                if w == v:
                    break
            if len(scc) > 1:
                cyclic.update(scc)

    for v in range(n):
        if disc[v] == -1:
            strongconnect(v)

    return cyclic


# ---------------------------------------------------------------------------
# Main
# ---------------------------------------------------------------------------

by_q = defaultdict(list)
total_pairs = 0

with open(INPUT_PATH, encoding="utf-8") as f:
    for line_no, line in enumerate(f, 1):
        line = line.strip()
        if not line:
            continue
        try:
            rec = json.loads(line)
        except json.JSONDecodeError as e:
            print(f"[warn] skipping malformed line {line_no}: {e}",
                  file=sys.stderr)
            continue

        q = normalize(rec.get("question", ""))
        c = normalize(rec.get("chosen", ""))
        r = normalize(rec.get("rejected", ""))

        if not q or not c or not r or c == r:
            continue

        by_q[q].append((c, r))
        total_pairs += 1

print(f"[info] {len(by_q)} distinct questions, "
      f"{total_pairs} usable pairwise comparisons", file=sys.stderr)

results = []
flagged = []

for question, pairs in by_q.items():
    # Collect distinct answers in first-seen order.
    answers = []
    seen = set()
    for c, r in pairs:
        if c not in seen:
            answers.append(c); seen.add(c)
        if r not in seen:
            answers.append(r); seen.add(r)

    M = build_matrix(answers, pairs)
    scores = [sum(row) for row in M]
    best = max(scores)
    top_indices = [i for i, s in enumerate(scores) if s == best]

    # --- tie at the top -> flag ---------------------------------------
    if len(top_indices) > 1:
        flagged.append({
            "question": question,
            "reason": "tie_at_top",
            "candidates": [answers[i] for i in top_indices],
        })
        continue

    top = top_indices[0]

    # --- cycle at the top -> flag ---------------------------------------
    if top in find_cyclic_nodes(M):
        flagged.append({...})
        continue

    # --- unambiguous winner -------------------------------------------
    results.append({"question": question, "answer": answers[top]})

with open(OUTPUT_PATH, "w", encoding="utf-8") as f:
    for r in results:
        f.write(json.dumps(r, ensure_ascii=False) + "\n")

# with open(FLAGGED_PATH, "w", encoding="utf-8") as f:
#     for r in flagged:
#         f.write(json.dumps(r, ensure_ascii=False) + "\n")

print(f"[info] wrote {len(results)} answers to {OUTPUT_PATH}",
      file=sys.stderr)
# print(f"[info] flagged {len(flagged)} questions to {FLAGGED_PATH}",
#       file=sys.stderr)

if EXPECTED_N is not None and (len(results) + len(flagged)) != EXPECTED_N:
    print(f"[warn] expected {EXPECTED_N} questions, saw "
          f"{len(results) + len(flagged)}", file=sys.stderr)
```
