# Mini Case Report (EO Analytics): Solar Underperformance

## Problem statement
We are given 1 week of **actual solar site production** at 5‑minute resolution and an **expected (forecast) baseline** at hourly resolution.  
The objective is to identify **when** the site underperforms, determine whether the signal looks like a **true operational issue** (outage/derate) vs a **data quality gap**, and quantify the business impact in **lost energy (MWh)**.

---

## Method (what the notebook does)
### 1) Align the time series
- Parse `LocalTime` into a timestamp index (`ts`)
- Resample expected hourly MW → **5‑minute MW** using time interpolation
- Build a single “fact” table:
  - `actual_mw`, `expected_mw`
  - `gap_mw = expected - actual`
  - `gap_mw_pos = max(gap_mw, 0)` (only counts lost generation)

### 2) Daily triage (dashboard view)
- Convert 5‑minute power (MW) to daily energy (MWh):  
  `MWh = sum(MW) * (5/60)`
- Compute daily performance ratio: `actual_mwh / expected_mwh`
- Flag the **worst day** (lowest ratio) for drilldown.

### 3) 5‑minute drilldown + confirmed underperformance window
We evaluate underperformance during daylight, and confirm it only when sustained:
- daylight filter: `expected_mw > 0.5`
- midday focus: 10:00–15:00
- high expectation: `expected_mw >= 25`
- underperformance trigger:
  - `actual_mw <= 8` OR `actual_mw <= 0.30 * expected_mw`
- confirmation: ≥ 6 consecutive points (≥ 30 minutes)

This produces:
- `flag_outage_like`
- `outage_run_len` (streak counter)
- `flag_outage_confirmed`

### 4) Impact estimation
For the confirmed window:
- duration = `(# points) * 5 minutes`
- lost energy (MWh) = `sum(gap_mw_pos) * (5/60)`
- severity proxy = average `ratio` in the window

---

## Interpreting the outputs
### Daily dashboard
- **Most days** track expected closely → normal operation / baseline alignment
- A **single day** shows a sharp drop in actual energy while expected remains stable → strong underperformance candidate

### 5‑minute drilldown
- Confirms whether the underperformance is:
  - a sustained operational dip (outage/derate-like), or
  - a data issue (missing actuals, timestamp gaps, etc.)

### Monitoring checks (pipeline health)
Simple metrics help separate “real ops” vs “data pipeline”:
- % missing actual
- % missing expected
- rows per day (should be 288 for 5‑minute cadence)
- timestamps covered (earliest/latest)

---

## Key assumptions / limitations
- Expected (forecast) is treated as the baseline (no weather/irradiance covariates)
- The outage rule is intentionally simple and interpretable; in production you’d:
  - calibrate thresholds per site/season
  - incorporate irradiance / POA / inverter status / curtailment flags if available
  - add multi-site support + alerting thresholds

---

## Suggested next steps (if this were production)
- Add site capacity normalization (expected vs capacity)
- Add curtailment and comms-loss signals if available
- Add alert routing: severity tiers based on lost MWh + duration
- Track recurring patterns (same hours/day-of-week) for preventive maintenance
