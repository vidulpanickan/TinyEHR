# About the Data

TinyEHR is derived from the MIMIC-IV Clinical Database Demo v2.2 (Johnson et al., 2023) and the MIMIC-IV Demo Data in the OMOP Common Data Model v0.9. The OMOP CDM is a standardized data model for observational health data maintained by OHDSI (Observational Health Data Sciences and Informatics). This document describes every transformation applied to the original data and the reasoning behind each decision.

TinyEHR ships in two formats:

**MIMIC format**: Matches the original MIMIC-IV Demo exactly in column names, data types, and values, with two transformations (date shifting and ICD code formatting) and one addition (synthetic clinical notes). Scripts written for MIMIC-IV Demo work with TinyEHR without modification.

**OMOP format**: Uses the OHDSI MIMIC-to-OMOP reference data as-is, with date shifting and synthetic note insertion. ICD (International Classification of Diseases) codes in OMOP `source_value` fields are stored without decimals, following the OMOP convention of preserving the original billing/claims format.

---

## What Changed

### 1. Date Shifting

The original MIMIC-IV Demo uses synthetic dates in the 2100+ range to protect patient privacy. TinyEHR shifts these to realistic but fictional years (2010s-2020s) for easier use in teaching and prototyping.

**How it works:**
- Each patient's `anchor_year_group` (a MIMIC field giving a 3-year range around the patient's true admission year, e.g., "2011 - 2013") is used to select a random real year
- A per-patient year offset is computed: `offset = original_anchor_year - real_year`
- All date and datetime columns are shifted backward by this offset
- Leap year edge cases are handled (Feb 29 becomes Feb 28 when the target year is not a leap year)
- Individual patient offsets are saved in `metadata/date_offsets.csv` for reproducibility

**Tables modified (MIMIC format):** admissions, transfers, services, labevents, microbiologyevents, prescriptions, pharmacy, emar, emar_detail, poe, hcpcsevents, diagnoses_icd, procedures_icd, drgcodes, icustays, chartevents, datetimeevents, inputevents, outputevents, procedureevents, ingredientevents, patients (dod column)

**Tables modified (OMOP format):** visit_occurrence, visit_detail, condition_occurrence, procedure_occurrence, drug_exposure, device_exposure, measurement, observation, death, observation_period, specimen, condition_era, drug_era, dose_era, person (year_of_birth)

### 2. ICD Code Formatting (MIMIC format only)

ICD codes are the standard classification system used worldwide to record diagnoses and procedures. The original MIMIC-IV Demo stores these codes without decimal points (e.g., `4139` instead of `413.9`). TinyEHR's MIMIC format inserts decimal points to match how codes appear in clinical practice, EHR systems, medical coding textbooks, and official CDC (Centers for Disease Control) / WHO code listings.

| Code System | Rule | Example | Source |
|---|---|---|---|
| ICD-9-CM Diagnosis (older US diagnosis codes) | Decimal point after 3rd digit | `4139` → `413.9` | CDC ICD-9-CM Guidelines |
| ICD-9-CM Procedure (older US procedure codes) | Decimal point after 2nd digit | `3961` → `39.61` | CMS Appendix F, AAPC |
| ICD-10-CM Diagnosis (current US diagnosis codes) | Decimal point after 3rd character | `E119` → `E11.9` | CMS ICD-10-CM Guidelines |
| ICD-10-PCS Procedure (inpatient procedure codes) | No decimal point | `02H633Z` unchanged | ICD-10-PCS specification |

> [!IMPORTANT]
> ICD-9 diagnosis codes and ICD-9 procedure codes have different decimal point placement rules. Diagnosis codes place the decimal point after the 3rd digit (category code), while procedure codes place the decimal point after the 2nd digit (two-digit category).

**Tables affected:** diagnoses_icd, d_icd_diagnoses, procedures_icd, d_icd_procedures

> [!NOTE]
> **Why OMOP format does not add decimal points**
>
> The OMOP CDM convention is that `source_value` fields store codes exactly as they appear in the source system. In billing and claims data, ICD codes are submitted without decimal points because CMS (Centers for Medicare & Medicaid Services) requires this on claim forms. This means the same diagnosis appears differently in each format:

| Format | Example | Convention |
|--------|---------|------------|
| TinyEHR MIMIC | `413.9` | Clinical display (textbooks, EHR systems) |
| TinyEHR OMOP | `4139` | Billing/claims format (CMS, Medicare) |

> [!TIP]
> This is intentional and reflects a real-world difference that learners will encounter when working with clinical vs billing data.

### 3. Synthetic Clinical Notes

The original MIMIC-IV Demo does not include clinical notes. TinyEHR adds 4,580 synthetic notes generated using a large language model, grounded in each patient's demographics, diagnoses, and admission data.

**14 note types:** Discharge summary (275), Physician (1,249), Nursing (1,249), Radiology (332), Nursing/other (314), ECG (246), Rehab Services (203), Respiratory (193), Echo (128), General (128), Case Management (109), Nutrition (68), Social Work (51), Consult (35)

**MIMIC format:** Added as `noteevents.csv` in `tinyehr_mimic_format/notes/` with columns: `note_id`, `subject_id`, `hadm_id`, `note_type`, `chartdate`, `charttime`, `text`. Each `note_id` follows the MIMIC-IV convention: `{subject_id}-{type_code}-{sequence}` (e.g., `10000032-DS-0001`).

**OMOP format:** Mapped to the `note` table with proper concept IDs:
- `note_type_concept_id` uses OMOP Note Type vocabulary. Concept IDs are standardized numeric identifiers that map local codes to universal meanings (e.g., 44814637 = Discharge summary)
- `note_class_concept_id` uses LOINC (Logical Observation Identifiers Names and Codes) Document Ontology (e.g., 706531 = Discharge summary, 706550 = Progress)
- `encoding_concept_id` = 32678 (UTF-8), `language_concept_id` = 4180186 (English)

The MIMIC-IV OMOP distribution does not include note-related concept IDs because its note table is empty. TinyEHR adds 19 concepts from the [Athena](https://athena.ohdsi.org/) OMOP vocabulary (OHDSI's standard vocabulary browser and download service) (10 Note Type, 7 LOINC Document Ontology, 2 utility), increasing `2b_concept.csv` from 3,885 to 3,904 rows.

---

## What Is Not Changed

- **All clinical values** (lab results, vital signs, medication doses, procedure codes) are identical to the original MIMIC-IV Demo
- **Patient demographics** (gender, anchor_age, anchor_year_group) are unchanged
- **Table structure** (column names, data types) matches the original schema
- **Referential integrity** (subject_id, hadm_id, stay_id linkages) is preserved
- **Row counts** for all original tables are identical to MIMIC-IV Demo v2.2

<details>
<summary><b>Table Reference (row counts)</b></summary>

### MIMIC-IV Format (33 tables)

#### Hospital tables (hosp/)

| Table | Rows | Description |
|-------|------|-------------|
| patients | 100 | Patient demographics |
| admissions | 275 | Hospital admissions |
| transfers | 1,190 | Patient transfers between units |
| services | 319 | Clinical service assignments |
| diagnoses_icd | 4,506 | ICD diagnosis codes per admission |
| procedures_icd | 722 | ICD procedure codes per admission |
| drgcodes | 454 | Diagnosis-related group codes |
| hcpcsevents | 61 | HCPCS procedure codes |
| labevents | 107,727 | Laboratory test results |
| microbiologyevents | 2,899 | Microbiology cultures and sensitivities |
| prescriptions | 18,087 | Medication prescriptions |
| pharmacy | 15,306 | Pharmacy dispensing records |
| emar | 35,835 | Electronic medication administration records |
| emar_detail | 72,018 | Detailed eMAR entries |
| poe | 45,154 | Provider order entries |
| poe_detail | 3,795 | Provider order details |
| omr | 2,964 | Online medical record (outpatient observations) |
| provider | 40,508 | Provider identifiers |
| d_labitems | 1,622 | Lab item dictionary |
| d_icd_diagnoses | 109,775 | ICD diagnosis code dictionary |
| d_icd_procedures | 85,257 | ICD procedure code dictionary |
| d_hcpcs | 89,200 | HCPCS code dictionary |

#### ICU tables (icu/)

| Table | Rows | Description |
|-------|------|-------------|
| icustays | 140 | ICU stay records |
| chartevents | 668,862 | Bedside charting (vitals, assessments) |
| datetimeevents | 15,280 | ICU datetime entries |
| inputevents | 20,404 | IV fluids and medications |
| outputevents | 9,362 | Fluid output measurements |
| procedureevents | 1,468 | ICU procedures |
| ingredientevents | 25,728 | IV infusion ingredients |
| d_items | 4,014 | ICU item dictionary |
| caregiver | 15,468 | Caregiver identifiers |

#### Notes (notes/)

| Table | Rows | Description |
|-------|------|-------------|
| noteevents | 4,580 | Synthetic clinical notes (14 types) |

**Total: 33 tables, 100 patients, 275 admissions, 140 ICU stays**

### OMOP CDM v5.3.1 Format (23 populated tables)

| Table | Rows | Description |
|-------|------|-------------|
| person | 100 | Patient demographics |
| death | 15 | Death records |
| observation_period | 100 | Time each patient is observed |
| visit_occurrence | 852 | Hospital admissions and visits |
| visit_detail | 14,479 | Unit-level visit details |
| condition_occurrence | 16,441 | Diagnoses (mapped to SNOMED) |
| procedure_occurrence | 18,447 | Procedures (mapped to SNOMED) |
| drug_exposure | 18,229 | Medications (mapped to RxNorm) |
| device_exposure | 3,855 | Medical device usage |
| measurement | 338,550 | Labs and vitals (mapped to LOINC) |
| observation | 31,390 | Clinical observations |
| note | 4,580 | Synthetic clinical notes |
| specimen | 150 | Biological specimens |
| condition_era | 3,771 | Grouped condition periods |
| drug_era | 7,931 | Grouped drug exposure periods |
| dose_era | 117 | Grouped dose periods |
| fact_relationship | 1,752 | Relationships between clinical facts |
| location | 1 | Geographic location |
| care_site | 31 | Hospital/clinic sites |
| cdm_source | 1 | Dataset metadata |
| 2b_concept | 3,904 | OMOP concept vocabulary |
| 2b_concept_relationship | 7,716 | Concept-to-concept mappings |
| 2b_vocabulary | 27 | Vocabulary definitions |

Empty tables (headers only): attribute_definition, cohort, cohort_attribute, cohort_definition, cost, metadata, note_nlp, payer_plan_period, provider

These tables are part of the standard OMOP CDM table structure and are included with headers only to maintain schema completeness, even though the MIMIC data does not populate them.

</details>

---

## Design Decisions

### OMOP Procedure Codes Are Not ICD

The OHDSI MIMIC-to-OMOP ETL does not carry ICD-9 procedure codes into `procedure_occurrence.procedure_source_value`. Despite the original MIMIC-IV Demo containing 401 ICD-9 procedure rows in `procedures_icd`, the OHDSI ETL maps procedures to standard vocabularies:

| Code Type | Count | Source | Example |
|---|---|---|---|
| CPT4/HCPCS (billing codes for procedures and services) | 35 | Procedure codes | `19301` |
| ICU (Intensive Care Unit) item IDs | 18,107 | d_items | `221214` |
| ICD-10-PCS | 305 | Procedure codes | `009600Z` |

TinyEHR does not modify any `source_value` fields in the OMOP format.

### OMOP condition_source_value Contains Mixed Data Types

The `condition_occurrence.condition_source_value` column in the OHDSI reference contains a mix of data types, not just ICD codes:

| Type | Count | Example |
|---|---|---|
| ICD-9/10 codes | 4,488 | `78791  ` |
| ICU item IDs | 174 | `225085` |
| ICU rhythm charting | 11,779 | `SR (Sinus Rhythm)` |

The rhythm entries come from ICU bedside charting (chartevents). The OHDSI ETL maps these into `condition_occurrence` as observed clinical states. TinyEHR does not modify these values.

### Medical Codes Stored as Strings

Medical codes (ICD, NDC, CPT, LOINC) are stored as strings in Parquet (a columnar file format optimized for data analysis) to preserve leading zeros and formatting. For example, `009.0` stored as float becomes `9.0`, and NDC `00002751001` stored as integer loses the leading zeros.

| Code Type | Data Type | Why |
|---|---|---|
| ICD codes | String | Letters, decimals, leading zeros (`009.0`) |
| NDC drug codes (National Drug Code, 11-digit medication identifier) | String | Leading zeros are significant (`00002751001`) |
| CPT/HCPCS | String | May contain letters |
| LOINC codes | String | Alphanumeric format |
| OMOP `source_value` | String | OMOP CDM specification (VARCHAR) |

Numeric identifiers (`concept_id`, `person_id`, `subject_id`) remain as integers.


### Nullable Int64 for OMOP IDs

OMOP uses 64-bit hashed IDs (e.g., `person_id = 3589912774911670296`). Some ID columns allow nulls. When pandas reads a CSV column with integers + nulls, it defaults to `float64`, which only has 53 bits of precision, silently corrupting large IDs. TinyEHR's Parquet export uses pandas `Int64` (nullable integer) to preserve full 64-bit precision.

> [!WARNING]
> **For CSV users:** Load with `dtype=str` or specify `dtype={'column_name': 'Int64'}` for ID columns. The Parquet format does not have this issue.

### OMOP Visit Count (852 vs 275)

> [!NOTE]
> OMOP `visit_occurrence` has 852 rows compared to MIMIC's 275 admissions. The OHDSI conversion process creates additional outpatient visits from lab and specimen events that have no hospital admission ID. This is standard OHDSI behavior, not a data mismatch.

---

## Glossary

| Term | Definition |
|------|-----------|
| **MIMIC-IV** | Medical Information Mart for Intensive Care: a publicly available critical care database from MIT |
| **OMOP CDM** | Observational Medical Outcomes Partnership Common Data Model: a standardized schema for health data |
| **OHDSI** | Observational Health Data Sciences and Informatics: an open-science community that maintains the OMOP CDM |
| **ICD** | International Classification of Diseases: the standard coding system for diagnoses and procedures |
| **ICD-9-CM / ICD-10-CM** | Older (v9) and current (v10) US clinical modifications of ICD for diagnoses |
| **ICD-10-PCS** | ICD-10 Procedure Coding System: used for inpatient hospital procedures |
| **NDC** | National Drug Code: an 11-digit identifier for medications |
| **CPT / HCPCS** | Current Procedural Terminology / Healthcare Common Procedure Coding System: billing codes for procedures |
| **LOINC** | Logical Observation Identifiers Names and Codes: standard codes for lab tests and clinical observations |
| **SNOMED** | Systematized Nomenclature of Medicine: a comprehensive clinical terminology used as the standard vocabulary in OMOP |
| **Athena** | OHDSI's vocabulary browser and download service ([athena.ohdsi.org](https://athena.ohdsi.org/)) |
| **PhysioNet** | An open-access repository for health-related data ([physionet.org](https://physionet.org/)) |
| **Parquet** | A columnar file format optimized for data analysis, used for distributing TinyEHR on HuggingFace |
| **ETL** | Extract, Transform, Load: the process of moving and reshaping data between systems |
| **concept ID** | A standardized numeric identifier in OMOP that maps local codes to universal meanings |
| **source_value** | An OMOP field that stores the original code exactly as it appeared in the source system |
| **CMS** | Centers for Medicare & Medicaid Services: the US federal agency that administers Medicare and Medicaid |
| **CDC** | Centers for Disease Control and Prevention |
| **ICU** | Intensive Care Unit |

---

## References

### Source Data

1. Johnson, A., Bulgarelli, L., Pollard, T., Horng, S., Celi, L. A., & Mark, R. (2023). MIMIC-IV, a freely accessible electronic health record dataset. *Scientific Data*, 10(1), 1. https://doi.org/10.1038/s41597-022-01899-x

2. Johnson, A., Bulgarelli, L., Pollard, T., Horng, S., Celi, L. A., & Mark, R. (2023). MIMIC-IV Clinical Database Demo (version 2.2). *PhysioNet*. https://doi.org/10.13026/dp1f-ex47

3. Kallfelz, M., Tsvetkova, A., Pollard, T., Kwong, M., Lipori, G., Huser, V., Osborn, J., Hao, S., & Williams, A. (2021). MIMIC-IV Demo Data in the OMOP Common Data Model (version 0.9). *PhysioNet*. https://doi.org/10.13026/p1f5-7x35

### Standards and Conventions

4. OHDSI. Observational Medical Outcomes Partnership Common Data Model. https://ohdsi.github.io/CommonDataModel/

5. OHDSI. *The Book of OHDSI*. https://ohdsi.github.io/TheBookOfOhdsi/

6. OHDSI. MIMIC-IV to OMOP CDM ETL. https://github.com/OHDSI/MIMIC

### Coding Guidelines

7. Centers for Disease Control and Prevention. *ICD-9-CM Official Guidelines for Coding and Reporting* (2011). https://www.cdc.gov/nchs/data/icd/icd9cm_guidelines_2011.pdf

8. Centers for Medicare & Medicaid Services. *ICD-10-CM Official Guidelines for Coding and Reporting, FY 2025*. https://www.cms.gov/files/document/fy-2025-icd-10-cm-coding-guidelines.pdf

9. Research Data Assistance Center (ResDAC). *International Classification of Disease (ICD) Codes in Medicare Files*. https://resdac.org/articles/international-classification-disease-icd-codes-medicare-files
