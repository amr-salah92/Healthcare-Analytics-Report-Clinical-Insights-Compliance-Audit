# Healthcare Analytics Report: Clinical Insights & Compliance Audit  
**Prepared for:** MediCare Analytics Division  
**Date:** June 27, 2025  

---

## Table of Contents  
1. [Project Name](#1-project-name)  
2. [Project Background](#2-project-background)  
3. [Project Goals](#3-project-goals)  
4. [Data Collection and Sources](#4-data-collection-and-sources)  
5. [Formal Data Governance](#5-formal-data-governance)  
6. [Regulatory Reporting](#6-regulatory-reporting)  
7. [Methodology](#7-methodology)  
8. [Data Structure & Initial Checks](#8-data-structure--initial-checks)  
9. [Documenting Issues](#9-documenting-issues)  
10. [Executive Summary](#10-executive-summary)  
11. [Insights Deep Dive](#11-insights-deep-dive)  
12. [Recommendations](#12-recommendations)  
13. [Future Work](#13-future-work)  
14. [Technical Details](#14-technical-details)  
15. [Assumptions and Caveats](#15-assumptions-and-caveats)  

---

## 1. Project Name  
**MediCare Analytics: Clinical Prevalence, Operational Efficiency & GDPR/HIPAA Compliance Audit**  

---

## 2. Project Background  
MediCare operates 22 hospitals and 150 clinics across the U.S., serving 1.2M+ patients annually since 2005. The organization uses an HL7/FHIR-compliant EHR system storing 50K+ patient records. Key metrics include:  
- **Patient Volume:** 50,000 de-identified records (2020–2025)  
- **Clinical Activity:** 500K diagnoses, 300K procedures, 500K observations  
- **Compliance Requirements:** 10K monthly audit log checks under GDPR/HIPAA  

---

## 3. Project Goals  
1. **Identify Top Clinical Patterns:** Determine prevalent ICD10/CPT codes to optimize resource allocation  
2. **Demographic Disparity Analysis:** Quantify diagnosis/procedure rates by age, gender, and ZIP3  
3. **Data Governance Compliance:** Audit GDPR/HIPAA adherence in audit logs and consent records  
4. **Operational Efficiency:** Evaluate readmission rates linked to procedures/diagnoses  

---

## 4. Data Collection and Sources  
| **Source**               | **Description**                                  | **Volume**    |  
|--------------------------|-------------------------------------------------|---------------|  
| EHR System (FHIR API)     | Patient encounters, observations, diagnoses     | 1.2M records  |  
| CMS Terminology Service  | ICD-10/CPT code lookups                         | 17 codes      |  
| Consent Management DB     | Patient consent records                         | 35K records   |  
| Audit Trail System        | GDPR/HIPAA access logs                          | 10K entries   |  

---

## 5. Formal Data Governance  
**Framework:** HL7/FHIR R4 for clinical data interoperability  
**Improvements Implemented:**  
- Mapped `INVALID_ACTION` to `UNKNOWN` in audit logs  
- Standardized ICD10/CPT descriptions using CMS benchmarks

## 6. Regulatory Reporting  
**GDPR/HIPAA Alignment:**  
- **Pseudonymization:** `Patient_Hash_ID` (SHA-256) replaces PHI  
- **Consent Tracking:** 35K records mapped to `Patient_Surrogate_ID`  
- **Audit Log Remediation:**  
  - 4,044 entries with invalid `Purpose_Code` recategorized as `UNKNOWN`  
  - 1,974 `INVALID_ACTION` entries quarantined  

---

## 7. Methodology  
**Statistical Analysis:**  
- Prevalence rates for ICD10/CPT codes (frequency distribution)  
- Readmission rate = Patients with encounters ≤30 days post-discharge ÷ Total patients  
**Tools:**  
- Python (Pandas, Seaborn) for cleaning/trend analysis  
- SQL for cohort identification (e.g., demographic segments)  

---

## 8. Data Structure & Initial Checks  
### Tables Overview  

### **1. Audit Logs (`data_audit`)**  
- **Rows:** 10,000  
- **Purpose:** GDPR/HIPAA compliance tracking  
- **Key Columns:**  
  | Column             | Description                                       | Initial Checks                         |  
  |--------------------|---------------------------------------------------|----------------------------------------|  
  | `Audit_ID`         | Unique auto-incremented audit identifier          | No duplicates                          |  
  | `User_ID`          | User performing action (e.g., `jeremyjones`)      | All populated                          |  
  | `Table_Name`       | Target table (`Procedure`, `Encounter`, etc.)     | 0 missing values                       |  
  | `Action_Type`      | Operation type (`SELECT`/`INSERT`/etc.)           | **1,974 invalid entries** (`INVALID_ACTION`) |  
  | `Purpose_Code`     | Legal basis (`TREATMENT`/`RESEARCH`/etc.)         | **2,016 nulls → filled with `UNKNOWN`** |  
  | `Action_Timestamp` | Action date/time                                  | Future dates removed                   |  

---

### **2. Patient Consents (`data_consents`)**  
- **Rows:** 35,000  
- **Purpose:** GDPR lawful basis documentation  
- **Key Columns:**  
  | Column                 | Description                                      | Initial Checks                         |  
  |------------------------|--------------------------------------------------|----------------------------------------|  
  | `Patient_Surrogate_ID` | Pseudonymized patient ID                         | All valid references                   |  
  | `Consent_Type_Code`    | Consent type (`RESEARCH`, `TREATMENT`, etc.)     | 0 missing values                       |  
  | `Consent_Given_Date`   | Date consent was obtained                        | **Coerced to datetime**                |  
  | `Consent_Expires_Date` | Consent expiration date                          | **13,985 nulls → filled with (Given_Date + 3 years)** |  

---

### **3. CPT Codes (`data_cpt`)**  
- **Rows:** 7 (6 valid + 1 null)  
- **Purpose:** Procedure code standardization  
- **Key Columns:**  
  | Column        | Description                                      | Initial Checks              |  
  |---------------|--------------------------------------------------|-----------------------------|  
  | `CPT_Code`    | Medical procedure code (e.g., `99213`)           | **1 null → corrected**      |  
  | `Description` | Human-readable procedure name                    | Standardized to CMS definitions |  

---

### **4. Diagnoses (`data_diagnoses`)**  
- **Rows:** 500,000  
- **Purpose:** Clinical condition documentation  
- **Key Columns:**  
  | Column               | Description                                  | Initial Checks       |  
  |----------------------|----------------------------------------------|----------------------|  
  | `ICD10_Code`         | Diagnosis code (e.g., `K219` for GERD)       | 0 nulls              |  
  | `Diagnosis_Rank`     | Primary (`1`) vs. secondary (`>1`)           | All populated        |  
  | `Encounter_ID`       | Links to encounter record                    | Valid references     |  

---

### **5. Encounters (`data_encounters`)**  
- **Rows:** ~250,000  
- **Purpose:** Patient visit tracking  
- **Key Columns:**  
  | Column                   | Description                              | Initial Checks               |  
  |--------------------------|------------------------------------------|------------------------------|  
  | `Patient_Surrogate_ID`   | Pseudonymized patient ID                 | **Converted to `int64`**     |  
  | `Encounter_Type_Code`    | Visit type (`inpatient`, `outpatient`)   | **41,588 nulls** (16.6% missing) |  
  | `Location_Code`          | Facility ID                              | **41,692 nulls** (16.7% missing) |  
  | `Attending_Prov_ID`      | Practitioner reference                    | **512 nulls** (0.2% missing) |  

---

### **6. ICD10 Codes (`data_icd10`)**  
- **Rows:** 8  
- **Purpose:** Diagnosis code definitions  
- **Key Columns:**  
  | Column        | Description                              | Initial Checks           |  
  |---------------|------------------------------------------|--------------------------|  
  | `ICD10_Code`  | Standard diagnosis code (e.g., `E11.9`)  | 0 nulls                  |  
  | `Description` | Clinical meaning of code                 | Corrected inaccurate entries |  

---

### **7. Observations (`data_observations`)**  
- **Rows:** 500,000  
- **Purpose:** Lab/test result storage  
- **Key Columns:**  
  | Column               | Description                      | Initial Checks              |  
  |----------------------|----------------------------------|-----------------------------|  
  | `Value_Num`          | Numeric result                   | **2,389 nulls** (0.5% missing) |  
  | `Value_Unit`         | Measurement unit                 | **7,631 nulls** (1.5% missing) |  

---

### **8. Patients (`data_patients`)**  
- **Rows:** 50,000  
- **Purpose:** De-identified patient registry  
- **Key Columns:**  
  | Column                 | Description          | Initial Checks                     |  
  |------------------------|----------------------|------------------------------------|  
  | `Gender_Code`          | `M`/`F`/`O`/`U`      | **7,091 nulls → `Missing`**        |  
  | `Birth_Year`           | Year of birth        | **1,986 nulls** (4% missing)       |  
  | `Race_Ethnicity_Code`  | Race/ethnicity       | **7,192 nulls → `Missing`**        |  

---

### **9. Practitioners (`data_practitioners`)**  
- **Rows:** 500  
- **Purpose:** Provider directory  
- **Key Columns:**  
  | Column             | Description        | Initial Checks                   |  
  |--------------------|--------------------|----------------------------------|  
  | `Specialty_Code`   | Clinical specialty | **66 nulls → `Missing`**         |  

---

### **10. Procedures (`data_procedures`)**  
- **Rows:** 300,000  
- **Purpose:** Medical intervention logging  
- **Key Columns:**  
  | Column               | Description            | Initial Checks         |  
  |----------------------|------------------------|------------------------|  
  | `Procedure_DateTime` | Timestamp of procedure | **Converted to datetime** |

## Critical Data Quality Observations

### High Null Rates in Encounters
- **16.7%** missing `Location_Code`  
- **16.6%** missing `Encounter_Type_Code`  

### Compliance Gaps in Audit Logs
- **20.2%** invalid `Purpose_Code`  
- **19.7%** invalid `Action_Type`  

### Clinical Data Incompleteness
- **1.5%** lab results missing units (`Value_Unit`)  
- **4%** patients missing `Birth_Year`  

## 9. Documenting Issues

| Table        | Column                | Issue                                      | Magnitude | Solvable | Resolution                           |
|--------------|------------------------|--------------------------------------------|-----------|----------|--------------------------------------|
| Audit_Log    | Purpose_Code           | 2,016 nulls; invalid values                | High      | Yes      | Mapped to UNKNOWN                    |
| Patient      | Gender_Code            | 7,091 nulls                                | Medium    | Yes      | Filled with Missing                  |
| Observations | Value_Unit             | 7,631 nulls                                | Medium    | Partial  | Use LOINC standard units             |
| Encounters   | Encounter_Type_Code    | 41,588 nulls                               | High      | Partial  | Infer from procedure type            |

---

## 10. Executive Summary

**For:** Chief Medical Officer & Compliance Director

### Top 3 Insights:

- **Clinical:** 15K+ patients have GERD (*ICD10: K219*) paired with metabolic panels (*CPT: 80053*).
- **Compliance:** 52% of data accesses (5,206 audit logs) lack valid purpose codes (*GDPR risk*).
- **Operational:** Readmission rates spike to 20% for invalid ICD10 codes (e.g., *e999*).

📈 ![2726c08b-1909-42bb-b344-78566ed95e75](https://github.com/user-attachments/assets/25545bac-81cc-4e76-a06f-54a816319964)

## 11. Insights Deep Dive

### Category 1: Clinical Prevalence & Patterns
- **Insight 1**: Top diagnosis-procedure pair: `K219 (GERD)` + `80053 (metabolic panel)` in **15,226 encounters**
- **Insight 2**: Readmission rates for invalid ICD10 codes (`e999`, `X9d9`) are **2–3× higher (20%)** vs. valid codes (**5–7%**)
- **Insight 3**: **Outpatient encounters (75%)** drive **68%** of observation tests
- **Insight 4**: **40%** of glucose tests (`LOINC: 2339-0`) exceed **140 mg/dL** for diabetic patients (`ICD10: E1165`)

### Category 2: Demographic Disparities
- **Insight 1**: Patients born **1980–1999** have **22.1 diagnoses/patient** vs. **21.9** for those born **before 1940**
- **Insight 2**: `ZIP3 fid` has **79 diagnoses/patient** (**3.5× average**) – likely data entry error
- **Insight 3**: Gender `U (unknown)` shows **highest procedure rates (22.2/patient)**
- **Insight 4**: **Asian patients (A)** have **12% lower procedure volumes** than **White patients**

### Category 3: Data Governance & Quality
- **Insight 1**: **41,588 encounters (16.6%)** lack `Encounter_Type_Code`, hindering site comparisons
- **Insight 2**: **7,631 lab results** missing `Value_Unit` risk misinterpretation (e.g., glucose in `mmol/L` vs. `mg/dL`)
- **Insight 3**: **66 practitioners (13.2%)** have **undefined specialties** – impacts referral analysis
- **Insight 4**: **1,974 audit entries** use `INVALID_ACTION` – system logging error

### Category 4: Compliance & Security
- **Insight 1**: **52% of data accesses (5,206 logs)** lack valid `Purpose_Code` (e.g., `jeremyjones` deleting procedures)
- **Insight 2**: **35% of DELETE actions** occur **overnight (12 AM–5 AM)** – security red flag
- **Insight 3**: **13,985 consent records** had **null expiration dates**; defaulted to **3-year terms**
- **Insight 4**: Users like `qallen` combine `INVALID_ACTION` with **invalid purpose codes**


## 12. Recommendations

### Clinical
- Create care bundles for **GERD (K219)** and **metabolic panels (80053)** to standardize treatment.
- Investigate readmissions tied to invalid ICD-10 codes (e.g., `E999`).

### Demographic
- Audit outlier `ZIP3` fields for data entry errors.
- Target outreach for **Asian patients** showing **12% lower procedure uptake**.

### Data Quality
- Enforce mandatory `Encounter_Type_Code` in EHR workflows.
- Add unit validation rules for `LOINC` codes.

### Compliance
- Block `DELETE` actions without valid `Purpose_Code`.
- Investigate overnight access by users such as `mckinneyandrea`.

---

## 13. Future Work

- **Predictive Modeling**: Forecast readmissions using diagnosis-procedure pairs.
- **Consent Expiry Alerts**: Real-time dashboards for consent lifecycle management.
- **Anomaly Detection**: ML model to flag suspicious audit log patterns.

## 14. Technical Details

**Tools:**

- **Python (Pandas/Seaborn):** Data cleaning, trend analysis

---

## 15. Assumptions and Caveats

- **Birth Year Nulls:** Imputed using median age (56) from non-null records  
- **2025 Data Drop:** Post-May 2025 records excluded due to ETL incomplete sync  
- **Invalid Codes:** `INVALID_ACTION` in audit logs treated as non-compliant  
- **ZIP3 Outliers:** `fid` retained but flagged for investigation  
- **Readmissions:** Defined as encounters ≤30 days post-discharge; transfers excluded  

