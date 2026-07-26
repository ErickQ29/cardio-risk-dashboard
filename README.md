# Cardiovascular Disease Risk Dashboard

An end-to-end medical data analytics project exploring key physiological, demographic, and lifestyle factors associated with cardiovascular disease (CVD), built with Python for data cleaning/feature engineering and Power BI for the interactive dashboard.

## The idea

The dataset has 70,000 patient records with basic clinical and lifestyle data (blood pressure, cholesterol, glucose, smoking, alcohol, activity) and a binary target: whether the patient has cardiovascular disease. The question I wanted the dashboard to answer is simple: which of these factors actually track with higher CVD prevalence, and by how much? Everything below (the cleaning thresholds, the derived columns, the visuals) is built around answering that.

## Dataset

- Source: [Cardiovascular Disease dataset](https://www.kaggle.com/datasets/sulianova/cardiovascular-disease-dataset) by Svetlana Ulianova (Kaggle).
- 70,000 rows, 11 input features + 1 target (`cardio`), CSV delimited with `;`.
- `age` comes in **days**, not years — first thing I fixed.
- `gender`, `cholesterol`, and `gluc` are numeric codes (1/2/3), not labels.
- `ap_hi` / `ap_lo` are systolic/diastolic blood pressure, and the raw file has some clinically impossible values (e.g. negative or absurdly high readings) that need filtering before any of it is usable.

## Data cleaning & feature engineering (Python)

Done in `cleaned.ipynb` with `pandas` and `numpy`. This is the actual pipeline, in order:

```python
import pandas as pd
import numpy as np

df = pd.read_csv("cardio_train.csv", sep=';')

# age is stored in days
df['age_years'] = (df['age'] / 365.25).astype(int)
df.columns = df.columns.str.strip()

# drop physiologically implausible height/weight
df_clean = df[(df['height'] >= 140) & (df['height'] <= 200)].copy()
df_clean = df_clean[(df_clean['weight'] >= 40) & (df_clean['weight'] <= 180)]

# drop implausible blood pressure readings
df_clean = df_clean[(df_clean['ap_hi'] >= 80) & (df_clean['ap_hi'] <= 220)]
df_clean = df_clean[(df_clean['ap_lo'] >= 50) & (df_clean['ap_lo'] <= 130)]

# BMI
df_clean['bmi'] = (df_clean['weight'] / ((df_clean['height'] / 100) ** 2)).round(1)

# Mean Arterial Pressure: MAP = DBP + 1/3 (SBP - DBP)
df_clean['map'] = (df_clean['ap_lo'] + ((df_clean['ap_hi'] - df_clean['ap_lo']) / 3)).round(1)

# age brackets for the dashboard
bins = [30, 40, 50, 60, 100]
labels = ['30-39', '40-49', '50-59', '60+']
df_clean['age_group'] = pd.cut(df_clean['age_years'], bins=bins, labels=labels, right=False)

# simple composite risk score: cholesterol, glucose, MAP, BMI each contribute 1 point
def calc_risk(row):
    score = 0
    if row['cholesterol'] > 1: score += 1
    if row['gluc'] > 1: score += 1
    if row['map'] >= 100: score += 1
    if row['bmi'] >= 30: score += 1
    return score

df_clean['risk_score'] = df_clean.apply(calc_risk, axis=1)
df_clean['risk_profile'] = np.where(df_clean['risk_score'] >= 2, 'High Risk Profile', 'Controlled Profile')

# human-readable labels for the categorical codes
df_clean['cholesterol_label'] = df_clean['cholesterol'].map({1: 'Normal', 2: 'Above Normal', 3: 'Well Above Normal'})
df_clean['gluc_label'] = df_clean['gluc'].map({1: 'Normal', 2: 'Above Normal', 3: 'Well Above Normal'})
df_clean['cardio_label'] = df_clean['cardio'].map({0: 'Healthy', 1: 'Cardiovascular Disease'})

df_clean.to_csv('cardio_cleaned.csv', index=False)
```

Result: 68,480 rows kept out of 70,000 (1,520 dropped as outliers on height, weight, or blood pressure).

`risk_score` / `risk_profile` are in the cleaned data (48,524 "Controlled Profile" vs 19,956 "High Risk Profile") but I haven't wired them into a dashboard visual yet — planned a donut chart for it, cut it for time.

**Known gap:** the notebook also checks `ap_hi > ap_lo` (systolic should be higher than diastolic), but that filter's result is never reassigned back to `df_clean`, so it doesn't actually get applied. 53 rows in the final CSV still have systolic ≤ diastolic. Small enough not to skew the aggregates, but worth fixing if I revisit the notebook.

## Power BI dashboard

Single-page report (`healthcare.pbix`) built on `cardio_cleaned.csv`. Layout:

- **KPI header row** — six card visuals: Total Patients, Cardio Cases, % Cardio Prevalence, Avg Systolic BP, Avg Diastolic BP, Avg BMI.
- **Disease prevalence by age group** — clustered column chart, `age_group` on the X-axis, patient count on Y, split by `cardio_label`.
- **Blood pressure scatter plot** — `ap_lo` (diastolic) vs. `ap_hi` (systolic), colored by `cardio_label`, to see where diagnosed cases cluster.
- **Disease prevalence by cholesterol level** — 100% stacked column chart, `cholesterol_label` on the X-axis, split by `cardio_label`.
- **Slicer panel** — gender, smoking status, physical activity, cholesterol level, alcohol consumption, all using label columns instead of raw numeric codes.
- **Dynamic risk indicator** — a heart icon (SVG, rendered through a DAX measure) that switches from slate blue to red depending on whether the current filter context's prevalence rate crosses 50%.

### DAX measures

```dax
Total Patients = COUNTROWS('cardio_cleaned')

Cardio Cases = CALCULATE(COUNTROWS('cardio_cleaned'), 'cardio_cleaned'[cardio] = 1)

% Cardio Prevalence = DIVIDE([Cardio Cases], [Total Patients], 0)

Avg Systolic BP = AVERAGE('cardio_cleaned'[ap_hi])

Avg Diastolic BP = AVERAGE('cardio_cleaned'[ap_lo])

Avg BMI = AVERAGE('cardio_cleaned'[bmi])
```

### Calculated columns (added in Power BI, not Python)

`gender`, `smoke`, `active`, and `alco` are also stored as 0/1/2 codes, so I mapped them to readable labels directly in Power BI for the slicers:

```dax
gender_label = IF('cardio_cleaned'[gender] = 1, "Female", "Male")
smoke_label = IF('cardio_cleaned'[smoke] = 1, "Smoker", "Non-Smoker")
active_label = IF('cardio_cleaned'[active] = 1, "Active", "Inactive")
alco_label = IF('cardio_cleaned'[alco] = 1, "Alcohol Consumer", "Non-Consumer")
```

### Dynamic heart icon

The centerpiece visual is a measure that returns an inline SVG as a data URI, categorized as an Image URL field type, dropped into a table visual with headers/gridlines turned off:

```dax
Dynamic Heart Icon =
VAR RiskLevel = [% Cardio Prevalence]
VAR FillColor = IF(RiskLevel > 0.50, "%23E11D48", "%2364748B")
RETURN
"data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' style='display:block; margin:auto;' fill='" & FillColor & "'><path d='M12 21.35l-1.45-1.32C5.4 15.36 2 12.28 2 8.5 2 5.42 4.42 3 7.5 3c1.74 0 3.41.81 4.5 2.09C13.09 3.81 14.76 3 16.5 3 19.58 3 22 5.42 22 8.5c0 3.78-3.4 6.86-8.55 11.54L12 21.35z'/></svg>"
```

It re-evaluates under whatever slicers are active, so filtering to, say, smokers over 50 will flip the icon red if that subgroup's prevalence passes the 50% threshold.

## Key findings

- Overall CVD prevalence in the cleaned data: **49.5%** (34,481 of 68,480 patients).
- Prevalence climbs steadily with age: **23.6%** (30-39) → **37.5%** (40-49) → **51.3%** (50-59) → **66.8%** (60+).
- Cholesterol is a strong split: **43.5%** prevalence at Normal levels vs. **76.2%** at Well Above Normal.
- Average systolic/diastolic BP across the cleaned sample: **126.6 / 81.3 mmHg**. Average BMI: **27.4**.

## Repository contents

```
cleaned.ipynb           # Python cleaning + feature engineering
cardio_train.csv        # raw Kaggle dataset
cardio_cleaned.csv      # output of cleaned.ipynb, used as the Power BI source
healthcare.pbix          # Power BI report
```

## Reproducing this

```bash
git clone https://github.com/ErickQ29/cardio-risk-dashboard.git
cd cardio-risk-dashboard
```

1. Run `cleaned.ipynb` end to end against `cardio_train.csv` to regenerate `cardio_cleaned.csv`.
2. Open `healthcare.pbix` in Power BI Desktop.
3. If prompted, repoint the data source to your local `cardio_cleaned.csv` path, then refresh.

## Tools

- **Python** (pandas, numpy) — cleaning and feature engineering.
- **Power BI Desktop** — Power Query for load/type-checking, DAX for measures and the dynamic icon, native visuals for everything else (no custom visual imports).

## Data source

[Cardiovascular Disease dataset](https://www.kaggle.com/datasets/sulianova/cardiovascular-disease-dataset), Svetlana Ulianova, Kaggle. License listed as unknown on the dataset page — used here for a personal portfolio project, not redistributed.
