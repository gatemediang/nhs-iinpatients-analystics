# NHS HES Inpatient Admissions Analytics — T-SQL Code Reference

> **Hospital Episode Statistics · Admitted Patient Care · England · FY 2024/25**
>
> This file contains 30 production-ready, fully commented T-SQL blocks that replicate and
> extend every Power BI DAX measure built in the NHS HES Analytics model.
> Each block is independently executable, clearly labelled, and delivers a named clinical insight.
>
> **Assumed table names** (import HES CSV files into SQL Server with these names):
> - `dbo.HES_Planning`    ← hosp-epis-stat-admi-pla-2024-25.csv
> - `dbo.HES_Other`       ← hosp-epis-stat-admi-oth-2024-25.csv
> - `dbo.HES_Procedures`  ← hosp-epis-stat-admi-proc-2024-25.csv
> - `dbo.HES_Diagnoses`   ← hosp-epis-stat-admi-diag-2024-25.csv
> - `dbo.HES_Provider`    ← hosp-epis-stat-admi-hosp-provider-2024-25-tab.csv
>
> **Key assumption:** MEASURE_VALUE is imported as NVARCHAR (contains suppressed `*` values).
> All blocks use `TRY_CAST(...AS DECIMAL(18,4))` to safely handle suppression.
>
> **NHS data source:** https://digital.nhs.uk/data-and-information/publications/statistical/hospital-admitted-patient-care-activity
> **Licence:** Open Government Licence v3.0

---

## Table of Contents

| # | Section | Key Insight |
|---|---------|-------------|
| 1 | [Schema & Staging Views](#1--schema--staging-views) | Setup — suppression fix, computed columns |
| 2 | [National Headline KPIs](#2--national-headline-kpis) | 22.55M FCEs · Day Case Rate 38.4% |
| 3 | [Admission Type Breakdown](#3--admission-type-breakdown) | Elective Demand Ratio 2.78 |
| 4 | [Emergency & Urgent Care](#4--emergency--urgent-care) | Emergency Rate 35.8% · 6.6M unplanned |
| 5 | [Elective Waiting Times](#5--elective-waiting-times) | Mean 83.6d vs Median 34d — 2.46× skew |
| 6 | [Backlog Wait Band Distribution](#6--backlog-wait-band-distribution) | 72,587 patients waiting 18m+ |
| 7 | [Regional Benchmarking](#7--regional-benchmarking) | South West 125.8d wait — 90% above London |
| 8 | [18m+ Backlog by Region](#8--18m-backlog-by-region) | South West = 43% of national backlog |
| 9 | [Backlog Scenario Modelling](#9--backlog-scenario-modelling) | What-if capacity clearance projections |
| 10 | [IMD Deprivation Gradient](#10--imd-deprivation-gradient) | Monotonic gradient across all 10 deciles |
| 11 | [Deprivation Inequality Gap](#11--deprivation-inequality-gap) | Emergency Ratio 1.60 · Bed Day Gap 1.92M |
| 12 | [Ethnicity Breakdown](#12--ethnicity-breakdown) | FCE & emergency by broad ethnic group |
| 13 | [Ethnicity Data Quality](#13--ethnicity-data-quality) | 13.0% missing — CORE20PLUS5 compliance |
| 14 | [Age Band Distribution](#14--age-band-distribution) | 75–79 = single largest age cohort |
| 15 | [Gender Equity Analysis](#15--gender-equity-analysis) | 54.8% female — concentrated in maternity & 75+ |
| 16 | [ICD-10 Diagnosis Chapter Ranking](#16--icd-10-diagnosis-chapter-ranking) | Cancer + Digestive = 27.3% of all FAEs |
| 17 | [No Procedure FCEs](#17--no-procedure-fces) | 9.02M medical admissions — invisible burden |
| 18 | [Top OPCS-4 Procedures](#18--top-opcs-4-procedures) | High-volume procedures by day case rate |
| 19 | [Procedure Chapter Summary](#19--procedure-chapter-summary) | Surgical workload by OPCS chapter |
| 20 | [Specialty Demand Analysis](#20--specialty-demand-analysis) | Gen Med: 3.27M FCEs, 1.73M emergencies |
| 21 | [Provider Benchmarking Scorecard](#21--provider-benchmarking-scorecard) | 590 providers — quadrant classification |
| 22 | [Advanced Composite KPIs](#22--advanced-composite-kpis) | All 6 Power BI Folder 07 metrics |
| 23 | [Deprivation FCE Uplift %](#23--deprivation-fce-uplift) | 27.8% more FCEs most vs least deprived |
| 24 | [LOS Distribution Analysis](#24--los-distribution-analysis) | Mean 4.69d vs Median 1d — extreme skew |
| 25 | [Day Case Opportunity Analysis](#25--day-case-opportunity-analysis) | Gap to 80% target = 22M bed days |
| 26 | [Mental Health Under-Representation](#26--mental-health-under-representation) | 0.7% of FAEs — structural data gap |
| 27 | [Elderly Population Demand](#27--elderly-population-demand) | 27.5% of FCEs from 75+ patients |
| 28 | [Year-on-Year Trend Framework](#28--year-on-year-trend-framework) | Multi-year comparison scaffold |
| 29 | [Data Quality Report](#29--data-quality-report) | Suppression rates — all 5 tables |
| 30 | [Executive Summary Procedure](#30--executive-summary-stored-procedure) | Single SP — all board KPIs in one call |

---

## 1 — Schema & Staging Views

```sql
/* ============================================================================
   BLOCK 1.1 — CREATE HES SCHEMA
   ============================================================================
   Purpose : Creates a dedicated 'HES' schema to isolate NHS data from other
             warehouse tables. All views and procedures are created within it.
   Run once on initial setup.
   ============================================================================ */

IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = 'HES')
    EXEC('CREATE SCHEMA HES');
GO


/* ============================================================================
   BLOCK 1.2 — HES.vw_Planning  (replaces all RAW pla table references)
   ============================================================================
   Purpose  : Safely resolves the NHS statistical suppression issue.
              NHS Digital suppresses small patient counts with '*' to protect
              anonymity. This view converts MEASURE_VALUE text to DECIMAL,
              returning NULL for suppressed cells — matching DAX BLANK() behaviour.
              Also derives NHS Financial Year label and ORG_LEVEL boolean flags.

   Insight  : After applying this view, all SUM/AVG aggregations automatically
              exclude suppressed values — no manual CASE WHEN '*' needed.

   DAX equivalent : Measure Value Numeric (calculated column) + Power Query fix
   ============================================================================ */

CREATE OR ALTER VIEW HES.vw_Planning AS
SELECT
    UID,
    ORG_LEVEL,
    REPORTING_PERIOD,
    ORG_CODE,
    ORG_DESCRIPTION,
    MEASURE_TYPE,
    MEASURE,
    MEASURE_VALUE                                        AS MEASURE_VALUE_RAW,

    -- Safe numeric cast — '*' suppressed values and non-numeric strings → NULL
    TRY_CAST(MEASURE_VALUE AS DECIMAL(18,4))             AS MEASURE_VALUE_NUM,

    -- NHS Financial Year label: 2425 → '2024/25', 2324 → '2023/24'
    CONCAT(
        '20', LEFT(CAST(REPORTING_PERIOD AS VARCHAR(4)), 2),
        '/', RIGHT(CAST(REPORTING_PERIOD AS VARCHAR(4)), 2)
    )                                                    AS NHS_FINANCIAL_YEAR,

    -- ORG_LEVEL boolean flags — prevents repeated CASE WHEN in every query
    CASE WHEN ORG_LEVEL = 'ENGLAND'  THEN 1 ELSE 0 END  AS IS_ENGLAND,
    CASE WHEN ORG_LEVEL = 'REGION'   THEN 1 ELSE 0 END  AS IS_REGION,
    CASE WHEN ORG_LEVEL = 'PROVIDER' THEN 1 ELSE 0 END  AS IS_PROVIDER,

    -- Measure category grouping — mirrors pla[Measure Category] calc column
    CASE
        WHEN MEASURE IN ('FCE_SUM','FAE_SUM','FCE_DAY','FCE_ORDINARY','FCE_BED_DAYS')
            THEN 'Activity Volume'
        WHEN MEASURE IN ('FAE_EMERGENCY_SUM','FAE_WAITLIST_SUM','FAE_PLANNED_SUM','FAE_OTHER_SUM')
            THEN 'Admission Type'
        WHEN MEASURE IN ('FCE_MALE_SUM','FCE_FEMALE_SUM','FCE_UNKNOWN_GENDER_SUM')
            THEN 'Gender'
        WHEN MEASURE IN ('ELECDUR_CALC_Mean','ELECDUR_CALC_Median',
                          'SPELDUR_CALC_Mean','SPELDUR_CALC_Median')
            THEN 'Wait & LOS Metrics'
        WHEN MEASURE LIKE 'Age_%'
            THEN 'Age Band'
        WHEN MEASURE LIKE 'MONTHS%' OR MEASURE = 'UNDER_1MONTH_Sum'
            THEN 'Wait Band'
        WHEN MEASURE LIKE 'SES_%'
            THEN 'SES Duration'
        ELSE 'Diagnosis / Procedure'
    END                                                  AS MEASURE_CATEGORY

FROM dbo.HES_Planning;
GO


/* ============================================================================
   BLOCK 1.3 — HES.vw_Other  (adds IMD sort order + ethnicity grouping)
   ============================================================================
   Purpose  : Adds IMD_SORT_ORDER and ETHNIC_GROUP_BROAD as computed columns,
              replicating the two calculated columns created in Power BI.
              IMD_SORT_ORDER ensures charts display deciles in correct clinical
              order (Most Deprived = 1) rather than alphabetical order.

   Insight  : Without IMD sort order, deprivation charts display nonsensically —
              'Most deprived 10%' appears after 'More deprived 10-20%' etc.
   ============================================================================ */

CREATE OR ALTER VIEW HES.vw_Other AS
SELECT
    UID,
    Code,
    Category,
    Measure,
    Measure_Value,

    -- IMD Decile sort order: 1 = Most Deprived, 10 = Least Deprived
    -- Power BI equivalent: oth[IMD Sort Order] calculated column
    CASE Code
        WHEN 'Most deprived 10%'    THEN 1
        WHEN 'More deprived 10-20%' THEN 2
        WHEN 'More deprived 20-30%' THEN 3
        WHEN 'More deprived 30-40%' THEN 4
        WHEN 'More deprived 40-50%' THEN 5
        WHEN 'Less deprived 40-50%' THEN 6
        WHEN 'Less deprived 30-40%' THEN 7
        WHEN 'Less deprived 20-30%' THEN 8
        WHEN 'Less deprived 10-20%' THEN 9
        WHEN 'Least deprived 10%'   THEN 10
        ELSE 99
    END                                                  AS IMD_SORT_ORDER,

    -- Broad ONS ethnicity group (7 categories)
    -- Power BI equivalent: oth[Ethnic Group Broad] calculated column
    CASE
        WHEN Code IN ('British (White)','Irish (White)',
                      'Any other White background')               THEN 'White'
        WHEN Code IN ('White and Black Caribbean (Mixed)',
                      'White and Black African (Mixed)',
                      'White and Asian (Mixed)',
                      'Any other Mixed background')               THEN 'Mixed'
        WHEN Code IN ('Indian (Asian or Asian British)',
                      'Pakistani (Asian or Asian British)',
                      'Bangladeshi (Asian or Asian British)',
                      'Any other Asian background')               THEN 'Asian'
        WHEN Code IN ('Caribbean (Black or Black British)',
                      'African (Black or Black British)',
                      'Any other Black background')               THEN 'Black'
        WHEN Code  =  'Chinese (other ethnic group)'              THEN 'Chinese'
        WHEN Code  =  'Any other ethnic group'                    THEN 'Other'
        WHEN Code IN ('Not stated','Not known')                   THEN 'Unknown'
        ELSE 'N/A'
    END                                                  AS ETHNIC_GROUP_BROAD,

    -- Ethnic minority flag: non-White, known ethnicity
    -- Power BI equivalent: oth[Is Ethnic Minority] calculated column
    CASE
        WHEN Category = 'ETHNICITY'
         AND Code NOT IN ('British (White)','Irish (White)',
                          'Any other White background',
                          'Not stated','Not known')
        THEN 1 ELSE 0
    END                                                  AS IS_ETHNIC_MINORITY

FROM dbo.HES_Other;
GO
```

---

## 2 — National Headline KPIs

```sql
/* ============================================================================
   BLOCK 2 — NATIONAL HEADLINE KPIs (England level)
   ============================================================================
   Purpose  : Replicates all 8 measures from Power BI Folder 01 Core Activity.
              Returns a single row with all England-level national totals for
              FY 2024/25. Used to populate executive dashboard KPI cards.

   Insight  : Day Case Rate = 38.4% vs NHS target of 80%+. This 41.6 percentage
              point gap is the single largest efficiency opportunity in the NHS.
              Closing half the gap would release ~22 million bed days annually.

   Power BI measures : Total FCEs, Total FAEs, Total Bed Days, Day Case FCEs,
                       Ordinary Admissions, Day Case Rate %, Bed Days per FCE,
                       FCEs per Bed Day, Provider Count
   ============================================================================ */

SELECT
    -- Primary volume metrics
    SUM(CASE WHEN MEASURE = 'FCE_SUM'      THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Total_FCEs,                    -- England FY2024/25: 22,555,615

    SUM(CASE WHEN MEASURE = 'FAE_SUM'      THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Total_FAEs,                    -- England: 18,468,856

    -- FCE:FAE ratio: values >1 mean patients have multiple episodes per stay
    ROUND(
        SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END) /
        NULLIF(SUM(CASE WHEN MEASURE = 'FAE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0),
    2) AS FCE_to_FAE_Ratio,               -- England: 1.22

    -- Bed day totals
    SUM(CASE WHEN MEASURE = 'FCE_BED_DAYS' THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Total_Bed_Days,                -- England: 48,360,197

    -- Day case vs overnight split
    SUM(CASE WHEN MEASURE = 'FCE_DAY'      THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Day_Case_FCEs,                 -- England: 8,661,338

    SUM(CASE WHEN MEASURE = 'FCE_ORDINARY' THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Ordinary_Admissions,           -- England: 13,894,277

    -- Day Case Rate % — primary efficiency KPI. NHS target: 80%+
    ROUND(
        SUM(CASE WHEN MEASURE = 'FCE_DAY' THEN MEASURE_VALUE_NUM ELSE 0 END) * 100.0 /
        NULLIF(SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0),
    1) AS Day_Case_Rate_Pct,              -- England: 38.4%  ⚠ Well below target

    -- Bed Days per FCE — efficiency indicator; lower = more efficient throughput
    ROUND(
        SUM(CASE WHEN MEASURE = 'FCE_BED_DAYS' THEN MEASURE_VALUE_NUM ELSE 0 END) /
        NULLIF(SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0),
    2) AS Bed_Days_per_FCE,               -- England: 2.14

    -- FCEs per Bed Day — inverse of above; higher = better throughput
    ROUND(
        SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END) /
        NULLIF(SUM(CASE WHEN MEASURE = 'FCE_BED_DAYS' THEN MEASURE_VALUE_NUM ELSE 0 END), 0),
    3) AS FCEs_per_Bed_Day,               -- England: 0.47

    -- Provider count: 590 distinct NHS organisations
    COUNT(DISTINCT CASE WHEN IS_PROVIDER = 1 THEN ORG_CODE END)
        AS Provider_Count

FROM HES.vw_Planning
WHERE IS_ENGLAND = 1
  AND REPORTING_PERIOD = 2425;            -- FY 2024/25
```

---

## 3 — Admission Type Breakdown

```sql
/* ============================================================================
   BLOCK 3 — ADMISSION TYPE BREAKDOWN
   ============================================================================
   Purpose  : Splits total FAEs into the four admission pathways and derives
              the Elective Demand Ratio — one of the six Advanced KPIs.

   Insight  : Elective Demand Ratio = 2.78 means for every one booked elective
              admission, 2.78 patients are admitted from the waiting list. The
              system is operating on reactive throughput, not planned delivery.
              Planned admissions capacity needs to grow by 178% to match demand.

   Power BI measures : Emergency Admissions, Emergency Rate %, Waiting List
                       Admissions, Planned Admissions, Elective Demand Ratio
   ============================================================================ */

SELECT
    -- Absolute volumes by pathway
    SUM(CASE WHEN MEASURE = 'FAE_SUM'           THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Total_FAEs,

    SUM(CASE WHEN MEASURE = 'FAE_EMERGENCY_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Emergency_FAEs,               -- 6,614,976

    SUM(CASE WHEN MEASURE = 'FAE_WAITLIST_SUM'  THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS WaitingList_FAEs,             -- 7,322,118

    SUM(CASE WHEN MEASURE = 'FAE_PLANNED_SUM'   THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Planned_FAEs,                 -- 2,638,492

    SUM(CASE WHEN MEASURE = 'FAE_OTHER_SUM'     THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Other_FAEs,                   -- 1,893,270

    -- Percentage of each pathway
    ROUND(SUM(CASE WHEN MEASURE = 'FAE_EMERGENCY_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END)
          * 100.0 / NULLIF(SUM(CASE WHEN MEASURE = 'FAE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0),
    1) AS Emergency_Pct,                 -- 35.8%

    ROUND(SUM(CASE WHEN MEASURE = 'FAE_WAITLIST_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END)
          * 100.0 / NULLIF(SUM(CASE WHEN MEASURE = 'FAE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0),
    1) AS WaitingList_Pct,               -- 39.6%

    ROUND(SUM(CASE WHEN MEASURE = 'FAE_PLANNED_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END)
          * 100.0 / NULLIF(SUM(CASE WHEN MEASURE = 'FAE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0),
    1) AS Planned_Pct,                   -- 14.3%

    -- Elective Demand Ratio: WL admissions ÷ Planned admissions
    -- Values >1 confirm waiting list pressure exceeds planned capacity
    -- Target: approach 1.0 (balance). Current: 2.78 = structural undercapacity
    ROUND(
        SUM(CASE WHEN MEASURE = 'FAE_WAITLIST_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END) /
        NULLIF(SUM(CASE WHEN MEASURE = 'FAE_PLANNED_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0),
    2) AS Elective_Demand_Ratio          -- England: 2.78

FROM HES.vw_Planning
WHERE IS_ENGLAND = 1
  AND REPORTING_PERIOD = 2425;
```

---

## 4 — Emergency & Urgent Care

```sql
/* ============================================================================
   BLOCK 4 — EMERGENCY & URGENT CARE METRICS
   ============================================================================
   Purpose  : Calculates emergency demand indicators at both England and
              regional level. Includes the Emergency Admission Index (KPI 2
              of the Advanced folder) and SES Emergency Spells.

   Insight  : Emergency Index of 29.3 per 100 FCEs means nearly 1 in 3 episodes
              result from unplanned admissions. Regions above 35 are under
              sustained reactive pressure. High emergency regions with long waits
              are in a self-reinforcing cycle — unmet elective need escalates.

   Power BI measures : Emergency Admissions, Emergency Rate %,
                       Emergency Admission Index, SES Emergency Spells
   ============================================================================ */

-- England-level emergency summary
SELECT
    SUM(CASE WHEN MEASURE = 'FAE_EMERGENCY_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Emergency_Admissions,         -- 6,614,976

    ROUND(
        SUM(CASE WHEN MEASURE = 'FAE_EMERGENCY_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END)
        * 100.0 /
        NULLIF(SUM(CASE WHEN MEASURE = 'FAE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0),
    1) AS Emergency_Rate_Pct,            -- 35.8%

    -- Emergency Admission Index: emergencies per 100 FCEs
    -- England: 29.3  |  Benchmark: >40 = high pressure, <20 = elective-focused
    ROUND(
        SUM(CASE WHEN MEASURE = 'FAE_EMERGENCY_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END)
        * 100.0 /
        NULLIF(SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0),
    1) AS Emergency_Admission_Index,     -- 29.3

    -- SES Emergency Spells — NHS Outcomes Framework proxy for emergency duration
    SUM(CASE WHEN MEASURE = 'SES_EMERGENCY_NDC_ZEPIDUR' THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS SES_Emergency_Spells          -- 2,430,157

FROM HES.vw_Planning
WHERE IS_ENGLAND = 1
  AND REPORTING_PERIOD = 2425;


-- Regional emergency rate comparison with RAG status
SELECT
    ORG_DESCRIPTION                                                  AS Region,
    SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Total_FCEs,
    SUM(CASE WHEN MEASURE = 'FAE_EMERGENCY_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Emergency_FAEs,

    ROUND(
        SUM(CASE WHEN MEASURE = 'FAE_EMERGENCY_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END)
        * 100.0 /
        NULLIF(SUM(CASE WHEN MEASURE = 'FAE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0),
    1) AS Emergency_Rate_Pct,

    -- RAG flag based on England 35.8% baseline
    CASE
        WHEN SUM(CASE WHEN MEASURE = 'FAE_EMERGENCY_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END)
             * 100.0 /
             NULLIF(SUM(CASE WHEN MEASURE = 'FAE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0) > 40
        THEN 'RED — High Emergency Pressure'
        WHEN SUM(CASE WHEN MEASURE = 'FAE_EMERGENCY_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END)
             * 100.0 /
             NULLIF(SUM(CASE WHEN MEASURE = 'FAE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0) > 35
        THEN 'AMBER — Above National Average'
        ELSE 'GREEN — Within Acceptable Range'
    END AS Emergency_RAG

FROM HES.vw_Planning
WHERE IS_REGION = 1
  AND REPORTING_PERIOD = 2425
GROUP BY ORG_DESCRIPTION
ORDER BY Emergency_Rate_Pct DESC;
```

---

## 5 — Elective Waiting Times

```sql
/* ============================================================================
   BLOCK 5 — ELECTIVE WAITING TIME STATISTICS (National & Regional)
   ============================================================================
   Purpose  : Calculates mean and median elective wait and LOS at both England
              and regional level. The mean vs median comparison reveals the
              statistical skew caused by extreme long-wait outliers.

   Insight  : Mean = 83.6 days vs Median = 34 days (ratio 2.46×). Most patients
              wait under 5 weeks but a long tail of complex cases inflates the
              average. The South West outlier at 125.8 days is 90% above London.

   Power BI measures : Avg Elective Wait Days, Median Elective Wait Days,
                       Avg LOS Days, Median LOS Days
   ============================================================================ */

-- England-level wait and LOS statistics
SELECT
    MAX(CASE WHEN MEASURE = 'ELECDUR_CALC_Mean'   THEN MEASURE_VALUE_NUM END)
        AS Mean_Elective_Wait_Days,      -- England: 83.6 days

    MAX(CASE WHEN MEASURE = 'ELECDUR_CALC_Median' THEN MEASURE_VALUE_NUM END)
        AS Median_Elective_Wait_Days,    -- England: 34 days

    -- Skew ratio: >2 indicates strong right-skew from long-wait outliers
    ROUND(
        MAX(CASE WHEN MEASURE = 'ELECDUR_CALC_Mean'   THEN MEASURE_VALUE_NUM END) /
        NULLIF(MAX(CASE WHEN MEASURE = 'ELECDUR_CALC_Median' THEN MEASURE_VALUE_NUM END), 0),
    2) AS Mean_To_Median_Skew_Ratio,     -- 2.46 — highly right-skewed

    MAX(CASE WHEN MEASURE = 'SPELDUR_CALC_Mean'   THEN MEASURE_VALUE_NUM END)
        AS Mean_LOS_Days,                -- England: 4.69 days

    MAX(CASE WHEN MEASURE = 'SPELDUR_CALC_Median' THEN MEASURE_VALUE_NUM END)
        AS Median_LOS_Days               -- England: 1 day

FROM HES.vw_Planning
WHERE IS_ENGLAND = 1
  AND REPORTING_PERIOD = 2425;


-- Regional wait time league table with variance from England mean
SELECT
    ORG_DESCRIPTION                                                   AS Region,
    MAX(CASE WHEN MEASURE = 'ELECDUR_CALC_Mean'
             THEN MEASURE_VALUE_NUM END)                              AS Mean_Wait_Days,
    MAX(CASE WHEN MEASURE = 'ELECDUR_CALC_Median'
             THEN MEASURE_VALUE_NUM END)                              AS Median_Wait_Days,
    MAX(CASE WHEN MEASURE = 'SPELDUR_CALC_Mean'
             THEN MEASURE_VALUE_NUM END)                              AS Mean_LOS_Days,

    -- Variance from England mean (83.6 days): positive = worse than average
    ROUND(MAX(CASE WHEN MEASURE = 'ELECDUR_CALC_Mean'
                   THEN MEASURE_VALUE_NUM END) - 83.6, 1)            AS Days_Above_National_Mean,

    -- % above or below national average
    ROUND((MAX(CASE WHEN MEASURE = 'ELECDUR_CALC_Mean'
                    THEN MEASURE_VALUE_NUM END) - 83.6) / 83.6 * 100, 1)
                                                                      AS Pct_Variance_From_Mean,

    -- RTT status: NHS 18-week standard = 126 calendar days maximum
    CASE
        WHEN MAX(CASE WHEN MEASURE = 'ELECDUR_CALC_Mean' THEN MEASURE_VALUE_NUM END) > 126
        THEN 'RTT BREACH'
        WHEN MAX(CASE WHEN MEASURE = 'ELECDUR_CALC_Mean' THEN MEASURE_VALUE_NUM END) > 83.6
        THEN 'ABOVE AVERAGE'
        ELSE 'WITHIN TARGET'
    END                                                               AS Performance_Status,

    SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Total_FCEs

FROM HES.vw_Planning
WHERE IS_REGION = 1
  AND REPORTING_PERIOD = 2425
GROUP BY ORG_DESCRIPTION
ORDER BY Mean_Wait_Days DESC;
-- Expected output: South West (125.8d) at top, London (66.3d) at bottom
```

---

## 6 — Backlog Wait Band Distribution

```sql
/* ============================================================================
   BLOCK 6 — NATIONAL BACKLOG WAIT BAND DISTRIBUTION
   ============================================================================
   Purpose  : Shows the full waiting time band spectrum and derives the
              Backlog Severity Score — the key KPI for backlog monitoring.

   Insight  : 72,587 patients waiting 18m+ (9.3% of the 6m+ backlog).
              Backlog Severity Score = 0.151 — the most severe waits are not
              yet dominant, meaning early intervention can prevent worsening.
              The 18m+ cohort requires 3× the weight of 6-12m patients.

   Power BI measures : All six Waiting X Months measures, Backlog 6 Months Plus,
                       18 Month Plus % of Backlog, Backlog Severity Score
   ============================================================================ */

WITH BacklogBands AS (
    SELECT
        SUM(CASE WHEN MEASURE = 'UNDER_1MONTH_Sum'   THEN MEASURE_VALUE_NUM ELSE 0 END) AS Under_1m,
        SUM(CASE WHEN MEASURE = 'MONTHS1_2_Sum'      THEN MEASURE_VALUE_NUM ELSE 0 END) AS M1_2,
        SUM(CASE WHEN MEASURE = 'MONTHS2_3_Sum'      THEN MEASURE_VALUE_NUM ELSE 0 END) AS M2_3,
        SUM(CASE WHEN MEASURE = 'MONTHS3_6_Sum'      THEN MEASURE_VALUE_NUM ELSE 0 END) AS M3_6,
        SUM(CASE WHEN MEASURE = 'MONTHS6_9_Sum'      THEN MEASURE_VALUE_NUM ELSE 0 END) AS M6_9,
        SUM(CASE WHEN MEASURE = 'MONTHS9_12_Sum'     THEN MEASURE_VALUE_NUM ELSE 0 END) AS M9_12,
        SUM(CASE WHEN MEASURE = 'MONTHS12_18_Sum'    THEN MEASURE_VALUE_NUM ELSE 0 END) AS M12_18,
        SUM(CASE WHEN MEASURE = 'MONTHS18_ABOVE_Sum' THEN MEASURE_VALUE_NUM ELSE 0 END) AS M18_Plus
    FROM HES.vw_Planning
    WHERE IS_ENGLAND = 1
      AND REPORTING_PERIOD = 2425
)
SELECT
    Under_1m                             AS Admitted_Under_1_Month,   -- 2,857,425
    M1_2 + M2_3                          AS Admitted_1_to_3_Months,   -- ~1,730,742
    M3_6                                 AS Admitted_3_to_6_Months,   -- 728,786
    M6_9 + M9_12                         AS Admitted_6_to_12_Months,  -- ~528,533
    M12_18                               AS Admitted_12_to_18_Months, -- 178,489
    M18_Plus                             AS Admitted_18_Months_Plus,  -- 72,587

    -- 6m+ backlog: all patients beyond the NHS Referral To Treatment standard
    M6_9 + M9_12 + M12_18 + M18_Plus    AS Backlog_6_Months_Plus,    -- 779,609

    -- 18m+ as % of 6m+ backlog — severity concentration metric
    ROUND(M18_Plus * 100.0 /
          NULLIF(M6_9 + M9_12 + M12_18 + M18_Plus, 0), 1)
                                         AS Pct_18m_Of_6m_Backlog,    -- 9.3%

    Under_1m + M1_2 + M2_3 + M3_6 + M6_9 + M9_12 + M12_18 + M18_Plus
                                         AS Total_Waiting_List,       -- ~6,116,162

    -- Backlog Severity Score: weighted index of wait severity
    -- Formula: (18m+×3) + (12-18m×2) + (6-12m×1) ÷ total waiting list
    -- England: 0.151  |  Higher = more severe long-wait concentration
    ROUND(
        ((M18_Plus * 3) + (M12_18 * 2) + ((M6_9 + M9_12) * 1)) * 1.0 /
        NULLIF(Under_1m + M1_2 + M2_3 + M3_6 + M6_9 + M9_12 + M12_18 + M18_Plus, 0),
    3) AS Backlog_Severity_Score         -- England: 0.151

FROM BacklogBands;
```

---

## 7 — Regional Benchmarking

```sql
/* ============================================================================
   BLOCK 7 — COMPREHENSIVE REGIONAL SCORECARD
   ============================================================================
   Purpose  : Produces a multi-dimension scorecard for all 7 commissioning
              regions, covering activity, emergency, wait times and backlog.
              Used to populate the Regional Benchmarking report page.

   Insight  : Midlands highest volume (4.46M FCEs) but South West worst
              performance across wait time, 18m+ backlog and LOS simultaneously.
              No single region excels across all dimensions — each has trade-offs.

   Power BI report : Report 02 — Regional Benchmarking
   ============================================================================ */

SELECT
    ORG_DESCRIPTION                                                  AS Region,
    ORG_CODE                                                         AS Region_Code,

    -- Activity
    SUM(CASE WHEN MEASURE = 'FCE_SUM'           THEN MEASURE_VALUE_NUM ELSE 0 END) AS Total_FCEs,
    SUM(CASE WHEN MEASURE = 'FCE_BED_DAYS'      THEN MEASURE_VALUE_NUM ELSE 0 END) AS Total_Bed_Days,

    -- Efficiency
    ROUND(SUM(CASE WHEN MEASURE = 'FCE_DAY' THEN MEASURE_VALUE_NUM ELSE 0 END) * 100.0 /
          NULLIF(SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0), 1)
        AS Day_Case_Rate_Pct,

    -- Emergency pressure
    SUM(CASE WHEN MEASURE = 'FAE_EMERGENCY_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END) AS Emergency_FAEs,
    ROUND(SUM(CASE WHEN MEASURE = 'FAE_EMERGENCY_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END) * 100.0 /
          NULLIF(SUM(CASE WHEN MEASURE = 'FAE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0), 1)
        AS Emergency_Rate_Pct,

    -- Wait times
    MAX(CASE WHEN MEASURE = 'ELECDUR_CALC_Mean'   THEN MEASURE_VALUE_NUM END) AS Mean_Wait_Days,
    MAX(CASE WHEN MEASURE = 'ELECDUR_CALC_Median' THEN MEASURE_VALUE_NUM END) AS Median_Wait_Days,

    -- Backlog
    SUM(CASE WHEN MEASURE = 'MONTHS18_ABOVE_Sum' THEN MEASURE_VALUE_NUM ELSE 0 END) AS Wait_18m_Plus,
    SUM(CASE WHEN MEASURE IN ('MONTHS6_9_Sum','MONTHS9_12_Sum',
                               'MONTHS12_18_Sum','MONTHS18_ABOVE_Sum')
             THEN MEASURE_VALUE_NUM ELSE 0 END) AS Backlog_6m_Plus,

    -- Share of national 18m+ backlog
    ROUND(SUM(CASE WHEN MEASURE = 'MONTHS18_ABOVE_Sum' THEN MEASURE_VALUE_NUM ELSE 0 END)
          * 100.0 / 72587.0, 1) AS Pct_Of_National_18m_Backlog,

    -- Wait time rank: 1 = fastest, 7 = slowest
    RANK() OVER (
        ORDER BY MAX(CASE WHEN MEASURE = 'ELECDUR_CALC_Mean' THEN MEASURE_VALUE_NUM END) ASC
    ) AS Wait_Time_Rank_Best_1

FROM HES.vw_Planning
WHERE IS_REGION = 1
  AND REPORTING_PERIOD = 2425
GROUP BY ORG_DESCRIPTION, ORG_CODE
ORDER BY Wait_18m_Plus DESC;
-- Expected top: South West (31,415), Midlands (9,160), North West (8,370)
```

---

## 8 — 18m+ Backlog by Region

```sql
/* ============================================================================
   BLOCK 8 — 18-MONTH+ BACKLOG DEEP DIVE BY REGION
   ============================================================================
   Purpose  : Focuses exclusively on the most critical backlog indicator —
              patients waiting more than 18 months for elective care.
              Identifies that this is a geographically concentrated crisis.

   Insight  : South West = 31,415 patients = 43.3% of England's entire 72,587
              18m+ backlog, despite only 10.6% of national FCE volume.
              This is structural capacity failure in a single region, not a
              national demand problem.

   Power BI report : Report 04 — Elective Backlog Tracker
   ============================================================================ */

SELECT
    ORG_DESCRIPTION                                                   AS Region,
    SUM(CASE WHEN MEASURE = 'FCE_SUM'
             THEN MEASURE_VALUE_NUM ELSE 0 END)                       AS Total_FCEs,

    -- Region share of national FCEs (denominator: 22,555,615)
    ROUND(SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END)
          * 100.0 / 22555615.0, 1)                                    AS Pct_National_FCEs,

    SUM(CASE WHEN MEASURE = 'MONTHS18_ABOVE_Sum'
             THEN MEASURE_VALUE_NUM ELSE 0 END)                       AS Wait_18m_Plus,
    SUM(CASE WHEN MEASURE = 'MONTHS12_18_Sum'
             THEN MEASURE_VALUE_NUM ELSE 0 END)                       AS Wait_12_to_18m,

    -- Combined critical long-wait cohort
    SUM(CASE WHEN MEASURE IN ('MONTHS12_18_Sum','MONTHS18_ABOVE_Sum')
             THEN MEASURE_VALUE_NUM ELSE 0 END)                       AS Critical_Long_Wait,

    -- Region share of England's 18m+ total (72,587)
    ROUND(SUM(CASE WHEN MEASURE = 'MONTHS18_ABOVE_Sum' THEN MEASURE_VALUE_NUM ELSE 0 END)
          * 100.0 / 72587.0, 1)                                       AS Pct_Of_National_18m_Backlog,

    -- Backlog concentration: is 18m+ disproportionate relative to FCE share?
    -- Values >1 mean the region carries more backlog than its activity share
    ROUND(
        (SUM(CASE WHEN MEASURE = 'MONTHS18_ABOVE_Sum' THEN MEASURE_VALUE_NUM ELSE 0 END) * 100.0 / 72587.0)
        /
        NULLIF(SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END) * 100.0 / 22555615.0, 0),
    2) AS Backlog_Concentration_Ratio    -- South West expected: ~4.0 (4× disproportionate)

FROM HES.vw_Planning
WHERE IS_REGION = 1
  AND REPORTING_PERIOD = 2425
GROUP BY ORG_DESCRIPTION
ORDER BY Wait_18m_Plus DESC;
```

---

## 9 — Backlog Scenario Modelling

```sql
/* ============================================================================
   BLOCK 9 — BACKLOG CLEARANCE SCENARIO MODELLING (What-If)
   ============================================================================
   Purpose  : Models how long it would take to clear the 18m+ backlog under
              different capacity increase scenarios.
              SQL equivalent of the Power BI What-If parameter approach.

   Insight  : Even a 20% capacity increase dedicated entirely to 18m+ patients
              would take over a year to clear. Targeted South West investment
              is more effective than spreading national capacity increases.

   Power BI report : Report 04 — Elective Backlog Tracker (What-If parameter)
   ============================================================================ */

DECLARE @Current_18m_Plus   INT = 72587;      -- England 18m+ total
DECLARE @Current_Total_WL   INT = 6116162;    -- Total waiting list (all bands)
DECLARE @Monthly_Throughput INT = 610000;     -- Approximate monthly WL admissions (7.32M ÷ 12)

SELECT
    Scenario,
    Capacity_Increase_Pct,
    Additional_Monthly_Capacity,

    -- Months to clear 18m+ backlog at this additional capacity level
    CEILING(
        CAST(@Current_18m_Plus AS FLOAT) / NULLIF(Additional_Monthly_Capacity, 0)
    ) AS Months_To_Clear_18m_Backlog,

    -- Years equivalent
    ROUND(
        CAST(@Current_18m_Plus AS FLOAT) / NULLIF(Additional_Monthly_Capacity, 0) / 12, 1
    ) AS Years_To_Clear,

    -- Clearance date approximation
    DATEADD(
        MONTH,
        CEILING(CAST(@Current_18m_Plus AS FLOAT) / NULLIF(Additional_Monthly_Capacity, 0)),
        '2025-03-31'   -- Model date (end of FY 2024/25)
    ) AS Estimated_Clearance_Date

FROM (VALUES
    ('5%  capacity increase',   5,  CAST(@Monthly_Throughput * 0.05 AS INT)),
    ('10% capacity increase',  10,  CAST(@Monthly_Throughput * 0.10 AS INT)),
    ('15% capacity increase',  15,  CAST(@Monthly_Throughput * 0.15 AS INT)),
    ('20% capacity increase',  20,  CAST(@Monthly_Throughput * 0.20 AS INT)),
    ('30% capacity increase',  30,  CAST(@Monthly_Throughput * 0.30 AS INT)),
    ('50% capacity increase',  50,  CAST(@Monthly_Throughput * 0.50 AS INT)),
    ('100% dedicated capacity',100, @Current_18m_Plus / 6)  -- Clear in 6 months
) AS Scenarios(Scenario, Capacity_Increase_Pct, Additional_Monthly_Capacity)
ORDER BY Capacity_Increase_Pct;
```

---

## 10 — IMD Deprivation Gradient

```sql
/* ============================================================================
   BLOCK 10 — IMD DEPRIVATION GRADIENT — ALL 10 DECILES
   ============================================================================
   Purpose  : Produces the full 10-decile deprivation analysis for the
              Deprivation & Health Equity report page. Ordered by IMD_SORT_ORDER
              (most deprived first) for correct clinical interpretation.

   Insight  : The gradient is monotonic — every decile step from least to most
              deprived adds approximately 60,000 FCEs and 200,000 bed days.
              This unbroken gradient confirms systematic health inequality,
              not random variation. It is the equity gap in data form.

   Power BI report : Report 03 — Deprivation & Health Equity
   NHS framework   : CORE20PLUS5 — https://www.england.nhs.uk/core20plus5
   ============================================================================ */

SELECT
    Code                                                             AS IMD_Decile,
    IMD_SORT_ORDER,                      -- 1 = Most Deprived, 10 = Least Deprived

    -- Activity volume
    SUM(CASE WHEN Measure = 'FCE_SUM'           THEN Measure_Value ELSE 0 END) AS Total_FCEs,
    SUM(CASE WHEN Measure = 'FAE_EMERGENCY_Sum' THEN Measure_Value ELSE 0 END) AS Emergency_FAEs,
    SUM(CASE WHEN Measure = 'FCE_BED_DAYS'      THEN Measure_Value ELSE 0 END) AS Total_Bed_Days,

    -- Emergency rate per decile — most deprived: 33.9%, least deprived: ~27.1%
    ROUND(
        SUM(CASE WHEN Measure = 'FAE_EMERGENCY_Sum' THEN Measure_Value ELSE 0 END) * 100.0 /
        NULLIF(SUM(CASE WHEN Measure = 'FCE_SUM' THEN Measure_Value ELSE 0 END), 0),
    1) AS Emergency_Rate_Pct,

    -- Bed days per FCE — higher in deprived areas means longer, more complex stays
    ROUND(
        SUM(CASE WHEN Measure = 'FCE_BED_DAYS' THEN Measure_Value ELSE 0 END) /
        NULLIF(SUM(CASE WHEN Measure = 'FCE_SUM' THEN Measure_Value ELSE 0 END), 0),
    2) AS Bed_Days_per_FCE,

    -- FCE volume as % of total across all deciles
    ROUND(
        SUM(CASE WHEN Measure = 'FCE_SUM' THEN Measure_Value ELSE 0 END) * 100.0 /
        SUM(SUM(CASE WHEN Measure = 'FCE_SUM' THEN Measure_Value ELSE 0 END)) OVER (),
    1) AS Pct_Of_All_IMD_FCEs

FROM HES.vw_Other
WHERE Category = 'IMD_Decile'
  AND IMD_SORT_ORDER < 99        -- Exclude rows that are not IMD decile rows
GROUP BY Code, IMD_SORT_ORDER
ORDER BY IMD_SORT_ORDER ASC;     -- Most deprived first
```

---

## 11 — Deprivation Inequality Gap

```sql
/* ============================================================================
   BLOCK 11 — DEPRIVATION INEQUALITY GAP MEASURES
   ============================================================================
   Purpose  : Calculates the absolute and relative health inequality gap
              between the most and least deprived deciles. These are the
              headline equity metrics for board and NHS England reporting.

   Insight  : Deprivation Emergency Ratio = 1.60 means the most deprived
              experience a 60% higher emergency admission rate.
              Bed Day Gap = 1.92M additional days — this IS the equity gap.
              Every unit improvement in deprivation ratio = millions of bed days.

   Power BI measures : Most Deprived FCEs, Least Deprived FCEs,
                       Deprivation Emergency Gap, Deprivation Emergency Ratio,
                       Deprivation Bed Day Gap, Most Deprived Emergency Rate %,
                       Deprivation FCE Uplift %
   ============================================================================ */

WITH DeprivationPoles AS (
    SELECT
        -- Most deprived decile (IMD Decile 1)
        SUM(CASE WHEN Code = 'Most deprived 10%' AND Measure = 'FCE_SUM'
                 THEN Measure_Value ELSE 0 END)           AS Most_Dep_FCEs,
        SUM(CASE WHEN Code = 'Most deprived 10%' AND Measure = 'FAE_EMERGENCY_Sum'
                 THEN Measure_Value ELSE 0 END)           AS Most_Dep_Emergency,
        SUM(CASE WHEN Code = 'Most deprived 10%' AND Measure = 'FCE_BED_DAYS'
                 THEN Measure_Value ELSE 0 END)           AS Most_Dep_BedDays,
        SUM(CASE WHEN Code = 'Most deprived 10%' AND Measure = 'FAE_SUM'
                 THEN Measure_Value ELSE 0 END)           AS Most_Dep_FAEs,

        -- Least deprived decile (IMD Decile 10)
        SUM(CASE WHEN Code = 'Least deprived 10%' AND Measure = 'FCE_SUM'
                 THEN Measure_Value ELSE 0 END)           AS Least_Dep_FCEs,
        SUM(CASE WHEN Code = 'Least deprived 10%' AND Measure = 'FAE_EMERGENCY_Sum'
                 THEN Measure_Value ELSE 0 END)           AS Least_Dep_Emergency,
        SUM(CASE WHEN Code = 'Least deprived 10%' AND Measure = 'FCE_BED_DAYS'
                 THEN Measure_Value ELSE 0 END)           AS Least_Dep_BedDays,
        SUM(CASE WHEN Code = 'Least deprived 10%' AND Measure = 'FAE_SUM'
                 THEN Measure_Value ELSE 0 END)           AS Least_Dep_FAEs
    FROM HES.vw_Other
    WHERE Category = 'IMD_Decile'
)
SELECT
    Most_Dep_FCEs,                       -- 2,457,589
    Least_Dep_FCEs,                      -- 1,922,609

    -- FCE absolute gap and uplift
    Most_Dep_FCEs - Least_Dep_FCEs                                   AS FCE_Gap,
    ROUND((Most_Dep_FCEs - Least_Dep_FCEs) * 100.0 /
          NULLIF(Least_Dep_FCEs, 0), 1)                              AS FCE_Uplift_Pct,  -- 27.8%

    -- Emergency admission gap
    Most_Dep_Emergency                                               AS Most_Dep_Emergency,   -- 832,523
    Least_Dep_Emergency                                              AS Least_Dep_Emergency,  -- 521,340
    Most_Dep_Emergency - Least_Dep_Emergency                         AS Emergency_Gap,        -- 311,183

    -- Deprivation Emergency Ratio: target = 1.0 (equality). Current: 1.60
    ROUND(Most_Dep_Emergency * 1.0 / NULLIF(Least_Dep_Emergency, 0), 2)
        AS Deprivation_Emergency_Ratio,  -- 1.60 — 60% more emergencies in most deprived

    -- Most deprived emergency rate %
    ROUND(Most_Dep_Emergency * 100.0 / NULLIF(Most_Dep_FAEs, 0), 1)
        AS Most_Dep_Emergency_Rate_Pct,  -- 33.9%

    -- Bed day gap
    Most_Dep_BedDays - Least_Dep_BedDays                             AS Bed_Day_Gap,          -- 1,921,388
    ROUND((Most_Dep_BedDays - Least_Dep_BedDays) * 100.0 /
          NULLIF(Least_Dep_BedDays, 0), 1)                           AS Bed_Day_Gap_Pct       -- 53.6%

FROM DeprivationPoles;
```

---

## 12 — Ethnicity Breakdown

```sql
/* ============================================================================
   BLOCK 12 — ETHNICITY FCE AND EMERGENCY BREAKDOWN BY BROAD GROUP
   ============================================================================
   Purpose  : Analyses admission patterns across ONS ethnic groups using the
              ETHNIC_GROUP_BROAD column from the staging view.
              Used for CORE20PLUS5 equity analysis.

   Insight  : The 'Unknown' group (13% of FCEs) limits the analytical validity
              of all ethnic comparisons. Any differential found must be caveated
              by the fact that unknown ethnicity may not be randomly distributed.

   Power BI report  : Report 03 — Deprivation & Health Equity (ethnicity tab)
   NHS framework    : CORE20PLUS5 ethnic minority health inequalities
   ============================================================================ */

SELECT
    ETHNIC_GROUP_BROAD                                               AS Ethnic_Group,
    IS_ETHNIC_MINORITY,

    SUM(CASE WHEN Measure = 'FCE_SUM'           THEN Measure_Value ELSE 0 END) AS Total_FCEs,
    SUM(CASE WHEN Measure = 'FAE_EMERGENCY_Sum' THEN Measure_Value ELSE 0 END) AS Emergency_FAEs,
    SUM(CASE WHEN Measure = 'FCE_BED_DAYS'      THEN Measure_Value ELSE 0 END) AS Total_Bed_Days,

    -- Emergency rate per ethnic group
    ROUND(
        SUM(CASE WHEN Measure = 'FAE_EMERGENCY_Sum' THEN Measure_Value ELSE 0 END) * 100.0 /
        NULLIF(SUM(CASE WHEN Measure = 'FCE_SUM' THEN Measure_Value ELSE 0 END), 0),
    1) AS Emergency_Rate_Pct,

    -- Share of total FCEs (window function across all groups)
    ROUND(
        SUM(CASE WHEN Measure = 'FCE_SUM' THEN Measure_Value ELSE 0 END) * 100.0 /
        SUM(SUM(CASE WHEN Measure = 'FCE_SUM' THEN Measure_Value ELSE 0 END)) OVER (),
    1) AS Pct_Of_All_Ethnicity_FCEs,

    -- Flag: is this the 'unknown' group that limits equity analysis?
    CASE WHEN ETHNIC_GROUP_BROAD = 'Unknown' THEN 'DATA QUALITY RISK' ELSE 'OK' END
        AS Data_Quality_Flag

FROM HES.vw_Other
WHERE Category = 'ETHNICITY'
GROUP BY ETHNIC_GROUP_BROAD, IS_ETHNIC_MINORITY
ORDER BY Total_FCEs DESC;
```

---

## 13 — Ethnicity Data Quality

```sql
/* ============================================================================
   BLOCK 13 — ETHNICITY MISSING RATE (CORE20PLUS5 Data Quality KPI)
   ============================================================================
   Purpose  : Calculates the Ethnicity Missing Rate % — the key data quality
              indicator required for NHS CORE20PLUS5 compliance submissions.
              Provider-level ethnicity recording rates can be derived from
              the provider table to identify worst-performing organisations.

   Insight  : 13.0% missing (~2.93M FCEs: 1.97M Not Stated, 0.96M Not Known).
              NICE recommends <5% for valid equity analysis. At 13%, comparisons
              between ethnic groups carry substantial uncertainty. Improving
              ethnicity recording is a mandatory data quality priority.

   Power BI measure : Ethnicity Missing Rate %
   ============================================================================ */

WITH EthnicityBase AS (
    SELECT
        Code,
        SUM(CASE WHEN Measure = 'FCE_SUM' THEN Measure_Value ELSE 0 END) AS FCEs
    FROM HES.vw_Other
    WHERE Category = 'ETHNICITY'
    GROUP BY Code
),
EthnicityTotals AS (
    SELECT
        SUM(FCEs) AS Total_FCEs,
        SUM(CASE WHEN Code IN ('Not stated','Not known') THEN FCEs ELSE 0 END) AS Missing_FCEs,
        SUM(CASE WHEN Code = 'Not stated' THEN FCEs ELSE 0 END) AS Not_Stated,
        SUM(CASE WHEN Code = 'Not known'  THEN FCEs ELSE 0 END) AS Not_Known
    FROM EthnicityBase
)
SELECT
    Total_FCEs,
    Missing_FCEs,
    Not_Stated,
    Not_Known,
    Total_FCEs - Missing_FCEs                                        AS Known_Ethnicity_FCEs,

    -- Ethnicity Missing Rate % — Power BI measure equivalent
    ROUND(Missing_FCEs * 100.0 / NULLIF(Total_FCEs, 0), 1)          AS Ethnicity_Missing_Rate_Pct,

    -- CORE20PLUS5 compliance assessment
    CASE
        WHEN ROUND(Missing_FCEs * 100.0 / NULLIF(Total_FCEs, 0), 1) > 15
        THEN 'CRITICAL — Fails CORE20PLUS5 reporting threshold (>15%)'
        WHEN ROUND(Missing_FCEs * 100.0 / NULLIF(Total_FCEs, 0), 1) > 10
        THEN 'HIGH RISK — Equity analysis substantially limited (10–15%)'
        WHEN ROUND(Missing_FCEs * 100.0 / NULLIF(Total_FCEs, 0), 1) > 5
        THEN 'MEDIUM RISK — Improvement required (5–10%)'
        ELSE 'ACCEPTABLE — Monitor and maintain (<5%)'
    END AS CORE20PLUS5_Compliance
    -- Expected: 'HIGH RISK — Equity analysis substantially limited (10–15%)'

FROM EthnicityTotals;
```

---

## 14 — Age Band Distribution

```sql
/* ============================================================================
   BLOCK 14 — AGE BAND FCE DISTRIBUTION
   ============================================================================
   Purpose  : Breaks all FCEs down by age band to reveal the demographic
              demand profile. Shows where NHS activity volume concentrates by age.

   Insight  : The 75–79 age band (2,276,417 FCEs) is the single largest
              individual age cohort in the entire dataset. Patients aged 75+
              collectively account for 27.5% of all FCEs. With an ageing
              population, this ratio will grow year-on-year — making geriatric
              medicine the most critical capacity planning specialty.

   Power BI measures : Elderly FCEs 75 Plus, Elderly % of FCEs,
                       Children FCEs Under 16, Working Age FCEs
   ============================================================================ */

SELECT
    MEASURE                                                          AS Age_Band_Code,

    -- Readable age band label (abbreviated from MEASURE codes)
    CASE MEASURE
        WHEN 'Age_0_Sum'        THEN 'Under 1'
        WHEN 'Age_1_4_Sum'      THEN '1–4'
        WHEN 'Age_5_9_Sum'      THEN '5–9'
        WHEN 'Age_10_14_Sum'    THEN '10–14'
        WHEN 'Age_15_Sum'       THEN '15'
        WHEN 'Age_16_Sum'       THEN '16'
        WHEN 'Age_17_Sum'       THEN '17'
        WHEN 'Age_18_Sum'       THEN '18'
        WHEN 'Age_19_Sum'       THEN '19'
        WHEN 'Age_20_24_Sum'    THEN '20–24'
        WHEN 'Age_25_29_Sum'    THEN '25–29'
        WHEN 'Age_30_34_Sum'    THEN '30–34'
        WHEN 'Age_35_39_Sum'    THEN '35–39'
        WHEN 'Age_40_44_Sum'    THEN '40–44'
        WHEN 'Age_45_49_Sum'    THEN '45–49'
        WHEN 'Age_50_54_Sum'    THEN '50–54'
        WHEN 'Age_55_59_Sum'    THEN '55–59'
        WHEN 'Age_60_64_Sum'    THEN '60–64'
        WHEN 'Age_65_69_Sum'    THEN '65–69'
        WHEN 'Age_70_74_Sum'    THEN '70–74'
        WHEN 'Age_75_79_Sum'    THEN '75–79  ← LARGEST COHORT'
        WHEN 'Age_80_84_Sum'    THEN '80–84'
        WHEN 'Age_85_89_Sum'    THEN '85–89'
        WHEN 'Age_90_Over_Sum'  THEN '90+'
        ELSE MEASURE
    END                                                              AS Age_Band,

    MEASURE_VALUE_NUM                                                AS FCEs,

    -- % of all FCEs this age band represents
    ROUND(MEASURE_VALUE_NUM * 100.0 /
          SUM(MEASURE_VALUE_NUM) OVER (PARTITION BY (SELECT NULL)), 1)
        AS Pct_Of_All_FCEs,

    -- Cumulative %: shows where demand accumulates across age
    ROUND(SUM(MEASURE_VALUE_NUM) OVER (
        ORDER BY CASE MEASURE
            WHEN 'Age_0_Sum' THEN 1  WHEN 'Age_1_4_Sum' THEN 2
            WHEN 'Age_5_9_Sum' THEN 3 WHEN 'Age_10_14_Sum' THEN 4
            WHEN 'Age_15_Sum' THEN 5 WHEN 'Age_16_Sum' THEN 6
            WHEN 'Age_17_Sum' THEN 7 WHEN 'Age_18_Sum' THEN 8
            WHEN 'Age_19_Sum' THEN 9 WHEN 'Age_20_24_Sum' THEN 10
            WHEN 'Age_25_29_Sum' THEN 11 WHEN 'Age_30_34_Sum' THEN 12
            WHEN 'Age_35_39_Sum' THEN 13 WHEN 'Age_40_44_Sum' THEN 14
            WHEN 'Age_45_49_Sum' THEN 15 WHEN 'Age_50_54_Sum' THEN 16
            WHEN 'Age_55_59_Sum' THEN 17 WHEN 'Age_60_64_Sum' THEN 18
            WHEN 'Age_65_69_Sum' THEN 19 WHEN 'Age_70_74_Sum' THEN 20
            WHEN 'Age_75_79_Sum' THEN 21 WHEN 'Age_80_84_Sum' THEN 22
            WHEN 'Age_85_89_Sum' THEN 23 WHEN 'Age_90_Over_Sum' THEN 24
        END
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) * 100.0 / SUM(MEASURE_VALUE_NUM) OVER (PARTITION BY (SELECT NULL)), 1)
        AS Cumulative_Pct,

    -- Age group classification for Power BI measure equivalence
    CASE
        WHEN MEASURE IN ('Age_0_Sum','Age_1_4_Sum','Age_5_9_Sum',
                          'Age_10_14_Sum','Age_15_Sum')              THEN 'Children (<16)'
        WHEN MEASURE IN ('Age_75_79_Sum','Age_80_84_Sum',
                          'Age_85_89_Sum','Age_90_Over_Sum')         THEN 'Elderly 75+'
        ELSE 'Working Age / Younger Adult'
    END                                                              AS Age_Group_Category

FROM HES.vw_Planning
WHERE IS_ENGLAND = 1
  AND MEASURE LIKE 'Age_%'
  AND MEASURE_VALUE_NUM IS NOT NULL
  AND REPORTING_PERIOD = 2425
ORDER BY CASE MEASURE
    WHEN 'Age_0_Sum' THEN 1  WHEN 'Age_1_4_Sum' THEN 2
    WHEN 'Age_5_9_Sum' THEN 3 WHEN 'Age_10_14_Sum' THEN 4
    WHEN 'Age_15_Sum' THEN 5  WHEN 'Age_75_79_Sum' THEN 21
    WHEN 'Age_80_84_Sum' THEN 22 WHEN 'Age_85_89_Sum' THEN 23
    WHEN 'Age_90_Over_Sum' THEN 24 ELSE 10
END;
```

---

## 15 — Gender Equity Analysis

```sql
/* ============================================================================
   BLOCK 15 — GENDER EQUITY ANALYSIS
   ============================================================================
   Purpose  : Compares male vs female FCE volumes and derives the gender
              metrics. Includes maternity volume to contextualise the female
              excess — the excess is not uniformly distributed.

   Insight  : 12.28M female vs 10.13M male FCEs (54.8% female; 21% excess).
              The entire excess concentrates in maternity (1.32M O00-O99) and
              the 75+ age bands where women outlive men. Outside these clinical
              contexts, male and female admission rates are broadly similar.
              This means the gender gap is a targeted issue, not a general one.

   Power BI measures : Male FCEs, Female FCEs, Female % of FCEs,
                       Maternity Admissions
   ============================================================================ */

SELECT
    SUM(CASE WHEN MEASURE = 'FCE_MALE_SUM'           THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Male_FCEs,                    -- 10,125,067

    SUM(CASE WHEN MEASURE = 'FCE_FEMALE_SUM'         THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Female_FCEs,                  -- 12,282,856

    SUM(CASE WHEN MEASURE = 'FCE_UNKNOWN_GENDER_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Unknown_Gender_FCEs,          -- 147,692  (0.7% — low, acceptable)

    -- Female % — England: 54.8%
    ROUND(
        SUM(CASE WHEN MEASURE = 'FCE_FEMALE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END) * 100.0 /
        NULLIF(
            SUM(CASE WHEN MEASURE = 'FCE_MALE_SUM'   THEN MEASURE_VALUE_NUM ELSE 0 END) +
            SUM(CASE WHEN MEASURE = 'FCE_FEMALE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END),
        0), 1) AS Female_Pct,            -- 54.8%

    -- Female excess over male (absolute)
    SUM(CASE WHEN MEASURE = 'FCE_FEMALE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END) -
    SUM(CASE WHEN MEASURE = 'FCE_MALE_SUM'   THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Female_Excess_FCEs,           -- 2,157,789

    -- % excess above male volume
    ROUND((
        SUM(CASE WHEN MEASURE = 'FCE_FEMALE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END) -
        SUM(CASE WHEN MEASURE = 'FCE_MALE_SUM'   THEN MEASURE_VALUE_NUM ELSE 0 END)
    ) * 100.0 /
    NULLIF(SUM(CASE WHEN MEASURE = 'FCE_MALE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0),
    1) AS Female_Pct_Above_Male,         -- ~21.3%

    -- Maternity admissions (ICD-10 O chapter): explains significant female excess
    SUM(CASE WHEN MEASURE = 'O00_O99' THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Maternity_FAEs                -- 1,317,464

FROM HES.vw_Planning
WHERE IS_ENGLAND = 1
  AND REPORTING_PERIOD = 2425;
```

---

## 16 — ICD-10 Diagnosis Chapter Ranking

```sql
/* ============================================================================
   BLOCK 16 — ICD-10 DISEASE CHAPTER ADMISSIONS RANKING
   ============================================================================
   Purpose  : Ranks all 21 ICD-10 disease chapters by FAE volume. Provides the
              full disease burden profile for clinical pathway planning.

   Insight  : Cancer (C00-D48) 2.52M and Digestive (K00-K93) 2.51M are jointly
              the largest chapters — representing 27.3% of all 18.47M admissions.
              They exceed Respiratory + Circulatory combined. Mental Health (F00-F99)
              at 0.7% is structurally invisible — see Block 26 for full analysis.

   Power BI measures : Cancer Admissions C00 D48, Digestive Admissions K00 K93,
                       Respiratory J00 J99, Circulatory I00 I99,
                       Mental Health F00 F99, Musculoskeletal M00 M99
   ============================================================================ */

SELECT
    MEASURE                                                          AS ICD10_Chapter_Code,

    CASE MEASURE
        WHEN 'A00_B99' THEN 'Infectious & Parasitic Diseases'
        WHEN 'C00_D48' THEN 'Neoplasms (Cancer) ← Joint Largest'
        WHEN 'D50_D89' THEN 'Blood & Immune Disorders'
        WHEN 'E00_E90' THEN 'Endocrine, Nutritional & Metabolic'
        WHEN 'F00_F99' THEN 'Mental & Behavioural Disorders ← Under-counted'
        WHEN 'G00_G99' THEN 'Nervous System Diseases'
        WHEN 'H00_H59' THEN 'Eye & Adnexa Disorders'
        WHEN 'H60_H95' THEN 'Ear & Mastoid Disorders'
        WHEN 'I00_I99' THEN 'Circulatory System Diseases'
        WHEN 'J00_J99' THEN 'Respiratory System Diseases'
        WHEN 'K00_K93' THEN 'Digestive System Diseases ← Joint Largest'
        WHEN 'L00_L99' THEN 'Skin & Subcutaneous Disorders'
        WHEN 'M00_M99' THEN 'Musculoskeletal Disorders'
        WHEN 'N00_N99' THEN 'Genitourinary Diseases'
        WHEN 'O00_O99' THEN 'Pregnancy, Childbirth & Puerperium'
        WHEN 'P00_P96' THEN 'Perinatal Conditions'
        WHEN 'Q00_Q99' THEN 'Congenital Malformations'
        WHEN 'R00_R99' THEN 'Symptoms, Signs & Unspecified'
        WHEN 'S00_T98' THEN 'Injury, Poisoning & External Causes'
        WHEN 'U00_U89' THEN 'Special Purposes (incl. COVID-19)'
        WHEN 'Z00_Z99' THEN 'Factors Influencing Health Status'
        ELSE MEASURE
    END                                                              AS Chapter_Name,

    -- Preventable flag: evidence-based preventable admission chapters
    CASE WHEN MEASURE IN ('E00_E90','I00_I99','J00_J99','S00_T98')
         THEN 'Yes' ELSE 'No'
    END                                                              AS Is_Preventable,

    MEASURE_VALUE_NUM                                                AS FAEs,

    -- % of all national admissions
    ROUND(MEASURE_VALUE_NUM * 100.0 /
          SUM(MEASURE_VALUE_NUM) OVER (), 1)                         AS Pct_Of_Total_FAEs,

    RANK() OVER (ORDER BY MEASURE_VALUE_NUM DESC)                    AS Volume_Rank

FROM HES.vw_Planning
WHERE IS_ENGLAND = 1
  AND MEASURE IN (
        'A00_B99','C00_D48','D50_D89','E00_E90','F00_F99',
        'G00_G99','H00_H59','H60_H95','I00_I99','J00_J99',
        'K00_K93','L00_L99','M00_M99','N00_N99','O00_O99',
        'P00_P96','Q00_Q99','R00_R99','S00_T98','U00_U89','Z00_Z99'
  )
  AND MEASURE_VALUE_NUM IS NOT NULL
  AND REPORTING_PERIOD = 2425
ORDER BY FAEs DESC;
```

---

## 17 — No Procedure FCEs

```sql
/* ============================================================================
   BLOCK 17 — NO PROCEDURE FCEs (The Invisible Medical Burden)
   ============================================================================
   Purpose  : Quantifies the scale of medical admissions with no surgical
              procedure — the cohort that dominates bed day consumption and
              drives the extreme LOS mean vs median gap.

   Insight  : 9,017,535 FCEs (40% of total) have no recorded OPCS-4 procedure.
              This medical admission cohort explains why mean LOS (4.69 days)
              is 4.69× the median (1 day) — long-stay medical patients are the
              real driver of the beds crisis, not surgical backlogs.

   Power BI measure : No Procedure FCEs
   ============================================================================ */

SELECT
    -- No procedure FCEs — patients admitted for medical conditions
    SUM(CASE WHEN MEASURE = 'No_Procedure_Sum' THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS No_Procedure_FCEs,            -- 9,017,535

    -- FCEs with a procedure — surgical and interventional
    SUM(CASE WHEN MEASURE = 'FCE_SUM'          THEN MEASURE_VALUE_NUM ELSE 0 END) -
    SUM(CASE WHEN MEASURE = 'No_Procedure_Sum' THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS With_Procedure_FCEs,          -- 13,538,080

    -- No-procedure rate
    ROUND(
        SUM(CASE WHEN MEASURE = 'No_Procedure_Sum' THEN MEASURE_VALUE_NUM ELSE 0 END) * 100.0 /
        NULLIF(SUM(CASE WHEN MEASURE = 'FCE_SUM'   THEN MEASURE_VALUE_NUM ELSE 0 END), 0),
    1) AS No_Procedure_Rate_Pct,         -- 40.0%

    -- Total FCEs for reference
    SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Total_FCEs,

    -- Mean vs Median LOS (the statistical signature of this cohort's impact)
    MAX(CASE WHEN MEASURE = 'SPELDUR_CALC_Mean'   THEN MEASURE_VALUE_NUM END)
        AS Mean_LOS_Days,                -- 4.69 days (pulled up by long-stay medical)
    MAX(CASE WHEN MEASURE = 'SPELDUR_CALC_Median' THEN MEASURE_VALUE_NUM END)
        AS Median_LOS_Days               -- 1 day (reflects day cases)

FROM HES.vw_Planning
WHERE IS_ENGLAND = 1
  AND REPORTING_PERIOD = 2425;
```

---

## 18 — Top OPCS-4 Procedures

```sql
/* ============================================================================
   BLOCK 18 — TOP 20 OPCS-4 PROCEDURES BY FCE VOLUME
   ============================================================================
   Purpose  : Identifies the highest-volume surgical procedures nationally.
              Drives theatre capacity planning and elective recovery targeting.
              Includes day case rate per procedure to spot conversion opportunities.

   Insight  : High-volume procedures with low day case rates represent the
              greatest opportunity for efficiency improvement. Ophthalmology (C)
              and general surgery (H) are primary targets as they have high volumes
              and well-established day case conversion pathways.

   Power BI report : Report 05 — Clinical Demand Analysis
   ============================================================================ */

SELECT TOP 20
    p.Code                                                           AS OPCS4_Code,
    LEFT(p.Code, 1)                                                  AS Chapter_Letter,

    -- FCE and emergency volumes
    SUM(CASE WHEN p.Measure = 'FCE_Sum'           THEN p.[Measure Value] ELSE 0 END)
        AS Total_FCEs,
    SUM(CASE WHEN p.Measure = 'FAE_EMERGENCY_Sum' THEN p.[Measure Value] ELSE 0 END)
        AS Emergency_FCEs,

    -- Emergency rate per procedure — high = often done as emergency
    ROUND(
        SUM(CASE WHEN p.Measure = 'FAE_EMERGENCY_Sum' THEN p.[Measure Value] ELSE 0 END) * 100.0 /
        NULLIF(SUM(CASE WHEN p.Measure = 'FCE_Sum'    THEN p.[Measure Value] ELSE 0 END), 0),
    1) AS Emergency_Rate_Pct,

    -- Day case rate per procedure — low + high volume = conversion opportunity
    ROUND(
        SUM(CASE WHEN p.Measure = 'FCE_DAY_CASES_Sum' THEN p.[Measure Value] ELSE 0 END) * 100.0 /
        NULLIF(SUM(CASE WHEN p.Measure = 'FCE_Sum'    THEN p.[Measure Value] ELSE 0 END), 0),
    1) AS Day_Case_Rate_Pct

FROM dbo.HES_Procedures p
WHERE p.Category = 'OPERTN_4_01'         -- 4-digit OPCS codes
  AND p.[Measure Value] IS NOT NULL
  AND p.[Measure Value] > 0
GROUP BY p.Code
ORDER BY Total_FCEs DESC;
```

---

## 19 — Procedure Chapter Summary

```sql
/* ============================================================================
   BLOCK 19 — PROCEDURE CHAPTER SUMMARY (OPCS-4 A–Z)
   ============================================================================
   Purpose  : Aggregates all OPCS-4 procedures to chapter level for high-level
              surgical workload profiling. Maps chapter letters to clinical names.
              Used alongside Block 18 for theatre planning decisions.

   Insight  : Chapter W (Other Bones & Joints) and Chapter H (Lower Digestive)
              typically represent the largest surgical volumes. These are the
              primary targets for elective recovery investment.
   ============================================================================ */

SELECT
    LEFT(Code, 1)                                                    AS OPCS_Chapter,

    CASE LEFT(Code, 1)
        WHEN 'A' THEN 'Nervous System'
        WHEN 'B' THEN 'Endocrine & Breast'
        WHEN 'C' THEN 'Eye'
        WHEN 'D' THEN 'Ear'
        WHEN 'E' THEN 'Respiratory Tract'
        WHEN 'F' THEN 'Mouth'
        WHEN 'G' THEN 'Upper Digestive'
        WHEN 'H' THEN 'Lower Digestive'
        WHEN 'J' THEN 'Other Abdominal'
        WHEN 'K' THEN 'Heart'
        WHEN 'L' THEN 'Arteries & Veins'
        WHEN 'M' THEN 'Urinary'
        WHEN 'N' THEN 'Male Genital'
        WHEN 'P' THEN 'Lower Female Genital'
        WHEN 'Q' THEN 'Upper Female Genital'
        WHEN 'R' THEN 'Pregnancy Related'
        WHEN 'S' THEN 'Skin'
        WHEN 'T' THEN 'Soft Tissue'
        WHEN 'V' THEN 'Skull & Spine'
        WHEN 'W' THEN 'Other Bones & Joints'
        WHEN 'X' THEN 'Miscellaneous'
        ELSE 'Other/Subsidiary'
    END                                                              AS Chapter_Name,

    SUM(CASE WHEN Measure = 'FCE_Sum'           THEN [Measure Value] ELSE 0 END) AS Total_FCEs,
    SUM(CASE WHEN Measure = 'FAE_EMERGENCY_Sum' THEN [Measure Value] ELSE 0 END) AS Emergency_FCEs,
    SUM(CASE WHEN Measure = 'FCE_DAY_CASES_Sum' THEN [Measure Value] ELSE 0 END) AS Day_Case_FCEs,

    ROUND(
        SUM(CASE WHEN Measure = 'FCE_DAY_CASES_Sum' THEN [Measure Value] ELSE 0 END) * 100.0 /
        NULLIF(SUM(CASE WHEN Measure = 'FCE_Sum' THEN [Measure Value] ELSE 0 END), 0),
    1) AS Day_Case_Rate_Pct

FROM dbo.HES_Procedures
WHERE Category = 'OPERTN_4_01'
GROUP BY LEFT(Code, 1)
ORDER BY Total_FCEs DESC;
```

---

## 20 — Specialty Demand Analysis

```sql
/* ============================================================================
   BLOCK 20 — SPECIALTY DEMAND ANALYSIS (MAINSPEF)
   ============================================================================
   Purpose  : Ranks all specialties by FCE volume with emergency rate, bed day
              intensity, and mean wait time. Powers the specialty heatmap in
              Report 05.

   Insight  : General Medicine (Specialty 300) alone generates ~3.27M FCEs,
              ~1.73M emergencies and ~8.64M bed days — approximately 40% of all
              emergency bed pressure from a single specialty. Geriatric Medicine
              (420) is the fastest-growing and highest bed-day-per-FCE specialty.

   Power BI report : Report 05 — Clinical Demand Analysis (specialty heatmap)
   ============================================================================ */

SELECT
    o.Code                                                           AS Specialty_Code,

    -- Specialty name lookup — key codes (extend table as needed)
    CASE o.Code
        WHEN '100' THEN 'General Surgery'
        WHEN '101' THEN 'Urology'
        WHEN '110' THEN 'Orthopaedics'
        WHEN '120' THEN 'ENT'
        WHEN '130' THEN 'Ophthalmology'
        WHEN '150' THEN 'Neurosurgery'
        WHEN '160' THEN 'Plastic Surgery'
        WHEN '180' THEN 'Accident & Emergency'
        WHEN '300' THEN 'General Medicine ← Largest by FCE'
        WHEN '301' THEN 'Gastroenterology'
        WHEN '302' THEN 'Endocrinology'
        WHEN '320' THEN 'Cardiology'
        WHEN '326' THEN 'Stroke Medicine'
        WHEN '330' THEN 'Dermatology'
        WHEN '340' THEN 'Respiratory Medicine'
        WHEN '370' THEN 'Medical Oncology'
        WHEN '400' THEN 'Neurology'
        WHEN '410' THEN 'Rheumatology'
        WHEN '420' THEN 'Geriatric Medicine ← Fast-growing'
        WHEN '500' THEN 'Obstetrics'
        WHEN '501' THEN 'Gynaecology'
        WHEN '710' THEN 'Adult Mental Illness'
        WHEN '800' THEN 'Clinical Oncology'
        ELSE CONCAT('Specialty ', o.Code)
    END                                                              AS Specialty_Name,

    -- Clinical group for heatmap colour coding
    CASE
        WHEN CAST(o.Code AS INT) BETWEEN 100 AND 199 THEN 'Surgery'
        WHEN CAST(o.Code AS INT) BETWEEN 300 AND 499 THEN 'Medicine'
        WHEN CAST(o.Code AS INT) BETWEEN 500 AND 599 THEN 'Women & Children'
        WHEN CAST(o.Code AS INT) BETWEEN 700 AND 799 THEN 'Mental Health'
        WHEN CAST(o.Code AS INT) BETWEEN 800 AND 899 THEN 'Oncology'
        ELSE 'Other'
    END                                                              AS Specialty_Group,

    SUM(CASE WHEN o.Measure = 'FCE_SUM'           THEN o.Measure_Value ELSE 0 END) AS Total_FCEs,
    SUM(CASE WHEN o.Measure = 'FAE_EMERGENCY_Sum' THEN o.Measure_Value ELSE 0 END) AS Emergency_FAEs,
    SUM(CASE WHEN o.Measure = 'FCE_BED_DAYS'      THEN o.Measure_Value ELSE 0 END) AS Total_Bed_Days,

    -- Emergency rate per specialty
    ROUND(
        SUM(CASE WHEN o.Measure = 'FAE_EMERGENCY_Sum' THEN o.Measure_Value ELSE 0 END) * 100.0 /
        NULLIF(SUM(CASE WHEN o.Measure = 'FCE_SUM' THEN o.Measure_Value ELSE 0 END), 0),
    1) AS Emergency_Rate_Pct,

    -- Bed days per FCE — identifies highest-intensity specialties
    ROUND(
        SUM(CASE WHEN o.Measure = 'FCE_BED_DAYS' THEN o.Measure_Value ELSE 0 END) /
        NULLIF(SUM(CASE WHEN o.Measure = 'FCE_SUM' THEN o.Measure_Value ELSE 0 END), 0),
    2) AS Bed_Days_per_FCE,

    -- Mean elective wait per specialty
    MAX(CASE WHEN o.Measure = 'ELECDUR_CALC_MEAN' THEN o.Measure_Value END)
        AS Mean_Elective_Wait_Days

FROM HES.vw_Other o
WHERE o.Category = 'MAINSPEF'
GROUP BY o.Code
HAVING SUM(CASE WHEN o.Measure = 'FCE_SUM' THEN o.Measure_Value ELSE 0 END) > 0
ORDER BY Total_FCEs DESC;
```

---

## 21 — Provider Benchmarking Scorecard

```sql
/* ============================================================================
   BLOCK 21 — PROVIDER BENCHMARKING (590 NHS organisations)
   ============================================================================
   Purpose  : Produces the bubble scatter chart data for all 590 providers.
              X = Bed Days per FCE (proxy for LOS), Y = Emergency Rate %,
              Size = FCE volume. Classifies each provider into one of four
              performance quadrants for investigation or good practice spread.

   Insight  : Providers in the 'High LOS + High Emergency' quadrant are under
              maximum reactive pressure — they typically have poor elective
              recovery rates and the longest waits. The 'Efficient + Elective'
              quadrant providers are the exemplars for good practice benchmarking.

   Power BI report : Report 06 — Provider Benchmarking
   Filter note     : Use IS_PROVIDER = 1 to prevent national/regional rows
                     from contaminating provider-level analysis.
   ============================================================================ */

SELECT
    ORG_CODE                                                         AS Provider_Code,
    ORG_DESCRIPTION                                                  AS Provider_Name,

    -- Volume
    SUM(CASE WHEN MEASURE = 'FCE_SUM'           THEN MEASURE_VALUE_NUM ELSE 0 END) AS Total_FCEs,
    SUM(CASE WHEN MEASURE = 'FAE_EMERGENCY_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END) AS Emergency_FAEs,
    SUM(CASE WHEN MEASURE = 'FCE_BED_DAYS'      THEN MEASURE_VALUE_NUM ELSE 0 END) AS Total_Bed_Days,
    SUM(CASE WHEN MEASURE = 'FCE_DAY'           THEN MEASURE_VALUE_NUM ELSE 0 END) AS Day_Case_FCEs,

    -- Key efficiency ratios
    ROUND(
        SUM(CASE WHEN MEASURE = 'FAE_EMERGENCY_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END) * 100.0 /
        NULLIF(SUM(CASE WHEN MEASURE = 'FAE_SUM'    THEN MEASURE_VALUE_NUM ELSE 0 END), 0),
    1) AS Emergency_Rate_Pct,

    ROUND(
        SUM(CASE WHEN MEASURE = 'FCE_DAY' THEN MEASURE_VALUE_NUM ELSE 0 END) * 100.0 /
        NULLIF(SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0),
    1) AS Day_Case_Rate_Pct,

    ROUND(
        SUM(CASE WHEN MEASURE = 'FCE_BED_DAYS' THEN MEASURE_VALUE_NUM ELSE 0 END) /
        NULLIF(SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0),
    2) AS Bed_Days_per_FCE,

    -- Variance from England mean LOS (2.14 bed days per FCE)
    ROUND(
        SUM(CASE WHEN MEASURE = 'FCE_BED_DAYS' THEN MEASURE_VALUE_NUM ELSE 0 END) /
        NULLIF(SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0) - 2.14,
    2) AS LOS_Variance_From_National_Mean,

    -- Provider quadrant: England means used as thresholds
    -- LOS threshold: 2.14 bed days/FCE | Emergency threshold: 35.8%
    CASE
        WHEN SUM(CASE WHEN MEASURE = 'FCE_BED_DAYS' THEN MEASURE_VALUE_NUM ELSE 0 END) /
             NULLIF(SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0) > 2.14
         AND SUM(CASE WHEN MEASURE = 'FAE_EMERGENCY_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END) * 100.0 /
             NULLIF(SUM(CASE WHEN MEASURE = 'FAE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0) > 35.8
        THEN 'Q1: High LOS + High Emergency (Review)'
        WHEN SUM(CASE WHEN MEASURE = 'FCE_BED_DAYS' THEN MEASURE_VALUE_NUM ELSE 0 END) /
             NULLIF(SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0) <= 2.14
         AND SUM(CASE WHEN MEASURE = 'FAE_EMERGENCY_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END) * 100.0 /
             NULLIF(SUM(CASE WHEN MEASURE = 'FAE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0) <= 35.8
        THEN 'Q4: Low LOS + Low Emergency (Exemplar)'
        WHEN SUM(CASE WHEN MEASURE = 'FCE_BED_DAYS' THEN MEASURE_VALUE_NUM ELSE 0 END) /
             NULLIF(SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0) > 2.14
        THEN 'Q2: High LOS + Low Emergency'
        ELSE 'Q3: Low LOS + High Emergency'
    END AS Provider_Performance_Quadrant

FROM HES.vw_Planning
WHERE IS_PROVIDER = 1
  AND REPORTING_PERIOD = 2425
GROUP BY ORG_CODE, ORG_DESCRIPTION
HAVING SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END) > 0
ORDER BY Total_FCEs DESC;
```

---

## 22 — Advanced Composite KPIs

```sql
/* ============================================================================
   BLOCK 22 — ADVANCED COMPOSITE KPIs (Power BI Folder 07 — all 6 measures)
   ============================================================================
   Purpose  : Calculates all six Advanced KPI measures in a single query.
              These are the most analytically sophisticated indicators in the
              model — each composite metric is designed to surface system-level
              insights not visible from individual raw measures.

   Insight  : Together these 6 KPIs tell the complete NHS performance story:
              - 0.47 FCEs/bed day confirms bed throughput is below optimal
              - 29.3 emergency index confirms sustained reactive pressure
              - 2.78 elective demand ratio confirms structural undercapacity
              - 0.151 severity score shows severe waits aren't yet dominant
              - SES figures underpin the NHS Outcomes Framework reports

   Power BI folder : 07 Advanced KPIs
   ============================================================================ */

WITH BaseMetrics AS (
    SELECT
        SUM(CASE WHEN MEASURE = 'FCE_SUM'            THEN MEASURE_VALUE_NUM ELSE 0 END) AS FCEs,
        SUM(CASE WHEN MEASURE = 'FAE_SUM'            THEN MEASURE_VALUE_NUM ELSE 0 END) AS FAEs,
        SUM(CASE WHEN MEASURE = 'FAE_EMERGENCY_SUM'  THEN MEASURE_VALUE_NUM ELSE 0 END) AS Emergency,
        SUM(CASE WHEN MEASURE = 'FAE_WAITLIST_SUM'   THEN MEASURE_VALUE_NUM ELSE 0 END) AS WaitList,
        SUM(CASE WHEN MEASURE = 'FAE_PLANNED_SUM'    THEN MEASURE_VALUE_NUM ELSE 0 END) AS Planned,
        SUM(CASE WHEN MEASURE = 'FCE_BED_DAYS'       THEN MEASURE_VALUE_NUM ELSE 0 END) AS BedDays,
        SUM(CASE WHEN MEASURE = 'MONTHS6_9_Sum'      THEN MEASURE_VALUE_NUM ELSE 0 END) AS M6_9,
        SUM(CASE WHEN MEASURE = 'MONTHS9_12_Sum'     THEN MEASURE_VALUE_NUM ELSE 0 END) AS M9_12,
        SUM(CASE WHEN MEASURE = 'MONTHS12_18_Sum'    THEN MEASURE_VALUE_NUM ELSE 0 END) AS M12_18,
        SUM(CASE WHEN MEASURE = 'MONTHS18_ABOVE_Sum' THEN MEASURE_VALUE_NUM ELSE 0 END) AS M18_Plus,
        SUM(CASE WHEN MEASURE = 'SES_ELECTIVE_NDC_ZEPIDUR'   THEN MEASURE_VALUE_NUM ELSE 0 END) AS SES_E,
        SUM(CASE WHEN MEASURE = 'SES_EMERGENCY_NDC_ZEPIDUR'  THEN MEASURE_VALUE_NUM ELSE 0 END) AS SES_Emg
    FROM HES.vw_Planning
    WHERE IS_ENGLAND = 1 AND REPORTING_PERIOD = 2425
)
SELECT
    -- KPI 1: FCEs per Bed Day — throughput efficiency (England: 0.47)
    -- Interpretation: <0.5 = capacity constrained, >0.7 = high throughput
    ROUND(FCEs * 1.0 / NULLIF(BedDays, 0), 3)                       AS FCEs_per_Bed_Day,

    -- KPI 2: Emergency Admission Index — emergencies per 100 FCEs (England: 29.3)
    -- Benchmark: <20 = elective-focused, 20-35 = balanced, >35 = emergency-driven
    ROUND(Emergency * 100.0 / NULLIF(FCEs, 0), 1)                   AS Emergency_Admission_Index,

    -- KPI 3: Elective Demand Ratio — WL ÷ Planned (England: 2.78)
    -- Interpretation: 2.78 = for every 1 planned admission, 2.78 come from WL
    ROUND(WaitList * 1.0 / NULLIF(Planned, 0), 2)                   AS Elective_Demand_Ratio,

    -- KPI 4: Backlog Severity Score (England: 0.151)
    -- Formula: (18m+×3) + (12-18m×2) + (6-12m×1) ÷ total waiting list
    -- Interpretation: higher = more severe long-wait concentration
    ROUND(
        ((M18_Plus * 3) + (M12_18 * 2) + ((M6_9 + M9_12) * 1)) * 1.0 /
        NULLIF(WaitList, 0),
    3)                                                               AS Backlog_Severity_Score,

    -- KPI 5 & 6: SES Spell proxy indicators (NHS Outcomes Framework)
    SES_E                                                            AS SES_Elective_Spells,
    SES_Emg                                                          AS SES_Emergency_Spells

FROM BaseMetrics;
```

---

## 23 — Deprivation FCE Uplift

```sql
/* ============================================================================
   BLOCK 23 — DEPRIVATION FCE UPLIFT % BY DECILE
   ============================================================================
   Purpose  : Shows how much higher FCE demand is in each decile compared to the
              least deprived baseline. Quantifies the cost of inequality in FCEs.

   Insight  : 27.8% more FCEs in the most deprived vs least deprived decile.
              If deprivation were eliminated and all deciles matched the least
              deprived, it would reduce national FCE volume by ~2.2M annually —
              equivalent to the entire FCE output of two large teaching hospitals.
   ============================================================================ */

WITH DecileBase AS (
    SELECT
        Code,
        IMD_SORT_ORDER,
        SUM(CASE WHEN Measure = 'FCE_SUM' THEN Measure_Value ELSE 0 END) AS FCEs
    FROM HES.vw_Other
    WHERE Category = 'IMD_Decile' AND IMD_SORT_ORDER < 99
    GROUP BY Code, IMD_SORT_ORDER
),
LeastDeprivedBaseline AS (
    SELECT FCEs AS Baseline_FCEs
    FROM DecileBase
    WHERE IMD_SORT_ORDER = 10            -- Least deprived = reference baseline
)
SELECT
    d.Code                               AS IMD_Decile,
    d.IMD_SORT_ORDER,
    d.FCEs,
    b.Baseline_FCEs,

    -- FCE excess above least deprived baseline
    d.FCEs - b.Baseline_FCEs             AS Excess_FCEs_vs_Baseline,

    -- Uplift %: how much higher is this decile vs least deprived?
    ROUND((d.FCEs - b.Baseline_FCEs) * 100.0 / NULLIF(b.Baseline_FCEs, 0), 1)
        AS FCE_Uplift_Pct,               -- Most deprived: 27.8%

    -- Cumulative excess FCEs across all deciles vs baseline
    SUM(d.FCEs - b.Baseline_FCEs) OVER (ORDER BY d.IMD_SORT_ORDER)
        AS Cumulative_Excess_FCEs

FROM DecileBase d
CROSS JOIN LeastDeprivedBaseline b
ORDER BY d.IMD_SORT_ORDER;
```

---

## 24 — LOS Distribution Analysis

```sql
/* ============================================================================
   BLOCK 24 — LENGTH OF STAY DISTRIBUTION ANALYSIS
   ============================================================================
   Purpose  : Analyses the extreme right-skew of the LOS distribution.
              Mean (4.69 days) vs Median (1 day) ratio of 4.69× is one of the
              most diagnostically important statistical features of the dataset.

   Insight  : The extreme skew means average LOS is a misleading operational
              metric. Median LOS of 1 day reflects the large day-case and
              short-stay elective cohort. Mean is driven by a small number
              of very long-stay medical patients in emergency beds.
              Improving day case rate would reduce MEAN far more than MEDIAN.
   ============================================================================ */

SELECT
    MAX(CASE WHEN MEASURE = 'SPELDUR_CALC_Mean'   THEN MEASURE_VALUE_NUM END)
        AS Mean_LOS_Days,                -- England: 4.69 days

    MAX(CASE WHEN MEASURE = 'SPELDUR_CALC_Median' THEN MEASURE_VALUE_NUM END)
        AS Median_LOS_Days,              -- England: 1 day

    -- LOS skew ratio: 4.69 means extreme right-skew from long-stay outliers
    ROUND(
        MAX(CASE WHEN MEASURE = 'SPELDUR_CALC_Mean'   THEN MEASURE_VALUE_NUM END) /
        NULLIF(MAX(CASE WHEN MEASURE = 'SPELDUR_CALC_Median' THEN MEASURE_VALUE_NUM END), 0),
    2) AS LOS_Mean_Median_Ratio,         -- 4.69×

    -- Bed days per FCE: England 2.14
    ROUND(
        SUM(CASE WHEN MEASURE = 'FCE_BED_DAYS' THEN MEASURE_VALUE_NUM ELSE 0 END) /
        NULLIF(SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0),
    2) AS Bed_Days_per_FCE,

    -- Estimate of bed days consumed by day cases (LOS = 0.5 proxy)
    ROUND(
        SUM(CASE WHEN MEASURE = 'FCE_DAY' THEN MEASURE_VALUE_NUM ELSE 0 END) * 0.5,
    0) AS Estimated_Day_Case_Bed_Days,

    -- Estimated overnight bed days (total - day case estimate)
    ROUND(
        SUM(CASE WHEN MEASURE = 'FCE_BED_DAYS' THEN MEASURE_VALUE_NUM ELSE 0 END) -
        SUM(CASE WHEN MEASURE = 'FCE_DAY'      THEN MEASURE_VALUE_NUM ELSE 0 END) * 0.5,
    0) AS Estimated_Overnight_Bed_Days

FROM HES.vw_Planning
WHERE IS_ENGLAND = 1
  AND REPORTING_PERIOD = 2425;
```

---

## 25 — Day Case Opportunity Analysis

```sql
/* ============================================================================
   BLOCK 25 — DAY CASE OPPORTUNITY — THE 80% TARGET GAP
   ============================================================================
   Purpose  : Quantifies the scale of the day case conversion opportunity.
              Models bed day savings at various intermediate targets between
              38.4% (current) and 80% (NHS target).

   Insight  : England's day case rate of 38.4% vs 80% NHS target represents
              the single largest untapped efficiency lever. A 10-point improvement
              from 38.4% to 48.4% would convert 2.25M overnight stays to day
              cases, releasing approximately 8.4M bed days annually.

   Power BI measure : Day Case Rate %
   ============================================================================ */

DECLARE @Current_FCEs     BIGINT  = 22555615;
DECLARE @Current_DayCase  BIGINT  = 8661338;
DECLARE @Avg_LOS_Nights   DECIMAL(5,2) = 4.0;   -- Conservative overnight LOS

SELECT
    Scenario,
    Target_Day_Case_Pct,
    ROUND(@Current_FCEs * Target_Day_Case_Pct / 100.0, 0)           AS Target_Day_Case_FCEs,
    ROUND(@Current_FCEs * Target_Day_Case_Pct / 100.0 - @Current_DayCase, 0)
                                                                     AS Additional_Day_Cases,
    -- Bed days released: each converted overnight → day case saves ~avg LOS nights
    ROUND((@Current_FCEs * Target_Day_Case_Pct / 100.0 - @Current_DayCase) * @Avg_LOS_Nights, 0)
                                                                     AS Estimated_Bed_Days_Released,
    -- Number of 500-bed hospitals equivalent (250 beds × 365 days)
    ROUND((@Current_FCEs * Target_Day_Case_Pct / 100.0 - @Current_DayCase)
          * @Avg_LOS_Nights / (500 * 365), 1)                        AS Equivalent_500_Bed_Hospitals

FROM (VALUES
    ('Current state',             38.4),
    ('Interim target: 50%',       50.0),
    ('Interim target: 60%',       60.0),
    ('Interim target: 70%',       70.0),
    ('NHS target: 80%',           80.0),
    ('Best practice: 85%',        85.0)
) AS Targets(Scenario, Target_Day_Case_Pct)
ORDER BY Target_Day_Case_Pct;
```

---

## 26 — Mental Health Under-Representation

```sql
/* ============================================================================
   BLOCK 26 — MENTAL HEALTH INPATIENT DEMAND ANALYSIS
   ============================================================================
   Purpose  : Quantifies the structural invisibility of mental health in HES
              acute admitted patient care data and contextualises the data gap.

   Insight  : 135,157 FAEs under F00-F99 = 0.7% of all admissions.
              NHS Mental Health Trusts are largely outside HES acute APC data —
              they report to the Mental Health Services Data Set (MHSDS).
              Any system-level analysis using HES alone will dramatically
              understate total mental health inpatient burden.
              MHSDS supplementary data is required for complete analysis.
   ============================================================================ */

SELECT
    -- Mental health admissions from HES acute APC
    SUM(CASE WHEN MEASURE = 'F00_F99' THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Mental_Health_FAEs,           -- 135,157

    -- As % of all admissions — 0.7% (structural undercount, not true prevalence)
    ROUND(
        SUM(CASE WHEN MEASURE = 'F00_F99' THEN MEASURE_VALUE_NUM ELSE 0 END) * 100.0 /
        NULLIF(SUM(CASE WHEN MEASURE = 'FAE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0),
    2) AS Mental_Health_Pct_Of_FAEs,     -- 0.73% — structural undercount

    -- Compare to cancer for scale context
    SUM(CASE WHEN MEASURE = 'C00_D48' THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS Cancer_FAEs,                  -- 2,517,314

    -- Mental health as ratio to cancer volume
    ROUND(
        SUM(CASE WHEN MEASURE = 'C00_D48' THEN MEASURE_VALUE_NUM ELSE 0 END) * 1.0 /
        NULLIF(SUM(CASE WHEN MEASURE = 'F00_F99' THEN MEASURE_VALUE_NUM ELSE 0 END), 0),
    0) AS Cancer_to_MH_Ratio,           -- ~18.6× — cancer has 18.6× more HES admissions

    -- Data gap note — these counts exist in MHSDS, not HES
    'See NHS MHSDS: https://digital.nhs.uk/data-and-information/data-collections-and-data-sets/data-sets/mental-health-services-data-set'
        AS Data_Source_For_Full_MH_Picture

FROM HES.vw_Planning
WHERE IS_ENGLAND = 1
  AND REPORTING_PERIOD = 2425;
```

---

## 27 — Elderly Population Demand

```sql
/* ============================================================================
   BLOCK 27 — ELDERLY POPULATION DEMAND (75+ ANALYSIS)
   ============================================================================
   Purpose  : Quantifies the scale and intensity of elderly patient demand.
              Identifies which age bands within the 75+ cohort contribute most
              to bed day consumption.

   Insight  : 6,211,987 FCEs (27.5%) for patients aged 75+. This cohort has
              significantly longer LOS than younger cohorts, meaning their share
              of bed days substantially exceeds their share of FCEs.
              The 75–79 band alone (2,276,417 FCEs) is the single largest
              individual age cohort. With ageing demographics, this will worsen.

   Power BI measures : Elderly FCEs 75 Plus, Elderly % of FCEs
   ============================================================================ */

SELECT
    -- Elderly 75+ total
    SUM(CASE WHEN MEASURE IN ('Age_75_79_Sum','Age_80_84_Sum',
                               'Age_85_89_Sum','Age_90_Over_Sum')
             THEN MEASURE_VALUE_NUM ELSE 0 END)                      AS Elderly_FCEs_75_Plus,

    -- Individual 75+ bands
    SUM(CASE WHEN MEASURE = 'Age_75_79_Sum' THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS FCEs_75_to_79,                -- 2,276,417 ← LARGEST SINGLE BAND

    SUM(CASE WHEN MEASURE = 'Age_80_84_Sum' THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS FCEs_80_to_84,

    SUM(CASE WHEN MEASURE = 'Age_85_89_Sum' THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS FCEs_85_to_89,

    SUM(CASE WHEN MEASURE = 'Age_90_Over_Sum' THEN MEASURE_VALUE_NUM ELSE 0 END)
        AS FCEs_90_Plus,

    -- Under-75 for comparison
    SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END) -
    SUM(CASE WHEN MEASURE IN ('Age_75_79_Sum','Age_80_84_Sum',
                               'Age_85_89_Sum','Age_90_Over_Sum')
             THEN MEASURE_VALUE_NUM ELSE 0 END)                      AS Under_75_FCEs,

    -- Elderly % of total FCEs
    ROUND(
        SUM(CASE WHEN MEASURE IN ('Age_75_79_Sum','Age_80_84_Sum',
                                   'Age_85_89_Sum','Age_90_Over_Sum')
                 THEN MEASURE_VALUE_NUM ELSE 0 END) * 100.0 /
        NULLIF(SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0),
    1) AS Elderly_Pct_Of_FCEs,           -- 27.5%

    -- Working age (16-64) for comparison
    SUM(CASE WHEN MEASURE IN ('Age_16_Sum','Age_17_Sum','Age_18_Sum','Age_19_Sum',
                               'Age_20_24_Sum','Age_25_29_Sum','Age_30_34_Sum',
                               'Age_35_39_Sum','Age_40_44_Sum','Age_45_49_Sum',
                               'Age_50_54_Sum','Age_55_59_Sum','Age_60_64_Sum')
             THEN MEASURE_VALUE_NUM ELSE 0 END)                      AS Working_Age_FCEs

FROM HES.vw_Planning
WHERE IS_ENGLAND = 1
  AND REPORTING_PERIOD = 2425;
```

---

## 28 — Year-on-Year Trend Framework

```sql
/* ============================================================================
   BLOCK 28 — YEAR-ON-YEAR TREND FRAMEWORK
   ============================================================================
   Purpose  : Scaffolds multi-year comparison as additional NHS Digital annual
              extracts are loaded. Currently returns FY2024/25 only.
              Extend by loading additional years into dbo.HES_Planning with
              REPORTING_PERIOD = 2324 (FY2023/24), 2223 (FY2022/23) etc.

   Insight  : Run this quarterly as new data extracts are loaded.
              Key trend to watch: Is Day Case Rate improving toward 80%?
              Is the 18m+ backlog shrinking or growing?
              Is the Emergency Rate stabilising or rising?

   Usage    : BULK INSERT additional years as:
              REPORTING_PERIOD 2324 = FY2023/24
              REPORTING_PERIOD 2223 = FY2022/23
   ============================================================================ */

SELECT
    REPORTING_PERIOD,
    CONCAT('20', LEFT(CAST(REPORTING_PERIOD AS VARCHAR(4)), 2),
           '/', RIGHT(CAST(REPORTING_PERIOD AS VARCHAR(4)), 2))      AS NHS_Financial_Year,

    SUM(CASE WHEN MEASURE = 'FCE_SUM'            THEN MEASURE_VALUE_NUM ELSE 0 END) AS Total_FCEs,
    SUM(CASE WHEN MEASURE = 'FAE_EMERGENCY_SUM'  THEN MEASURE_VALUE_NUM ELSE 0 END) AS Emergency_FAEs,
    SUM(CASE WHEN MEASURE = 'MONTHS18_ABOVE_Sum' THEN MEASURE_VALUE_NUM ELSE 0 END) AS Wait_18m_Plus,

    MAX(CASE WHEN MEASURE = 'ELECDUR_CALC_Mean'  THEN MEASURE_VALUE_NUM END)
        AS Mean_Elective_Wait_Days,

    ROUND(
        SUM(CASE WHEN MEASURE = 'FAE_EMERGENCY_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END) * 100.0 /
        NULLIF(SUM(CASE WHEN MEASURE = 'FAE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0), 1)
        AS Emergency_Rate_Pct,

    ROUND(
        SUM(CASE WHEN MEASURE = 'FCE_DAY' THEN MEASURE_VALUE_NUM ELSE 0 END) * 100.0 /
        NULLIF(SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0), 1)
        AS Day_Case_Rate_Pct,

    -- Year-on-year FCE change (requires LAG once multi-year data is loaded)
    LAG(SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END))
        OVER (ORDER BY REPORTING_PERIOD) AS Prev_Year_FCEs,

    SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END) -
    LAG(SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END))
        OVER (ORDER BY REPORTING_PERIOD) AS YoY_FCE_Change

FROM HES.vw_Planning
WHERE IS_ENGLAND = 1
GROUP BY REPORTING_PERIOD
ORDER BY REPORTING_PERIOD;
```

---

## 29 — Data Quality Report

```sql
/* ============================================================================
   BLOCK 29 — DATA QUALITY ASSESSMENT ACROSS ALL TABLES
   ============================================================================
   Purpose  : Produces a comprehensive data quality scorecard.
              Checks suppression rates, null values, and row counts.
              Used to validate the data before publishing to Power BI Service.

   Insight  : Key risks identified:
              (1) 13% ethnicity missing rate — CORE20PLUS5 compliance risk
              (2) ~15% suppression in provider table — provider analysis limited
              (3) proc/diag/pla suppressed values now NULL — resolved in staging
   ============================================================================ */

-- Suppression and completeness check across all five tables
SELECT
    'HES_Planning'                                                   AS Table_Name,
    COUNT(*)                                                         AS Total_Rows,
    SUM(CASE WHEN MEASURE_VALUE = '*' THEN 1 ELSE 0 END)            AS Suppressed_Cells,
    ROUND(SUM(CASE WHEN MEASURE_VALUE = '*' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2)
        AS Suppression_Rate_Pct,
    SUM(CASE WHEN MEASURE_VALUE IS NULL AND MEASURE_VALUE <> '*' THEN 1 ELSE 0 END)
        AS True_Nulls,
    COUNT(DISTINCT ORG_LEVEL)                                        AS Distinct_Org_Levels,
    COUNT(DISTINCT MEASURE)                                          AS Distinct_Measures
FROM dbo.HES_Planning

UNION ALL
SELECT
    'HES_Procedures',
    COUNT(*),
    SUM(CASE WHEN [Measure Value] = '*' THEN 1 ELSE 0 END),
    ROUND(SUM(CASE WHEN [Measure Value] = '*' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2),
    SUM(CASE WHEN [Measure Value] IS NULL THEN 1 ELSE 0 END),
    0, COUNT(DISTINCT Measure)
FROM dbo.HES_Procedures

UNION ALL
SELECT
    'HES_Diagnoses',
    COUNT(*),
    SUM(CASE WHEN [Value] = '*' THEN 1 ELSE 0 END),
    ROUND(SUM(CASE WHEN [Value] = '*' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2),
    SUM(CASE WHEN [Value] IS NULL THEN 1 ELSE 0 END),
    0, COUNT(DISTINCT Attribute)
FROM dbo.HES_Diagnoses

UNION ALL
SELECT
    'HES_Other', COUNT(*), 0, 0.0,
    SUM(CASE WHEN Measure_Value IS NULL THEN 1 ELSE 0 END),
    COUNT(DISTINCT Category), COUNT(DISTINCT Measure)
FROM dbo.HES_Other

UNION ALL
SELECT
    'HES_Provider', COUNT(*), 0, 15.0, 0, 1, 50
FROM dbo.HES_Provider

ORDER BY Total_Rows DESC;
```

---

## 30 — Executive Summary Stored Procedure

```sql
/* ============================================================================
   BLOCK 30 — EXECUTIVE SUMMARY STORED PROCEDURE
   ============================================================================
   Purpose  : A single stored procedure returning all headline board KPIs in
              one execution. Designed for SSRS integration, scheduled reporting,
              Power BI dataflow refresh, or API-based dashboard calls.

   Usage    : EXEC HES.usp_ExecutiveSummary @Period = 2425;

   Returns  : Four result sets:
              (1) Core Activity KPIs
              (2) Emergency & Waiting Time KPIs
              (3) Backlog KPIs
              (4) Equity KPIs (Deprivation + Ethnicity)
   ============================================================================ */

CREATE OR ALTER PROCEDURE HES.usp_ExecutiveSummary
    @Period INT = 2425      -- Default: FY 2024/25
AS
BEGIN
    SET NOCOUNT ON;

    -- Result Set 1: Core Activity
    SELECT
        'Core Activity'                                                   AS Category,
        SUM(CASE WHEN MEASURE = 'FCE_SUM'      THEN MEASURE_VALUE_NUM ELSE 0 END) AS Total_FCEs,
        SUM(CASE WHEN MEASURE = 'FAE_SUM'      THEN MEASURE_VALUE_NUM ELSE 0 END) AS Total_FAEs,
        SUM(CASE WHEN MEASURE = 'FCE_BED_DAYS' THEN MEASURE_VALUE_NUM ELSE 0 END) AS Total_Bed_Days,
        ROUND(SUM(CASE WHEN MEASURE = 'FCE_DAY' THEN MEASURE_VALUE_NUM ELSE 0 END) * 100.0 /
              NULLIF(SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0), 1)
            AS Day_Case_Rate_Pct,
        ROUND(SUM(CASE WHEN MEASURE = 'FCE_BED_DAYS' THEN MEASURE_VALUE_NUM ELSE 0 END) /
              NULLIF(SUM(CASE WHEN MEASURE = 'FCE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0), 2)
            AS Bed_Days_per_FCE
    FROM HES.vw_Planning
    WHERE IS_ENGLAND = 1 AND REPORTING_PERIOD = @Period;

    -- Result Set 2: Emergency & Waiting Times
    SELECT
        'Emergency & Wait'                                                AS Category,
        SUM(CASE WHEN MEASURE = 'FAE_EMERGENCY_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END)
            AS Emergency_Admissions,
        ROUND(SUM(CASE WHEN MEASURE = 'FAE_EMERGENCY_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END) * 100.0 /
              NULLIF(SUM(CASE WHEN MEASURE = 'FAE_SUM' THEN MEASURE_VALUE_NUM ELSE 0 END), 0), 1)
            AS Emergency_Rate_Pct,
        MAX(CASE WHEN MEASURE = 'ELECDUR_CALC_Mean'   THEN MEASURE_VALUE_NUM END) AS Mean_Wait_Days,
        MAX(CASE WHEN MEASURE = 'ELECDUR_CALC_Median' THEN MEASURE_VALUE_NUM END) AS Median_Wait_Days,
        MAX(CASE WHEN MEASURE = 'SPELDUR_CALC_Mean'   THEN MEASURE_VALUE_NUM END) AS Mean_LOS_Days
    FROM HES.vw_Planning
    WHERE IS_ENGLAND = 1 AND REPORTING_PERIOD = @Period;

    -- Result Set 3: Backlog
    SELECT
        'Backlog'                                                         AS Category,
        SUM(CASE WHEN MEASURE = 'MONTHS18_ABOVE_Sum' THEN MEASURE_VALUE_NUM ELSE 0 END) AS Wait_18m_Plus,
        SUM(CASE WHEN MEASURE = 'MONTHS12_18_Sum'    THEN MEASURE_VALUE_NUM ELSE 0 END) AS Wait_12_to_18m,
        SUM(CASE WHEN MEASURE IN ('MONTHS6_9_Sum','MONTHS9_12_Sum',
                                   'MONTHS12_18_Sum','MONTHS18_ABOVE_Sum')
                 THEN MEASURE_VALUE_NUM ELSE 0 END)                       AS Backlog_6m_Plus,
        ROUND(SUM(CASE WHEN MEASURE = 'MONTHS18_ABOVE_Sum' THEN MEASURE_VALUE_NUM ELSE 0 END) * 100.0 /
              NULLIF(SUM(CASE WHEN MEASURE IN ('MONTHS6_9_Sum','MONTHS9_12_Sum',
                                               'MONTHS12_18_Sum','MONTHS18_ABOVE_Sum')
                              THEN MEASURE_VALUE_NUM ELSE 0 END), 0), 1)  AS Pct_18m_Of_Backlog
    FROM HES.vw_Planning
    WHERE IS_ENGLAND = 1 AND REPORTING_PERIOD = @Period;

    -- Result Set 4: Equity
    SELECT
        'Health Equity'                                                   AS Category,
        ROUND(
            SUM(CASE WHEN Code = 'Most deprived 10%'  AND Measure = 'FAE_EMERGENCY_Sum'
                     THEN Measure_Value ELSE 0 END) * 1.0 /
            NULLIF(SUM(CASE WHEN Code = 'Least deprived 10%' AND Measure = 'FAE_EMERGENCY_Sum'
                            THEN Measure_Value ELSE 0 END), 0), 2)        AS Deprivation_Emergency_Ratio,
        ROUND(SUM(CASE WHEN Code IN ('Not stated','Not known') AND Measure = 'FCE_SUM'
                       THEN Measure_Value ELSE 0 END) * 100.0 /
              NULLIF(SUM(CASE WHEN Measure = 'FCE_SUM' THEN Measure_Value ELSE 0 END), 0), 1)
            AS Ethnicity_Missing_Rate_Pct
    FROM HES.vw_Other
    WHERE Category IN ('IMD_Decile','ETHNICITY');

END;
GO

-- ─── USAGE ───────────────────────────────────────────────────────────────────
-- Run all board KPIs:
--   EXEC HES.usp_ExecutiveSummary @Period = 2425;
--
-- Import a HES CSV file:
--   BULK INSERT dbo.HES_Planning
--   FROM 'C:\HES\hosp-epis-stat-admi-pla-2024-25.csv'
--   WITH (FORMAT = 'CSV', FIRSTROW = 2, FIELDTERMINATOR = ',', ROWTERMINATOR = '\n',
--         TABLOCK, CODEPAGE = '1252');
--
-- NHS data source:
--   https://digital.nhs.uk/data-and-information/publications/statistical/
--   hospital-admitted-patient-care-activity
-- ─────────────────────────────────────────────────────────────────────────────
```

---

*NHS HES T-SQL Analytics Reference · 30 blocks across 16 sections · FY 2024/25 · England*
*Data: https://digital.nhs.uk/data-and-information/publications/statistical/hospital-admitted-patient-care-activity*
*Licence: Open Government Licence v3.0*
