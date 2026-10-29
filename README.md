# Hospital Readmission Risk: Diabetic Patients

**A healthcare data analytics case study (SQL Server)**

*By a registered nurse building a career in healthcare data analytics.*

---

## In one minute

About 9 in every 100 diabetic patients in this dataset came back to hospital within 30 days of leaving. I used SQL to find out **who comes back most often**, so that follow-up care could be focused where it matters most.

**The three main findings:**

1. **Earlier hospital stays are the strongest warning sign.** Patients with 3 or more earlier stays came back 26.5% of the time, compared with 8.1% for patients with none.
2. **Where a patient goes after discharge matters.** Patients transferred to a rehab facility came back 26.4% of the time. Patients who went home came back 6.9% of the time.
3. **Sicker, more complex patients return more often.** Longer stays, more medications and more diagnoses all go with higher readmission rates.

> These are **links, not causes**. See [Cautions and limits](#cautions-and-limits) and [Data privacy and responsible use](#data-privacy-and-responsible-use).

---

## Contents

1. [The question](#the-question)
2. [The data](#the-data)
3. [Tools and skills](#tools-and-skills)
4. [Data cleaning](#data-cleaning)
5. [Validation](#validation)
6. [What I found](#what-i-found)
7. [Illustrative risk score](#illustrative-risk-score)
8. [What I recommend](#what-i-recommend)
9. [Cautions and limits](#cautions-and-limits)
10. [Data privacy and responsible use](#data-privacy-and-responsible-use)
11. [Power BI dashboard (planned)](#power-bi-dashboard-planned)
12. [How to run this project](#how-to-run-this-project)
13. [Files](#files)
14. [Data source and attribution](#data-source-and-attribution)

---

## The question

> Which diabetic patients are most likely to return to hospital within 30 days, and what could a hospital do about it?

Readmissions within 30 days are costly for hospitals and hard on patients. If a hospital can spot the patients most likely to return, it can plan extra support such as a follow-up call or a medication check before they leave.

### Problem statement

While the initial request focused on diabetic readmission rates, a comprehensive review from a health professional's point of view needs to look beyond the primary diagnosis. Readmission is multifactorial. I therefore ran an exploratory analysis of demographic, clinical utilization and comorbidity factors that are known clinically to influence outcomes for patients with diabetes, to find which factors are most strongly **associated** with readmission within 30 days.

## The data

- **Source:** the public "Diabetes 130-US Hospitals for Years 1999-2008" dataset from the UCI Machine Learning Repository. I did not create this dataset.
- **Coverage:** ten years of hospital records from 130 US hospitals and care networks.
- **Who is in it:** inpatient stays of 1 to 14 days for patients with diabetes, where laboratory tests were done. So it does **not** cover every diabetic patient.
- **Size:** 101,766 hospital visits, reduced to **69,990 patients** after cleaning.
- **What it holds:** age band, race, gender, length of stay, number of medications and lab tests, earlier hospital visits, main diagnosis, how the patient was admitted and discharged, test results, and whether they came back within 30 days.

## Tools and skills

| Tool | How I used it |
|---|---|
| **SQL Server (T-SQL)** | All cleaning, grouping, validation and analysis |
| **SQL Server Management Studio** | Writing and running the queries |
| **Power BI** | Dashboard (planned) |
| **GitHub** | Sharing the script and this case study |

**SQL skills shown in the script:** backups and working copies, `NULLIF` for missing values, `CASE` grouping, joins to lookup tables, common table expressions (CTEs), window functions (`RANK`, `SUM() OVER`), `TRY_CAST`, reusable views, `HAVING` filters for small groups, and row-count validation. Every query has a plain-English comment, and the main results are written beside it.

---

## Data cleaning

The rule: **the original data is never changed. All work happens on copies.**

| Step | What I did | Result |
|---|---|---|
| 1. Backup | Made an untouched copy of the raw table, then worked on a separate clean copy | 101,766 rows in both |
| 2. Missing values | The dataset uses `?` for "unknown". I turned `?` into proper empty values (NULL) in race, payer code, medical specialty and the three diagnosis columns | SQL no longer counts `?` as real data |
| 3. Drop weight | Weight was empty for 96.9% of records (98,569 of 101,766), so it has no analysis value | Column removed from the clean copy |
| 4. Keep, but flag | Payer code (39.7% missing) and medical specialty (48.9% missing) were kept but not relied on. Missing specialty is grouped as "Unknown" | Limits noted below |
| 5. Lookup tables | Built three clean lookup tables to translate ID numbers into words: admission type (8 rows), discharge disposition (30 rows) and admission source (25 rows). The text `'NULL'` in the mapping file became `'Unknown'` | Counts checked |
| 6. Remove patients who cannot return | Removed 2,423 visits that ended in death or hospice care (discharge codes 11, 13, 14, 19, 20, 21), before any other step | 99,343 rows left |
| 7. One row per patient | Many patients appear more than once (one has 40 visits). I kept each patient's **first** recorded visit | **69,990 patients** |
| 8. Age as a number | Turned age bands like `[70-80)` into their middle value (75) so age can be sorted and compared | New column added |
| 9. Group the data | Grouped main diagnosis into body systems using ICD-9 code ranges, and grouped earlier visits, medications and diagnoses into tiers | One reusable view holds all groupings |

## Validation

Every stage was checked, so nothing is lost or counted twice by accident.

| Check | Expected | Result |
|---|---|---|
| Raw table rows | 101,766 | 101,766 |
| Backup rows | 101,766 | 101,766 |
| Rows after removing death and hospice | 99,343 | 99,343 |
| Patients after one-row-per-patient | 69,990 rows = 69,990 distinct patients | Matched |
| Lookup table rows | 8 / 30 / 25 | 8 / 30 / 25 |
| Readmission groups add up | 41,479 + 22,226 + 6,285 = 69,990 | Matched |
| Other group totals (race, admission type, A1C, risk groups) | Each adds up to 69,990 | Matched |
| Overall rate by hand | 6,285 / 69,990 = 8.98% | Matched |
| Patients with a missing readmission value | 0 | 0 |
| Re-run safety | The clean tables rebuild from the backup every time | Yes |

The risk score was also sense-checked: the readmission rate rises at every step of the score (see below).

---

## What I found

Based on clinical best practice for readmission reduction, the following factors were investigated proactively, in addition to the primary diagnosis. They are the same kinds of factors that established readmission tools use, such as length of stay, how urgent the admission was, comorbidity and emergency visits (for example the LACE index).

**Overall:** 6,285 of 69,990 patients (**8.98%**) were readmitted within 30 days.

### 1. Prior Healthcare Utilization (the strongest signal)

*Clinical view:* patients who needed hospital care several times in the year before are more likely to have unstable or poorly controlled illness. I counted inpatient stays and emergency visits in the year before the admission.

| Inpatient stays in the year before | Patients | Came back within 30 days |
|---|---|---|
| None | 61,795 | 8.1% |
| 1 | 5,798 | 12.9% |
| 2 | 1,501 | 18.5% |
| 3 or more | 896 | **26.5%** |

| Emergency visits in the year before | Patients | Came back within 30 days |
|---|---|---|
| None | 64,889 | 8.7% |
| 1 | 3,883 | 11.3% |
| 2 | 789 | 13.4% |

Groups with more visits than this are too small to read (fewer than 250 patients each), so I did not rely on them.

> **Key insight: prior hospital use is the strongest warning sign.** The 30-day return rate rises with every earlier inpatient stay, from 8.1% with none to 26.5% with three or more, which is more than three times higher. Emergency visits point the same way (8.7% with none, 13.4% with two). This is an association, not proof that earlier stays cause readmission.

### 2. Demographic Risk Stratification

*Clinical view:* patient background can influence how well people follow their care plan and how they do afterwards. I checked whether any group is clearly more at risk.

| Group | Patients | Came back within 30 days |
|---|---|---|
| Caucasian | 52,305 | 9.2% |
| African American | 12,627 | 8.7% |
| Asian | 488 | 8.4% |
| Hispanic | 1,501 | 8.1% |
| Other | 1,150 | 6.9% |
| Race not recorded | 1,919 | 7.3% |
| Men | 32,748 | 8.9% |
| Women | 37,239 | 9.0% |

- **Race and gender:** the differences are small and were not adjusted for anything else, so I draw no conclusion from them. Small groups (Asian, Other) are less reliable, and the source does not document how race was recorded.
- **Age** shows the clearest pattern: 2.0% under 10 (153 patients, too few to trust), 7.4% at 40-50, 9.0% at 60-70, 10.2% at 70-80 and 10.8% at 80-90.

### 3. Clinical Utilization and Care Intensity Review

*Clinical view:* length of stay, lab procedures and medications were analyzed as indicators of illness severity. The hypothesis was that too little care (a short stay, few labs) might contribute to early readmission.

| Factor | Lower end | Higher end |
|---|---|---|
| Length of stay | 1 day: 6.5% | 8 days: 12.3% |
| Lab procedures | 0-20: 7.3% | 61 or more: 10.2% |
| Medications | 0-5: 5.9% | 16 or more: 10.1% |

**The hypothesis was not supported.** The shortest stays and the fewest labs had the *lowest* return rates, not the highest. Return rates rise with stay length, labs and medications, which most likely reflects sicker patients rather than the quality of care. This data cannot separate the two.

### 4. Comorbidity Burden Assessment

*Clinical view:* patients with diabetes rarely have diabetes alone. To assess multimorbidity, I looked at the number of diagnoses and at the main diagnosis group.

- **Number of diagnoses:** 3.1% with 1 diagnosis, rising to 10.0% with 9 (30,719 patients, 44% of everyone). Groups with 10 or more diagnoses have 30 patients or fewer, so I did not rely on them.
- **Main diagnosis:** patients admitted for an injury (10.8%), circulatory disease (9.7%) or diabetes (9.1%) were above the overall average.

### 5. Transitions of Care

*Clinical view:* the point of discharge is a known risk moment, especially when a patient moves to another facility.

| Discharged to | Patients | Came back within 30 days |
|---|---|---|
| Another rehab facility | 1,411 | **26.4%** |
| Another type of inpatient institution | 913 | 20.6% |
| Another short-term hospital | 1,541 | 13.8% |
| Skilled nursing facility | 8,790 | 13.4% |
| Home | 44,322 | 6.9% |

| Admission type | Patients | Came back within 30 days |
|---|---|---|
| Emergency | 35,480 | 9.2% |
| Urgent | 12,803 | 9.0% |
| Elective (planned) | 13,787 | 8.3% |

Patients moved to another facility came back far more often than patients who went home. Emergency admissions are the largest group and slightly above planned ones. "Unknown", "Not available" and "Not mapped" are data-quality gaps, and Newborn (9) and Trauma Center (18) are too small to read.

### 6. Medications and Glucose Monitoring

- **Medication changed during the stay:** 9.4%, against 8.6% when no medication was changed. This is a link only. Sicker patients are more likely to have medicines changed.
- **Glucose and A1C testing:** 8.7% for patients with at least one test, 9.1% for patients with none. Tested patients may differ in other ways, so this does not show that testing changes the outcome.

---

## Illustrative risk score

To show how this could work in practice, I built a simple points rule:

| Factor | Points |
|---|---|
| Earlier stays: 0 / 1 / 2 / 3 or more | 0 / 2 / 3 / 4 |
| Age 70 or over | +1 |
| Injury or circulatory main diagnosis | +1 |
| Medication changed during the stay | +1 |
| Discharged to rehab, another inpatient institution, a short-term hospital or a nursing facility | +2 |

Race and gender are **not** used in the score.

| Risk group | Patients | Share of patients | Came back within 30 days | Average stay | Average earlier stays |
|---|---|---|---|---|---|
| Low (score 0-1) | 33,694 | 48.1% | 6.2% | 3.8 days | 0.00 |
| Medium (score 2-3) | 25,150 | 35.9% | 9.6% | 4.5 days | 0.13 |
| High (score 4 or more) | 11,146 | 15.9% | **16.0%** | 5.4 days | 0.80 |

The High group is about 16% of patients but accounts for roughly 28% of all 30-day readmissions. Its return rate is about 2.6 times the Low group's. The rate also rises at every step of the score, from 5.5% at score 0 to 22.7% at score 7. (Scores 8 and 9 have fewer than 120 patients, so I did not rely on them.)

> This score is a **teaching example only**. I chose the points myself, and I checked it on the same data that suggested the rules, so this shows the rule is consistent, **not** that it would predict new patients. It is **not a clinical tool** and must not be used to make decisions about real patients.

## What I recommend

1. **Focus follow-up on the highest-risk groups.** Start with patients who have earlier hospital stays and patients transferred to another facility.
2. **Try simple actions before discharge.** For example, a follow-up call within 2 days and a medication review.
3. **Test it as a small pilot.** Measure whether the 30-day return rate falls before rolling it out.

---

## Cautions and limits

- **Links, not causes.** This is an exploratory risk-factor analysis, not a root-cause analysis. It shows which factors are associated with early readmission, not why they happen.
- **Old data.** It covers 130 US hospitals between 1999 and 2008. Today's practice, coding (ICD-9 was later replaced by ICD-10) and patients may differ.
- **Not every diabetic patient.** Only stays of 1 to 14 days with laboratory tests are included, so the results do not describe all diabetic care.
- **First visit only.** I kept each patient's first recorded visit and assumed a lower encounter ID means an earlier visit.
- **Patients removed on purpose.** I removed visits that ended in death or hospice, because those patients cannot be readmitted. The results apply to patients who left hospital alive and not into hospice.
- **Missing data.** Weight (96.9% missing) was left out. Payer code (39.7% missing) and specialty (48.9% missing) were kept but not relied on.
- **Small groups.** Rates from very small groups (for example 6 or more earlier stays, or 10 or more diagnoses) are not reliable, so I did not draw conclusions from them.
- **Data-quality groups.** "Unknown", "Not Available" and "Not Mapped" are data gaps, not findings.
- **The risk score is not validated.** It was built and checked on the same data. It would need testing on separate patients before anyone could trust it.
- **Only a 30-day yes/no flag.** I did not look at readmissions after 30 days, cost, or the reason for the return.

## Data privacy and responsible use

- **The data is public and de-identified.** It has no names, addresses or exact dates. Age is given only in 10-year bands, and patient and visit numbers are anonymous IDs.
- **No attempt to identify anyone.** I did not try to identify any patient, and nobody should combine this data with other sources to do so.
- **No patient-level data in this repository.** This repository holds the code and the written results only. To reproduce the work, download the data from the source link below.
- **Only group results are shown.** The tables show counts and rates for groups, not individual patients.
- **Race and gender.** They are shown only to describe the dataset. They are not used in the risk score, and these results must not be used to treat or judge any individual patient or group differently.
- **Using real hospital data.** If I use data like this at work, I will follow the organization's data-protection rules and applicable law (for example HIPAA in the US), and share only what I am permitted to share.
- **Not medical advice.** This project is for learning and portfolio purposes only.

---

## Power BI dashboard (planned)

A dashboard is planned and **not yet built**. It will be built on the SQL views in this project. Planned content:

- **Overview:** overall rate (8.98%), patients, readmissions
- **Who returns:** earlier stays, age, diagnosis group
- **After discharge:** discharge destination and admission source
- **Risk groups:** Low, Medium and High, with the score caveat shown on the page

## How to run this project

1. Download the data (`diabetic_data.csv` and `IDS_mapping.csv`) from the source link below and import them into SQL Server as `diabetic_data_raw` and `IDS_mapping_raw` in a database called `Hospital_Readmission_Risk`.
2. Open `readmission_final.sql` and run it from the top, one section at a time. `GO` separates the sections.
3. Check the validation results against the table above.
4. Run each lookup-table block once. Running an `INSERT` twice doubles every count that joins to that table.

## Files

| File | What it is |
|---|---|
| `readmission_final.sql` | The full SQL script. Every query has a plain-English comment, and the main results are written beside it. |
| `README.md` | This case study |

## Data source and attribution

- **Dataset:** Clore, J., Cios, K., DeShazo, J., & Strack, B. (2014). *Diabetes 130-US Hospitals for Years 1999-2008* [Dataset]. UCI Machine Learning Repository. https://doi.org/10.24432/C5230J. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
- **Original study:** Strack, B., DeShazo, J. P., Gennings, C., Olmo, J. L., Ventura, S., Cios, K. J., & Clore, J. N. (2014). Impact of HbA1c measurement on hospital readmission rates: analysis of 70,000 clinical database patient records. *BioMed Research International*, 2014, Article 781670. https://doi.org/10.1155/2014/781670
- **My work:** the cleaning, analysis, findings and risk score are my own. The dataset and the original study belong to their authors. I changed the data by cleaning, filtering and grouping it, as described above.
- **Disclaimer:** this project is for learning and portfolio purposes only and is not medical advice.
