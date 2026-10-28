# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run
it, and record what actually happened. See the starters in the Instructions, or design your own (your
own experiment earns more credit).

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):**
- **Command I ran:**
- **What I predicted:**
- **What actually happened (paste the key output line):**
- **How this differs from the unperturbed run:**

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):**
- **Command I ran:**
- **What I predicted:**
- **What actually happened (paste the key output line):**
- **How this differs from the unperturbed run:**

---

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):**
- **Command I ran:**
- **What I predicted:**
- **What actually happened (paste the key output line):**
- **How this differs from the unperturbed run:**

## Experiment 1 — Insurance Policy: Missing Premium

**Project:** Validated, Routed Insurance Policy Extraction Pipeline

**Input:** POL-2025-001.txt

**Perturbation:** Replaced the explicitly stated premium amount
($1,847.62) with `NOT PROVIDED` in an isolated copy of the policy.
The original source document was not modified.

**Prediction:** The extractor should not invent a premium.
The validator should detect a missing required field, classify it
as `missing_source`, and escalate the record without auto-approval.

**Observed result:**
- Pipeline exit status: 0
- Validation event: `validation_failed`
- Pattern: `premium_amount_absent`
- Category: `missing_source`
- Escalations: 1
- Routing decisions written: 0
- Auto-approvals: 0
- Routing JSON: empty array (`[]`)
- One API request appeared in the log

**Conclusion:** The pipeline detected missing premium information
and escalated the record instead of auto-approving it. The log
supports early termination of the processing path. The exact
extracted field value was not separately captured.

**Evidence:**
- 01-policy-pipeline/perturbation-input/POL-2025-001.txt
- 01-policy-pipeline/perturbation-run.txt
- 01-policy-pipeline/perturbation-routing.json

## Experiment 2 — Mortgage Income Mathematical Discrepancy

**Project:** Resilient Mortgage Document Extraction System

**Input:** income_sum_mismatch.txt

**Perturbation:** Changed TOTAL MONTHLY EARNINGS from
$10,892.17 to $11,392.17 in an isolated copy of the paystub.
The five individual earnings components were unchanged.

**Prediction:** The extractor should preserve the modified
stated total and the validator should detect a $1,750.00
mathematical discrepancy.

**Observed result:**
- Extraction completed and returned structured JSON
- Calculated monthly income: $9,642.17
- Stated monthly income: $11,392.17
- Delta: -$1,750.00
- Validation consistent: false
- Discrepancy field: total_monthly_income
- Process exit status: 1

**Conclusion:** The system correctly identified the altered
income total and reported the exact mathematical discrepancy.
The CLI returned exit status 1; its intended meaning should
be checked separately.

**Evidence:**
- 02-mortgage-extraction/perturbation-input/income_sum_mismatch.txt
- 02-mortgage-extraction/perturbation-run.txt
- 02-mortgage-extraction/tests.txt
- 02-mortgage-extraction/static-checks.txt

## Supply Chain Risk Investigation — Logistics Timeout

**Perturbation:** Run the Meridian investigation with
`--offline --simulate-timeout`.

**Prediction:** The logistics source will become unavailable.
The coordinator should continue using remaining sources,
explicitly report the timeout, and avoid inventing logistics metrics.

**Observed:**
- Baseline exit status: 0.
- Timeout exit status: 0.
- The timeout briefing explicitly reports:
  "Sources unavailable: logistics unavailable (timeout)".
- Average lead time remains 12 days, but corroboration drops
  from two sources to one.
- Late shipment count becomes incomplete with the reason
  "timeout reading logistics".
- The 95% versus 78% on-time-delivery disagreement disappears
  from the contested section because the logistics source is absent.
- Financial-distress ambiguity remains escalated.

**Result:** PASS — graceful degradation and explicit missing-source
reporting were demonstrated.

**Limitation:** Source loss can conceal disagreements. A single-source
value should not be interpreted as stronger evidence merely because
the conflicting source timed out.

**Evidence:**
- 03-supply-chain/offline-run.txt
- 03-supply-chain/perturbation-run.txt
- 03-supply-chain/tests.txt
- 03-supply-chain/ruff-check.txt
- 03-supply-chain/mypy-check.txt
