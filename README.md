# CGM Payer ROI Model on Medicare Claims

**When does continuous glucose monitoring (CGM) pay for itself for a health plan?**

This project builds a claims analytics pipeline on CMS Medicare claims, measures the cost and hospital use of members with diabetes, and models the payer return on CGM using published effect sizes. It then uses real 2024 CMS state data to decide where a targeted program should start.

## Key findings

1. **Most of the diabetic cost gap is other illness.** Diabetics cost 3.9x non-diabetics per member per month before adjustment. After comparing members with the same number of other chronic conditions, the gap is 1.3x to 1.6x. About 58% of the raw gap comes from comorbidity mix.
2. **Hospital stays drive the excess.** Inpatient care is 56% of the extra spend on diabetics, so avoided admissions are the main savings lever.
3. **Acute diabetes events are too rare to carry the case.** DKA and hypoglycemia admissions are under 1% of diabetic admissions. Any credible ROI has to rest on all-cause admissions and ED visits.
4. **At $250 a month, CGM does not pay back for the average diabetic** (base ROI -67%). Return rises about 10x from the healthiest to the sickest diabetics, and members with 5+ other conditions come closest (-43%, break-even at about $143 a month).
5. **Price and outcome decide the case.** In high-risk diabetics, every 5 points of admission reduction supports about $45 a month of CGM price. This points to targeted coverage with outcomes-based pricing.
6. **Where to start:** Florida and Michigan combine large Medicare Advantage markets with high hospital and ED use.

## Data

| Source | Use | Notes |
|---|---|---|
| [CMS DE-SynPUF Sample 1](https://www.cms.gov/data-research/statistics-trends-and-reports/medicare-claims-synthetic-public-use-files/cms-2008-2010-data-entrepreneurs-synthetic-public-use-file-de-synpuf/de10-sample-1) | Member, inpatient and outpatient claims, 2008 to 2010 | Synthetic. Structure and codes are real; records and relationships between variables are altered |
| [CMS Medicare Geographic Variation PUF](https://data.cms.gov/summary-statistics-on-use-and-payments/medicare-geographic-comparisons/medicare-geographic-variation-by-national-state-county) | State-level MA enrollment, admissions and ED use, 2024 | Real data |
| Published CGM studies (below) | Effect sizes for the ROI model | Low / base / high ranges |

Files used from DE-SynPUF: Beneficiary Summary 2008 and 2009, Inpatient Claims, Outpatient Claims. Carrier and Part D files were skipped; the beneficiary file carries annual carrier reimbursement, and the model counts medical savings only.

## Method

**Eligibility and denominators**

| Step | Member-years | Removed |
|---|---|---|
| All beneficiary records, 2008 and 2009 | 230,890 | |
| Exclude any HMO / Medicare Advantage months | 170,670 | 60,220 |
| Exclude zero Part A or Part B months | 152,900 | 17,770 |

- Member months = the lesser of Part A and Part B months, so only months with full cost visibility count.
- PMPM = sum of Medicare paid / sum of member months (ratio of sums, not average of ratios).
- Utilization per 1,000 = events / member months x 12,000.

**Claims handling rules**

- All code columns loaded as text (ICD-9 V codes and HCPCS J codes break numeric types; leading zeros matter).
- Events counted with `COUNT(DISTINCT CLM_ID)`; claims split into segments are counted once.
- Dollars summed across all segment rows; segment 2 rows carry additional payment.
- Dates taken from segment 1, which is the only segment with dates.

**Measures**

- **ED visits:** outpatient claims with HCPCS 99281 to 99285 in any of 45 code columns (treat-and-release only; ED visits ending in admission sit in inpatient claims).
- **Admissions:** distinct inpatient claims by year of claim start.
- **30-day readmissions:** `LEAD()` window function over each member's stays across all years, so December discharges see January readmissions. Gaps of 1 to 30 days count; same-day transfers are excluded.
- **Acute diabetes events:** principal diagnosis ICD-9 2501x (DKA), 2502x (hyperosmolar), 2508x, 2510 to 2512 (hypoglycemia).
- **Comorbidity adjustment:** stratification into bands of other chronic conditions (0, 1-2, 3-4, 5+), then direct standardization to the diabetic band mix.

## Results (2009, eligible fee-for-service members)

| | Non-diabetic | Diabetic |
|---|---|---|
| Members | 40,545 | 33,067 |
| PMPM | $170 | $655 |
| Inpatient / outpatient / physician PMPM | $75 / $33 / $62 | $347 / $112 / $196 |
| Admissions per 1,000 | 88 | 417 |
| ED visits per 1,000 | 128 | 392 |
| 30-day readmission rate | 4.4% | 8.0% |

**Comorbidity-stratified PMPM**

| Other chronic conditions | Non-diabetic | Diabetic | Ratio |
|---|---|---|---|
| 0 | $30 | $163 | 5.4x |
| 1-2 | $191 | $299 | 1.6x |
| 3-4 | $425 | $565 | 1.3x |
| 5+ | $719 | $1,062 | 1.5x |

Diabetics with depression: $803 PMPM and 543 admissions per 1,000, against $554 and 331 without depression.

## ROI model

```
users           = members x eligible share x adherence
savings / user  = (admits per member x admission reduction x cost per admission
                 + ED visits per member x ED reduction x cost per ED visit) x inflation
program cost    = users x CGM monthly cost x 12
ROI             = (savings - program cost) / program cost
```

| Input | Low | Base | High | Source |
|---|---|---|---|---|
| All-cause admission reduction | 10% | 15% | 32% | Miller et al. 2021 (HR 0.85); J Endocr Soc 2021 (HR 0.68); low case discounts for before/after bias |
| ED visit reduction | 10% | 20% | 30% | Dennis et al. 2025 (acute care -23%) |
| CGM cost per user per month | $400 | $250 | $150 | Retail ~$321 to $572 (SingleCare, Jan 2026); Medicare paid 69% above supplier cost (HHS OIG, Nov 2025) |
| 12-month adherence | 60% | 70% | 85% | Working assumption |
| Cost per admission / ED visit | | $9,724 / $369 | | From the claims, 2009 diabetics |
| Inflation 2009 to 2025 | | 1.55 | | CPI medical care, FRED CUUS0000SAM |

**Results:** low -86%, base -67%, high +16% for the average diabetic. By band, base ROI is -94%, -88%, -74% and -43%, with break-even CGM prices of $15, $31, $66 and $143 a month.

**Break-even grid, members with 5+ other conditions (ROI %)**

| CGM price / month | 10% | 15% | 20% | 25% | 32% |
|---|---|---|---|---|---|
| $100 | -3 | +44 | +90 | +136 | +200 |
| $150 | -35 | -4 | +26 | +57 | +100 |
| $200 | -51 | -28 | -5 | +18 | +50 |
| $250 | -61 | -43 | -24 | -6 | +20 |
| $300 | -68 | -52 | -37 | -21 | 0 |

## Market prioritization (2024, real data)

| State | MA members | MA share | Admits per 1,000 | ED per 1,000 | Est. MA admissions |
|---|---|---|---|---|---|
| CA | 3.47M | 56% | 217 | 587 | 753k |
| FL | 2.85M | 60% | 262 | 621 | 745k |
| TX | 2.51M | 58% | 227 | 584 | 569k |
| NY | 1.99M | 57% | 252 | 579 | 502k |
| PA | 1.55M | 58% | 240 | 565 | 372k |
| MI | 1.38M | 66% | 244 | 638 | 337k |
| OH | 1.40M | 60% | 239 | 620 | 335k |

Estimated MA admissions = MA members x admissions per 1,000. Admission rates are fee-for-service rates used as a proxy for MA members, and cover all members rather than diabetics.

## Limitations

- **Synthetic claims.** DE-SynPUF alters records and weakens relationships between variables. Diabetes prevalence (41.7%) is above real Medicare (about 27%), and repeat admissions are rarer than in real data (8.0% diabetic readmission rate against roughly 15 to 20% published). Use the pipeline for method, not for real effect sizes.
- **Observational effect sizes.** The CGM studies are before/after or matched claims analyses, which tend to overstate effects (regression to the mean). The low case is there for this reason.
- **No insulin flag.** Without Part D data the model cannot isolate insulin users, the population CGM coverage targets. Eligible share (30%) is a working assumption; it changes dollar totals, not ROI %.
- **Association, not causation.** Stratification adjusts for one observed confounder only.
- **Not modeled:** physician fees per avoided event, readmissions avoided, drug cost effects, Medicare Advantage Star Ratings.
- **Small undercount of ED visits:** segment 2 rows have no dates, so their HCPCS codes are not assigned to a year.

## Repository

```
claims_analysis.ipynb      pipeline, analysis and ROI model (DuckDB + pandas)
CGM_Payer_ROI_Deck.pptx    8-slide payer deck
README.md
data/                      not committed; download links above
```

To run: download the four DE-SynPUF Sample 1 files and the Geographic Variation CSV into `data/`, `pip install duckdb pandas`, and run the notebook top to bottom.

## Sources

- Miller E. et al. Flash CGM associated with event reduction in nonintensive diabetes therapy. *Am J Manag Care.* 2021;27(11):e372-e377.
- Flash CGM is associated with reduced diabetes events and hospitalizations in insulin-treated type 2 diabetes. *J Endocr Soc.* 2021. PubMed 33644623.
- Bergenstal R.M. et al. *Diabetes Obes Metab.* 2024 (CGM in type 2 diabetes, Optum claims).
- Dennis C. et al. *Diabetes Technol Ther.* December 2025 (matched cohort, prandial insulin users).
- Isaacson B. et al. *J Diabetes Sci Technol.* 2022;16(2):383-389.
- HHS OIG, Medicare payments for CGMs, November 2025.
- CMS Medicare CGM coverage update, effective April 16, 2023.
