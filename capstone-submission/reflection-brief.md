# Reflection Brief — Evaluation and Observability Capstone

**Name:** Kuppa Venkata Sai Raja
**Date:** 2026-10-28

## 0. Environment

| Field | Value |
|---|---|
| OS & version | Linux 4.19.0-gvisor, x86_64, Vocareum VS Code |
| Python version | 3.11.16 |
| Date run | October 28, 2026 |
| Ran any system live? | Insurance policy pipeline using the Vocareum Claude gateway; mortgage extraction used recorded responses; supply-chain investigation used offline news replay |

Each final-exercise solution used its own Python 3.11 virtual environment.

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 45 offline tests |
| Routing output file | 01-policy-pipeline/routing_decisions.json |
| auto_approve / human_review / spot_check counts | 0 / 9 / 0 |

### 1a. Retry boundary

In the deliberately modified `POL-2025-001` document, the required premium amount was replaced with `NOT PROVIDED`.

The perturbation run reported:

- `validation_failed`
- `decisions_written: 0`
- `escalations: 1`
- `premium_amount_absent`, category `missing_source`

Evidence: `01-policy-pipeline/perturbation-run.txt` and `01-policy-pipeline/perturbation-routing.json`.

The run made one extraction API call, as shown by the single successful HTTP request in the captured output. The implementation in `policy_extractor/retry.py` returns `RetryFutileEscalation` immediately for `missing_source`. Repeating extraction cannot recover information genuinely absent from the document. Retrying would add cost and latency while increasing the opportunity to fabricate a premium.

### 1b. Reading the router

Policy `POL-2025-001` was sent to `human_review`.

Its routing record shows:

- `fields_below_threshold: []`
- `integration_failures: []`
- `reviewer_disagreements: ["coverage_limit", "endorsements"]`

The independent reviewer disagreement, rather than low confidence or an integration failure, caused the human-review decision. The extraction confidence was 1.0 for `coverage_limit` and 0.9 for `endorsements`. Trusting confidence alone could have allowed disputed values to bypass human review.

Evidence: `01-policy-pipeline/routing_decisions.json`.

### 1c. Where the aggregate lies

The calibration report shows:

- `umbrella exclusions`: n=2, confidence=0.93, accuracy=0.00, Brier=0.865.
- Overall Brier score: 0.291.

Evidence: `01-policy-pipeline/calibration-report.txt`.

The umbrella-exclusions slice shows severe overconfidence in this small sample. A single overall score blends that failure with better-performing fields, such as auto premium amount. Slicing by `policy_type × field` exposes localized failure modes that aggregate metrics can conceal.

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 |
| Document run | 02-mortgage-extraction/extract-run.txt |
| Classified type | Appraisal/property document |

### 2a. Two guarantees

The discrepancy run extracted monthly income components totaling 9,642.17, but the document stated 10,892.17.

The validator reported:

- `consistent: false`
- `field: total_monthly_income`
- `calculated: 9642.17`
- `stated: 10892.17`
- `delta: -1250.0`

Evidence: `02-mortgage-extraction/discrepancy-run.txt`.

Schema-enforced tool output guarantees that values have the required structure and types. It does not guarantee that the numbers agree mathematically. The deterministic consistency validator detects arithmetic contradictions, but cannot independently guarantee that every extracted component matches the original document. Both checks are needed.

### 2b. Refusing to fabricate

In the missing-bonus run, `bonus_monthly` was `null`, while `base_monthly` was 5673.08.

Evidence: `02-mortgage-extraction/missing-bonus-run.txt`.

The schema explicitly permits `bonus_monthly` to be either a number or null in `mortgage_extractor/schema.py`. The corresponding model declares `bonus_monthly: float | None = None`.

A missing bonus is not equivalent to a zero bonus. Returning null preserves the distinction between unavailable information and a verified numerical value.

### 2c. Normalization

The appraisal example normalized an approximately 2,400-square-foot property description to:

`gross_living_area_sqft: 2400`

Evidence: `02-mortgage-extraction/extract-run.txt`.

Normalizing measurements and categorical values during extraction gives downstream validators and consumers a consistent representation, reducing repeated parsing and comparison errors.

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 |
| Briefing file | 03-supply-chain/briefing.md |
| Section the conflict landed in | Contested |

### 3a. Annotate, don't arbitrate

The baseline briefing preserved two conflicting on-time delivery rates:

- 95.0 percent — supplier_audit, as of 2026-04-10.
- 78.0 percent — logistics, as of 2026-04-05.

The metric appeared in the `Contested` section with an escalation for a high-impact cross-source conflict.

Evidence: `03-supply-chain/briefing.md`.

Keeping both values allows a reviewer to investigate differences in reporting period, methodology, or source reliability. Selecting a single figure would conceal a potentially important operational risk.

### 3b. Source goes dark

The simulated-timeout run displayed:

`Sources unavailable: logistics unavailable (timeout)`

It also moved `late_shipment_count` into the Incomplete section with:

`missing source: timeout reading logistics`

Evidence: `03-supply-chain/timeout-run.txt`.

An unreachable source is an evidence-coverage failure, not a valid observation that there were no late shipments. The coordinator continued with audit, quality, and news information and exited successfully with status 0.

The timeout also removed the logistics value of 78 percent from the on-time delivery comparison, leaving only the audit's 95 percent. This demonstrates why source availability must remain visible: missing evidence can hide a disagreement.

### 3c. Dates as a guardrail

Two dated claims for Meridian were:

- On-time delivery rate: 95.0 percent — supplier_audit, 2026-04-10.
- On-time delivery rate: 78.0 percent — logistics, 2026-04-05.

Evidence: `03-supply-chain/briefing.md`.

Requiring source dates makes it possible to investigate whether a disagreement reflects different observation periods rather than assuming that the sources describe the same moment. The dates do not resolve this conflict by themselves, but they provide necessary context for reconciliation.

## 4. Synthesis

### 4a. One principle

The clearest example of "evaluate the output, don't trust the model's word" was the mortgage mathematical consistency check.

The extraction was structurally valid, but its stated monthly total of 10,892.17 disagreed with the calculated total of 9,642.17. The deterministic validator flagged a 1,250.00 discrepancy.

Evidence: `02-mortgage-extraction/discrepancy-run.txt`.

A system that accepted well-formed model output without checking its meaning could have passed the inconsistent financial result downstream.

### 4b. Confidence is not correctness

The insurance pipeline showed this most clearly. The umbrella-exclusions calibration slice reported 0.93 average confidence but 0.00 accuracy across two examples.

Evidence: `01-policy-pipeline/calibration-report.txt`.

High self-reported confidence did not establish correctness. Independent review, deterministic routing, and field-level calibration provided stronger safeguards than confidence alone. Because the sample contained only two examples, this result should not be generalized into a population-wide accuracy claim.

### 4c. Apply it

In a mortgage document-processing workflow, an LLM might extract borrower income, loan amounts, property measurements, and employment information from inconsistent forms and statements.

I would first apply schema-enforced extraction followed by deterministic validation and escalation for missing or inconsistent critical fields. I would instrument extraction completeness, null rates by field, arithmetic discrepancy counts, retry counts, escalation reasons, source-document provenance, and human correction rates.

A useful alert would identify an increase in income-total discrepancies or missing required values, rather than relying only on overall model confidence or successful API responses.
