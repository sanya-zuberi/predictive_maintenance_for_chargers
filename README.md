# Predictive Maintenance for Chargers

Using utilisation, session duration and energy patterns to find which EV chargers may need maintenance first, and testing how far the data can predict trouble.

## The Question

Which chargers should be inspected or serviced first, and can session data predict which ones will fail? This project builds health indicators for every charger, trains a model to predict a proxy for trouble, checks the model against simple rules, and ranks chargers for maintenance. The dataset records no faults or downtime, so every model result concerns a proxy, not real failures.

## Repo Structure

```
├── notebooks/        → full analysis notebook
├── output/           → CSV outputs
├── dashboard/        → Power BI .pbix file + page screenshots
├── slides/           → presentation deck
└── documentation/    → project documentation 
```

## Data

The source is the EV charging dataset: `stations` (33 stations), `chargers` (199 chargers: 95 rated 50 kW, 69 rated 150 kW and 35 rated 300 kW) and `sessions` (294,024 sessions, January 2022 – December 2024). The raw file is not included in this repo. It is hosted as a Kaggle Dataset: https://www.kaggle.com/datasets/sanyazuberi/ev-charging-raw-dataset.

None of the three sheets has a fault, status, outage or maintenance column, and there are no missing values or duplicate sessions.

Output files (in `output/`):
- `charger_health_indicators.csv`: one row per charger: utilisation, sessions and energy per day, session duration, power ratio, overlap rate, quiet-spell measures and recent trends
- `indicator_reliability.csv`: for five indicators, how much charger type and station size explain, and whether differences repeat between odd and even months
- `model_evaluation.csv`: AUC, average precision and top-10% event rate for the model and the simple baselines
- `risk_calibration_by_decile.csv`: predicted against actual inactivity rate by risk decile
- `feature_importance.csv`: permutation and gain importance for the 17 model features
- `risk_scores_2024.csv`: the model's risk score and the actual outcome for each charger-month in 2024
- `maintenance_priority_list.csv`: one row per charger: risk, wear, priority score, rank, tier, flags and action

## How Health and Risk Were Measured

- **Utilisation** is the share of a charger's available time that it is in use. Overlapping sessions on the same charger were merged before busy time was counted. Available time runs from 1 January of the station's activation year to the end of 2024.
- **Power ratio** is the average power delivered in a session divided by the charger's rated power.
- **Quiet spells** are measured as the days between two session starts on the same charger. The **gap ratio** compares a charger's longest quiet spell with what random session arrivals would typically give for its traffic (1 means as expected).
- **Trend indicators** compare the last 90 days of 2024 with the previous 12 months, for sessions per day and for power ratio.
- **The label is a proxy.** With no failure records, the model predicts an *inactivity event*: a charger with a gap of 3 or more days between two session starts in the next month. This cannot tell a fault from a quiet charger.
- **The model** is XGBoost, trained on charger-months from 2022-2023 (1,740 rows) and tested on January to November 2024 (2,189 rows). It uses 17 features: that month's indicators, 3-month history, wear (cumulative energy and months in service), rated power and station size.
- **Evaluation** compares the model with three simple baselines, checks it within each station size, and compares predicted risk with actual events by decile. Feature importance is measured by permutation (how much the AUC falls when a feature is shuffled).
- **The priority list** scores each charger using December 2024 data. The priority score is the average of its wear rank (total sessions and total energy delivered) and its inactivity-risk rank. Chargers are placed in tiers (top 10%, next 20%, rest) and flagged for inspection by three screening rules: a gap ratio above 1.5, sessions below 85% of the previous year's rate, or delivered power down more than 1%.

## Key Findings

- **The dataset holds no failure information.** No fault, status or maintenance fields exist, so a real failure model could not be built or checked.
- **The sessions show no signs of trouble.** Delivered power is always 64% to 90% of rated power (median 80%), every session lasts at least 15 minutes and delivers at least 21.7 kWh, and the longest idle gap on any charger is 8.2 days. Only 3 chargers ever go quiet for over a week.
- **Chargers differ by type and station size, not individually.** Charger type and station size explain 97% to 100% of the differences between chargers. After removing those effects, behaviour hardly repeats between odd and even months (correlations from -0.11 to 0.17, where anything below about 0.14 is noise).
- **Health indicators are flat.** Average utilisation is 5.4% (range 2.2% to 11.5%) and every charger delivers 77% to 80% of rated power. Recent sessions per day are between 0.75 and 1.36 times the previous year's rate, and recent power ratio is within 2% of its earlier level. Only 3 chargers have a gap ratio above 1.5 (STN_025_CH_02, STN_004_CH_02 and STN_004_CH_05).
- **The inactivity label mostly reflects station size.** 14.6% of labelled charger-months are followed by an inactivity event, rising from 0.4% at 4-charger stations to 28.4% at 8-charger stations. Each station receives roughly the same number of sessions whatever its size, so chargers at bigger stations get fewer sessions each and go quiet more often by chance.
- **The model does not beat simple rules.** On 2024 data it reaches an AUC of 0.728, while ranking by low recent traffic alone gives 0.753. Among chargers at the same kind of station its ranking is no better than chance (AUC 0.41). Its scores are overconfident at the top: the highest-risk decile is predicted at 44% but has an actual rate of 28%.
- **Two features carry the model.** Recent traffic (3-month sessions) cuts the AUC by 0.107 when shuffled and station size by 0.024. Every other feature changes it by 0.004 or less, and 9 of 17 have zero or negative importance. The wear measures add nothing.
- **8.3% of sessions overlap.** 24,425 sessions start before the previous session on the same charger has ended, which points to an allocation or logging problem, not a hardware fault.

## Model Results

| Model (tested on 2024) | AUC | Event rate in riskiest 10% |
|---|---|---|
| Full XGBoost model (17 features) | 0.728 | 28.0% |
| Baseline: low recent traffic only | 0.753 | 28.4% |
| Baseline: station size only | 0.753 | 24.8% |
| Baseline: XGBoost on station size and traffic | 0.736 | 24.8% |

The overall test event rate is 15.4%.

## Maintenance Priority List

| Tier | Chargers | Median sessions | Share of all sessions |
|---|---|---|---|
| Tier 1 (top 10%) | 20 | 1,727 | 11.6% |
| Tier 2 (next 20%) | 40 | 1,931 | 30.1% |
| Tier 3 | 139 | 995 | 58.3% |

- Tier 1 is made up of chargers from the 2022 and 2023 sites (10 each), and 14 of the 20 are at 8-charger stations. Tiers 1 and 2 are 30% of chargers but handled 42% of all sessions.
- The top five by priority score are STN_004_CH_08, STN_005_CH_01, STN_005_CH_08, STN_016_CH_03 and STN_005_CH_05.
- 14 chargers are flagged for inspection whatever their tier: 3 with an unusually long quiet spell, 6 with falling sessions and 7 with falling delivered power (two chargers have two flags). The thresholds are screening rules, and because charger differences do not repeat from month to month, some flags will be chance.

## Recommendations

1. **Treat the risk model as a screening aid, not a failure predictor.** It matches simple traffic and station-size rules, so do not schedule repairs from its scores alone.
2. **Start recording real failures.** Log fault and status codes, downtime, work orders and parts replaced, then retrain this pipeline to predict outages in the next 30 days.
3. **Service by wear, not by calendar.** The busiest charger has delivered about 15 times the energy of the least busy (292,098 kWh against 19,646), and 2022 chargers average 2,825 sessions each against 1,550 for 2023 and 836 for 2024. Work through Tier 1, then Tier 2.
4. **Inspect the 14 flagged chargers now,** starting with STN_004_CH_02, STN_004_CH_05 and STN_025_CH_02, and confirm whether the cause is hardware, allocation or something else.
5. **Fix charger allocation and logging** so health monitoring can separate a real availability problem from a logging one.
6. **Monitor monthly with alert limits,** to be tuned once fault data exists: gap ratio above 2, sessions trend below 0.70, delivered power down more than 3%, or a quiet spell over 10 days. Each sits just beyond the largest deviation seen in this data.
7. **Check whether the lower power ratio on 300 kW chargers is by design.** They deliver 77.7% of rated power, against 80.0% for the others.

## Dashboard

Built in Power BI, three pages:
- **Charger Health**: slicers for charger type, station size and region, KPI cards, utilisation by charger type, sessions per day by station size, and the noise-check table
- **Risk Model**: average predicted risk against actual rate, model AUC against the baselines, the evaluation table, predicted vs actual rate by risk decile, and feature importance
- **Maintenance Priorities**: slicers for tier, region and charger type, KPI cards, a wear-rank vs risk-rank scatter, chargers by tier and station size, and the top 20 priority table

See `dashboard/` for the `.pbix` file and page screenshots.

## Reproducing This Project

1. Download the raw dataset from the Kaggle link above.
2. Run the notebook in `notebooks/` from top to bottom in a fresh session. It writes the CSV files and the `figures/` folder.
3. Open the `.pbix` in `dashboard/` and refresh the data sources if the CSV paths have changed.

## Tools Used

Python (pandas, NumPy, scikit-learn, XGBoost, Matplotlib, Seaborn) · Kaggle Notebooks · Power BI

## Limitations

- There are no failure records, so every model result concerns a proxy (inactivity), not failure.
- The dataset appears simulated. Each station receives roughly the same number of sessions whatever its size, and chargers show no consistent differences beyond type and station size, so the data may not contain real wear or fault patterns.
- Wear is measured only by sessions and energy delivered. Temperature, connector condition and component age are not in the data.
- The priority score averages two ranks with equal weights, which is a judgement call. The weights can be changed in the notebook.
- Flag thresholds and alert limits are starting points set from the spread in this data, not validated limits.
- Available time assumes each station operates from 1 January of its activation year, because the installation dates conflict with session activity.
- Risk scores can differ slightly between XGBoost versions.
