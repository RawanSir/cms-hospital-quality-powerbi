# US Hospital Performance & Patient Experience Dashboard
**Power BI Portfolio Project | Rawan Al-Sir**

---

## Project Overview

This project analyses real hospital performance and patient experience data published by the **Centers for Medicare & Medicaid Services (CMS)** — the US federal agency responsible for hospital quality reporting. The project demonstrates end-to-end business intelligence work: data sourcing, data modelling, DAX development, and dashboard design across two focused executive dashboards.

This dataset was chosen over synthetic alternatives specifically because every number is real, auditable, and defensible. CMS updates this data quarterly and it is used by healthcare executives, policy makers, and the public to evaluate hospital quality across the United States.

---

## Data Source

**Source:** [CMS Provider Data Catalog](https://data.cms.gov/provider-data)
**Coverage:** 5,432 hospitals across all US states and territories
**Period:** 2024 (Q1–Q2)

### Files Used

| File | Description | Role in Model |
|---|---|---|
| Hospital_General_Information.csv | Hospital name, location, type, ownership, overall rating | Dim_Hospital |
| Complications_and_Deaths-Hospital.csv | Mortality rates and complication scores vs national benchmarks | Fact_Complications |
| Timely_and_Effective_Care-Hospital.csv | Care timeliness scores by condition | Fact_TimelyCare |
| HCAHPS-Hospital.csv | Patient satisfaction survey results and star ratings | Fact_PatientSurvey |

---

## Data Model — Star Schema

The project uses a **star schema** with 3 fact tables and 4 dimension tables. All relationships are one-to-many with single-direction filtering from dimension to fact tables.

```
Dim_Hospital ─────────────── Fact_Complications
     │                              │
     │                         Dim_Measure
     │
     ├─────────────────────── Fact_PatientSurvey
     │
     └─────────────────────── Fact_TimelyCare
                                    │
                              Dim_Condition
```

### Modelling Decisions

**Why a star schema?**
Dimension tables act as the filter layer — stable, unique, and standardised. Fact tables store the transactional records. This separation ensures clean filter propagation across all visuals and slicers.

**Why Dim_Measure was extracted separately:**
Measure IDs and Names were repeated across every row in Fact_Complications. Extracting them into a dedicated dimension table eliminates redundancy and demonstrates proper normalisation.

**Why Dim_Condition was created separately from Dim_Measure:**
The Timely and Effective Care table has a Condition column (6 categories grouping multiple measures) and a Measure ID column. Adding Condition to Dim_Measure would introduce nulls for all Fact_Complications measures which don't have conditions. A separate Dim_Condition keeps both tables clean.

---

## Power Query — Data Cleaning Decisions

**Facility ID stored as Text:**
Facility IDs like "01014F" are identifiers, not numbers. Storing as text prevents leading zero loss (01014 → 1014) which would break all relationships across tables.

**Score and HCAHPS Answer Percent kept as Text:**
Both columns contain mixed values — numbers alongside "Not Available" text. Converting to numeric before handling the text values causes errors. Both are kept as text and handled in DAX using IFERROR(VALUE(...), BLANK()).

**EDV (Emergency Department Volume) discovered during cleaning:**
While cleaning Fact_TimelyCare, the Score column was found to contain both numeric values and categorical values (High/Medium/Low). Investigation revealed this was specific to one Measure ID — EDV — which measures emergency department patient volume as a relative rating rather than an absolute number. This measure is kept as text and visualised separately from numeric measures.

**Date parsing using locale:**
CMS dates are in MM/DD/YYYY US format. Power Query's default date parsing failed because the environment locale expected DD/MM/YYYY. Resolved using a custom column with Date.FromText([Date Column], [Format="MM/dd/yyyy"]).

**Timely and Effective Care included despite timeline pressure:**
Initially scoped out to save time, this table was included after reviewing its analytical value — it adds timeliness of care data across 6 condition categories which strengthens Dashboard 1 significantly.

---

## DAX Measures

### Hospital Quality Dashboard

```dax
-- Count all hospitals
Total Hospitals Numeric = COUNTROWS(Dim_Hospital)

-- Count only hospitals with a 1-5 star rating
Total Rated Hospitals = 
CALCULATE(
    COUNTROWS(Dim_Hospital),
    Dim_Hospital[Hospital overall rating] <> "Not Available"
)

-- Average mortality rate for a specific condition
-- Score is stored as text so VALUE() converts row by row
-- IFERROR handles "Not Available" text gracefully
Avg Mortality Rate - Heart Attack = 
CALCULATE(
    AVERAGEX(
        'Fact_Complications_and_Deaths-Hospital',
        IFERROR(VALUE('Fact_Complications_and_Deaths-Hospital'[Score]), BLANK())
    ),
    'Fact_Complications_and_Deaths-Hospital'[Measure ID] = "MORT_30_AMI"
)

-- Hospitals performance vs national average
Hospitals Better Than National = 
CALCULATE(
    COUNTROWS('Fact_Complications_and_Deaths-Hospital'),
    'Fact_Complications_and_Deaths-Hospital'[Compared to National] = "Better Than the National Rate"
)

-- Dynamic state ranking that responds to field parameter selection
-- Uses order number instead of text label to avoid composite key error
Dynamic State Rank = 
VAR SelectedMeasure = 
    SELECTEDVALUE('Mortality Measure'[Mortality Measure Order])
RETURN
RANKX(
    ALL(Dim_Hospital[State]),
    SWITCH(
        SelectedMeasure,
        1, [Avg Mortality Rate - Heart Attack],
        2, [Avg Mortality Rate - Heart Failure],
        3, [Avg Mortality Rate - Pneumonia],
        [Avg Mortality Rate - Heart Attack]
    ),
    ,
    DESC
)
```

### Patient Experience Dashboard

```dax
-- Overall summary star rating per hospital
-- H_STAR_RATING is CMS's pre-calculated overall summary score
-- One row per hospital, distinct from category-level ratings
Avg Overall Experience Rating = 
CALCULATE(
    AVERAGEX(
        Fact_PatientSurvey,
        IFERROR(VALUE(Fact_PatientSurvey[Patient Survey Star Rating]), BLANK())
    ),
    Fact_PatientSurvey[HCAHPS Measure ID] = "H_STAR_RATING"
)

-- Response rate averaged at hospital level to avoid row duplication
-- MAX per hospital because the value repeats across all question rows
Total Surveys Completed = 
SUMX(
    VALUES(Fact_PatientSurvey[Facility ID]),
    CALCULATE(MAX(Fact_PatientSurvey[Number of Completed Surveys]))
)
```

---

## Performance Category Column

A calculated column was added to Fact_Complications to consolidate the "Rate" and "Value" variants in the Compared to National column:

```dax
Performance Category = 
SWITCH(
    TRUE(),
    'Fact_Complications_and_Deaths-Hospital'[Compared to National] = "Better Than the National Rate" || 
    'Fact_Complications_and_Deaths-Hospital'[Compared to National] = "Better Than the National Value", 
    "Better Than National",
    ...
)
```

**Why:** CMS uses "Rate" for percentage-based measures and "Value" for index-based measures. Both mean the same thing for comparison purposes. Consolidating to 3 categories instead of 6 makes the visual immediately readable for an executive audience.

---

## Dashboard 1 — Hospital Quality

**Audience:** Healthcare executives, ministry of health analysts, policy makers

**Business Questions Answered:**
1. Which hospitals are rated best and worst overall?
2. How do mortality rates compare across hospitals for key conditions?
3. Which hospitals perform better, same, or worse than the national average?
4. Which states have the highest average mortality rates?

**Key Visuals:**
- KPI cards: Total Hospitals, Total Rated Hospitals, Avg Mortality Rate by condition
- Bar chart: Hospital distribution by overall rating (1-5)
- Bar chart: Hospital performance vs national average (color coded)
- Dynamic bar chart: Top 10 states by mortality rate with field parameter switching between conditions

**Color System:**
- Green (#16A34A): Better Than National
- Amber (#D97706): No Different Than National
- Red (#DC2626): Worse Than National

**Key Findings:**
- Most hospitals cluster at rating 3-4. Very few achieve a 5-star rating.
- The vast majority of hospitals perform no differently than the national average on clinical outcomes — genuine outliers in either direction are rare.
- Mississippi and Puerto Rico consistently show the highest mortality rates across heart attack, heart failure, and pneumonia conditions.

---

## Dashboard 2 — Patient Experience

**Audience:** Hospital administrators, patient experience teams, healthcare policy analysts

**Business Questions Answered:**
1. Which hospitals have the highest patient satisfaction scores?
2. What are patients most and least satisfied about?
3. How does satisfaction vary by hospital type and ownership?
4. Is there a relationship between survey response rate and satisfaction scores?

**Key Visuals:**
- KPI cards: Total Hospitals, Total Rated Hospitals, Avg Experience Rating, Avg Survey Response Rate, Total Surveys Completed
- Bar chart: Patient satisfaction by care category (color coded by performance threshold)
- Bar chart: Patient outcome measures — overall rating and recommendation
- Bar chart: Highest rated hospitals by patient experience
- Bar chart: Survey response rate by star rating

**Color Coding Logic for Care Categories:**
- Red (#DC2626): Average rating below 3.0 — needs immediate improvement
- Amber (#D97706): Average rating between 3.0 and 3.2 — underperforming
- Teal (#0891B2): Average rating above 3.2 — performing adequately

Thresholds were set based on the distribution of scores across all 8 categories. The national average sits at 3.29, so anything below 3.0 represents a significant gap.

**Key Findings:**
- Communication about Medicines scores 2.66 out of 5 — significantly below all other categories and the only one rated below 3.0 nationally. This is the single largest gap in the US patient experience data.
- Patients are most likely to recommend their hospital (3.64) even when specific experience aspects score lower — suggesting overall sentiment is driven by clinical outcomes more than operational factors.
- Survey response rate correlates positively with satisfaction score. 5-star hospitals have approximately 35% response rates vs 15% for 1-star hospitals. This could reflect either higher patient motivation to share positive experiences, or better patient engagement programs at higher-quality hospitals.
- The average survey response rate nationally is 22.64% — raising a data quality consideration. Insights are based on roughly 1 in 5 patients responding.

---

## Analytical Decisions Worth Noting

**Why CMS data over synthetic datasets:**
Synthetic healthcare datasets (like the commonly used Prasad Patil Kaggle dataset) have randomly generated values with no real patterns. CMS data contains genuine hospital performance information that can be defended in an interview and compared against publicly available benchmarks.

**Why Recommend Hospital and Overall Hospital Rating were separated from care categories:**
Nurse communication, doctor communication, cleanliness, and quietness are operational experience measures — things that happen during the hospital stay that hospitals can directly improve. Recommend Hospital and Overall Hospital Rating are outcome measures — patient verdicts on the total experience. Mixing them distorts the operational picture.

**Why the Top 10 Hospitals chart ranks by survey volume rather than rating:**
The H_STAR_RATING column uses a 1-5 whole number scale, creating many tied hospitals at 5 stars. Ranking by maximum survey count within the 4+ star filter selects the hospitals where the 5-star rating has the most statistical evidence behind it. A hospital with 10 responses and a 5-star rating is less meaningful than one with 2,000 responses and the same rating.

**Why the SWITCH approach was used for Dynamic State Rank instead of referencing the field parameter directly:**
Power BI field parameter tables use composite keys. SELECTEDVALUE on a composite key column throws an error. Using SELECTEDVALUE on the Order column (a simple integer key) and mapping to measures via SWITCH avoids the composite key issue entirely.

---

## Tools Used

- Power BI Desktop
- Power Query (M language) for data transformation
- DAX for measures and calculated columns
- DuckDB SQL (separate project) for exploratory analysis
- CMS Provider Data Catalog as data source

---

## Repository Structure

```
/
├── README.md
├── US_Hospital_Dashboard.pbix
├── data/
│   ├── Hospital_General_Information.csv
│   ├── Complications_and_Deaths-Hospital.csv
│   ├── Timely_and_Effective_Care-Hospital.csv
│   └── HCAHPS-Hospital.csv
└── screenshots/
    ├── dashboard_1_hospital_quality.png
    └── dashboard_2_patient_experience.png
```

---

## About

Built as part of a data analytics portfolio by Rawan Al-Sir.
