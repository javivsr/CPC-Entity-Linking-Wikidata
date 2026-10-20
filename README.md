# Entity Linking over Historical Documents: CPC Records to Wikidata

![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?style=flat&logo=python&logoColor=white)
![Wikidata](https://img.shields.io/badge/Knowledge%20Base-Wikidata-006699?style=flat&logo=wikidata&logoColor=white)

An end-to-end entity linking pipeline that maps **152,589 records of the Casellario
Politico Centrale (CPC)** — a historical Italian police archive — to their corresponding
entities in **Wikidata**.

## The problem

The CPC is a historical registry of individuals compiled by the Italian police between
1894 and the 1980s. Each record exists only as free natural-language text, which makes the
collection essentially unusable for search, aggregation or semantic analysis.

Turning it into structured, queryable data means resolving, for every record, *which*
entity in a public knowledge base that record refers to. That breaks down into four
classic sub-problems:

1. **Mention detection** — find the referring expression inside the record text.
2. **Candidate generation** — retrieve plausible Wikidata entities for that mention.
3. **Disambiguation** — select the correct candidate.
4. **Abstention** — decide when *no* candidate is correct.

The fourth problem matters as much as the others. A linking system that always returns an
answer is worse than useless on a real historical corpus, full of ambiguous names and
records that simply have no entity. The pipeline therefore implements an explicit
**NIL detection** mechanism and reports how often it abstains.

## Pipeline

1. **Labelling process** — builds and populates the master file from the CPC source.
2. **Linking** — mention detection, candidate generation and disambiguation against
   Wikidata.
3. **NIL detection** — abstains when no candidate clears the confidence bar.
4. **Evaluation** — scores linking quality against the held-out annotated sample.

## Data

- **Input:** 152,589 CPC records, one row each.
- **Output:** `data/production/master.csv`, one row per CPC record.

The `qid` field carries the linking outcome:

| Value | Meaning |
| --- | --- |
| `-` | Record not yet processed |
| `NIL` | Processed, but no entity could be confidently linked |
| `Q…` | Wikidata QID of the linked entity |

## Results

Four classification models were evaluated on two datasets — a labelling set and a
disjoint test set — using accuracy, precision, recall and F1. Precision was prioritised
over recall, since an incorrect link is more harmful than a missed one, so the decision
threshold was fixed at 0.85.

| Model / set | Accuracy | Precision | Recall | F1 |
| --- | --- | --- | --- | --- |
| Heuristic — labelling | 96.2% | 100.0% | 93.6% | 0.967 |
| Heuristic — test | 97.3% | 100.0% | 96.4% | 0.981 |
| Logistic Regression — labelling | 93.2% | 92.6% | 96.2% | 0.943 |
| Logistic Regression — test | 96.0% | 100.0% | 94.5% | 0.972 |
| Random Forest — labelling | 96.2% | 100.0% | 93.6% | 0.967 |
| Random Forest — test | 97.3% | 100.0% | 96.4% | 0.981 |
| Gradient Boosting — labelling | 99.2% | 100.0% | 98.7% | 0.994 |
| Gradient Boosting — test | **100.0%** | **100.0%** | **100.0%** | **1.000** |

**Gradient Boosting is the best model**, misclassifying a single case on the labelling set
and classifying every test case correctly (F1 = 1.000). Logistic Regression is the
weakest of the four. Notably, the Random Forest reproduces the heuristic's results exactly
— the same cases are misclassified — indicating that its decision trees largely replicate
the rule the heuristic encodes.

### Datasets

The labelling set and the test set were built by the same pipeline over different records:

- **Labelling set:** after candidate retrieval and enrichment, 5,465 pairs remained;
  removing non-person entries left 4,857, and removing pairs with a birth-year gap above
  one year left 132 pairs (122 persons) — 78 positive (59.1%) and 54 negative (40.9%).
- **Test set:** 3,061 randomly chosen persons. 2,330 returned no candidate, and after
  filtering 75 pairs (74 distinct persons) remained — 55 positive (73.3%) and 20 negative
  (26.7%).

### Models compared

- **Heuristic** — predicts a link when `eval_occupation = 1` or `eval_city > threshold`, a
  rule derived from the two most discriminative features.
- **Logistic Regression** — scikit-learn; largest positive coefficients are `eval_city`
  (+2.04), `eval_country` (+1.37) and `eval_occupation` (+1.33).
- **Random Forest** — 300 trees, max depth 3, class-imbalance weighting. Feature
  importance dominated by `eval_city` (0.43), `eval_birth` (0.22), `eval_occupation` (0.16).
- **Gradient Boosting** — 100 estimators, depth 3, imbalance weighting. Feature importance
  concentrated on `eval_city` (0.80) and `eval_occupation` (0.15).

## Repository structure
```
src/                                 # labelling process, linking pipeline, evaluation
data/                                # CPC source data, annotated sets, production output
Entity_Linking_Report.pdf / .docx    # full report
```


## Usage

### Setup - create and populate the master file (run once)

```python
from src.labellingprocess import *

create_master_file(FILE_CPC_ALL, FILE_MASTER)
load_data(FILE_MASTER, 'data/labelling')
```

## Processing CPC records

Two modes are supported:

- Pass an explicit list of CPC record IDs to process.
- Request the first n records, to process in batches.

## Reports and data

- Full report ([PDF](https://github.com/javivsr/EntityLinkingJavierVaca/blob/main/Entity_Linking_Report.pdf))

- Full report ([Word](https://github.com/javivsr/EntityLinkingJavierVaca/blob/main/Entity_Linking_Report.docx))

- CPC database ([zip](https://github.com/javivsr/EntityLinkingJavierVaca/blob/main/data/cpc_records.zip), decompress before use)

- CPC database, [CSV](https://github.com/javivsr/EntityLinkingJavierVaca/blob/main/data/cpc_csv.zip)

## Evaluation 

```
from src.evaluation import *
```

## Context

Developed at the Universidad de Valladolid, across 37 commits covering the labelling process, the linking pipeline and the evaluation suite.

Author: Javier Vaca Sánchez
