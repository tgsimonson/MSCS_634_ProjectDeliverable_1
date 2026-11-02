# MSCS 634 Project Deliverable 1: Data Collection, Cleaning, and Exploration

**Todd Simonson**
MSCS-634: Advanced Big Data and Data Mining
Dr. Satish Penmatsa

## Dataset Summary

This project uses the **CMS Nursing Home Care Compare Provider Information** file (October 2026 release), published by the Centers for Medicare & Medicaid Services. Each record is one nursing home certified to participate in Medicare, Medicaid, or both.

| | |
|---|---|
| Source | https://data.cms.gov/provider-data/dataset/4pq5-n9py |
| Raw size | 14,686 facilities × 102 attributes |
| Cleaned size | 14,620 facilities × 42 attributes |
| Coverage | All 50 states, Washington DC, Puerto Rico, and Guam |
| Attribute groups | Location, ownership and chain structure, beds and census, star ratings, nurse staffing hours, staff turnover, inspection deficiencies, fines and penalties, abuse and Special Focus Facility flags |

The dataset was chosen because it exceeds the size requirements, falls in the healthcare domain, and contains a natural target for every later deliverable:

- **Regression:** total nurse staffing hours per resident per day
- **Classification:** the CMS abuse icon
- **Clustering:** facility segments based on ownership, size, staffing, and quality
- **Association rules:** patterns among ownership, staffing, turnover, and rating levels

It also raises substantial ethical questions for the final deliverable, since it concerns a vulnerable population and the fairness of public quality ratings.

## Repository Contents

| Path | Description |
|---|---|
| `MSCS_634_ProjectDeliverable_1.ipynb` | Notebook with dataset description, cleaning, EDA, and insights |
| `data/raw/NH_ProviderInfo_Oct2026.csv` | Raw CMS file, unmodified |
| `data/nh_cleaned.csv` | Cleaned analysis dataset produced by the notebook |
| `requirements.txt` | Python packages needed to run the notebook |

## How to Run

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
jupyter notebook MSCS_634_ProjectDeliverable_1.ipynb
```

Run all cells from the top. The notebook reads the raw file and writes `data/nh_cleaned.csv`.

## Data Cleaning Steps

1. **Loading.** CCN and ZIP code were read as text. A default load reads ZIP codes as integers and drops the leading zero from 1,145 of them (for example, Connecticut ZIP 06095 becomes 6095).
2. **Duplicates.** No exact duplicate rows and no repeated facility IDs (CCN) were found. Sixteen rows share an address with another row, but they are 8 pairs of separately certified units with different CCNs, names, and bed counts, so all were kept. Repeated provider names (215 rows) are the same generic names used in different places.
3. **Inconsistent data.**
   - Provider, legal, city, and chain names were standardized: 5,126 provider names were not in consistent case, and 76 contained double spaces.
   - Seven Y/N text fields were converted to booleans.
   - Ownership type was split into a sector (for profit, non-profit, government) and a legal form.
   - The placeholder "Data Not Available" was converted to a missing value.
   - Text dates were parsed into dates.
4. **Missing values, handled by reason:**
   - **Explained by footnotes, left missing:** CMS footnote codes explain every missing rating and staffing value. The 126 missing overall ratings are 87 Special Focus Facilities that CMS does not rate (footnote 18) and 39 newly certified facilities (footnote 1). The 367 missing staffing values are facilities that did not submit staffing data or submitted data CMS could not validate. Ratings and the staffing target were **not imputed**, because that would invent quality judgments and training labels.
   - **Structurally blank, filled with a meaningful constant:** chain fields for independent facilities (30.4%), Special Focus status, and resident council.
   - **Too sparse, dropped:** infection-control citations (93.2% empty, and an empty cell does not mean zero).
   - **Feature gaps, imputed:** turnover measures (7.5% to 12.8% missing) were filled with the median of the facility's ownership sector, and a flag column records which values were imputed.
   - **Removed:** 66 rows (0.45%) with no bed count or census.
   - **Metadata, dropped:** the 15 footnote columns, after their information was used.
5. **Noisy data.**
   - **Occupancy:** 32 facilities reported more than 120% occupancy (one had 159.5 residents in 12 beds). Those occupancy values were set to missing, and the rest of each row was kept.
   - **Staffing:** 32 facilities reported implausible staffing, below 1.5 or above 12 hours per resident per day; the lowest reported value was 0.004 hours. Those staffing values were set to missing.
   - **High staffing outliers kept:** 709 facilities fall outside the 1.5 × IQR range for staffing but report realistic values of 5 to 11 hours, so they were kept.
   - **Log transforms:** added for the heavily skewed fines and inspection scores (fines skewness drops from 4.3 to 0.3).
6. **Column selection.** Identifiers, contact details, and redundant columns were removed. The 42 columns kept include two derived features, facility age and occupancy rate.

## Key Insights from the EDA

- **Ownership is the dominant pattern.** For-profit facilities provide a median of 3.61 nursing hours per resident per day, versus 4.28 for non-profits, about 40 minutes less care per resident every day. 21.7% of for-profit facilities have 1 star, compared with 9.9% of non-profits, and mean ratings are 2.87 versus 3.62.
- **Staffing and stability track quality.** From 1-star to 5-star facilities, median staffing rises from 3.47 to 4.15 hours, RN hours rise by about 70%, and annual nursing turnover falls from 52.1% to 38.1%.
- **The abuse icon is rare and imbalanced.** 9.5% of facilities carry it. The rate climbs with size, from 4.7% for facilities under 60 beds to 19.4% for those with 200 or more. Flagged facilities have twice the median recent deficiencies (14 vs. 7) and a median of $37,440 in fines versus $0.
- **Ratings and citations depend on the state.** Star ratings are spread almost evenly (19% to 22% per level), consistent with CMS grading inspections relative to other facilities in the same state. Abuse icon rates range from 0% (Maine, New Hampshire, Hawaii) to 28.8% (Illinois) and 25.2% (Colorado), suggesting differences in state inspection practice as well as in care.
- **Several features are nearly redundant.** Total and weekend staffing correlate at 0.97, and the overall and inspection ratings correlate at 0.89.

## How the Insights Guide the Next Deliverables

- **Regression:** predict `total_nurse_hprd`.
  - Drop the 340 rows without valid staffing data, and exclude the staffing and overall ratings, which are derived from staffing, to prevent leakage.
  - Test a log-transformed target, since staffing is right-skewed (skewness 2.45).
  - Compare linear regression with Ridge and Lasso, since several predictors are highly correlated.
- **Classification:** predict `abuse_icon`.
  - Use stratified splits and class weights, and evaluate with F1 and ROC/AUC rather than accuracy.
  - Compare models with and without inspection-derived features.
- **Clustering:** use standardized staffing, turnover, log fines, occupancy, size, and ratings, keeping one variable from each redundant group.
- **Association rules:** bin staffing, turnover, and ratings into levels, and combine them with ownership, chain, and size items.
- **Ethics:** cover state-relative grading, state differences in abuse citations, the profit motive in elder care, and the exclusion of unrated Special Focus Facilities.

## Challenges and How They Were Addressed

1. **Missing values that were not really missing.** Most blanks are concentrated in columns where a blank means "does not apply" or "explained in a footnote." Treating every blank as unknown would have produced misleading imputations. The footnote codes were matched against each missing value (for example, all 87 Special Focus Facilities line up exactly with footnote 18), and a treatment was chosen for each reason.
2. **Deciding what counts as noise.** A standard 1.5 × IQR rule flags 709 facilities for staffing, but most of them are legitimately well staffed. Only values outside any realistic operating range were treated as errors, and the strong-but-real outliers were kept, because they are part of the variation the regression model must explain.
3. **Silent type conversion.** Pandas silently dropped leading zeros from ZIP codes on a default load. The problem was caught by comparing the ZIP code column with the full address string, which kept the zeros, and was fixed by reading identifiers as text.
4. **Leakage risks.** Several columns are calculated from other columns. The staffing rating comes from staffing hours, the overall rating from the inspection and staffing ratings, and the abuse icon from inspection citations. These relationships were documented now, so that the modeling deliverables do not accidentally use an answer to predict itself.
