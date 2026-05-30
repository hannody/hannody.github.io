---
title: "Before You Train, Audit: A TinyStories Case Study"
date: 2026-05-30
draft: false
slug: "before-you-train-audit-tinystories"
tags:
  - data-audit
  - tinystories
  - eda
  - duplication
  - training-quality
  - synthetic-datasets
  - best-practices
---


If you are building a small language model from scratch, TinyStories is usually one of the first datasets you reach for. I did too.
Before touching tokenization and training, I ran a quick data audit on the parquet train split. That one step surfaced something worth sharing: a large number of exact duplicate stories sitting quietly in the data.
This post is a practitioner-level writeup, not an accusation. TinyStories is still a solid dataset. The point is just this: verify your data pipeline the same way you verify your model pipeline.

## 1. Why This Audit Exists

Something I keep noticing in junior engineers, in tutorials, in official courses and documentation is that downloading a dataset gets treated as the finish line. The code runs, the rows load, the training loop starts. Nobody actually looked at the data.
I once asked a few junior engineers if they had done any EDA on their training data. "It's from Hugging Face," one said. "The authors are from Microsoft Research!" That was the whole answer.
It is getting easier to skip this step, not harder. An AI coding assistant will hand you a working load_dataset call in seconds, which is great but the code working is not the same as the data being clean.

TinyStories ([paper](https://arxiv.org/abs/2305.07759), [HuggingFace Dataset](https://huggingface.co/datasets/roneneldan/TinyStories)) is a well-known, accessible dataset that sees a lot of use for small LLM experiments simple enough to iterate on, small enough to train on a laptop or a single GPU. I was using it as a baseline while training a tiny language model from scratch, having already worked with a few other text datasets including Biblical texts and children's stories. Given that the original paper has similar goals to my setup, I expected a quick EDA pass and then move on.
Instead the parquet train shards had **320,470** exact duplicate stories and **230** empty rows out of 2.1 million total. Not catastrophic, but not something you want to discover after a training run either.
Pandas and PyArrow will catch this in under fifty lines of code. Worth the fifteen minutes.

## 2. Dataset Background

TinyStories (Eldan and Li, 2023) is a synthetic children-story dataset commonly used to train coherent sub-10M parameter models.

In common Hugging Face workflows, users consume the parquet split with four train shards:

- train-00000-of-00004-2d5a1467fff1081b.parquet
- train-00001-of-00004-5852b56a2bd28fd9.parquet
- train-00002-of-00004-a26307300439e943.parquet
- train-00003-of-00004-d243063613e5a057.parquet

Total rows in these four shards: 2,119,719.

Because the data is synthetic, some duplication can happen naturally. The question is how much.

## 3. EDA Methodology and Reproducible Script

The following script is an exact-match audit on the text column. It runs in few minutes on a MacBook Pro M4 laptop.

```python
import glob
import pandas as pd

train_files = sorted(glob.glob("./data/tinystories/data/train-*.parquet"))
train = pd.concat([pd.read_parquet(f, columns=["text"]) for f in train_files], ignore_index=True)

total_rows = len(train)
unique_rows = train["text"].nunique(dropna=False)
extra_rows = total_rows - unique_rows
dup_rate = extra_rows / total_rows * 100

empty_rows = (train["text"].fillna("").str.strip() == "").sum()

print("Total rows:", total_rows)
print("Unique rows:", unique_rows)
print("Extra duplicate rows:", extra_rows)
print("Duplicate rate (%):", round(dup_rate, 2))
print("Empty-string rows:", int(empty_rows))
```

Observed results from this audit:

- Total rows: 2,119,719
- Unique rows (including the empty string as one value): 1,799,249
- Extra duplicate rows: 320,470
- Exact-duplicate rate: 320,470 / 2,119,719 = 15.12%
- Empty-string rows: 230

Per-shard duplicate counts (exact string duplicates within each shard):

```python
for f in train_files:
    d = pd.read_parquet(f, columns=["text"])
    dupes = d["text"].duplicated().sum()
    print(f"{f.split('/')[-1]}: {dupes} duplicates out of {len(d)}")
```

Observed:

- train-00000-of-00004-2d5a1467fff1081b.parquet: 20,304
- train-00001-of-00004-5852b56a2bd28fd9.parquet: 18,232
- train-00002-of-00004-a26307300439e943.parquet: 21,087
- train-00003-of-00004-d243063613e5a057.parquet: 21,062

These within-shard duplicates sum to 80,685. Since the corpus-wide total of extra duplicate rows is 320,470, the remaining 239,785 are cross-shard duplicates (the same story appearing in more than one shard). In other words, duplication exists both inside individual shards and across them, which rules out a merge or concatenation mistake on my side as the cause.

The even spread across shards suggests this is upstream data reality, not an artifact of how the shards are combined.

### Verifying the numbers without loading everything into memory

The `pd.concat` approach above materializes all ~2.1M strings at once, which is convenient but memory-hungry. I was having vLLM with Qwen3.5-4B loaded in my GPU and wanted to avoid memory contention. To independently confirm the totals, I re-ran the audit with a streaming, memory-safe script that processes one shard at a time and keeps only a fixed-size 16-byte BLAKE2b hash per row instead of the full text. A global counter over the hashes gives the corpus-wide unique and duplicate totals; a per-shard counter gives the within-shard duplicates.

```python
import glob
import hashlib
from collections import Counter

import pyarrow.parquet as pq


def content_hash(text: str) -> bytes:
    return hashlib.blake2b(text.encode("utf-8"), digest_size=16).digest()


files = sorted(glob.glob("./data/tinystories/data/train-*.parquet"))

global_counts: Counter[bytes] = Counter()
total_rows = 0
empty_total = 0

for f in files:
    col = pq.read_table(f, columns=["text"]).column("text").to_pylist()
    total_rows += len(col)

    local: Counter[bytes] = Counter()
    empties = 0
    for t in col:
        t = "" if t is None else t
        if t.strip() == "":
            empties += 1
        hv = content_hash(t)
        local[hv] += 1
        global_counts[hv] += 1

    shard_dupes = sum(c - 1 for c in local.values() if c > 1)
    empty_total += empties
    print(f"{f.split('/')[-1]}: rows={len(col)} within_shard_dupes={shard_dupes} empties={empties}")
    del col, local

unique = len(global_counts)
extra = total_rows - unique
print("Total rows:", total_rows)
print("Unique rows (incl. empty string):", unique)
print("Extra duplicate rows:", extra)
print("Duplicate rate %:", round(extra / total_rows * 100, 2))
print("Empty-string rows:", empty_total)
```

This produced identical figures to the pandas version (2,119,719 total, 1,799,249 unique, 320,470 duplicates, 15.12%, 230 empties), which is the cross-check I trust before publishing any of these numbers.

## 4. Evidence You Can Manually Inspect

If you want manual validation, record duplicate groups as row-position lists.

Example format:

- [16761, 269824]
- [1, 4, 6]

Each group means all listed row positions contain the same story text. Note that `groupby(...).indices` returns positional offsets. The shards load with a default `RangeIndex`, so these positions coincide with the `.loc` labels used below; if you reindex the frame, switch the lookups to `.iloc`.

```python
# Duplicate groups for one shard by row index
import pandas as pd

tdf1 = pd.read_parquet("./data/tinystories/data/train-00000-of-00004-2d5a1467fff1081b.parquet", columns=["text"])

groups = tdf1.groupby("text", sort=False).indices
# keep only groups with more than one row index
dup_groups = [sorted(list(idxs)) for idxs in groups.values() if len(idxs) > 1]

with open("./data/duplicate_groups_train_1.txt", "w") as f:
    for g in sorted(dup_groups, key=lambda x: (len(x), x)):
        f.write(str(g) + "\n")

print("Wrote", len(dup_groups), "duplicate groups to ./data/duplicate_groups_train_1.txt")
```

Then inspect any pair directly:

```python
print(tdf1.loc[16761, "text"])
print("#" * 80)
print(tdf1.loc[269824, "text"])
print("Identical:", tdf1.loc[16761, "text"] == tdf1.loc[269824, "text"])
```

## 5. Prior Art: Known Concern, Missing Quantification on Parquet

This issue is not entirely new, but the parquet-specific quantification gap is real.

- Finke et al. (SimpleStories, NeurIPS 2025) explicitly note that TinyStories contains duplications and other artifacts, though they do not quantify them [see introduction in section 1](https://neurips.cc/virtual/2025/loc/san-diego/poster/121425).
- Karpathy's tinystories-gpt4-clean dataset card acknowledges known quality issues and describes a cleaning process focused on encoding and normalization rather than exact-text deduplication, and it operates on the text source rather than these parquet shards.
- Sajith et al. (2024) study duplication effects on TinyStories experimentally and report that quality is sensitive to duplication rate.

I have not independently reproduced the prior results; the citations are provided so readers can check the original sources. What is additive here is a concrete, reproducible audit of the canonical parquet train shards with exact counts.

## 6. Why 15% Duplication Matters in Practice

Exact duplicate stories increase repeated token sequences. This can make training look stable while reducing effective data diversity. Combined with already formulaic starts and short story templates, this can bias the model toward narrow patterns. From my own earlier experiments with tiny LMs such as 1M up to 18M parameters models trained on such data, tend to perform well on training loss, but the output quality will suffer, diversity will be low, even when I set temperature to 1.0 and TopK to 40 or more. The model is just memorizing and regurgitating the same stories instead of learning to generalize.

A practical takeaway: your loss curve can improve while your generalization quality plateaus.

## 7. Fix: Deduplicate Before Training

Minimal cleanup pipeline:

```python
import pandas as pd
import glob

train_files = sorted(glob.glob("./data/tinystories/data/train-*.parquet"))
train = pd.concat([pd.read_parquet(f, columns=["text"]) for f in train_files], ignore_index=True)

clean = train.copy()
clean = clean[clean["text"].fillna("").str.strip() != ""]
clean = clean.drop_duplicates(subset="text", keep="first").reset_index(drop=True)

clean.to_parquet("./data/tinystories/data/train_full_clean.parquet", index=False)
print("Clean rows:", len(clean))
```

Result from this run:

- Clean rows: 1,799,248

Note the one-row difference from the 1,799,249 unique count reported earlier. The unique count treats the empty string as a single distinct value; here we drop empty rows entirely before deduplicating, so that one bucket disappears. Order does not matter: dropping empties then deduplicating, or deduplicating then dropping empties, both yield 1,799,248.

This is the dataset I recommend using as a baseline for training experiments.

## 8. What Comes Next

The next step is dataset augmentation. But the order matters:

1. Deduplicate and sanitize first.
2. Add complementary story sources after the baseline is clean.

This keeps improvements attributable and avoids compounding hidden data issues.

## 9. Closing Takeaway

EDA is not optional, even for trusted datasets. Synthetic corpora are especially likely to carry generation-side duplication and formatting artifacts that survive into popular conversions.

TinyStories remains useful. But if you train on the parquet split as-is, you should account for a non-trivial exact duplicate rate and empty rows.

The full streaming audit used to verify every number in this post is available as `audit_duplicates.py` in the repository, so you can reproduce the figures on your own copy of the shards in a couple of minutes. Also, the refernces section has links to the original papers and dataset cards for further reading, as there are more nuances to the dataset and its quality issues than I could cover in this post.

## References

1. Finke et al., SimpleStories (arXiv): [https://arxiv.org/pdf/2504.09184](https://arxiv.org/pdf/2504.09184)
2. Finke et al., SimpleStories (OpenReview / NeurIPS 2025): [https://openreview.net/pdf/8d789832b5dd91a1ba6936f835de732eff38cdb2.pdf](https://openreview.net/pdf/8d789832b5dd91a1ba6936f835de732eff38cdb2.pdf)
3. Karpathy cleaned TinyStories dataset card: [https://huggingface.co/datasets/karpathy/tinystories-gpt4-clean](https://huggingface.co/datasets/karpathy/tinystories-gpt4-clean)
4. Sajith et al. (duplication impact study): [https://arxiv.org/pdf/2411.15821](https://arxiv.org/pdf/2411.15821)
5. SimpleStories dataset card: [https://huggingface.co/datasets/SimpleStories/SimpleStories](https://huggingface.co/datasets/SimpleStories/SimpleStories)
6. TinyStories topic overview: [https://www.emergentmind.com/topics/tinystories-dataset](https://www.emergentmind.com/topics/tinystories-dataset)
