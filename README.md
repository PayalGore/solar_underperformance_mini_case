# Mini Case: Investigating an Underperforming Solar Site (EO Analytics)

This notebook demonstrates an end-to-end, ops-ready approach to diagnosing solar underperformance using time-series analytics.

## Goal
Given **actual site production** and an **expected (forecast) baseline**, we:
1. Detect when the site is underperforming (daily triage → 5‑minute drilldown)
2. Distinguish **operational underperformance** (outage/derate) vs **data gaps**
3. Estimate impact (**lost MWh** + duration window)
4. Produce monitoring artifacts an EO team can use (charts + event tables + pipeline checks)
5. Provide a Snowflake-style SQL equivalent of the outage-flag logic

---

## Data used
- **Actual generation (MW)** at **5-minute** resolution (1 week → **2016 rows**)
- **Expected generation (MW)** at **hourly** resolution (1 week → **168 rows**)
- Expected is **resampled to 5-minute intervals** (time interpolation) to align with actuals.

### Important note about zeros
Overnight values are often **0 MW** — this is normal for solar.  
Performance metrics are evaluated primarily during **daylight** (by default: `expected_mw > 0.5`) to avoid nighttime bias.

---

## Run behavior (real vs synthetic)
The notebook runs in either mode:
- **Real CSV mode:** if raw CSVs exist in `data/raw/`, they are loaded.
- **Synthetic mode:** if raw CSVs are not present, the notebook generates a realistic synthetic dataset (including a planted outage + missing-data window) so it runs end-to-end.

The notebook prints `Source: real_csv` or `Source: synthetic`.

---

## What you’ll see in this notebook
- **Daily dashboard view:** expected vs actual energy + daily performance ratio (triage)
- **5‑minute drilldown:** expected vs actual power on the event day
- **Outage/derate detection rules:** practical, interpretable thresholds + “sustained” confirmation
- **Impact estimation:** confirmed window, duration, estimated lost MWh, average derate severity
- **Monitoring checks:** basic pipeline-health indicators (missingness %, row counts per day, etc.)
- **Snowflake-style SQL:** equivalent window-based logic for the outage confirmation rule

---

## Key outputs
- **Daily table**: `actual_mwh`, `expected_mwh`, `lost_mwh`, `ratio`
- **Event window table**: 5‑minute points with flags and run-length counters
- **Impact summary**: confirmed points, window start/end, duration (minutes), estimated lost MWh

---

## Underperformance rule (interpretable & ops-friendly)
The “confirmed underperformance” signal is designed to avoid false positives:
- **Daylight only** (expected > 0.5 MW)
- **Midday focus** (default: 10:00–15:00)
- **High-expectation filter** (default: expected ≥ 25 MW)
- Underperformance if **actual is very low** or **drops sharply vs expected**:
  - `actual <= 8 MW` **OR**
  - `actual <= 0.30 * expected`
- **Sustained requirement**: ≥ 30 minutes (6 consecutive 5‑min points)

> These thresholds are easy to tune per site (capacity, seasonality, curtailment patterns).

---

## How to run (Google Colab)
1. Open the notebook in Colab.
2. (Optional) Upload raw CSVs into `data/raw/`:
   - `Actual_..._5_Min.csv`
   - `DA_..._60_Min.csv`
3. **Run all cells** (Runtime → Run all).
4. Review outputs in this order: **Daily dashboard → event day drilldown → impact summary → pipeline checks → SQL snippet**.

---

## Repository contents
- `Underperformance_mini_case.ipynb` — main notebook (Colab-ready)
- `data/raw/` — raw CSV input folder (optional)
- `requirements.txt` — minimal Python dependencies
- `REPORT.md` — short, shareable write-up of results & interpretation

