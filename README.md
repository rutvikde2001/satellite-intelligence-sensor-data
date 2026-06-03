# Sensor data processing — deliverables

## 1. Data quality audit

Polars-based audit of `resources/parcel_readings.csv` (3,447 rows) and `resources/parcel_metadata.csv` (28 rows). All columns were loaded as strings to surface formatting and join issues. Full checks and code are in [`1.data_quality_audit.ipynb`](1.data_quality_audit.ipynb).

### Summary

| Dataset | Field | Issue | Count | % | Action | Justification |
|---------|-------|-------|------:|---:|--------|---------------|
| metadata | parcel_id | Metadata parcels with zero readings | 3 | 10.7% | flag | Expected operational variance (e.g., new parcels). Flag to monitor pipeline coverage without dropping metadata. |
| readings | date | Non-ISO date format: DD/MM/YYYY | 686 | 19.9% | repair | Parse to canonical Date using explicit format list. |
| readings | date | Non-ISO date format: DD-Mon-YYYY | 330 | 9.6% | repair | Parse to canonical Date using explicit format list. |
| readings | date | Ambiguous DD/MM slash dates (both parts ≤ 12) | 279 | 8.1% | Repair + Verify | Safe parsing: Infer locale (DD/MM vs MM/DD) from unambiguous rows in the same batch before converting to ISO and set a flag `is_data_ambiguous = 1`. |
| readings | date | Duplicate parcel_id + date combinations | 8 | 0.2% | Drop + Flag | Deduplicate by keeping the first occurrence or field with proper all fields are valid; log/flag the incident to trace upstream ingestion bugs. |
| readings | ndvi_value | NDVI outside valid range [-1.0, 1.0] | 104 | 3.0% | Flag + Nullify | Physically impossible telemetry. Flag for data lineage tracking, coerce invalid values to NaN to avoid skewing ML models. |
| readings | parcel_id | Orphan parcel_id not in metadata | 40 | 1.2% | flag | No metadata to join; exclude from joined output or alert upstream. |
| readings | rainfall_mm | Mixed int/float numeric representation (whole vs fractional) | 2833 | 82.2% | repair | Cast to Float64 for consistent numeric operations in pipeline. |
| readings | sensor_status | Inconsistent status casing/spelling (OK/error/ERROR/ok etc.) | 3353 | 97.3% | repair | Normalize to canonical OK / ERROR / Missing after stripping whitespace. |
| readings | sensor_status | Leading/trailing whitespace in sensor_status | 110 | 3.2% | repair | Strip whitespace before normalizing status values. |
| readings | sensor_status | Empty, NA, or NaN sensor_status | 94 | 2.7% | flag | Missing sensor health; exclude from analysis where status matters. |
| readings | temperature_c | Mixed int/float numeric representation (whole vs fractional) | 349 | 10.1% | repair | Cast to Float64 for consistent numeric operations in pipeline. |

**Total distinct issues:** 12

### Readings — dates

`date` mixes ISO (`YYYY-MM-DD`), slash (`DD/MM/YYYY`), and abbreviated month (`DD-Mon-YYYY`) formats. A subset of slash dates is ambiguous when both day and month are ≤ 12 (279 rows, 8.1%); infer locale from unambiguous rows in the batch, parse to ISO, and set `is_data_ambiguous = 1` where needed. Eight duplicate `(parcel_id, date)` keys (0.2%) should be deduplicated (keep first valid row) and flagged for upstream tracing. After multi-format parsing, all reading dates fall within Jan–May 2026 (no out-of-window rows).

### Readings — values and status

`ndvi_value` has 104 rows (3.0%) outside the valid [-1, 1] range — flag for lineage and null invalid values (NaN) before analysis. `temperature_c` and `rainfall_mm` are stored as strings with mixed whole-number vs fractional representation; cast to Float64 in the pipeline (not a data-loss issue). `sensor_status` needs whitespace stripping (110 rows), normalization to OK / ERROR / Missing (97.3% of rows use non-canonical values), and flagging of 94 missing/NA rows (2.7%) for exclusion where sensor health matters.

### Cross-file and metadata

40 reading rows (1.2%) reference `parcel_id` values with no metadata match — exclude from joined output or alert upstream. Three metadata parcels have no readings: `PARCEL_050`, `PARCEL_051`, `PARCEL_052` (10.7% of metadata rows) — treated as expected operational variance; flag for coverage monitoring without dropping metadata. Metadata otherwise passed null/empty, whitespace, duplicate `parcel_id`, and `sowing_date` parse checks.

## 2. Clean pipeline

**Notebook:** [`2.clean_pipeline.ipynb`](2.clean_pipeline.ipynb)  
**Output:** [`cleaned_parcel_timeseries.csv`](cleaned_parcel_timeseries.csv) — **3,399 rows** (one per `parcel_id` × `date`)

### Approach

| Audit decision | Pipeline behavior |
|----------------|-------------------|
| Date formats (ISO / DD/MM / DD-Mon) | Parse via `dateutil.parser`; canonical `Date`; ambiguous slash dates use batch MDY/DMY vote + `is_data_ambiguous` |
| Ambiguous slash dates | Infer DD/MM vs MM/DD from unambiguous batch rows; default DD/MM; set `is_data_ambiguous = 1` |
| Duplicate `(parcel_id, date)` | Keep best row (valid NDVI, then OK status, then first); set `is_duplicate = 1` on kept row |
| NDVI out of range | Set `is_ndvi_out_of_range = 1`; null `ndvi_value` |
| Orphan readings | **Inner join** on `parcel_id` (40 rows excluded) |
| `temperature_c` / `rainfall_mm` | Cast to Float64 |
| `sensor_status` | Strip whitespace; normalize to `OK` / `ERROR` / `MISSING` (raw nulls → `MISSING`) |
| Metadata | Strip strings; parse `sowing_date`; cast `area_hectares` |
| Metadata with no readings | Not in output (reading-driven join); logged in validation |

### Output columns

`parcel_id`, `date`, `ndvi_value`, `temperature_c`, `rainfall_mm`, `sensor_status`, `mill_id`, `crop_type`, `sowing_date`, `area_hectares`, `is_data_ambiguous`, `is_ndvi_out_of_range`, `is_duplicate`

CSV is written with quoted fields (`quote_style="always"`) so null `ndvi_value` appears as `""` and columns stay aligned in Excel.

## 3. Quick analysis

**Notebook:** [`3.quick_analysis.ipynb`](3.quick_analysis.ipynb)  
**Input:** [`cleaned_parcel_timeseries.csv`](cleaned_parcel_timeseries.csv)

### Method

- **Filter:** keep `sensor_status == OK` only (exclude ERROR and MISSING); drop null `ndvi_value` → **3,017** rows used (382 excluded)
- **Before window:** `[sowing_date − 30 days, sowing_date)` (sowing day excluded)
- **After window:** `(sowing_date, sowing_date + 30 days]`
- **Aggregation:** per-parcel mean NDVI in each window, then mean across parcels per `crop_type`; `n_parcels` = distinct parcels per crop in the filtered set

### Results

| crop_type | mean_ndvi_before | mean_ndvi_after | n_parcels |
|-----------|-----------------:|----------------:|----------:|
| soybean | 0.175 | 0.314 | 4 |
| sugarcane | 0.177 | 0.336 | 19 |
| wheat | 0.174 | 0.311 | 2 |

### Interpretation

After keeping only `sensor_status == OK` readings, mean NDVI rises in the 30 days after sowing for every crop type — roughly 0.17 before vs 0.31–0.34 after — which matches expected canopy growth once crops are established. Sugarcane shows the highest post-sowing NDVI (~0.34) and the largest sample (19 parcels), while soybean and wheat are similar to each other but based on only 4 and 2 parcels respectively, so those two estimates should be treated as indicative rather than definitive.

## 4. Production-readiness reflection

If this pipeline ran daily on a 100x larger dataset, here is what I would change, monitor, and worry about — grounded in the decisions actually made in Tasks 1–3.

### Three things I would change

1. **Vectorize date parsing.** At this scale I deliberately chose `dateutil.parser` applied row-wise via `map_elements` in [`2.clean_pipeline.ipynb`](2.clean_pipeline.ipynb) for robustness and development speed — it absorbs mixed/odd formats without me enumerating every pattern. At 100x daily this row-wise UDF becomes the throughput bottleneck because it does not vectorize. Since the audit already pinned the real formats (ISO, `DD/MM/YYYY`, `DD-Mon-YYYY`), the safe migration is a vectorized `pl.coalesce` of native `str.to_date` calls with that explicit format list — same correctness, far higher throughput. It is a robustness-vs-throughput tradeoff with a known, low-risk migration path, not a defect.
2. **Incremental, partitioned storage.** Today the pipeline reads and rewrites the full `cleaned_parcel_timeseries.csv` every run. At 100x I would switch to columnar storage (Parquet) partitioned by date (and possibly `mill_id`) with incremental daily appends, so a daily run processes only the new partition instead of re-reading all history. This also makes downstream Task 3-style window queries cheaper.
3. **Per-source, persisted date-locale decision.** The ambiguous `DD/MM` vs `MM/DD` choice currently comes from a single whole-batch vote (`infer_slash_use_mdy`). At scale, with many sources/days, one global vote can flip between runs. I would resolve locale per source (or per parcel) and persist that decision, so a day's dates cannot be silently relabeled by a shift in batch composition.

### What I would monitor in production

- **Volume and schema drift:** daily row counts and column types vs an expected baseline (sudden drops/spikes flag upstream breakage).
- **Data-quality rates as alerting metrics:** % NDVI out of range, % `sensor_status` MISSING/ERROR, % orphan `parcel_id`, and % ambiguous dates — each with thresholds so a regression pages someone instead of quietly degrading the output.
- **Join coverage:** orphan reading rate and metadata-without-readings count (the `PARCEL_050`–`052` style gap) to catch metadata/ingestion desync.
- **Freshness and runtime:** lateness of incoming readings and pipeline runtime against an SLA.

### Most likely thing to silently break

The **date parsing and ambiguous-slash locale vote**. It is dangerous precisely because it fails *silently*: a wrong `DD/MM` vs `MM/DD` decision still yields valid-looking dates, which shifts readings into the wrong before/after-sowing window and biases the Task 3 NDVI numbers with no error raised. The guardrails are explicit metrics — an unparseable-date count (should stay 0) and an ambiguity-rate alarm — so a locale flip surfaces as a monitored signal rather than a quiet correctness bug.

### How AI tools were used

I used Cursor (Claude) to scaffold the Polars audit, cleaning, and analysis notebooks, to debug two concrete issues (the `unnest` duplicate-`date` column error and the CSV null-`ndvi` column-alignment problem), and to draft these README sections. All cleaning decisions, thresholds, and interpretations were reviewed and chosen by me.
