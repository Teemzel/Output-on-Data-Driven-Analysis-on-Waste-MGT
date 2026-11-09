# Data-Driven Analysis and Predicting Waste Generation in Nigeria

A reproducible workflow that cleans Nigerian waste data, maps where waste is generated, tests forecasting methods on the Lagos dumpsite record, and projects national and state waste generation to 2035 under low, central and high scenarios.

---
# Contents

1. [Project overview](#project-overview)
2. [Research questions](#research-questions)
3. [Key results](#key-results)
4. [Input data](#input-data)
5. [Setup](#setup)
6. [How to run](#how-to-run)
7. [Methods in brief](#methods-in-brief)
8. [Assumptions and limitations](#assumptions-and-limitations)
9. [Sources](#sources)
---

# Project overview

Published estimates of waste generation in Nigeria disagree, monthly records are rare, and projections are seldom tested. This project addresses that in three steps:

| Step | What it does |
|---|---|
| **Describe** | Cleans the available data, documents every correction, and maps waste and organic share by state and zone |
| **Test** | Compares seven forecasting methods on the Lagos dumpsite record (Jan 2022 to Dec 2024) |
| **Project** | Projects generation and organic waste for each state to 2035 using population growth, urbanisation and composition change |

# Research questions

1. Which of the available waste data can be trusted, and what corrections does it need?
2. Where in Nigeria is waste generated, and how much of it is organic?
3. Which simple method best predicts waste reaching the Lagos dumpsites, and what share of Lagos's estimated waste reaches them?
4. How much waste and organic waste will Nigeria generate by 2035 if population and urbanisation grow as projected?

# Key results

| Result | Value |

| National MSW, 2023 (state-file baseline) | 27.5 Mt (58.8% organic) |
| Share from five states (Lagos, Kano, Rivers, Edo, Katsina) | 40.0% |
| Lagos dumpsite receipts, 2022 / 2023 / 2024 | 1.90 / 1.75 / 1.39 Mt |
| Best Lagos forecasting model | 12-month linear trend (MAPE 6.4%) |
| Central projection, 2035 | 38.9 Mt (range 34.7 to 42.1 Mt) |
| Organic share, 2023 to 2035 (central) | 58.8% to 56.8% |
| Organic waste, 2023 to 2035 (central) | 16.2 Mt to 22.1 Mt |

The baseline **level** is the main uncertainty: the state file gives 27.5 Mt for 2023, whereas interpolating World Bank figures gives about 43.5 Mt (scaling factor about 1.585).


# Input data

| File | Content | Notes |
|---|---|---|
| `lagos_waste_data.xlsx` | Monthly trips and tonnes for five Lagos dumpsites (Olushosun, Solous, Badagry, Epe, Ikorodu), Jan 2022 to Dec 2024 | Records waste **deposited**, not waste generated. Source: eko360 portal (confirm publisher and access date). |
| `State_Level_OFMSW_Amounts_Generated_-_Nigeria.xlsx` | Per capita rate, 2006 and 2023 population, organic share, annual MSW for 36 states and the FCT | Rates and organic shares. 2023 population is the 2006 figure x 1.547 for every state. |
| `nigeria_state_boundaries_geojson.txt` | 37 state polygons (WGS84, 2016 boundaries) | Joined on state name (`FCT` renamed `Federal Capital Territory`). |


# Setup

Requires Python 3.10 or later.
```bash
python -m venv .venv
# Windows:  .venv\Scripts\activate        macOS/Linux:  source .venv/bin/activate
pip install pandas numpy openpyxl matplotlib statsmodels geopandas
```

# How to run
```bash
# 1. Cleaning, map join, Lagos forecasting and maps
python waste_pipeline.py

# 2. Projection to 2035
python waste_projection.py
```

`waste_projection.py` prints the calibration, a check that state rows add up to national totals, and a summary table. It stops with an error if the check fails.

# Methods in brief

**Cleaning**
- **State file:** drop the national total row, rename `FCT`, recompute MSW = rate x population x 365 / 1000 (reproduces the file to rounding), assign six geopolitical zones, compute area in an equal-area projection (EPSG:6933).
- **Lagos file:** tonnes per trip are compared with each site's median. A value is flagged if it is more than the larger of 5 robust standard deviations (1.4826 x MAD) and 25% of the median away. The one flagged value (Olushosun, June 2023: 163,616 t for 9,471 trips) is replaced with trips x median (91,015 t) and logged.
- **Structural break:** Solous trips fall about 50% from July 2023 (kept and reported).
- **June 2023 surge:** kept in the cleaned data, replaced by the mean of adjacent months in the modelling series only.

# Lagos forecasting
Seven models are compared by **rolling-origin evaluation** (7 origins, training windows of 24 to 30 months, 6-month horizon, 42 forecasts per model): naive, mean of last 6 months, seasonal naive, simple exponential smoothing, damped-trend smoothing, ARIMA(0,1,1) with drift, and a linear trend on the last 12 months. Metrics: MAE, RMSE, MAPE, bias. The 2025 and 2026 results are shown as **scenarios** (low = damped trend, central = mean of the three best models, high = simple smoothing), not statistical intervals.

# Projection to 2035
```
G(s,t) = G(s,2023) x P(t) x M(t) x kappa          generation, state s, year t
P(t)   = N(t) / N(2023)                             population index (UN WPP 2024)
M(t)   = (U(t) / U(2023)) ^ epsilon                 per-capita multiplier from urban share
o(s,t) = o(s,2023) + beta x c(s) x (M(t) - 1)       organic share (%)
O(s,t) = G(s,t) x o(s,t) / 100                      organic waste
```

| Scenario | epsilon | Meaning |
|---|---|---|
| Low | 0 | Per-capita waste stays flat |
| Central | about 0.60 | **Calibrated** so per-capita growth matches the World Bank 2016 to 2030 projection |
| High | 1.0 | Per-capita waste grows as fast as the urban share |

`beta` (about -39.6 points per kg/person/day, r = -0.71) comes from regressing organic share on per-capita generation across the 37 areas. Set `composition_shift` to `0` in `PARAMS` to hold organic shares fixed.


# Assumptions and limitations

1. **Baseline level.** The state file applies city-level per capita rates from an earlier compilation to whole-state populations. Its total (27.5 Mt) is below World Bank-based estimates. Set `baseline_scale` in `PARAMS` (about 1.585) to rescale; all tonnages scale linearly.
2. **Uniform growth.** Every state grows at the national UN rate because the source has no state-specific growth.
3. **Calibrated elasticity.** Calibrated to a single World Bank projection and applied nationally, so the match in growth rates is partly built in.
4. **Composition shift.** Uses a cross-state relationship (partly driven by Lagos; slope is -27.4 without it) as a stand-in for change over time.
5. **Lagos series.** Short (36 months), contains a structural break, measures disposal rather than generation, and tonnage appears closely tied to trips. Forecasts are scenarios; the backtest covers only the 2023 to 2024 decline.
6. **Capture ratio.** Receipts divided by estimated Lagos generation (2023 estimate scaled by national population). Read as indicative, not a measured collection rate.
7. The series ends in December 2024, so 2025 and 2026 observations now exist and should be used to test the scenarios if obtainable.

# Sources

- Alamu, A. S. (2025). *Journal of Sustainability and Environmental Management, 4*(3), 165-171. https://doi.org/10.3126/josem.v4i3.93144
- Kaza, S., Yao, L. C., Bhada-Tata, P., & Van Woerden, F. (2018). *What a Waste 2.0*. World Bank. https://doi.org/10.1596/978-1-4648-1329-0
- Africa Check (2019). *What a waste: Fact-checking four claims about Nigeria's garbage problem*. https://africacheck.org/reports/what-a-waste-fact-checking-four-claims-about-nigerias-garbage-problem/
- Atughwe, R., Gadkari, S., & Short, M. (2026). National and Subnational Quantitative Estimates of Agricultural Wastes, OFMSW, and UCO in Nigeria, Kenya, and South Africa (Version v.3) [Data set]. Zenodo. https://doi.org/10.5281/zenodo.22206719.
- Dickson, E. M., Hastings, A., & Smith, J. (2023). *Frontiers in Sustainability, 4*, 1173474. https://doi.org/10.3389/frsus.2023.1173474
- Maton, S. M., et al. (2016). *J. Geography, Environment and Earth Science International, 6*(3). https://dspace.unijos.edu.ng/jspui/handle/123456789/1873
- United Nations (2024). *World Population Prospects 2024*. https://population.un.org/wpp/ (accessed via Worldometers and StatisticsTimes)



