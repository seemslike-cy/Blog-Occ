# Blog Fine-Grained Occupation Benchmark

Evaluation benchmark for **fine-grained occupation inference** from long-form user-generated text.

The benchmark is built on the public [Blog Authorship Corpus](https://www.cs.bgu.ac.il/~yoavd/download.html) (Schler et al., 2006): 19,320 bloggers and roughly 681k posts with **12 coarse industry labels only**. We extend it with **120 O\*NET-SOC–aligned occupation classes** and **human fine-grained labels** on a curated evaluation subset.

## Release scope (read this first)

**This repository is an annotation overlay, not a standalone text dataset.**

We release only the **user–label correspondence** for 2,000 Blog Authorship Corpus bloggers:

| What we provide | What we do **not** provide |
|-----------------|----------------------------|
| `user_id` → fine-grained occupation label mapping | Blog post text |
| Human annotator votes and gold labels | Raw corpus files |
| Profile metadata and summary stats from the original corpus | A self-contained inference-ready text bundle |

To run models or reproduce evaluation, you **must**:

1. Obtain the original [Blog Authorship Corpus](https://www.cs.bgu.ac.il/~yoavd/download.html) under its license terms.
2. Join corpus posts to this release by **`user_id`**.
3. Use the joined `(user_id, posts)` pairs with labels from `evaluation_2000.jsonl` as evaluation instances.

Without the original corpus, this repository alone is **insufficient** for text-based occupation inference—it only defines *which users were labeled* and *what their labels are*.

---

## What is included

| File | Description |
|------|-------------|
| `evaluation_2000.jsonl` | **User–label correspondence** for 2,000 bloggers: `user_id`, profile fields, human occupation labels, per-annotator votes, and summary stats (~1 MB). **No post text.** |
| `occupation_taxonomy_120.json` | 120-class occupation taxonomy (label definitions, coarse industry, per-class scope notes) |
| `README.md` | Dataset description, join instructions, and citation |

Each JSONL line is one labeled user. The join key is always **`user_id`**.

---

## How to use (required workflow)

```
Blog Authorship Corpus          This repository
(post text, by user_id)    +    (labels, by user_id)
         │                              │
         └──────── join on user_id ──────┘
                        │
                        ▼
              evaluation instances
         (text history + gold occupation)
```

**Step-by-step:**

1. **Download** the [Blog Authorship Corpus](https://www.cs.bgu.ac.il/~yoavd/download.html).
2. **Load** the 2,000 `user_id` values from `evaluation_2000.jsonl`.
3. **Extract** all posts for those IDs from the corpus.
4. **Sort** posts chronologically per user (matching our annotation protocol).
5. **Evaluate** your model on the joined text; compare predictions to `profile.occupation` or `annotation.annotator_votes`.

We do not host or mirror post bodies here—to keep the release small and to respect the original corpus distribution terms.

---

## Construction pipeline

The pipeline matches the dataset description in our paper (§4.1.1).

1. **Eligibility.** Bloggers with at least **30 posts** in the original corpus are retained (4,512 eligible profiles in the paper-aligned filter).

2. **Sampling priority (GPT-4o).**  
   `gpt-4o-2024-11-20` scores each eligible profile for (i) coarse industry fit and (ii) sampling priority based on whether the posting history plausibly supports a fine-grained occupation label.  
   **GPT-4o does not assign fine-grained occupation labels**; it is used only to rank histories for annotation.

3. **Human annotation.**  
   Three trained annotators independently assign **one primary occupation** per sampled blogger, reading the **full post history** and choosing from the 120-class taxonomy (with per-class scope definitions to reduce boundary ambiguity).

4. **Quality control.**  
   Profiles with three-way disagreement, insufficient occupational evidence, or unresolved multi-role ambiguity are excluded or marked uncertain.  
   Profiles with **at least two-of-three annotator agreement** enter the evaluation set.

5. **Evaluation subset.**  
   The released 2,000 users are the highest-priority labelable profiles under the above protocol.

---

## Evaluation-set statistics

Computed from `evaluation_2000.jsonl` (label release):

| Metric | Value |
|--------|------:|
| Users | 2,000 |
| Mean posts per user | 147.7 |
| Median posts per user | 83 |
| Users with occupational cue terms | 100% |
| Unique occupation labels (in eval) | 70 |
| Taxonomy size | 120 |

Top labels (count): Secondary School Teacher (384), Police Officer (205), Physician (200), Executive (126), Waiter (125).

**Interpretation.** Results on this benchmark characterize fine-grained occupation inference when long-form text contains sufficient occupational evidence—they are **not** representative of arbitrary social-media users without such cues.

---

## Data format

### One user record (JSONL line)

```json
{
  "user_id": "3780951",
  "source": "blog_authorship_corpus",
  "profile": {
    "gender": "female",
    "age": 17,
    "industry_coarse": "Student",
    "industry_12": "Education",
    "occupation": "Secondary School Teacher"
  },
  "annotation": {
    "occupation": "Secondary School Teacher",
    "annotator_votes": [
      "Secondary School Teacher",
      "Secondary School Teacher",
      "Secondary School Teacher"
    ]
  },
  "stats": {
    "post_count": 50,
    "word_count": 5961,
    "job_term_hits": 92,
    "job_term_ratio": 15.43,
    "sampling_priority": 1.0
  },
  "occupation_id": 12,
  "occupation_coarse": "Education"
}
```

- **`user_id`**: **required join key** to Blog Authorship Corpus posts; without it, labels cannot be linked to text.
- **`profile.occupation`**: gold label (majority / agreed label).
- **`annotation.annotator_votes`**: three human annotator decisions.
- **`stats`**: corpus-derived summary statistics (no post text).
- **`occupation_id` / `occupation_coarse`**: links to `occupation_taxonomy_120.json`.

### Taxonomy entry

```json
{
  "id": 12,
  "label": "Secondary School Teacher",
  "coarse": "Education",
  "scope": "Teaches middle or high school subjects."
}
```

---

## Quick start

Load the user–label correspondence:

```python
import json

labels_by_user = {}
with open("evaluation_2000.jsonl", encoding="utf-8") as f:
    for line in f:
        user = json.loads(line)
        labels_by_user[user["user_id"]] = {
            "gold": user["profile"]["occupation"],
            "votes": user["annotation"]["annotator_votes"],
            "occupation_id": user["occupation_id"],
        }

# You must separately load Blog Authorship Corpus posts and join:
# posts_by_user[user_id] -> list of post texts from the original corpus
# evaluation instance = (posts_by_user[user_id], labels_by_user[user_id]["gold"])
```

**`user_id` is the only link between this release and the corpus text.**

Load taxonomy:

```python
with open("occupation_taxonomy_120.json", encoding="utf-8") as f:
    taxonomy = json.load(f)
labels = {c["id"]: c for c in taxonomy["classes"]}
```

---

## License and use

- **Original Blog Authorship Corpus (required):** follow Schler et al. (2006) license terms. Post text must be obtained from the official corpus and joined via `user_id`; this repository provides labels only.
- **Taxonomy and labels:** cite Schler et al. (2006) for the underlying corpus and our paper for the fine-grained labels and user–label mapping.
- **Ethics:** bloggers did not consent to modern LLM-based attribute inference. Use for **offline research evaluation only**; do not deploy for high-stakes profiling or identification of real individuals.

---

## Citation

If you use this benchmark, please cite:

```bibtex
@inproceedings{fast2026,
  title   = {Fine-Grained User Attribute Inference via Dual-View Specificity Tracing},
  author  = {...},
  booktitle = {Proceedings of ...},
  year    = {2026}
}

@inproceedings{schler2006,
  title   = {Effects of Age and Gender on Blogging},
  author  = {Schler, Jonathan and Koppel, Moshe and Argamon, Shlomo and Pennebaker, James},
  booktitle = {AAAI Spring Symposium on Computational Approaches to Analyzing Weblogs},
  year    = {2006}
}
```

---

## Contact

Issues and questions: open a GitHub issue in the repository where this benchmark is hosted.
