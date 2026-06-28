# Predicting HEDIS A1c Testing Gaps in Diabetic Medicare Patients

## Overview

This project predicts which diabetic Medicare patients are at risk of missing
their annual HbA1c test, using the NCQA HEDIS "Comprehensive Diabetes Care"
measure as the clinical target. The goal is to identify patients for proactive
outreach before a care gap occurs, rather than after.

Unlike most public HEDIS-related projects, this one is built entirely from
**real CMS claims data structure** (the DE-SynPUF dataset) rather than a
purely synthetic dataset with a manufactured label. The compliance label
itself is derived from actual procedure codes in claims, the same way a real
health plan's quality team would calculate it.

## Data

[CMS DE-SynPUF](https://www.cms.gov/data-research/statistics-trends-reports/medicare-claims-synthetic-public-use-files/cms-2008-2010-data-entrepreneurs-synthetic-public-use-file-de-synpuf) (2008-2010), Sample 1 (~10,000 Medicare beneficiaries).

SynPUF data is synthetic at the patient level but preserves real claims
structure: ICD-9 diagnosis codes, HCPCS procedure codes, chronic condition
flags, and three years of beneficiary summary, inpatient, outpatient,
carrier claims, and prescription drug event files.

One important limitation: CMS deliberately "thins" (randomly drops/perturbs)
records in SynPUF to protect privacy. This suppresses some of the real-world
signal, particularly around regular visit patterns, and is the most likely
reason model performance here is more modest than published HEDIS studies
using full claims warehouses.

## Step 1 — Building the denominator

The HEDIS measure defines who's eligible to be measured. A patient qualifies if:

- Diagnosed with diabetes (`SP_DIABETES` flag in the beneficiary summary file)
- Age 18-75 as of the measurement year (2010)
- Continuously enrolled in Medicare for at least 11 months
- Alive through the end of the measurement year

Applying these filters to ~9,700 beneficiaries left **1,380 eligible patients** —
this is the population the measure actually applies to.

## Step 2 — Building the compliance label

There's no pre-built "tested for A1c" flag in claims data — it has to be
derived. The HbA1c lab test has two specific HCPCS procedure codes:

- `83036` — Hemoglobin A1c
- `83037` — Hemoglobin A1c, FDA-cleared home use device

I scanned both Carrier Claims files (~200MB of physician/lab claims) for
any line containing one of these codes, for each of the 1,380 denominator
patients, restricted to 2010. Patients with at least one matching claim were
labeled compliant.

Result: **27% compliance rate** (372 of 1,380 patients tested).

This is well below the ~85-90% compliance rates reported nationally by NCQA.
That gap is expected given SynPUF's data thinning, and is itself worth noting —
it's a real constraint of working with privacy-protected synthetic data, not
a processing error.

## Step 3 — Features and modeling

**First pass** used only demographics and static chronic condition flags
(heart failure, kidney disease, depression, etc.) plus a single utilization
count. This produced weak results — ROC AUC of 0.54-0.58 across logistic
regression, random forest, and XGBoost, barely better than chance.

The issue wasn't the modeling approach, it was the features: chronic disease
flags don't capture whether a patient actually engages with the healthcare
system, and engagement is what drives whether someone shows up for a lab test.

**Second pass** added utilization-based features instead:

- Prior-year (2008-2009) A1c testing history
- Prior-year and current-year outpatient visit counts
- Days since last outpatient visit
- Inpatient stay count
- Total prescription fills and prescription spend
- Total carrier (physician/lab) claim volume

This moved logistic regression from AUC 0.576 to **0.672**, and PR-AUC
(more meaningful here given the 27%/73% class imbalance) to 0.446 against
a 0.27 random baseline. The top predictors were overall claims volume,
days since last visit, and prior-year visit history — confirming that
**general healthcare engagement, not disease severity, is the dominant
driver of testing compliance** in this dataset.

| Model | ROC AUC | PR AUC |
|---|---|---|
| Logistic Regression | 0.672 | 0.446 |
| Random Forest | 0.631 | 0.381 |
| XGBoost | 0.569 | 0.341 |

Logistic regression outperforming the tree-based models is itself a finding:
it suggests the relationship between engagement and testing is close to
linear, and the extra model complexity isn't buying anything on this
feature set and sample size (1,380 patients, ~345 in the test set).

## What this would need to be production-ready

- Full SynPUF sample set (20 samples, ~200k patients) instead of one 10k sample,
  for a more stable estimate
- NDC-to-drug-class mapping to flag actual diabetes medications
  (metformin, insulin) rather than generic prescription volume
- Diagnosis code-based comorbidity scoring (e.g. Elixhauser) instead of
  the six chronic condition flags in the beneficiary summary file
- Validation against a non-thinned claims source, since SynPUF's privacy
  protections likely suppress real signal
