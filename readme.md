# End-to-End Finance ABSA with Shared Data Pipeline

## Project Status
This repository contains the **shared data pipeline, task-specific exports, smoke-test notebooks, and model interfaces** for our finance-domain Aspect-Based Sentiment Analysis (ABSA) project.

Train/Val/Test 80/10/10 split for the tasks

The goal of this project is to build an **end-to-end ABSA system** with three main branches:

- **ATE** — Aspect Term Extraction
- **ASC** — Aspect Sentiment Classification
- **Merge / Joint / Pipeline Integration** — combine aspect extraction and sentiment classification into final `(aspect, sentiment)` predictions

This README is designed to help:

- what the project is doing,
- what each phase already produced,
- which files are the source of truth,
- how ATE / ASC / merge connect to each other,
- what is ready now,
- and what still needs attention.

---

# 1. Project Overview

## Scientific Question
**Does jointly training ATE and ASC in a shared-encoder architecture improve end-to-end ABSA performance compared with training them separately as a pipeline?**

## Why this project
ABSA is a fine-grained sentiment task: instead of predicting one global sentiment for a sentence, we identify **what entity/aspect is being discussed** and **what sentiment is attached to that aspect**.

Example:

**Input**  
`"Silver rebounds while gold also gains in global trade."`

**Output**
- `("Silver", positive)`
- `("gold", positive)`

This project is built around a finance-domain dataset so that the task is more realistic and domain-relevant than generic review sentiment.

---

# 2. Main Task Definitions

## ATE — Aspect Term Extraction
Given a sentence/headline, identify all aspect/entity spans using BIO tagging.

Example:
- Sentence: `SpiceJet to issue 6.4 crore warrants to promoters`
- Output BIO: `B-ASP O O O O O O O O O`

## ASC — Aspect Sentiment Classification
Given a sentence and a known aspect, classify the sentiment polarity.

Example:
- Sentence: `SpiceJet to issue 6.4 crore warrants to promoters`
- Aspect: `SpiceJet`
- Output: `neutral`

## End-to-End Goal
Given only a raw sentence/headline, output a list of `(aspect, sentiment)` pairs.

This is the final shared output space used by pipeline/joint evaluation.

---

# 3. Dataset

## Raw dataset
- **Dataset file**: `SEntFiN-v1.1.csv`
- **Domain**: financial news / finance headlines
- **Total headlines**: **10,753**
- Each row contains a headline and a `Decisions` field mapping one or more entities/aspects to sentiment labels.

## Why this dataset
We chose the finance dataset because it is already ABSA-shaped and lower risk than building a new domain-specific annotation scheme from scratch.

---

# 4. Backbone Model

## Default backbone
`microsoft/deberta-v3-base`

This is the shared encoder family used across:
- ATE
- ASC
- later merge / joint model work

## Why one shared backbone
Using one backbone keeps:
- preprocessing consistent,
- tokenizer behavior consistent,
- and later joint integration simpler.

## Practical note
For smoke tests we prioritize **stability and interface validation**, not final performance tuning.

---

# 5. Shared Data Philosophy

## Single source of truth
The repository should be treated as having **one main canonical data layer**.

### Raw input
- `SEntFiN-v1.1.csv`

### Canonical source of truth
- `finance_absa_phase1_canonical.jsonl`
- then split-specific canonical files:
  - `train_canonical.jsonl`
  - `val_canonical.jsonl`
  - `test_canonical.jsonl`

These canonical files are the shared headline-level representation that downstream tasks depend on.


All training-related work should start from the canonical / split exports.

---

# 6. End-to-End Workflow

## Phase 0 — Raw Audit
Purpose: verify that the raw finance CSV is structurally usable.

Done in this phase:
- load the raw CSV,
- verify schema,
- parse `Decisions`,
- validate sentiment labels,
- inspect label distribution,
- check whether raw file order is safe,
- inspect duplicates / conflicts.

Outputs:
- audited CSV
- audit report
- duplicate report

---

## Phase 1 — Canonical ABSA Readiness Audit
Purpose: convert finance headline data into ABSA-ready supervision.

Done in this phase:
- validate `Words` as metadata,
- inspect aspect/entity key quality,
- generate pseudo-spans,
- audit match quality,
- isolate ambiguous cases into a review queue,
- export canonical headline-level JSON.

Key idea:
We do not only know that an aspect has a sentiment — we also try to ground that aspect back into the headline text via character spans. This is critical for later ATE and end-to-end evaluation.

Outputs:
- canonical JSONL
- canonical CSV
- span review queue
- words validation report
- aspect quality report

---

## Phase 2 — Duplicate-Aware Split and Task Export
Purpose: create reproducible, leakage-safe train/val/test splits and task-specific exports.

Done in this phase:
- isolate unresolved review cases into holdout,
- split by `group_key` instead of raw row,
- use fixed seed and split manifest,
- export:
  - `train/val/test canonical`
  - `ATE jsonl`
  - `ASC jsonl`
- run leakage validation,
- run ATE / ASC validation.

Key idea:
By the end of Phase 2, we no longer merely “have data”.  
We have a **shared, reusable training interface** for all model branches.

Outputs:
- `train_canonical.jsonl`
- `val_canonical.jsonl`
- `test_canonical.jsonl`
- `ate_train/val/test.jsonl`
- `asc_train/val/test.jsonl`
- split manifest
- split stats report
- review holdout files

---

## Phase 3 — Model-Facing Preprocessing and Interface Validation
Purpose: verify that ATE / ASC / merge can all consume the split exports correctly.

Done in this phase:
- load shared tokenizer,
- validate ATE JSONL loading,
- validate ASC JSONL loading,
- align ATE word-level BIO labels to subwords,
- tokenize ASC sentence-aspect inputs,
- build minimal batch collators,
- define shared prediction schemas,
- run smoke tests for ATE / ASC / merge interfaces.

Key idea:
This phase is about **I/O alignment**, not full training.

Outputs:
- ATE smoke report
- ASC smoke report
- merge smoke report
- phase 3 checklist

---

## Phase 4A / 4B / 4C — Minimal Smoke Runs
Purpose: prove that the current branches are trainable / runnable.

- **4A** = ASC minimal train-val smoke run
- **4B** = ATE minimal train-val smoke run
- **4C** = merge / pipeline / joint smoke validation

Important:
These smoke runs are intentionally lightweight and are **not final experiments**.

---

# 7. Current Known Issues

## Issue 1 — ATE supervision mismatch
There is a known Phase 2 issue involving **56 ATE supervision mismatches**.  
This comes from a small subset of substring-based matches where an aspect is marked as matched in canonical data, but is not fully preserved in token-level BIO export.

Current interpretation:
- this does **not** invalidate the whole data pipeline,
- but it means ATE final gold consistency is not yet perfect,
- so formal ATE experiments should treat this as a known risk until resolved.

## Issue 2 — Smoke-test hyperparameters are temporary
For quick smoke tests, current minimal runs may use extremely simplified settings:
- small LR,
- zero weight decay,
- frozen backbone,
- head-only training,
- rollback / finite-check style debugging.

These are **not final training settings**. They are only for early verification.

## Issue 3 — Merge design is still open (Solved)
~~The current assumption is that merge/joint will use a **shared encoder** and learn ATE + ASC together.~~  
~~However, the exact comparison strategy for merge baselines is still open and may be updated later.~~

Current status:
- Phase 4C currently implements a **shared-encoder joint ATE + ASC** smoke run.
- This is **not** a pipeline design such as `ASC -> ATE` or `ATE -> ASC`.
- So for the current engineering baseline, merge/joint should be understood as the **shared-encoder joint approach**.
- If this approach later proves unstable or underperforms, we can then add and compare a **pipeline baseline** as the next alternative.

## Issue 4 Conflicting duplicate supervision still exists in model-ready data (Already Group to avoid leakage, not a big issue at this moment)
Phase 0 already identified a small set of **duplicate/conflicting rows** in the raw dataset.
Although Phase 2 prevents split leakage by assigning the same `group_key` to the same split, it does **not yet remove all conflicting duplicate supervision** from the model-ready pool.

Current interpretation:
- this is **not** a train/val/test leakage problem,
- but it does mean a small number of duplicate or near-duplicate headlines still carry **different gold labels**,
- so some ASC records may present the same `(sentence, aspect, span)` with conflicting sentiments,
- and some ATE records may present the same headline with different BIO supervision,
- therefore this should be treated as a **known training-stability risk** before full baseline experiments.

Practical implication:
- Phase 4 smoke runs are still valid as engineering checks,
- but later full training should ideally isolate, review, or reconcile these conflicting duplicate groups before being treated as final baselines.

---

# 8. Data Files and Their Roles

## Raw source
- `SEntFiN-v1.1.csv`

## Audited intermediate files
- audited CSV
- audit reports
- duplicate reports

## Canonical files
- `finance_absa_phase1_canonical.jsonl`
- `train_canonical.jsonl`
- `val_canonical.jsonl`
- `test_canonical.jsonl`

## Task-specific files
### ATE
- `ate_train.jsonl`
- `ate_val.jsonl`
- `ate_test.jsonl`

### ASC
- `asc_train.jsonl`
- `asc_val.jsonl`
- `asc_test.jsonl`

## Control / metadata files
- review holdout files
- split manifest
- split stats report
- smoke reports
- checklists

---

# 9. I/O Interface Specification

## 9.1 Canonical headline-level record
This is the master representation.

Example structure:

```json
{
  "raw_id": 1,
  "doc_id": "sentfin_000001",
  "text": "SpiceJet to issue 6.4 crore warrants to promoters",
  "group_key": "spicejet to issue 64 crore warrants to promoters",
  "num_aspects": 1,
  "label_signature": "neutral",
  "aspects": [
    {
      "term": "SpiceJet",
      "sentiment": "neutral",
      "char_from": 0,
      "char_to": 8,
      "match_status": "matched",
      "match_type": "whole_word_unique"
    }
  ]
}
```

### Meaning
- `doc_id`: shared ID across all branches
- `text`: headline text used by downstream task exports
- `group_key`: used for duplicate-aware splitting
- `aspects`: all aspect/sentiment annotations for this headline

---

## 9.2 ATE input format
ATE uses token-level BIO tagging.

Example structure:

```json
{
  "doc_id": "sentfin_000001",
  "raw_id": 1,
  "split": "train",
  "text": "SpiceJet to issue 6.4 crore warrants to promoters",
  "tokens": ["SpiceJet", "to", "issue", "6", ".", "4", "crore", "warrants", "to", "promoters"],
  "token_offsets": [[0, 8], [9, 11], [12, 17], [18, 19], [19, 20], [20, 21], [22, 27], [28, 36], [37, 39], [40, 49]],
  "bio_labels": ["B-ASP", "O", "O", "O", "O", "O", "O", "O", "O", "O"],
  "aspects": [...]
}
```

### Used by
- Member 3 (ATE baseline)
- merge / pipeline integration later

---

## 9.3 ASC input format
ASC uses one `(sentence, aspect)` pair per record.

Example structure:

```json
{
  "doc_id": "sentfin_000001",
  "raw_id": 1,
  "split": "train",
  "sentence": "SpiceJet to issue 6.4 crore warrants to promoters",
  "aspect": "SpiceJet",
  "sentiment": "neutral",
  "char_from": 0,
  "char_to": 8,
  "match_type": "whole_word_unique",
  "input_text_template": "SpiceJet to issue 6.4 crore warrants to promoters [SEP] SpiceJet"
}
```

### Used by
- Member 4 (ASC baseline)
- pipeline sentiment step
- merge / joint evaluation later

---

## 9.4 Shared prediction schemas

### ATE prediction schema
```json
{
  "doc_id": "sentfin_xxxxxx",
  "predicted_spans": [
    {
      "term": "example aspect",
      "char_from": 0,
      "char_to": 14
    }
  ]
}
```

### ASC prediction schema
```json
{
  "doc_id": "sentfin_xxxxxx",
  "aspect_predictions": [
    {
      "term": "example aspect",
      "char_from": 0,
      "char_to": 14,
      "sentiment": "positive"
    }
  ]
}
```

### Final merged ABSA output schema
```json
{
  "doc_id": "sentfin_xxxxxx",
  "predicted_pairs": [
    {
      "term": "example aspect",
      "char_from": 0,
      "char_to": 14,
      "sentiment": "positive"
    }
  ]
}
```

This merged format is the one that should be used later for:
- pipeline evaluation,
- joint model comparison,
- end-to-end qualitative examples.

---

# 10. What Each Branch Should Read

## If you are working on ATE
Start from:
- `ate_train.jsonl`
- `ate_val.jsonl`
- `ate_test.jsonl`



## If you are working on ASC
Start from:
- `asc_train.jsonl`
- `asc_val.jsonl`
- `asc_test.jsonl`



## If you are working on merge / joint
Start from:
- `train_canonical.jsonl`
- `val_canonical.jsonl`
- `test_canonical.jsonl`
- plus shared prediction schemas from ATE / ASC



---

# 11. Recommended Working Order

## Already done
- shared data pipeline
- canonical layer
- split export
- ATE / ASC export
- interface smoke tests

## Recommended next order
1. **ASC minimal train-val smoke run** √
2. **ATE minimal train-val smoke run** √
3. **merge / pipeline smoke run** √
4. after that, convert smoke notebooks into real baseline training notebooks
5. only then move into:
   - ablation
   - qualitative analysis
   - full report writing
   - final evaluation

---

# 12. Evaluation Plan

The proposal’s main evaluation target is:

- **Primary metric**: End-to-End F1
- **Diagnostic metrics**:
  - ATE F1
  - ASC Accuracy / Macro-F1

This matches the proposal’s main scientific question: compare **pipeline vs shared/joint encoder** fairly in the same output space.

For now, smoke tests are **not** final evaluation.

---



# 15. Current Repository Mental Model

Think of the project as four layers:

## Layer 1 — Raw source
`SEntFiN-v1.1.csv`

## Layer 2 — Canonical truth
headline-level canonical JSON

## Layer 3 — Task-specific training views
- ATE JSONL
- ASC JSONL

## Layer 4 — Prediction / integration layer
- ATE predictions
- ASC predictions
- merged `(aspect, sentiment)` pairs


---

