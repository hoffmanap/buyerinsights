# Findings: https://hoffmanap.github.io/buyerinsights/ 

# Methodology Documentation: Multi-Stage Record Linkage Pipeline

This documents the actual, verified behavior of the linkage pipeline used to reconcile MLS transaction logs, HMDA LAR records, and Data Axle mover profiles for El Paso County. Every claim below has been checked directly against the delivered data (`Fuzzy_Enhanced_Database.csv`, `Data_Axle_Mover.csv`, `LAR_Data_2024.csv`, `LAR_Data_2025.xlsx`). Where the original design intent and the observed output diverge, both are stated.

## 1. Core Linkage Architectural Concept

The pipeline's starting point is a probabilistic corroboration grid built from Data Axle mover profiles and address-string matching. That framing is accurate. The specifics of how well each stage actually performs are corrected below.

## 2. Pipeline Stage Mechanics

### Stage 1: Spatial & Chronological Anchoring

**MLS-side census tract.** Confirmed: 10,777 of 12,901 records (83.5%) carry a populated 11-digit tract ID, consistent with a geocoding step run against the property side of the data.

### Stage 2: Financial Signature Isolation

**Value field validation (±1%) — confirmed, and confirmed tight.** This is accurate and stronger than "within a tolerance": of the 4,341 records that resolved to an HMDA record, **100% fall within 1.0% of the reported HMDA property value**, with a median deviation of 0.25%. This is clearly a hard filter in the pipeline, not a soft scoring input.

**Financing product cross-reference.** Confirmed. Loan type is populated for all 4,341 HMDA-matched records and distributes across Conventional (1,431), FHA (1,524), and VA (1,384), with cash purchases separately flagged and excluded (1,697 records under `Exempt: Cash Purchase`) — consistent with the documented design.

### Stage 3: Probabilistic Corroboration & Demographic Triage

**The demographic weight matrix exists as a field, but is barely active.** `match_confidence_score` is populated and takes the values described in principle, but in practice:
- 582 records score exactly 5 — and every one of these corresponds to `Deterministic Unique Match` (a single unambiguous candidate in the tract/year/price/loan-type bucket), not an accumulated demographic score.
- Of the 3,759 records flagged `Probabilistic Resolved Match` — the population this scoring layer exists to adjudicate — **3,751 (99.8%) carry a score of exactly 0.** Only 8 records show any nonzero demographic corroboration (score 1 or 2).

In other words, the gender/race/income point system is implemented in the schema but did not fire for nearly all of the ambiguous matches it was designed to resolve. Whatever actually broke ties for those 3,751 probabilistic matches, it wasn't the documented demographic weight matrix.

## 3. Data-Tiering Classifications

The tier table (100 / 85–99 / 70–84 / <70) describes the **address fuzzy-match score** (`data_axle_fuzzy_match_score`), which governs the MLS-to-Data-Axle-mover linkage specifically — it is a separate score from the HMDA value/tract/loan-type match described in Stage 2, even though both feed the same output table. Measured directly against the documented cutoffs:

| Tier | Documented rule | Records | Share |
|---|---|---|---|
| 100 | Tier 1 — automatically locked | 1,108 | 8.6% |
| 85–99 | Tier 2 — confident high | 6,436 | 49.9% |
| 70–84 | Tier 3 — flagged for audit, needs +3 demographic points to merge | 2,977 | 23.1% |
| <70 | Programmatic miss — rejected and purged | 2,380 | 18.4% |




