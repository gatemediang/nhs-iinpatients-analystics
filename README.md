# 🏥 NHS HES Inpatient Admissions Analytics — Power BI Intelligence Platform

> **Hospital Episode Statistics (HES) · Admitted Patient Care · England · Financial Year 2024/25**
> A complete T-SQL/Power BI data model and analytics platform built on 22.5 million NHS inpatient episodes across 590 provider organisations.

---

## 📋 Table of Contents

1. [Project Overview](#project-overview)
2. [The Problem This Analysis Solves](#the-problem-this-analysis-solves)
3. [Data Source](#data-source)
4. [Data Dictionary](#data-dictionary)
5. [Power BI Model Architecture](#power-bi-model-architecture)
6. [DAX Measures Reference](#dax-measures-reference)
7. [Calculated Columns Reference](#calculated-columns-reference)
8. [Key Insights & Findings](#key-insights--findings)
9. [Report Design Guide](#report-design-guide)
10. [Data Quality Notes](#data-quality-notes)
11. [How to Use This Model](#how-to-use-this-model)
12. [References](#references)

---

## Project Overview

This project delivers a production-grade Power BI analytical model built on NHS England's Hospital Episode Statistics (HES) Admitted Patient Care data for the financial year 2024/25. The model provides a single trusted analytical layer covering inpatient activity, elective waiting times, health inequalities, demographic demand, clinical diagnosis patterns, and provider-level benchmarking.

| Metric | Value |
|--------|-------|
| **Total FCEs analysed** | 22,555,615 |
| **Total FAEs analysed** | 18,468,856 |
| **Total Bed Days** | 48,360,197 |
| **NHS Provider Organisations** | 590 |
| **DAX Measures** | 54 (across 7 display folders) |
| **Calculated Columns** | 22 (across 4 fact tables) |
| **Model Tables** | 12 (5 fact + 7 dimension/bridge) |
| **Relationships** | 7 |
| **Reporting Period** | April 2024 – March 2025 |
| **Data Source** | NHS England — NHS Digital HES |

---

## The Problem This Analysis Solves

The NHS faces three simultaneous crises in 2024/25. This model is built to provide the evidence base for addressing each of them.

### 1. 🚨 The Elective Backlog Crisis
72,587 patients in England are waiting more than 18 months for elective care — a direct consequence of COVID-19 disruption compounded by chronic capacity constraints. The South West Commissioning Region alone holds 31,415 of these patients (43% of the national total) despite representing only 10.6% of national FCE volume. Without granular, provider-level data that can be interrogated interactively, commissioning teams cannot identify where to target capacity investment to achieve the fastest backlog clearance.

**This model solves it by:** Providing 12 wait band measures from under 1 month through 18 months+, a Backlog Severity Score, regional comparison visuals, and a What-If parameter framework for scenario modelling.

### 2. ⚖️ Health Inequalities — Deprivation & Ethnicity
The most deprived 10% of England's population generate 832,523 emergency admissions per year versus 521,340 in the least deprived — a ratio of 1.60 (60% higher unplanned admission rate). They also consume 1,921,388 more bed days. This deprivation gradient is systematic, monotonic, and represents a direct failure to achieve the NHS CORE20PLUS5 health equity ambitions. Additionally, 13% of FCE records have ethnicity coded as Not Stated or Not Known, making equity analysis incomplete.

**This model solves it by:** Providing a full 10-decile IMD deprivation analysis, a Deprivation Emergency Ratio measure, Ethnic Group Broad calculated columns, and Ethnicity Missing Rate % as a data quality indicator.

### 3. 📉 Operational Inefficiency — Day Case Rate
England's day case rate is 38.4% (8,661,338 of 22,555,615 FCEs). The NHS target is 80%+ for suitable elective procedures. If this gap were halved, approximately 4.7 million overnight stays could be converted to day cases annually, releasing an estimated 22 million bed days — equivalent to the capacity of several district general hospitals. Without specialty-level and provider-level visibility, it is impossible to identify where day case conversion opportunities are greatest.

**This model solves it by:** Providing Day Case Rate % as a core measure, enabling provider-level benchmarking, and supporting specialty-specific decomposition analysis.

### Additional Problems Addressed

| Problem | How the Model Addresses It |
|---------|---------------------------|
| Ageing demand — 27.5% of FCEs are patients aged 75+ | Elderly FCEs 75 Plus measure + age band calculated columns |
| Gender equity — female admissions 21% higher than male | Male FCEs / Female FCEs / Female % of FCEs measures |
| Clinical pathway — 40% of FCEs have no procedure (medical burden) | No Procedure FCEs measure flagging invisible demand |
| Disease burden — Cancer & Digestive jointly represent 27.3% of all FAEs | 7 clinical diagnosis chapter measures (ICD-10) |
| Provider variation — 590 providers with unknown performance spread | Emergency Admission Index + Bed Days per FCE benchmarking |
| Emergency demand — 35.8% of all admissions are unplanned | Emergency Rate % + SES Emergency Spells measures |

---

## Data Source

### Primary Source
**NHS England — Hospital Episode Statistics (HES), Admitted Patient Care**

| Field | Detail |
|-------|--------|
| **Publisher** | NHS England (formerly NHS Digital) |
| **Dataset** | HES Admitted Patient Care |
| **Financial Year** | 2024–25 (April 2024 – March 2025) |
| **Geographic Coverage** | England |
| **Primary URL** | https://digital.nhs.uk/data-and-information/publications/statistical/hospital-admitted-patient-care-activity |
| **HES Overview** | https://digital.nhs.uk/data-and-information/data-tools-and-services/data-services/hospital-episode-statistics |
| **Data Access** | https://digital.nhs.uk/services/data-access-request-service-dars |
| **Open Data Portal** | https://www.england.nhs.uk/statistics/statistical-work-areas/hospital-activity/ |
| **CORE20PLUS5 Framework** | https://www.england.nhs.uk/about/equality/equality-hub/national-healthcare-inequalities-improvement-programme/core20plus5/ |
| **ICD-10 Reference** | https://www.who.int/standards/classifications/classification-of-diseases |
| **OPCS-4 Reference** | https://digital.nhs.uk/services/terminology-and-classifications/opcs-classification |
| **IMD Data** | https://www.gov.uk/government/statistics/english-indices-of-deprivation-2019 |
| **Licence** | Open Government Licence v3.0 |

### Data Files Used

| File Name | Description |
|-----------|-------------|
| `hosp-epis-stat-admi-proc-2024-25.csv` | Procedure (OPCS-4) breakdown |
| `hosp-epis-stat-admi-pla-2024-25.csv` | Planning — Provider/Region/England summary |
| `hosp-epis-stat-admi-oth-2024-25.csv` | Other breakdowns (IMD, Ethnicity, Specialty) |
| `hosp-epis-stat-admi-hosp-provider-2024-25-tab.csv` | Hospital provider summary table |
| `hosp-epis-stat-admi-diag-2024-25.csv` | Diagnosis (ICD-10) breakdown |

---

## Data Dictionary

### Table 1: `hosp-epis-stat-admi-pla-2024-25` — Planning Table
**Description:** Primary fact table containing FCE, FAE, bed day, waiting time and demographic metrics at England, Commissioning Region and NHS Provider level. 66,323 rows. The most important table for national and regional analysis.

#### Source Columns

| Column | Data Type | Description | Example Values |
|--------|-----------|-------------|----------------|
| `UID` | Integer | Unique row identifier | 1, 2, 3... |
| `ORG_LEVEL` | Text | Organisational aggregation level | `ENGLAND`, `REGION`, `PROVIDER` |
| `REPORTING_PERIOD` | Integer | NHS financial year as 4-digit integer (YYMM format) | `2425` = FY2024/25 |
| `ORG_CODE` | Text | NHS organisation code | `RJ1`, `Q50`, `E40000001` |
| `ORG_DESCRIPTION` | Text | Full organisation name | `LONDON COMMISSIONING REGION` |
| `MEASURE_TYPE` | Text | Category of measure rows | `Summary`, `Time Waited LOS`, `FCE patient age`, `FCE procedure`, `FAE primary diagnosis chapter`, `FCE main specialty consultant` |
| `MEASURE` | Text | Specific metric identifier (see Measure Glossary below) | `FCE_SUM`, `FAE_EMERGENCY_SUM`, `ELECDUR_CALC_Mean` |
| `MEASURE_VALUE` | Text | Numeric value of the measure (text type — suppressed values stored as `*`) | `22555615`, `83.6`, `*` |
| `Table_Order` | Integer | Sort order for table display | 1, 2, 3... |
| `Level_Order` | Integer | Sort order within organisation level | 1, 2, 3... |

#### Measure Glossary — MEASURE Column Values

| MEASURE Code | Full Name | Description | Unit |
|--------------|-----------|-------------|------|
| `FCE_SUM` | Finished Consultant Episodes | Total inpatient episodes under a single consultant | Count |
| `FAE_SUM` | Finished Admission Episodes | Total patient stays (one stay may span multiple FCEs) | Count |
| `FCE_DAY` | Day Case FCEs | Episodes with no overnight stay | Count |
| `FCE_ORDINARY` | Ordinary Admissions | Inpatient overnight admissions | Count |
| `FCE_BED_DAYS` | FCE Bed Days | Total bed days consumed by inpatient episodes | Days |
| `FCE_MALE_SUM` | Male FCEs | FCEs for male patients | Count |
| `FCE_FEMALE_SUM` | Female FCEs | FCEs for female patients | Count |
| `FCE_UNKNOWN_GENDER_SUM` | Unknown Gender FCEs | FCEs where gender is not recorded | Count |
| `FAE_EMERGENCY_SUM` | Emergency Admissions | Unplanned emergency FAEs | Count |
| `FAE_WAITLIST_SUM` | Waiting List Admissions | Patients admitted from elective waiting list | Count |
| `FAE_PLANNED_SUM` | Planned Admissions | Pre-booked elective admissions | Count |
| `FAE_OTHER_SUM` | Other Admissions | Admissions not classified as emergency, waiting list or planned | Count |
| `ELECDUR_CALC_Mean` | Mean Elective Wait | Average days from referral to elective admission | Days |
| `ELECDUR_CALC_Median` | Median Elective Wait | Median days from referral to elective admission | Days |
| `SPELDUR_CALC_Mean` | Mean Length of Spell | Average inpatient spell duration | Days |
| `SPELDUR_CALC_Median` | Median Length of Spell | Median inpatient spell duration | Days |
| `UNDER_1MONTH_Sum` | Waiting Under 1 Month | Patients admitted within 1 month of referral | Count |
| `MONTHS1_2_Sum` | Waiting 1–2 Months | Patients admitted after 1–2 months | Count |
| `MONTHS2_3_Sum` | Waiting 2–3 Months | Patients admitted after 2–3 months | Count |
| `MONTHS3_6_Sum` | Waiting 3–6 Months | Patients admitted after 3–6 months | Count |
| `MONTHS6_9_Sum` | Waiting 6–9 Months | Patients admitted after 6–9 months (RTT breach) | Count |
| `MONTHS9_12_Sum` | Waiting 9–12 Months | Patients admitted after 9–12 months | Count |
| `MONTHS12_18_Sum` | Waiting 12–18 Months | Patients admitted after 12–18 months | Count |
| `MONTHS18_ABOVE_Sum` | Waiting 18 Months+ | Patients admitted after more than 18 months | Count |
| `SES_ELECTIVE_NDC_ZEPIDUR` | SES Elective | Spell Equivalent Spells — Elective (NHS Outcomes Framework) | Count |
| `SES_EMERGENCY_NDC_ZEPIDUR` | SES Emergency | Spell Equivalent Spells — Emergency (NHS Outcomes Framework) | Count |
| `SES_OTHER_NDC_ZEPIDUR` | SES Other | Spell Equivalent Spells — Other | Count |
| `Age_0_Sum` through `Age_90_Over_Sum` | Age Band FCEs | FCEs by single year or 5-year age band | Count |
| `A00_B99` through `Z00_Z99` | ICD-10 Chapter FAEs | FAEs by ICD-10 diagnosis chapter code | Count |
| `No_Procedure_Sum` | No Procedure FCEs | FCEs with no recorded surgical procedure | Count |
| `Arteries_Veins_Sum` through `Upper_Female_Genital_Sum` | Procedure Group FCEs | FCEs by OPCS-4 procedure grouping | Count |

#### Calculated Columns (added by this model)

| Column | Data Type | Description |
|--------|-----------|-------------|
| `NHS Financial Year` | Text | Converts `REPORTING_PERIOD` integer to label e.g. `2024/25` |
| `Org Level Full Name` | Text | Human-readable label for `ORG_LEVEL` codes |
| `Is England Level` | Boolean | TRUE for England-level rows |
| `Is Region Level` | Boolean | TRUE for commissioning region rows |
| `Is Provider Level` | Boolean | TRUE for individual NHS provider rows |
| `Measure Value Numeric` | Decimal | Converts text `MEASURE_VALUE` to number; suppressed `*` values → BLANK |
| `Measure Category` | Text | Groups MEASURE values into analytical domains: Activity Volume, Admission Type, Gender, Wait & LOS Metrics, Age Band, Wait Band, SES Duration, Diagnosis/Procedure |

---

### Table 2: `hosp-epis-stat-admi-oth-2024-25` — Other Breakdowns
**Description:** Activity breakdowns by IMD Deprivation Decile, Ethnicity, Main Specialty (MAINSPEF), Treatment Specialty (TRETSPEF), and External Cause of Injury (CAUSE_3). 27,969 rows. Key table for equity analysis.

#### Source Columns

| Column | Data Type | Description | Example Values |
|--------|-----------|-------------|----------------|
| `UID` | Integer | Unique row identifier | 1, 2, 3... |
| `Code` | Text | Value within the category (e.g. deprivation decile name, ethnicity description, specialty code) | `Most deprived 10%`, `British (White)`, `300` |
| `Category` | Text | Type of breakdown | `IMD_Decile`, `ETHNICITY`, `MAINSPEF`, `TRETSPEF`, `CAUSE_3` |
| `Measure` | Text | Metric identifier (same vocabulary as pla table) | `FCE_SUM`, `FAE_EMERGENCY_Sum` |
| `Measure_Value` | Decimal | Numeric measure value | 2457589, 832523 |

#### Category Values Reference

| Category | Description | Example Code Values |
|----------|-------------|---------------------|
| `IMD_Decile` | English Index of Multiple Deprivation decile | `Most deprived 10%`, `More deprived 10-20%`... `Least deprived 10%` |
| `ETHNICITY` | ONS ethnicity classification | `British (White)`, `Indian (Asian or Asian British)`, `Caribbean (Black or Black British)`, `Not stated`, `Not known` |
| `MAINSPEF` | Main Specialty of the consultant responsible for the episode | `100` (General Surgery), `300` (General Medicine), `420` (Geriatrics) |
| `TRETSPEF` | Treatment Specialty — the specialty that actually provided treatment | Same numeric codes as MAINSPEF |
| `CAUSE_3` | ICD-10 external cause of injury (V/W/X/Y codes) | `Assault_Sum`, `Complication_Medical_Surgi_Sum`, `No_Ext_Cause_Sum` |

#### Calculated Columns (added by this model)

| Column | Data Type | Description |
|--------|-----------|-------------|
| `IMD Sort Order` | Integer | 1=Most Deprived → 10=Least Deprived. Critical for correct chart axis ordering |
| `Ethnic Group Broad` | Text | Maps 18 ONS ethnicity descriptions to 7 broad groups: White, Mixed, Asian, Black, Chinese, Other Ethnic Group, Unknown |
| `Is Ethnic Minority` | Boolean | TRUE for known non-White ethnic groups |
| `Category Full Name` | Text | Human-readable label for Category codes |
| `Is Deprivation Row` | Boolean | TRUE when Category = IMD_Decile |
| `Is Ethnicity Row` | Boolean | TRUE when Category = ETHNICITY |
| `Is Specialty Row` | Boolean | TRUE when Category is MAINSPEF or TRETSPEF |

---

### Table 3: `hosp-epis-stat-admi-proc-2024-25` — Procedures
**Description:** FCE activity broken down by OPCS-4 surgical procedure codes at 4-digit (OPERTN_4_01) and 3-digit chapter (OPERTN_3_01) levels. 369,250 rows. NHS suppressed values (`*`) replaced with BLANK.

#### Source Columns

| Column | Data Type | Description | Example Values |
|--------|-----------|-------------|----------------|
| `UID` | Integer | Unique row identifier | 1, 2, 3... |
| `Code` | Text | OPCS-4 procedure code | `A01.1`, `W37.1`, `H33.8` |
| `Category` | Text | Code granularity level | `OPERTN_3_01` (3-digit), `OPERTN_4_01` (4-digit) |
| `Measure` | Text | Metric identifier | `FCE_Sum`, `FAE_EMERGENCY_Sum`, `Age_45_49_Sum` |
| `Measure Value` | Decimal | Numeric value (NULL where NHS suppression applied) | 15234, BLANK |

#### OPCS-4 Chapter Reference

| Chapter Letter | Procedure Group |
|----------------|----------------|
| A | Nervous system |
| B | Endocrine system and breast |
| C | Eye |
| D | Ear |
| E | Respiratory tract |
| F | Mouth |
| G | Upper digestive system |
| H | Lower digestive system |
| J | Other abdominal organs |
| K | Heart |
| L | Arteries and veins |
| M | Urinary |
| N | Male genital organs |
| P | Lower female genital tract |
| Q | Upper female genital tract |
| R | Female genital tract (pregnancy) |
| S | Skin |
| T | Soft tissue |
| V | Bones and joints of skull and spine |
| W | Other bones and joints |
| X | Miscellaneous operations |
| Y | Subsidiary classification |
| Z | Subsidiary classification (sites) |

#### Calculated Columns (added by this model)

| Column | Data Type | Description |
|--------|-----------|-------------|
| `Procedure Code Level` | Text | Human-readable label: `OPCS-3 (3-digit chapter)` or `OPCS-4 (4-digit detailed)` |
| `Procedure Chapter` | Text | First letter of OPCS code = procedure chapter (A–Z) |
| `Measure Type Group` | Text | Groups Measure values: Total Count, Gender Split, Age Band, Admission Type, Duration Metric, Other |

---

### Table 4: `hosp-epis-stat-admi-diag-2024-25` — Diagnoses
**Description:** FCE activity broken down by ICD-10 primary diagnosis codes at 4-digit detailed (DIAG_4_01) and 3-digit chapter (DIAG_3_01) levels. 508,002 rows — largest table in the model. NHS suppressed values replaced with BLANK.

#### Source Columns

| Column | Data Type | Description | Example Values |
|--------|-----------|-------------|----------------|
| `UID` | Integer | Unique row identifier | 1, 2, 3... |
| `Code` | Text | ICD-10 diagnosis code | `C34.1`, `I21.0`, `J18.9` |
| `Category` | Text | Code granularity level | `DIAG_3_01` (3-digit), `DIAG_4_01` (4-digit) |
| `Attribute` | Text | Metric identifier (same vocabulary as other tables) | `FCE_SUM`, `FAE_EMERGENCY_Sum` |
| `Value` | Decimal | Numeric value (NULL where NHS suppression applied) | 45000, BLANK |

#### ICD-10 Chapter Reference

| ICD-10 Range | Chapter Name | Clinical Group | Preventable |
|---|---|---|---|
| A00–B99 | Infectious & Parasitic Diseases | Communicable | No |
| C00–D48 | Neoplasms (Cancer) | Oncology | No |
| D50–D89 | Blood & Immune Disorders | Non-communicable | No |
| E00–E90 | Endocrine, Nutritional & Metabolic | Non-communicable | Yes |
| F00–F99 | Mental & Behavioural Disorders | Mental Health | No |
| G00–G99 | Nervous System Diseases | Non-communicable | No |
| H00–H59 | Eye & Adnexa Disorders | Non-communicable | No |
| H60–H95 | Ear & Mastoid Disorders | Non-communicable | No |
| I00–I99 | Circulatory System Diseases | Cardiovascular | Yes |
| J00–J99 | Respiratory System Diseases | Respiratory | Yes |
| K00–K93 | Digestive System Diseases | Non-communicable | No |
| L00–L99 | Skin & Subcutaneous Disorders | Non-communicable | No |
| M00–M99 | Musculoskeletal Disorders | Non-communicable | No |
| N00–N99 | Genitourinary Diseases | Non-communicable | No |
| O00–O99 | Pregnancy, Childbirth & Puerperium | Maternity | No |
| P00–P96 | Perinatal Conditions | Maternity | No |
| Q00–Q99 | Congenital Malformations | Non-communicable | No |
| R00–R99 | Symptoms, Signs & Abnormal Findings | Unspecified | No |
| S00–T98 | Injury, Poisoning & External Causes | Injury | Yes |
| U00–U89 | Special Purposes (incl. COVID-19) | Other | No |
| Z00–Z99 | Factors Influencing Health Status | Other | No |

#### Calculated Columns (added by this model)

| Column | Data Type | Description |
|--------|-----------|-------------|
| `Diagnosis Code Level` | Text | `ICD-10 (3-digit chapter)` or `ICD-10 (4-digit detailed)` |
| `ICD10 Chapter Letter` | Text | First letter of ICD-10 code = clinical chapter (A–Z) |
| `Is Maternity Code` | Boolean | TRUE for O and P chapter codes |
| `Is Mental Health Code` | Boolean | TRUE for F chapter codes |
| `Is Cancer Code` | Boolean | TRUE for C and D chapter codes |

---

### Table 5: `hosp-epis-stat-admi-hosp-provider-2024-25-tab` — Provider Table
**Description:** One row per NHS provider organisation (607 providers). Wide format with 50 unnamed columns. Contains provider-level summary statistics. Approximately 15% of cells suppressed due to small patient counts.

| Column | Data Type | Description |
|--------|-----------|-------------|
| `Column1` | Text | NHS Provider Organisation Code |
| `Column2` | Text | NHS Provider Organisation Name |
| `Column3–Column50` | Various | Unnamed metric columns — require reshaping in Power Query before use |

> ⚠️ **Note:** This table requires unpivoting and column renaming in Power Query before it can be used in visualisations. The current model retains it in wide format.

---

### Dimension Tables

### `Calendar` — NHS Financial Year Calendar
**Description:** Date dimension covering April 2019 to March 2026. 2,557 rows. Marked as Date Table on the `Date` column. Bridges to fact tables via `_NHS_Year_Lookup`.

| Column | Data Type | Description |
|--------|-----------|-------------|
| `Date` | Date | Primary key — one row per calendar day |
| `Year` | Integer | Calendar year (e.g. 2024) |
| `Month Number` | Integer | Month as integer 1–12 |
| `Month Name` | Text | Full month name (e.g. April). Sorted by Month Number |
| `Month Short` | Text | 3-letter month abbreviation (e.g. Apr) |
| `Quarter Number` | Integer | Calendar quarter 1–4 |
| `Quarter` | Text | Calendar quarter label (e.g. Q1). Sorted by Quarter Number |
| `NHS Year` | Text | NHS financial year (e.g. 2024/25) |
| `NHS Year Label` | Text | Compact FY label (e.g. FY2024) |
| `Week Number` | Integer | ISO week number 1–53 |
| `Day of Week` | Text | Full day name (e.g. Monday). Sorted by Day Number |
| `Day Number` | Integer | Day of week: 1=Monday, 7=Sunday |
| `Is Weekend` | Boolean | TRUE for Saturday and Sunday |
| `Is Weekday` | Boolean | TRUE for Monday to Friday |
| `Month Year` | Text | Axis label (e.g. Apr 2024). Sorted by YYYYMM |
| `YYYYMM` | Integer | Numeric sort key (e.g. 202404) |
| `Day Name Short` | Text | 3-letter day abbreviation (e.g. Mon) |
| `Month Sort` | Integer | Hidden numeric sort key for Month Name |

---

### `_NHS_Year_Lookup` — Financial Year Bridge Table
**Description:** Bridge table with exactly one row per NHS financial year. Provides a unique key that enables clean Many-to-One relationships between the Calendar and fact tables without Many-to-Many cardinality issues.

| Column | Data Type | Description |
|--------|-----------|-------------|
| `NHS_Year` | Text | **Unique key** — NHS financial year (e.g. 2024/25) |
| `NHS_Year_Label` | Text | Short FY label (e.g. FY2024) |
| `Start_Year` | Integer | Calendar year the financial year starts (e.g. 2024) |
| `End_Year` | Integer | Calendar year the financial year ends (e.g. 2025) |
| `Is_Current_Year` | Boolean | TRUE for the current reporting year (2024/25) |

---

### Hidden Reference Tables

### `_Specialty_Ref` *(Hidden in Model View)*
**Description:** Maps 40+ MAINSPEF/TRETSPEF numeric specialty codes to human-readable names and clinical groups.

| Column | Description |
|--------|-------------|
| `Specialty_Code` | MAINSPEF/TRETSPEF numeric code as text (join key) |
| `Specialty_Name` | Human-readable name (e.g. General Medicine, Geriatrics) |
| `Specialty_Group` | Clinical grouping: Surgery, Medicine, Emergency, Oncology, Women & Children, Mental Health, Diagnostics, Support |
| `Is_Emergency_Specialty` | TRUE for primarily emergency-facing specialties |

### `_Diagnosis_Ref` *(Hidden in Model View)*
**Description:** Maps 21 ICD-10 diagnosis chapter codes to clinical names and groups.

| Column | Description |
|--------|-------------|
| `Chapter_Code` | ICD-10 chapter code as used in MEASURE column (e.g. C00_D48) |
| `Chapter_Name` | Full ICD-10 chapter name |
| `Chapter_Group` | Clinical grouping: Oncology, Cardiovascular, Respiratory, Mental Health, Maternity, Injury, etc. |
| `Is_Preventable` | TRUE for chapters with significant preventable component |

### `_IMD_Deprivation_Bridge` *(Hidden in Model View)*
**Description:** Provides sortable deprivation decile labels for correct chart axis ordering.

| Column | Description |
|--------|-------------|
| `IMD_Code` | Exact text match to Code column in oth table |
| `IMD_Sort_Order` | 1=Most Deprived, 10=Least Deprived |
| `IMD_Label_Short` | Short display label (e.g. Decile 1 - Most Deprived) |
| `IMD_Group` | Most Deprived / More Deprived / Less Deprived / Least Deprived |
| `Is_Most_Deprived_Half` | TRUE for deciles 1–5 |

### `_Ethnicity_Groups` *(Hidden in Model View)*
**Description:** Maps 18 ONS ethnicity descriptions to 7 broad groups.

| Column | Description |
|--------|-------------|
| `Ethnicity_Detail` | Exact ONS ethnicity description (join key) |
| `Ethnicity_Broad_Group` | White / Mixed / Asian / Black / Chinese / Other / Unknown |
| `Ethnicity_Sort` | Sort order 1–7 for consistent chart ordering |
| `Is_Ethnic_Minority` | TRUE for all non-White known ethnicity groups |

### `_Measures` — Dedicated Measures Container *(Not hidden)*
**Description:** Empty table that holds all 54 DAX measures. Contains one hidden placeholder column. Measures are organised into 7 display folders for easy navigation in the Fields pane.

---

## Power BI Model Architecture

```
Calendar ──────────────────────────────────────────────────────────────┐
    │ NHS Year (Many)                                                   │
    │ ↓                                                                 │
_NHS_Year_Lookup                                                        │
    │ NHS_Year (One)     (One)                                          │
    │ ↓                  ↓                                              │
    │ NHS Financial Year (Many)                                         │
    │ hosp-epis-stat-admi-pla-2024-25 ──── UID (1:1) ──── hosp-epis-stat-admi-proc-2024-25
    │                                  └── UID (1:1) ──── hosp-epis-stat-admi-oth-2024-25
    │                                  └── UID (1:1*) ─── hosp-epis-stat-admi-diag-2024-25
    └──────────────────────────────────────────────────────────────────┘

* = inactive relationship (ambiguous path)

Hidden Reference Tables (no relationships — used via DAX LOOKUPVALUE):
  _Specialty_Ref | _Diagnosis_Ref | _IMD_Deprivation_Bridge | _Ethnicity_Groups
```

### Relationship Summary

| Name | From | To | Cardinality | Active |
|------|------|----|-------------|--------|
| `Calendar_to_NHSYearLookup` | `Calendar[NHS Year]` | `_NHS_Year_Lookup[NHS_Year]` | Many-to-One | ✅ Yes |
| `pla_to_NHSYearLookup` | `pla[NHS Financial Year]` | `_NHS_Year_Lookup[NHS_Year]` | Many-to-One | ✅ Yes |
| Auto-detected | `pla[UID]` | `proc[UID]` | One-to-One | ✅ Yes |
| Auto-detected | `oth[UID]` | `proc[UID]` | One-to-One | ✅ Yes |
| Auto-detected | `proc[UID]` | `diag[UID]` | One-to-One | ✅ Yes |
| Auto-detected | `pla[UID]` | `diag[UID]` | One-to-One | ❌ Inactive |
| Auto-detected | `oth[UID]` | `diag[UID]` | One-to-One | ❌ Inactive |

---

## DAX Measures Reference

All 54 measures live in the `_Measures` table, organised into 7 display folders.

### Folder 01 — Core Activity

| Measure | Format | England Value (FY2024/25) | Description |
|---------|--------|---------------------------|-------------|
| `Total FCEs` | `#,##0` | 22,555,615 | Total Finished Consultant Episodes |
| `Total FAEs` | `#,##0` | 18,468,856 | Total Finished Admission Episodes |
| `Total Bed Days` | `#,##0` | 48,360,197 | Total inpatient bed days consumed |
| `Day Case FCEs` | `#,##0` | 8,661,338 | Day case (no overnight stay) FCEs |
| `Ordinary Admissions` | `#,##0` | 13,894,277 | Overnight inpatient admissions |
| `Day Case Rate %` | `0.0%` | 38.4% | Day cases as % of total FCEs. NHS target: 80%+ |
| `Bed Days per FCE` | `0.00` | 2.14 | Average bed days per FCE — efficiency metric |
| `Provider Count` | `#,##0` | 590 | Distinct NHS provider organisations |

### Folder 02 — Emergency & Urgent Care

| Measure | Format | England Value | Description |
|---------|--------|---------------|-------------|
| `Emergency Admissions` | `#,##0` | 6,614,976 | Total unplanned emergency FAEs |
| `Emergency Rate %` | `0.0%` | 35.8% | Emergency as % of all FAEs |
| `Waiting List Admissions` | `#,##0` | 7,322,118 | Admitted from elective waiting list |
| `Planned Admissions` | `#,##0` | 2,638,492 | Pre-booked elective admissions |
| `Elective % of Admissions` | `0.0%` | 54.2% | Planned + WL as % of all FAEs |
| `SES Emergency Spells` | `#,##0` | 2,430,157 | NHS Outcomes Framework emergency spell proxy |

### Folder 03 — Waiting Times

| Measure | Format | England Value | Description |
|---------|--------|---------------|-------------|
| `Avg Elective Wait Days` | `0.0` | 83.6 | Mean elective waiting time in days |
| `Median Elective Wait Days` | `0` | 34 | Median elective wait — less sensitive to outliers |
| `Avg LOS Days` | `0.0` | 4.69 | Mean inpatient spell length |
| `Median LOS Days` | `0` | 1 | Median spell length |
| `Waiting Under 1 Month` | `#,##0` | 2,857,425 | Admitted within 1 month |
| `Waiting 1 to 3 Months` | `#,##0` | 1,730,742 | Admitted after 1–3 months |
| `Waiting 3 to 6 Months` | `#,##0` | 728,786 | Admitted after 3–6 months |
| `Waiting 6 to 12 Months` | `#,##0` | 528,533 | RTT breach zone |
| `Waiting 12 to 18 Months` | `#,##0` | 178,489 | Critical backlog |
| `Waiting 18 Months Plus` | `#,##0` | 72,587 | Most critical backlog indicator. Target: 0 |
| `Backlog 6 Months Plus` | `#,##0` | 779,609 | Composite 6m+ backlog |
| `18 Month Plus % of Backlog` | `0.0%` | 9.3% | 18m+ as % of all 6m+ backlog |

### Folder 04 — Demographics

| Measure | Format | England Value | Description |
|---------|--------|---------------|-------------|
| `Male FCEs` | `#,##0` | 10,125,067 | FCEs for male patients |
| `Female FCEs` | `#,##0` | 12,282,856 | FCEs for female patients |
| `Female % of FCEs` | `0.0%` | 54.8% | Female share of total admissions |
| `Elderly FCEs 75 Plus` | `#,##0` | 6,211,987 | FCEs for patients aged 75+ |
| `Elderly % of FCEs` | `0.0%` | 27.5% | Elderly share of total FCEs |
| `Working Age FCEs` | `#,##0` | 10,325,007 | FCEs for patients aged 16–64 |
| `Children FCEs Under 16` | `#,##0` | 2,051,973 | FCEs for patients under 16 |
| `Maternity Admissions` | `#,##0` | 1,317,464 | Pregnancy & childbirth FAEs (O00–O99) |

### Folder 05 — Deprivation & Equity

| Measure | Format | England Value | Description |
|---------|--------|---------------|-------------|
| `Most Deprived FCEs` | `#,##0` | 2,457,589 | FCEs — most deprived 10% (IMD Decile 1) |
| `Least Deprived FCEs` | `#,##0` | 1,922,609 | FCEs — least deprived 10% (IMD Decile 10) |
| `Deprivation Emergency Gap` | `#,##0` | 311,183 | Absolute emergency admission gap (most vs least deprived) |
| `Deprivation Emergency Ratio` | `0.00` | 1.60 | Emergency rate ratio (most/least deprived). Target: 1.0 |
| `Deprivation Bed Day Gap` | `#,##0` | 1,921,388 | Bed day gap (most vs least deprived) |
| `Most Deprived Emergency Rate %` | `0.0%` | 33.9% | Emergency rate in most deprived 10% |
| `Ethnicity Missing Rate %` | `0.0%` | 13.0% | FCEs with unknown ethnicity — CORE20PLUS5 data quality risk |

### Folder 06 — Clinical Diagnosis

| Measure | Format | England Value | ICD-10 |
|---------|--------|---------------|--------|
| `Cancer Admissions C00 D48` | `#,##0` | 2,517,314 | Neoplasms |
| `Digestive Admissions K00 K93` | `#,##0` | 2,513,714 | Digestive system |
| `Respiratory Admissions J00 J99` | `#,##0` | 1,105,324 | Respiratory system |
| `Circulatory Admissions I00 I99` | `#,##0` | 1,014,464 | Cardiovascular |
| `Mental Health Admissions F00 F99` | `#,##0` | 135,157 | Mental & behavioural |
| `Musculoskeletal Admissions M00 M99` | `#,##0` | 1,306,159 | Musculoskeletal |
| `No Procedure FCEs` | `#,##0` | 9,017,535 | Medical admissions with no surgical procedure |

### Folder 07 — Advanced KPIs

| Measure | Format | England Value | Description |
|---------|--------|---------------|-------------|
| `FCEs per Bed Day` | `0.00` | 0.47 | Inverse of bed days per FCE — throughput efficiency |
| `Emergency Admission Index` | `0.0` | 29.3 | Emergency admissions per 100 FCEs. Values >40 = high pressure |
| `Elective Demand Ratio` | `0.00` | 2.78 | Waiting list admissions ÷ planned admissions. Values >1 = undercapacity |
| `Backlog Severity Score` | `0.00` | 0.151 | Weighted composite: (18m+×3)+(12–18m×2)+(6–12m×1) ÷ total WL |
| `SES Elective Spells` | `#,##0` | 220,081 | NHS Outcomes Framework elective spell proxy |
| `Deprivation FCE Uplift %` | `0.0%` | 27.8% | % more FCEs in most deprived vs least deprived decile |

---

## Key Insights & Findings

The following insights are drawn directly from DAX measures verified against the live Power BI model via the Analysis Services XMLA endpoint. All figures relate to England, financial year 2024/25.

---

### 🚨 Insight 1 — South West is a System Failure, Not a System Pressure

The South West Commissioning Region records a mean elective wait of **125.8 days** — 90% longer than London's 66.3 days and 50% above the national mean of 83.6 days. Crucially, the South West holds **31,415 of England's 72,587 patients waiting 18+ months (43% of the entire national backlog)**, despite the region accounting for only 10.6% of national FCE volume (2,385,115 of 22,555,615 FCEs).

> **Implication:** This is a structural capacity failure requiring targeted capital and workforce investment, not a demand management problem. The magnitude of the 18-month backlog in a single region is unprecedented and requires urgent commissioning intervention.

| Region | FCEs | Avg Wait (days) | 18m+ Backlog |
|--------|------|-----------------|--------------|
| Midlands | 4,458,065 | 80.9 | 9,160 |
| NE & Yorkshire | 3,671,825 | 77.4 | 5,955 |
| London | 3,369,800 | **66.3 ✅** | 6,025 |
| South East | 3,193,025 | 81.9 | 6,030 |
| North West | 3,016,805 | 79.4 | 8,370 |
| East of England | 2,460,895 | 87.6 | 5,620 |
| **South West** | 2,385,115 | **125.8 🔴** | **31,415 🔴** |

---

### 📊 Insight 2 — Deprivation Drives Emergency Demand with a 1.60× Multiplier

The most deprived 10% of England's population generate **832,523 emergency admissions** versus **521,340** in the least deprived — a Deprivation Emergency Ratio of **1.60**. They also consume **5,501,922 bed days** versus **3,580,534** in the least deprived — a gap of **1,921,388 additional bed days** (54% more). The gradient is monotonic across all 10 IMD deciles.

> **Implication:** Targeted public health investment in deprived communities would simultaneously reduce emergency demand and release bed capacity. This is the most powerful single lever for system-level efficiency improvement.

| IMD Decile | FCEs | Emergency Admissions | Bed Days |
|------------|------|---------------------|----------|
| Most deprived 10% | 2,457,589 | 832,523 | 5,501,922 |
| More deprived 10–20% | 2,300,041 | 735,929 | 5,090,735 |
| More deprived 20–30% | 2,245,227 | 681,027 | 5,375,858 |
| More deprived 30–40% | 2,225,333 | 660,283 | 4,756,658 |
| More deprived 40–50% | 2,204,574 | 642,752 | 4,824,792 |
| Less deprived 40–50% | 2,214,350 | 635,645 | 4,608,993 |
| Less deprived 30–40% | 2,147,008 | 608,432 | 4,289,480 |
| Less deprived 20–30% | 2,119,200 | 589,202 | 4,172,308 |
| Less deprived 10–20% | 2,069,656 | 570,623 | 3,922,045 |
| **Least deprived 10%** | 1,922,609 | 521,340 | 3,580,534 |

---

### 📈 Insight 3 — Day Case Rate at Half the NHS Target Represents the Largest Efficiency Lever

England's day case rate is **38.4%** (8,661,338 of 22,555,615 FCEs). The NHS target is **80%+** for suitable elective procedures — a gap of 41.6 percentage points. If half this gap were closed, approximately **4.7 million overnight episodes could be converted to day cases**, releasing an estimated **22 million bed days** annually.

> **Implication:** Day case rate improvement is the single largest available efficiency gain in the NHS. Targeting specialties with high surgical volumes and low current day case rates (Orthopaedics, Ophthalmology, General Surgery) would yield the fastest results.

---

### 🏥 Insight 4 — Cancer and Digestive Disorders are the Joint Largest Disease Burden

**Cancer (C00–D48)** accounts for **2,517,314 FAEs** and **Digestive (K00–K93)** for **2,513,714 FAEs** — together **27.3% of all 18,468,856 national admissions**. Both chapters exceed Respiratory (1,105,324) and Circulatory (1,014,464) combined.

> **Implication:** Theatre capacity planning, oncology workforce and digestive endoscopy services must be prioritised in commissioning cycles.

---

### 🔬 Insight 5 — 40% of FCEs Have No Procedure — The Invisible Medical Burden

**9,017,535 FCEs** (40% of total) have no recorded OPCS-4 surgical procedure code. This medical admission cohort is responsible for the dramatic statistical gap between **mean LOS of 4.69 days** and **median LOS of 1 day** — the median reflects the high proportion of short-stay and day-case admissions; the mean is pulled by long-stay medical patients.

> **Implication:** The beds crisis is fundamentally a medical, not a surgical, problem. General Medicine (Specialty 300) alone generates 3,265,887 FCEs and 8,643,068 bed days with 1,731,795 emergency admissions.

---

### 👥 Insight 6 — Female Admissions 21% Higher but Clinically Concentrated

**12,282,856 female FCEs** versus **10,125,067 male** (54.8% female). The excess is clinically concentrated in two areas: **maternity (1,317,464 O00–O99 admissions)** and the **75+ age bands** where women outnumber men due to greater longevity. Outside these cohorts, male and female admission rates are broadly similar.

> **Implication:** The gender gap is not a general policy issue but a targeted one requiring maternity pathway investment and geriatric care for older women.

---

### ⏱ Insight 7 — Waiting List is 2.78× the Size of Planned Admissions

With **7,322,118 waiting list admissions** versus **2,638,492 planned admissions**, the Elective Demand Ratio is **2.78**. The system operates on reactive throughput rather than planned care delivery. The **Backlog Severity Score of 0.151** indicates the most severe waits (18m+) do not yet dominate the backlog composition — early intervention can still prevent further deterioration.

> **Implication:** Planned admissions capacity must be expanded by 178% to match current waiting list demand. Without this, the backlog will continue to grow regardless of productivity improvements.

---

### 👴 Insight 8 — The 75+ Age Cohort is 27.5% of FCEs and Rising

**6,211,987 FCEs** for patients aged 75 and over represent **27.5% of total inpatient activity**. The **75–79 age band alone accounts for 2,276,417 FCEs** — the single largest individual age cohort in the entire dataset. This cohort has significantly longer LOS than younger age groups, meaning their bed day share substantially exceeds their FCE share.

> **Implication:** NHS demand forecasting models must account for accelerating elderly population growth. Geriatric Medicine (Specialty 420) capacity planning is a critical commissioning priority.

---

### 📋 Insight 9 — 13% Ethnicity Data Gap is a CORE20PLUS5 Compliance Risk

**13.0% of FCE records** have ethnicity coded as Not Stated or Not Known — approximately **2,929,044 FCEs** (1,973,824 Not Stated + 955,220 Not Known). This directly limits the ability to measure and report on ethnic health inequalities as required under the **NHS CORE20PLUS5 framework**, which mandates action on healthcare inequalities for the most deprived 20% and five clinical areas.

> **Implication:** Improving ethnicity data recording completeness is a mandatory quality improvement priority before the next annual HES submission. Provider-level ethnicity recording rates should be audited.

---

### 🧠 Insight 10 — Mental Health Inpatient Demand is Structurally Invisible

Only **135,157 FAEs (0.7% of all admissions)** are recorded under ICD-10 F00–F99 (Mental & Behavioural Disorders). NHS Mental Health Trusts and community services operate under separate data collections (MHSDS) and are largely excluded from HES acute admitted patient care data. This renders mental health inpatient demand structurally invisible in this dataset.

> **Implication:** Any NHS system-level analysis using HES data alone will dramatically understate the true mental health inpatient burden. MHSDS supplementary data must be joined for complete analysis.

---

## Report Design Guide

This model supports six Power BI report pages. See the full HTML analytics platform for interactive chart previews.

| Report | Audience | Key Visuals | Primary Measures |
|--------|----------|-------------|------------------|
| **01 Executive Summary** | Trust Board, CEO | 5 KPI cards, bullet chart vs targets | Total FCEs, Emergency Rate %, Waiting 18 Months Plus |
| **02 Regional Benchmarking** | Commissioning | Filled England map, sorted bar chart | Avg Elective Wait Days, Backlog 6 Months Plus |
| **03 Deprivation & Equity** | Public Health | 10-decile gradient bar, ethnicity donut | Deprivation Emergency Ratio, Ethnicity Missing Rate % |
| **04 Elective Backlog** | Operations, RTT | 100% stacked bar by wait band, What-If | Waiting 18 Months Plus, Backlog Severity Score |
| **05 Clinical Demand** | Medical Director | Specialty × age heatmap, ICD-10 bar | Cancer/Respiratory/Circulatory/Mental Health measures |
| **06 Provider Benchmarking** | Finance, Performance | Bubble scatter: LOS vs Emergency Rate | Emergency Admission Index, Bed Days per FCE |

### Key Filtering Rules

- **Always filter `ORG_LEVEL`** in pla table visuals — mixing England, Region and Provider rows produces double-counting
- **Use `Is Provider Level = TRUE`** on all Provider Benchmarking page visuals
- **Use `Is Deprivation Row = TRUE`** when displaying IMD analysis to avoid mixing ethnicity/specialty rows
- **Use `Is Ethnicity Row = TRUE`** when displaying ethnicity analysis
- **Default Calendar slicer to `Is_Current_Year = TRUE`** in _NHS_Year_Lookup for correct FY2024/25 display

---

## Data Quality Notes

| Issue | Affected Tables | Scale | Resolution |
|-------|----------------|-------|------------|
| NHS statistical suppression (`*`) | proc, pla, diag | 143 / ~200 / 587 rows | Replaced with BLANK in Power Query — excluded from aggregation |
| Provider table wide format | provider-tab | All 50 columns unnamed | Requires Power Query unpivot before use in visuals |
| Ethnicity missing data | oth (ETHNICITY) | 13.0% of FCEs | Flagged via `Ethnicity Missing Rate %` measure |
| Mental health under-count | diag, pla | Structural | MHSDS supplementary data required |
| Single-year model | All tables | REPORTING_PERIOD = 2425 only | Calendar supports FY2019–FY2026 for future multi-year expansion |
| UID relationship ambiguity | pla ↔ diag, oth ↔ diag | 2 inactive relationships | DAX measures filter tables directly — relationships not required for measure logic |

---

## How to Use This Model

### Prerequisites
- Power BI Desktop (March 2026 or later)
- Source CSV files from NHS England HES publication
- File paths must match those in the Power Query M expressions

### Setup Steps

1. **Download source data** from the NHS England HES publication URL above
2. **Open the Power BI Desktop file** — Power Query will attempt to load from the configured CSV paths
3. **Update file paths** in Power Query if your files are stored in a different directory:
   - Go to Transform Data → Data source settings → Change Source
4. **Refresh the model** — all 5 fact tables will load
5. **Mark Calendar as Date Table** — right-click Calendar table → Mark as Date Table → select `Date` column (if not already set)
6. **Verify relationships** in Model view — 7 relationships should be present including `Calendar_to_NHSYearLookup` and `pla_to_NHSYearLookup`
7. **Configure Row-Level Security** before publishing to Power BI Service if provider-level data access needs to be restricted by organisation

### Publishing to Power BI Service
- Configure RLS roles per organisational access requirements before publishing
- Provider-level data is commercially sensitive — apply appropriate access controls
- Schedule data refresh via Power BI Service gateway if connected to live NHS Digital feeds

---

## References

| Source | URL |
|--------|-----|
| NHS England HES Publications | https://digital.nhs.uk/data-and-information/publications/statistical/hospital-admitted-patient-care-activity |
| NHS Digital HES Overview | https://digital.nhs.uk/data-and-information/data-tools-and-services/data-services/hospital-episode-statistics |
| NHS Data Access Request Service (DARS) | https://digital.nhs.uk/services/data-access-request-service-dars |
| NHS England Open Statistics | https://www.england.nhs.uk/statistics/statistical-work-areas/hospital-activity/ |
| CORE20PLUS5 Framework | https://www.england.nhs.uk/about/equality/equality-hub/national-healthcare-inequalities-improvement-programme/core20plus5/ |
| ICD-10 Classification (WHO) | https://www.who.int/standards/classifications/classification-of-diseases |
| OPCS-4 Classification (NHS Digital) | https://digital.nhs.uk/services/terminology-and-classifications/opcs-classification |
| English Indices of Deprivation 2019 (IMD) | https://www.gov.uk/government/statistics/english-indices-of-deprivation-2019 |
| ONS Ethnicity Classifications | https://www.ons.gov.uk/methodology/classificationsandstandards/measuringequality/ethnicgroupnationalidentityandreligion |
| NHS RTT Waiting Times Policy | https://www.england.nhs.uk/rtt/ |
| Power BI DAX Reference | https://learn.microsoft.com/en-us/dax/ |
| Open Government Licence v3.0 | https://www.nationalarchives.gov.uk/doc/open-government-licence/version/3/ |

---

## Licence

The source NHS HES data is published under the **Open Government Licence v3.0** by NHS England.
This Power BI model, DAX measures, data dictionary and analysis are produced for analytical and educational purposes.

---

*Last updated: March 2026 | Model verified against Power BI Desktop Analysis Services XMLA endpoint | All figures relate to England, Financial Year 2024/25*
