# Loom Walkthrough Script (~8 min)

Sensor data processing take-home — Carnot assignment.

---

## Before you record

**Open these tabs/files:**

1. `Readme.md`
2. `1.data_quality_audit.ipynb` (outputs visible)
3. `2.clean_pipeline.ipynb`
4. `3.quick_analysis.ipynb`
5. `cleaned_parcel_timeseries.csv` (optional — quick peek at output)

**Tip:** Run all three notebooks once before recording so outputs are populated.

---

## Script overview

| Section | Time | File |
|---------|------|------|
| Intro + repo tour | 0:00–0:45 | README top |
| Task 1 — how you found issues | 0:45–2:15 | `1.data_quality_audit.ipynb` |
| Task 2 — 4 key decisions | 2:15–5:30 | `2.clean_pipeline.ipynb` |
| Task 3 — analysis | 5:30–6:30 | `3.quick_analysis.ipynb` |
| Production reflection (spoken) | 6:30–8:00 | README §4 or no screen |

---

## 0:00–0:45 — Intro & repo structure

**Show:** `Readme.md` (top of file)

**Say:**

> Hi — this is my take-home for the Carnot data engineering assignment. I had two raw CSVs: daily parcel sensor readings and parcel metadata. My deliverables are three notebooks, a cleaned timeseries CSV, and this README which documents every data quality issue, the cleaning approach, analysis results, and a production reflection.
>
> The flow is: audit the mess → clean and join → analyze NDVI around sowing date. I used Polars throughout and Cursor to help scaffold and debug, but every cleaning decision is mine and is traceable from the audit to the pipeline.

**Show:** Quick scroll of repo root — `1.data_quality_audit.ipynb`, `2.clean_pipeline.ipynb`, `3.quick_analysis.ipynb`, `cleaned_parcel_timeseries.csv`, `resources/`.

---

## 0:45–2:15 — Task 1: Data quality audit

**Show:** `Readme.md` → **§1 Data quality audit** → summary table (12 issues)

**Say:**

> Before writing any cleaning code, I documented every issue with count, percent, and my call — drop, flag, or repair. That table in the README is the source of truth; the audit notebook is where I actually discovered these problems.

**Show:** `1.data_quality_audit.ipynb`

### Block 1 — First look at readings

**Cell:** `# parcel_readings.csv` → load cell (`readings.head(5)` output)

**Say:**

> I loaded readings as strings on purpose — I didn't want Polars to silently coerce types. Immediately in `head()` you can see mixed date formats: `16/05/2026` next to `2026-01-27`, and inconsistent `sensor_status` like `error` vs `OK`.

### Block 2 — Dates

**Cell:** `### date` → format counts output (ISO / DD/MM / DD-Mon)

**Say:**

> About 70% of rows are ISO, but roughly 20% are slash dates and 10% abbreviated month. That's not cosmetic — wrong parsing would put readings in the wrong time window later.

**Cell:** next date cell — ambiguous slash count (279 rows) + duplicate keys (8)

**Say:**

> 279 slash dates are ambiguous when both parts are ≤ 12 — you can't tell DD/MM from MM/DD without context. There are also 8 duplicate parcel-plus-date keys. Both of those drove specific pipeline decisions I'll show next.

### Block 3 — sensor_status & NDVI

**Cell:** `### sensor_status` → raw value counts output

**Say:**

> `sensor_status` is a mess — OK, ok, ERROR, error, whitespace variants, NA, null. I normalize to three values: OK, ERROR, MISSING.

**Cell:** `### ndvi_value` → out-of-range count (104 rows, min/max outside [-1, 1])

**Say:**

> 104 NDVI values are physically impossible. I flag those and null the value rather than drop the whole row — temperature and rainfall might still be useful.

### Block 4 — Join surprise

**Cell:** `### join` → orphan rows output (40 rows, `PARCEL_098` / `PARCEL_099`)

**Say:**

> I only found orphan parcel IDs when I tried joining — 40 reading rows have no metadata match. And on the metadata side, three parcels have zero readings. Those are flagged, not silently ignored.

**Show:** `Readme.md` — cross-file paragraph (optional, 10 sec)

---

## 2:15–5:30 — Task 2: Clean pipeline (4 key decisions)

**Show:** `2.clean_pipeline.ipynb` → title cell

**Say:**

> The pipeline implements every README decision. I'll walk through the four decisions that matter most.

---

### Decision 1 — Ambiguous date parsing + flag (most important)

**Show:** **Cell 2** — `infer_slash_use_mdy()` and `parse_reading_date()`

**Say:**

> For dates I use `dateutil` row-wise for robustness at this scale. For ambiguous slash dates like `10/02/2026`, I don't guess per row — I vote across the whole batch: unambiguous dates where one part is > 12 tell me whether the source is DD/MM or MM/DD. Default is DD/MM on a tie. Any date that was ambiguous gets `is_data_ambiguous = 1` so downstream consumers know the parse had uncertainty. At 100x scale I'd vectorize this — I'll come back to that.

**Show:** **Cell 6** — `clean_readings()` → `use_mdy = infer_slash_use_mdy(...)` and `map_elements(_parse_row, ...)`

---

### Decision 2 — Dedup: keep the best row, don't blind-drop

**Show:** **Cell 6** — `clean_readings()` bottom half: `_ndvi_ok`, `_status_ok`, sort + `group_by(...).first()`, `is_duplicate`

**Say:**

> Eight rows share the same parcel and date. I dedupe by keeping the best row: valid NDVI first, then OK sensor status, then earliest ingestion order. The kept row gets `is_duplicate = 1` so we can trace upstream ingestion bugs without losing data silently.

---

### Decision 3 — Inner join: exclude orphans from output

**Show:** **Cell 10** — `readings_clean.join(metadata_clean, on="parcel_id", how="inner")`

**Say:**

> Orphan readings have no mill, crop, or sowing date — they're useless for the timeseries analysis. I use an inner join, which drops 40 rows. I log that count in validation rather than leaving null metadata columns that look like a join bug.

**Show:** **Cell 12** — validation output (`excluded 40 orphan rows`)

---

### Decision 4 — NDVI invalid + sensor_status for downstream analysis

**Show:** **Cell 6** — NDVI block: `ndvi_out` → null + `is_ndvi_out_of_range`

**Show:** **Cell 2** — `normalize_sensor_status()`

**Show:** **Cell 6** — missing flags (`is_sensor_status_missing`, drop missing parcel_id/date)

**Say:**

> NDVI outside [-1, 1] gets nulled with `is_ndvi_out_of_range = 1` — separate from truly missing values. Sensor status is normalized to OK / ERROR / MISSING, with a missing flag. For analysis I only trust OK readings — that's ~3,017 of 3,399 rows. ERROR and MISSING stay in the cleaned file for monitoring but don't bias NDVI means.

**Show:** **Cell 14** — `write_csv(..., quote_style="always")`

**Say:**

> Small but real detail: quoted CSV so null NDVI shows as empty string in Excel, not shifted columns.

**Show:** validation cell output — row counts, flag sums, `sensor_status` breakdown

---

## 5:30–6:30 — Task 3: Quick analysis

**Show:** `3.quick_analysis.ipynb`

### Block 1 — Filter

**Cell:** `### Filter sensor status`

**Say:**

> Analysis reads the cleaned CSV, keeps only `sensor_status == OK`, drops null NDVI. That leaves 3,017 rows — consistent with the audit decision to exclude bad sensor health.

### Block 2 — Window logic

**Cell:** `### NDVI windows around sowing` + code cell below

**Say:**

> For each parcel I compute mean NDVI in two half-open windows: 30 days before sowing, sowing day excluded, and 30 days after. Half-open matters — if you include sowing day in both windows you double-count.

### Block 3 — Results

**Cell:** `### Aggregate by crop_type` → output table

**Say:**

> Every crop type shows higher NDVI after sowing — roughly 0.17 before vs 0.31–0.34 after — which matches canopy growth. Sugarcane is highest at ~0.34 with 19 parcels. Soybean and wheat look similar but only 4 and 2 parcels, so I'd treat those as indicative, not definitive.

**Show:** `Readme.md` → **§3 Results** table (optional confirm)

---

## 6:30–8:00 — Production reflection (say this out loud)

**Show:** `Readme.md` §4 *or* just camera — brief requires verbal answer

**Say:**

> If this ran daily at 100x scale, three things I'd change:
>
> **One — vectorize date parsing.** I used row-wise `dateutil` for speed of development and because formats are messy. The audit already identified the real formats, so I'd migrate to vectorized Polars `str.to_date` with an explicit format list. Same correctness, much better throughput.
>
> **Two — incremental partitioned storage.** Today I rewrite the full CSV every run. At scale I'd use Parquet partitioned by date, maybe mill, with daily appends so each run only touches new data.
>
> **Three — persist the date locale decision per source.** Right now one batch-wide vote picks DD/MM vs MM/DD. If batch composition shifts, that vote can flip between runs and silently relabel dates.
>
> **What I'd monitor:** daily row counts and schema drift; data-quality rates — % NDVI out of range, % MISSING/ERROR status, % orphan parcel IDs, % ambiguous dates — each with alert thresholds; join coverage gaps like metadata parcels with no readings; pipeline freshness and runtime against an SLA.
>
> **What would silently break:** the ambiguous slash-date locale vote. A wrong DD/MM vs MM/DD choice still produces valid-looking dates — no error — but it shifts readings into the wrong before/after sowing window and biases the NDVI analysis. That's why I'd alarm on ambiguity rate and unparseable-date count, not just pipeline success.
>
> Thanks for watching — repo link is in the submission.

---

## Checklist (brief requirements)

- [ ] 5–10 minutes
- [ ] Code on screen
- [ ] **4 decisions narrated:** (1) date/locale + flag, (2) dedup strategy, (3) inner join orphans, (4) NDVI nullify + sensor_status OK filter
- [ ] Production reflection **spoken**, not only in README
- [ ] Mention AI tools briefly (README § "How AI tools were used")

---

## Optional cuts if you're over 10 min

- Skip `temperature_c / rainfall_mm` audit cell
- Skip CSV write / preview cells
- Shorten README table scroll — point at it, don't read every row

---

## Optional additions if you're under 5 min

- Run one audit cell live (date format counts)
- Open `cleaned_parcel_timeseries.csv` and show flag columns
- Expand interpretation with one parcel example from `parcel_metrics.head()`
