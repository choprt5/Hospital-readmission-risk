# Hospital Readmission — SQL Mini Case

## Dataset
This dataset contains hospital discharge records with demographic, clinical, utilization, and medication features. The target variable is whether a patient was readmitted within 30 days (`readmit_30`).

Tables:
- `X_discharge`: patient features at discharge
- `y_readmit30`: 30-day readmission outcome
- `cohort` (VIEW): joined feature + outcome table

## Objective
Use SQL to explore drivers of 30-day readmission risk, validate cohort integrity, and compute operationally meaningful metrics beyond basic EDA.

## Database Schema
The analysis uses a SQLite database with a derived VIEW:

- `cohort`: JOIN of `X_discharge` and `y_readmit30` on `row_id`
  - One row per hospital encounter
  - Includes all predictors and the readmission label

Using a VIEW avoids duplicating data while enabling reusable analysis queries.

## Analysis Queries

## Q1: How does readmission rate vary by age band?

**SQL:**

```sql
SELECT
  age,
  COUNT(*) AS n,
  AVG(readmit_30) AS readmit_rate
FROM cohort
GROUP BY age
ORDER BY age;
```

**Result (from notebook):**

```text
        age      n  readmit_rate
0    [0-10)    161      0.018634
1   [10-20)    691      0.057887
2   [20-30)   1657      0.142426
3   [30-40)   3775      0.112318
4   [40-50)   9685      0.106040
5   [50-60)  17256      0.096662
6   [60-70)  22483      0.111284
7   [70-80)  26068      0.117731
8   [80-90)  17197      0.120835
9  [90-100)   2793      0.110992
```

**Interpretation:** 
Readmission rate increases with age, with the lowest rates in younger bands and the highest rates in the oldest bands. Band sizes are very uneven (most rows are in older age bands), so comparisons for small-n bands should be treated as noisier.


## Q2: How common is 'Left AMA' discharge, and is the readmission rate higher for those cases?

**SQL:**

```sql
SELECT
  COUNT(*) AS n_total,
  SUM(CASE WHEN discharge_disposition_id = 7 THEN 1 ELSE 0 END) AS n_left_ama,
  AVG(CASE WHEN discharge_disposition_id = 7 THEN 1.0 ELSE 0.0 END) AS left_ama_rate,
  AVG(readmit_30) AS overall_readmit_rate,
  AVG(CASE WHEN discharge_disposition_id = 7 THEN readmit_30*1.0 ELSE NULL END) AS readmit_rate_left_ama,
  AVG(CASE WHEN discharge_disposition_id <> 7 THEN readmit_30*1.0 ELSE NULL END) AS readmit_rate_not_left_ama
FROM cohort;
```

**Result (from notebook):**

```text
n_total  n_left_ama  left_ama_rate  overall_readmit_rate  \
0   101766         623       0.006122              0.111599   

   readmit_rate_left_ama  readmit_rate_not_left_ama  
0               0.144462                   0.111397
```

**Interpretation:** 
Patients who left against medical advice (AMA) show a substantially higher readmission rate (~14.4%) compared to those discharged through standard pathways (~11.1%). Although AMA discharges represent a small fraction of total encounters (~0.6%), they are disproportionately associated with readmission risk, suggesting premature discharge or incomplete care plans.

## Q3: Does readmission rate change with `time_in_hospital`?

**SQL:**

```sql
SELECT
  time_in_hospital,
  COUNT(*) AS n,
  AVG(readmit_30) AS readmit_rate
FROM cohort
GROUP BY time_in_hospital
ORDER BY time_in_hospital;
```

**Result (from notebook):**

```text
    time_in_hospital      n  readmit_rate
0                  1  14208      0.081785
1                  2  17224      0.099396
2                  3  17756      0.106668
3                  4  13924      0.118070
4                  5   9966      0.120309
5                  6   7539      0.125879
6                  7   5859      0.128350
7                  8   4391      0.142337
8                  9   3002      0.137242
9                 10   2342      0.143467
10                11   1855      0.105121
11                12   1448      0.133287
12                13   1210      0.123140
13                14   1042      0.129559
```

**Interpretation:**
Readmission rate generally increases as length of stay increases (higher acuity/complexity tends to correlate with longer stays). At very long stays, counts drop and rates can wobble due to small n.

## Q4: Is readmission rate different by gender?

**SQL:**

```sql
SELECT
  gender,
  COUNT(*) AS n,
  AVG(readmit_30) AS readmit_rate
FROM cohort
GROUP BY gender
ORDER BY readmit_rate DESC;
```

**Result (from notebook):**

```text
gender      n  readmit_rate
0           Female  54708      0.112452
1             Male  47055      0.110615
2  Unknown/Invalid      3      0.000000
```

**Interpretation:** 
Female vs male rates are very close here, indicating that gender alone is not a strong predictor of 30-day readmission in this cohort. The  size of the “Unknown/Invalid” category is neglible. 

## Q5: Is readmission rate different by race categories in the dataset?

**SQL:**

```sql
SELECT
  race,
  COUNT(*) AS n,
  AVG(readmit_30) AS readmit_rate
FROM cohort
GROUP BY race
ORDER BY readmit_rate DESC;
```

**Result (from notebook):**

```text
race      n  readmit_rate
0        Caucasian  76099      0.112906
1  AfricanAmerican  19210      0.112181
2         Hispanic   2037      0.104075
3            Asian    641      0.101404
4            Other   1506      0.096282
5             None   2273      0.082710
```

**Interpretation:** 
Rates are relatively similar across the largest race groups; smaller groups have noisier estimates.

## Q6: Which admission types have the highest readmission rate (using the mapping table)?

**SQL:**

```sql
SELECT
  m.admission_type AS admission_type,
  COUNT(*) AS n,
  AVG(c.readmit_30) AS readmit_rate
FROM cohort c
LEFT JOIN admission_type_map m
  ON CAST(c.admission_type_id AS INTEGER) = m.admission_type_id
GROUP BY m.admission_type
ORDER BY readmit_rate DESC;
```

**Result (from notebook):**

```text
admission_type      n  readmit_rate
0                               Physician Referral  53990      0.115225
1                                  Clinic Referral  18480      0.111797
2       Transfer from another health care facility   5291      0.110754
3                    Discharged/transferred to SNF  18869      0.103927
4   Transfer from a Skilled Nursing Facility (SNF)   4785      0.103448
5                    Discharged/transferred to ICF     10      0.100000
6                            Court/Law Enforcement    320      0.084375
7                                   Emergency Room     21      0.000000
```

**Quick interpretation:** 
Referral-based admissions show higher readmission risk than transfers or emergency room cases, reinforcing that structured admissions often involve patients with ongoing care or underlying patient severity. Emergency Room admissions display a notably lower observed rate in this subset, likely reflecting small sample size rather than true reduced risk. 

## Q7: Among older age bands, what’s the average lab-procedure count and readmission rate?

**SQL:**

```sql
SELECT
  age,
  COUNT(*) AS n,
  AVG(readmit_30) AS readmit_rate,
  AVG(num_lab_procedures) AS avg_labs
FROM cohort
WHERE age IN ('[60-70)', '[70-80)', '[80-90)', '[90-100)')
GROUP BY age
ORDER BY age;
```

**Result (from notebook):**

```text
age      n  readmit_rate   avg_labs
0   [60-70)  22483      0.111284  42.600632
1   [70-80)  26068      0.117731  43.157396
2   [80-90)  17197      0.120835  44.085015
3  [90-100)   2793      0.110992  44.695310
```

**Quick interpretation:** 
Readmission rates slightly rise from ages 60–90, then level off in the oldest group, suggesting increasing clinical complexity with age but possible survivorship or care-path effects at extreme ages. However, the rate differences are modest; this is more of a 'resource use' signal than a strong discriminator by itself

## Q8: How does readmission rate vary with `number_inpatient` (previous inpatient visits)?

**SQL:**

```sql
SELECT
  number_inpatient,
  COUNT(*) AS n,
  AVG(readmit_30) AS readmit_rate
FROM cohort
GROUP BY number_inpatient
ORDER BY number_inpatient;
```

**Result (from notebook):**

```text
number_inpatient      n  readmit_rate
0                  0  67630      0.084371
1                  1  19521      0.129245
2                  2   7566      0.174333
3                  3   3411      0.202873
4                  4   1622      0.236128
5                  5    812      0.314039
6                  6    480      0.345833
7                  7    268      0.354478
8                  8    151      0.443709
9                  9    111      0.423423
10                10     61      0.426230
11                11     49      0.673469
12                12     34      0.500000
13                13     20      0.500000
14                14     10      0.400000
15                15      9      1.000000
16                16      6      0.333333
17                17      1      1.000000
18                18      1      0.000000
19                19      2      0.500000
20                21      1      1.000000
```

**Quick interpretation:** 
Patients with multiple previous inpatient stays show dramatically higher readmission rates, exceeding 30–40% at higher counts. This variable is one of the strongest stratifiers of readmission risk in the dataset. Howver, tiny counts, n, at high values can produce extreme rates.

## Q9: How does readmission rate vary with `number_emergency`?

**SQL:**

```sql
SELECT
  number_emergency,
  COUNT(*) AS n,
  AVG(readmit_30) AS readmit_rate
FROM cohort
GROUP BY number_emergency
ORDER BY number_emergency;
```

**Result (from notebook):**

```text
number_emergency      n  readmit_rate
0                  0  90383      0.104743
1                  1   7677      0.143546
2                  2   2042      0.182664
3                  3    725      0.202759
4                  4    374      0.307487
5                  5    192      0.244792
6                  6     94      0.234043
7                  7     73      0.260274
8                  8     50      0.320000
9                  9     33      0.363636
10                10     34      0.352941
11                11     23      0.217391
12                12     10      0.200000
13                13     12      0.333333
14                14      3      0.000000
15                15      3      0.333333
16                16      5      0.400000
17                18      5      0.200000
18                19      4      0.500000
19                20      4      0.500000
20                21      2      0.500000
21                22      6      0.500000
22                24      1      0.000000
23                25      2      0.000000
24                28      1      1.000000
```

**Quick interpretation:** 
An increasing number of prior emergency visits is associated with higher readmission rates. The trend suggests that frequent emergency utilization may act as a proxy for unstable disease management or limited outpatient care continuity. Though, the distribution is skewed since a few are 0.

## Q10: How does readmission rate vary with `number_diagnoses`?

**SQL:**

```sql
SELECT
  number_diagnoses,
  COUNT(*) AS n,
  AVG(readmit_30) AS readmit_rate
FROM cohort
GROUP BY number_diagnoses
ORDER BY number_diagnoses;
```

**Result (from notebook):**

```text
number_diagnoses      n  readmit_rate
0                  1    219      0.059361
1                  2   1023      0.060606
2                  3   2835      0.073721
3                  4   5537      0.082536
4                  5  11393      0.091547
5                  6  10161      0.104124
6                  7  10393      0.107669
7                  8  10616      0.118124
8                  9  49474      0.123802
9                 10     17      0.176471
10                11     11      0.272727
11                12      9      0.111111
12                13     16      0.187500
13                14      7      0.142857
14                15     10      0.200000
15                16     45      0.088889
```

**Quick interpretation:** 
Readmission tends to increase with more diagnoses up to the common range; very high counts are rare so rates there are unstable.

## Q11: Which discharge dispositions have the highest readmission rates (filtering small n)?

**SQL:**

```sql
SELECT
  discharge_disposition_id,
  COUNT(*) AS n,
  AVG(readmit_30) AS readmit_rate
FROM cohort
GROUP BY discharge_disposition_id
HAVING n >= 100
ORDER BY readmit_rate DESC;
```

**Result (from notebook):**

```text
discharge_disposition_id      n  readmit_rate
0                         28    139      0.366906
1                         22   1993      0.276969
2                          5   1184      0.208615
3                          2   2128      0.160714
4                          3  13954      0.146625
5                          7    623      0.144462
6                          8    108      0.138889
7                          4    815      0.127607
8                          6  12902      0.126957
9                         18   3691      0.124357
10                        25    989      0.093023
11                         1  60234      0.093004
12                        23    412      0.072816
13                        14    372      0.064516
14                        13    399      0.047619
15                        11   1642      0.000000
```

**Quick interpretation:** 
Discharge destination is a strong stratifier: some destinations (e.g., transfers to facilities) have much higher readmission than 'home'—consistent. These dispositions likely reflect greater severity of illness and ongoing care needs at discharge.

## Q12: Which admission sources have the highest readmission rates (filtering small n)?

**SQL:**

```sql
SELECT
  admission_source_id,
  COUNT(*) AS n,
  AVG(readmit_30) AS readmit_rate
FROM cohort
GROUP BY admission_source_id
HAVING n >= 100
ORDER BY readmit_rate DESC;
```

**Result (from notebook):**

```text
admission_source_id      n  readmit_rate
0                    3    187      0.155080
1                   20    161      0.136646
2                    5    855      0.118129
3                    7  57494      0.116882
4                    1  29565      0.105868
5                   17   6781      0.104114
6                    9    125      0.104000
7                    2   1104      0.100543
8                    4   3187      0.096956
9                    6   2264      0.093640
```

**Quick interpretation:** 
Differences exist but are smaller or noiser than prior inpatient or discharge disposition; watch sample sizes.

## Q13: Among specialties with n>=200, which have the highest readmission rates?

**SQL:**

```sql
SELECT
  medical_specialty,
  COUNT(*) AS n,
  AVG(readmit_30) AS readmit_rate
FROM cohort
GROUP BY medical_specialty
HAVING n >= 200
ORDER BY readmit_rate DESC
LIMIT 15;
```

**Result (from notebook):**

```text
medical_specialty      n  readmit_rate
0                 Hematology/Oncology    207      0.193237
1                            Oncology    348      0.189655
2                          Nephrology   1613      0.153751
3   PhysicalMedicineandRehabilitation    391      0.153453
4                    Surgery-Vascular    533      0.138837
5                          Psychiatry    854      0.121780
6              Family/GeneralPractice   7440      0.118683
7                             Missing  49949      0.115738
8                    InternalMedicine  14635      0.112470
9                    Emergency/Trauma   7565      0.111831
10                    Surgery-General   3099      0.110358
11                        Pulmonology    871      0.110218
12                   Gastroenterology    564      0.109929
13                        Orthopedics   1400      0.107857
14                            Urology    685      0.099270
```

**Quick interpretation:** 
Specialties like Hematology/Oncology and Nephrology appear higher-risk, but specialty is also correlated with case-mix and chronicity.

## Q14: Does readmission rise with the number of medications (bucketed)?

**SQL:**

```sql
SELECT
  CASE
    WHEN num_medications <= 5 THEN '0-5'
    WHEN num_medications <= 10 THEN '6-10'
    WHEN num_medications <= 20 THEN '11-20'
    ELSE '21+'
  END AS meds_bucket,
  COUNT(*) AS n,
  AVG(readmit_30) AS readmit_rate
FROM cohort
GROUP BY meds_bucket
ORDER BY n DESC;
```

**Result (from notebook):**

```text
meds_bucket      n  readmit_rate
0       11-20  52025      0.114599
1         21+  23880      0.127764
2        6-10  20795      0.094494
3         0-5   5066      0.074812
```

**Quick interpretation:** 
Higher medication burden (21+) corresponds to higher readmission rate; low-med groups are lower risk on average.

## Q15: How does readmission rate differ across insulin status categories?

**SQL:**

```sql
SELECT
  insulin,
  COUNT(*) AS n,
  AVG(readmit_30) AS readmit_rate
FROM cohort
GROUP BY insulin
ORDER BY readmit_rate DESC;
```

**Result (from notebook):**

```text
insulin      n  readmit_rate
0    Down  12218      0.138975
1      Up  11316      0.129905
2  Steady  30849      0.111284
3      No  47383      0.100374
```

**Quick interpretation:** 
Insulin 'Down/Up' categories have higher readmission than 'No'. Active insulin adjustment likely reflects unstable glycemic control, which may contribute to early readmission.

## Q16: How does readmission rate differ across metformin status categories?

**SQL:**

```sql
SELECT
  metformin,
  COUNT(*) AS n,
  AVG(readmit_30) AS readmit_rate
FROM cohort
GROUP BY metformin
ORDER BY readmit_rate DESC;
```

**Result (from notebook):**

```text
metformin      n  readmit_rate
0      Down    575      0.120000
1        No  81778      0.115165
2    Steady  18346      0.097133
3        Up   1067      0.082474
```

**Quick interpretation:** 
Patients with stable or increased metformin use demonstrate lower readmission rates than those not receiving metformin. Though, treatment variables are confounded by disease severity and contraindications.

## Q17: Within each insulin category, how does readmission rate change with `time_in_hospital` (with n>=50 filter)?

**SQL:**

```sql
SELECT
  insulin,
  time_in_hospital,
  COUNT(*) AS n,
  AVG(readmit_30) AS readmit_rate
FROM cohort
GROUP BY insulin, time_in_hospital
HAVING n >= 50
ORDER BY insulin, time_in_hospital;
```

**Result (from notebook):**

```text
insulin  time_in_hospital     n  readmit_rate
0     Down                 1  1158      0.111399
1     Down                 2  1808      0.129425
2     Down                 3  2044      0.130137
3     Down                 4  1700      0.158235
4     Down                 5  1293      0.129157
5     Down                 6   998      0.141283
6     Down                 7   839      0.140644
7     Down                 8   660      0.165152
8     Down                 9   448      0.169643
9     Down                10   355      0.197183
10    Down                11   296      0.104730
11    Down                12   234      0.132479
12    Down                13   209      0.153110
13    Down                14   176      0.142045
14      No                 1  8140      0.071622
15      No                 2  8616      0.090297
16      No                 3  8392      0.098785
17      No                 4  6275      0.103108
18      No                 5  4361      0.113277
19      No                 6  3267      0.111417
20      No                 7  2403      0.130670
21      No                 8  1732      0.129908
22      No                 9  1166      0.128645
23      No                10   925      0.128649
24      No                11   716      0.113128
25      No                12   538      0.131970
26      No                13   449      0.118040
27      No                14   403      0.119107
28  Steady                 1  3957      0.084407
29  Steady                 2  5337      0.097995
30  Steady                 3  5628      0.104300
31  Steady                 4  4407      0.120944
32  Steady                 5  3060      0.121569
33  Steady                 6  2266      0.132392
34  Steady                 7  1747      0.120206
35  Steady                 8  1318      0.149469
36  Steady                 9   921      0.125950
37  Steady                10   658      0.133739
38  Steady                11   517      0.079304
39  Steady                12   425      0.129412
40  Steady                13   349      0.106017
41  Steady                14   259      0.154440
42      Up                 1   953      0.121721
43      Up                 2  1463      0.120984
44      Up                 3  1692      0.125296
45      Up                 4  1542      0.126459
46      Up                 5  1252      0.132588
47      Up                 6  1008      0.142857
48      Up                 7   870      0.126437
49      Up                 8   681      0.138032
```

**Quick interpretation:** Longer hospital stays combined with insulin adjustments correspond to higher readmission rates. It is not perfectly monotone, but shows a trend.

## Q18: If we bucket by age, long stay (7+), and prior inpatient (>=1), which segments have the highest readmission?

**SQL:**

```sql
SELECT
  age,
  CASE WHEN time_in_hospital >= 7 THEN '7+' ELSE '<7' END AS los_bucket,
  CASE WHEN number_inpatient >= 1 THEN 'prior_inpatient' ELSE 'no_prior_inpatient' END AS prior_inp,
  COUNT(*) AS n,
  AVG(readmit_30) AS readmit_rate
FROM cohort
GROUP BY age, los_bucket, prior_inp
HAVING n >= 100
ORDER BY readmit_rate DESC
LIMIT 20;
```

**Result (from notebook):**

```text
age los_bucket           prior_inp     n  readmit_rate
0    [20-30)         <7     prior_inpatient   514      0.317121
1    [30-40)         7+     prior_inpatient   233      0.227468
2    [30-40)         <7     prior_inpatient   987      0.209726
3    [40-50)         7+     prior_inpatient   731      0.201094
4    [50-60)         7+     prior_inpatient  1257      0.182975
5    [40-50)         <7     prior_inpatient  2422      0.174236
6    [70-80)         7+     prior_inpatient  2337      0.172871
7    [60-70)         <7     prior_inpatient  5569      0.165918
8    [60-70)         7+     prior_inpatient  1857      0.163705
9    [80-90)         <7     prior_inpatient  4487      0.159126
10   [70-80)         <7     prior_inpatient  6666      0.155116
11   [50-60)         <7     prior_inpatient  4204      0.151998
12   [80-90)         7+     prior_inpatient  1673      0.146444
13  [90-100)         <7     prior_inpatient   704      0.144886
14  [90-100)         7+     prior_inpatient   216      0.120370
15   [70-80)         7+  no_prior_inpatient  3587      0.117647
16   [10-20)         <7     prior_inpatient   144      0.111111
17   [30-40)         7+  no_prior_inpatient   317      0.110410
18   [60-70)         7+  no_prior_inpatient  2872      0.109680
19   [80-90)         7+  no_prior_inpatient  2499      0.108443
```

**Quick interpretation:** The top segments are usually those with prior inpatient history; long stays further increase risk within an age band.

## Q19: What are the approximate Q1/median/Q3 of `time_in_hospital` (SQLite window functions)?

**SQL:**

```sql
WITH ordered AS (
  SELECT
    time_in_hospital,
    ROW_NUMBER() OVER (ORDER BY time_in_hospital) AS rn,
    COUNT(*) OVER () AS n
  FROM cohort
)
SELECT
  MAX(CASE WHEN rn = CAST(0.25*n AS INT) THEN time_in_hospital END) AS q1,
  MAX(CASE WHEN rn = CAST(0.50*n AS INT) THEN time_in_hospital END) AS median,
  MAX(CASE WHEN rn = CAST(0.75*n AS INT) THEN time_in_hospital END) AS q3
FROM ordered;
```

**Result (from notebook):**

```text
q1  median  q3
0   2       4   6
```

**Quick interpretation:** The distribution of the median hospital stay is centered around 4 days (median), with Q1≈2 and Q3≈6. 

## Q20: For a simple rule-based classifier, how many false positives do we generate per true positive?

**SQL:**

```sql
WITH preds AS (
  SELECT
    readmit_30,
    CASE
      WHEN number_inpatient >= 1 OR time_in_hospital >= 7 THEN 1
      ELSE 0
    END AS pred
  FROM cohort
),
cm AS (
  SELECT
    SUM(CASE WHEN pred=1 AND readmit_30=0 THEN 1 ELSE 0 END) AS fp,
    SUM(CASE WHEN pred=1 AND readmit_30=1 THEN 1 ELSE 0 END) AS tp
  FROM preds
)
SELECT
  fp,
  tp,
  (1.0*fp)/NULLIF(tp,0) AS fp_per_tp
FROM cm;
```

**Result (from notebook):**

```text
fp    tp  fp_per_tp
0  39839  7019    5.67588
```

**Quick interpretation:** The rule-based predictor catches true positives but generates ~5.7 false alarms per true catch. This is useful as an operational burden metric.

## Key Takeaways
- Readmission risk increases with length of stay and prior inpatient utilization.
- Discharge disposition and medication burden strongly stratify risk.
- Many extreme rates occur in small-n groups and should be interpreted cautiously.
- High-recall rules incur substantial false-positive cost in deployment settings.

## Notes & Limitations
- The dataset is observational; associations are not causal.
- Several categorical levels have small sample sizes.
- Readmission prevalence (~11%) creates precision–recall tradeoffs.