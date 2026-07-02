# Methodology Documentation: Multi-Stage Record Linkage Pipeline

This documentation details the programmatic and econometric methodology used in the Google Colab environment to reconcile and integrate physical housing market records with federal credit registries. 

---

## 1. Core Linkage Architectural Concept

The data linking pipeline adapts the architectural rigor of the **Avenancio-León & Howard (2022)** methodology to a localized context. Traditionally, this econometric framework joins public transaction records to Home Mortgage Disclosure Act (HMDA) Loan Application Register (LAR) records using a deterministic combination of four keys:
1. **Census Tract ID** (11-digit FIPS code)
2. **Transaction / Activity Year**
3. **Exact Loan Amount**
4. **Institutional Lender Name / Legal Entity Identifier (LEI)**

### The Local Modification Barrier
Because local Multiple Listing Service (MLS) logs lack designated institutional lender names or LEI strings, a direct deterministic join is impossible. To overcome this limitation, the pipeline substitutes the missing lender key with a **multi-stage probabilistic corroboration grid**, utilizing demographic arrays from **Data Axle Mover** profiles and property history characteristics from **RentCast**.

---

## 2. Pipeline Stage Mechanics

```
+-----------------------------+       +-----------------------------+
|      Authoritative Base     |       |      Credit Registry        |
|          MLS Logs           |       |        HMDA LAR             |
+--------------+--------------+       +--------------+--------------+
               |                                     |
               v                                     v
       [Geocoded via API]                    [Temporal Splitting]
       [11-Digit Census Tract]               [Reporting Year Windows]
               |                                     |
               +------------------+------------------+
                                  |
                                  v
                     [Financial Signature Matrix]
                     - Sale Price vs Property Value (±1%)
                     - Financing Type Matching
                                  |
                                  v
                     [Probabilistic Scoring Layer]
                     - Fuzzy String Matching (RapidFuzz)
                     - Address Normalization Tokenizer
                     - Demographic Triangulation (Mover)
                                  |
                                  v
                     [Linked Master Consolidated DB]
```

### Stage 1: Spatial & Chronological Anchoring
* **Spatial Transformation:** Physical property addresses from the MLS dataset are parsed and geocoded via the U.S. Census Bureau Geocoder API to append unique 11-digit FIPS Census Tract codes. This limits the initial search field within the massive 2024 and 2025 HMDA workbooks to matching tract coordinates.
* **Temporal Alignment:** Transaction dates from the MLS logs are isolated by calendar year. Trailing chronological buffers are introduced to account for natural institutional reporting delays between the physical property sale date and formal loan closing logs.

### Stage 2: Financial Signature Isolation
To replace the specific matching weight of the missing lender variable, the pipeline builds a high-dimensional financial signature overlay:
* **Value Field Validation:** The final MLS Sale Price is matched against the reported HMDA Property Value field within a tight numerical tolerance threshold ($\pm 1\%$).
* **Financing Product Cross-Reference:** The MLS "Terms of Sale" field is mapped to corresponding HMDA Loan Type variables (e.g., Conventional, FHA, VA). All-cash transactions are automatically flagged, separated, and excluded from the primary debt-registry linkage since they possess no institutional mortgage files.

### Stage 3: Probabilistic Corroboration & Demographic Triage
When multiple candidate applications in the credit register share duplicate spatial and financial parameters within the same census tract, the pipeline applies a probabilistic triage layer using consumer demographic data from **Data Axle Mover**:
* **Address Standardization Engine (`advanced_address_cleaner`):** Programmatically strips structural unit noise (e.g., `"Apt 2"`, `"Suite C"`) and standardizes suffixes (e.g., `"Avenue"` $
ightarrow$ `"Ave"`, `"Drive"` $
ightarrow$ `"Dr"`) across datasets to maximize string overlap.
* **Fuzzy Alignment Arrays:** The system uses the `rapidfuzz` library alongside pre-indexed numpy lookup arrays to calculate character distance metrics (`data_axle_fuzzy_match_score`).
* **Demographic Weight Matrix:** Matches are assigned confidence scoring based on intersecting consumer keys:
  * Matching Gender / Sex Configuration: **+1 Point**
  * Intersecting Race/Ethnicity Identifiers: **+2 Points** (due to elevated data specificity)
  * Intersecting Income Bracket Bounds: **+1 Point**

---

## 3. Data-Tiering Classifications

Output records are classified into distinct confidence tiers based on their cumulative match matrix score prior to downstream filtering and market analysis:

| Match Score Tier | Classification | Textual & Financial Rule Structure | Analytical Treatment |
| :--- | :--- | :--- | :--- |
| **Score 100** | Tier 1: Highly Confident | Absolute character-for-character match on standardized addresses, backed by identical financial variables. | Automatically approved and locked as an authoritative unique record pairing. |
| **Score 85 to 99** | Tier 2: Confident High | Minor phonetic variations, transposed abbreviations, or missing directionals (e.g., `Robert Dahl Dr` vs `Robert Dahl Drive`). | Evaluated against secondary demographic indicators to verify alignment. |
| **Score 70 to 84** | Tier 3: Potential Match | Shared surnames on matching block numbers; address variations requiring structural validation. | Flagged for manual audit; requires a minimum of +3 demographic points to merge. |
| **Score < 70** | Programmatic Miss | Complete string mismatch; distinct block numbers or separate regional grids. | Programmatically rejected and purged from the primary integrated database. |

---

## 4. Key Quantitative Insights Derived

By deploying this linkage architecture, the combined database isolates critical market vectors across El Paso County without relying on unverified policy assumptions:

1. **Broken Filtering Engine:** Long-term residents over 65 are locked into aging central core inventory due to severe property tax reset penalties. This creates an entry-level supply shortage, forcing younger home-seekers out of the urban center.
2. **The Core Underwriting Wall:** In Central Core Tracts (e.g., `48141010502`), purchase liquidity is constrained by institutional rejections. The average borrowing request sits at a conservative **$195,277.78**, yet applications face deep credit history, DTI, and collateral/appraisal failures.
3. **The Suburban Leverage Trap:** Displaced young buyers are migrating outward to periphery sub-markets like the Far East Montana Corridor (`ZIP 79938`, capturing **5,194 records**). Households are buying homes at an average price point of **$263,000** on lower relative household income bases ($\sim$\$92,500), pushing DTI tolerances to structural limits.
4. **The Workforce Purchasing Engine:** Local middle-class families earning between **$90,000 and $120,000** make up the primary volume engine for transactions in the competitive **$250,000 to $350,000** price brackets.
5. **Subsidy-Asset Mismatch:** New for-sale residential infill projects receiving 5-year public tax abatements routinely price at or above **$300,000**, completely overshooting the localized core credit capacity baseline (**~$195,000**).
