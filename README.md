# BUSA3022 Market Data Pipeline (S&P/ASX 200 + VIX)

A small, repeatable data pipeline that downloads two public data series, validates them, and produces one curated CSV for metric development.

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/npincini/BUSA3022/blob/main/BUSA3022_Data_Pipeline_%28S%26P_ASX_200_%2B_VIX%29.ipynb)

---

## 1. What it does

| Source | Series | Role | Access |
|---|---|---|---|
| Yahoo Finance via the `yfinance` library | S&P/ASX 200 (`^AXJO`): open, high, low, close, volume | **Market index series** (performance of the Australian share market) | Python library |
| FRED (St. Louis Fed) | CBOE VIX daily close (`VIXCLS`) | **Macro series** (expected volatility in the US share market) | Public CSV endpoint |

**Business question it supports:** how does US market volatility relate to Australian market activity, so bank performance can later be benchmarked against industry data?

## 2. Pipeline layers

| Layer | Folder | Rule |
|---|---|---|
| Raw | `data/raw/` | Saved exactly as received. Never edited. Each file is timestamped and recorded in `manifest.csv` |
| Transform | in the notebook | Reads from the raw files. Nothing is written back to raw. |
| Curated | `data/curated/` | The single handoff file plus a data dictionary. |
| Logs | `logs/` | Quality log, quarantine file, and run log for every run. |

## 3. Quick start on Google Colab

1. Click the **Open in Colab** badge above (or upload the file to Colab).
2. `Runtime > Run all`.
3. Download the outputs using the last cell (one zip containing `data/` and `logs/`), or find them in the Colab file browser (folder icon) under `data/` and `logs/`.

Colab's disk is temporary, so download results before the session ends.

## 4. Scheduled run on GitHub Actions

The workflow in `.github/workflows/pipeline.yml` runs the notebook on a schedule and on demand. No API keys or secrets are needed.

| Setting | Value |
|---|---|
| Schedule | 07:00 UTC, Monday to Friday (5pm AEST or 6pm AEDT in Sydney). This is after the ASX close and after the 16:30 Sydney cut-off used to quarantine unfinished bars. |
| Manual run | Open the **Actions** tab, select **Market data pipeline**, then choose **Run workflow**. |
| Dependencies | Installed from `requirements.txt`. |
| Command | `jupyter nbconvert --to notebook --execute "BUSA3022_Data_Pipeline_(S&P_ASX_200_+_VIX).ipynb" --output run_output.ipynb --ExecutePreprocessor.timeout=600` |
| Outputs | One artefact per run, `pipeline-outputs-<run id>`, containing `data/raw`, `data/curated` and `logs`. Kept for 14 days. |
| Fallback files | `data/raw` is cached between runs, so the "reuse the newest earlier raw file" fallback has a file to use. |
| Failure | The run is marked as failed in the Actions tab. The upload step still runs, so the logs are kept. |

Notes:

* GitHub cron times are in UTC, and scheduled workflows run from the default branch. A scheduled run can start later than its scheduled time.
* GitHub can disable scheduled workflows in a repository that has had no activity for a period, and deletes caches that have not been used for a period. See GitHub's documentation for the current limits.
* The manual file route in section 7 cannot be used on GitHub Actions, because files cannot be added to the runner. The cached raw files are the fallback there.

## 5. Outputs

| File | What it is |
|---|---|
| `data/curated/market_curated.csv` | The handoff file. One row per ASX trading date. |
| `data/curated/data_dictionary.csv` | Column definitions (also in section 8). |
| `data/raw/asx200_yahoo_<run>.csv`, `vix_fred_<run>.csv` | Untouched downloads. |
| `data/raw/manifest.csv` | Audit trail of every download. |
| `logs/quality_log_<run>.csv` | Every event that happened. |
| `logs/quarantine_<run>.csv` | Every rejected row with its reason and original contents. Always created, even if empty. |
| `logs/run_<run>.log` | The same events as readable text (Sydney time). |

`<run>` is the Sydney-time timestamp of the run, for example `20261026T143000` is 26 October 2026 at 2:30pm.

## 6. How the two sources are aligned (date and time zone handling)

The ASX and the US market trade in different time zones. For a Sydney date *D*, the ASX closes at about 4pm Sydney time. The US close for date *D* happens at 4pm New York time, which is the morning of D+1 in Sydney, so it does not exist yet while the ASX is trading on *D*.

**Rule:** for each ASX date *D*, use the latest VIX close from a date strictly before *D*.

* The ASX 200 is the date spine. The output has exactly one row per clean ASX date.
* Tuesday to Friday normally pair with the previous US day. Monday pairs with Friday.
* Using the same-date VIX would be look-ahead bias, so exact matches are not allowed. The pipeline checks this on every run and stops if it is violated.
* Daylight saving changes the gap between the markets by an hour or two but never changes which close happens first, so the rule holds all year.
* **ASX timestamps:** Yahoo returns midnight Sydney time with a changing UTC offset (`+10:00` or `+11:00`). These are parsed, converted explicitly to `Australia/Sydney`, then reduced to a plain date.
* **VIX dates:** FRED provides plain US market dates, which are used as-is.
* **Carried-forward values:** if the expected VIX date has no value (for example Christmas Day or New Year's Day, or because FRED has not published it yet), the previous close is used and the row is flagged `vix_is_filled = True`. The date actually used is always shown in `vix_source_date`.
* **Staleness limit:** a VIX close is never carried forward more than 7 days. Beyond that, `vix_close` is left blank and a warning is logged.

## 7. Data quality: nothing is silently dropped

Rows that fail a check are moved to `logs/quarantine_<run>.csv` with a reason code and their original contents. After each source is validated, the pipeline proves `raw rows = clean rows + quarantined rows` and stops with an error if the numbers do not balance.

**Row checks (row is quarantined):**

| Reason code | Meaning |
|---|---|
| `INVALID_DATE` | Date could not be read |
| `DUPLICATE_DATE` | Date appears more than once (last copy kept) |
| `INCOMPLETE_SESSION` | Today's ASX bar fetched before 16:30 Sydney time (a live, partial bar) |
| `MISSING_VALUE` | A required value is blank, `.`, `NA`, `null` and so on. VIX rows with a blank value are logged at INFO level; if an ASX date would have used one, the align step uses the most recent earlier close |
| `UNPARSEABLE_VALUE` | A value is present but not a number |
| `NON_POSITIVE_PRICE` | A price or VIX value is zero or negative |
| `OHLC_INCONSISTENT` | High below open/close/low, or low above them (0.01% tolerance) |
| `NEGATIVE_VOLUME` | Volume below zero |
| `ZERO_RANGE_BAR` | Open, high, low and close are all identical, so the bar has zero trading range |

**Dataset warnings (rows are kept, a warning is logged):** identical prices on consecutive trading days (`STALE_PRICES`), gaps over 5 calendar days (`CALENDAR_GAP`), median volume of zero (`ZERO_MEDIAN_VOLUME`), days with volume under 20% of the median (`LOW_VOLUME_OUTLIER`), an `open` that equals the previous close on more than half of days (`OPEN_EQUALS_PREVIOUS_CLOSE`), data older than 5 days (`STALE_SOURCE`), and VIX values that could not be matched (`NO_VIX_WITHIN_TOLERANCE`).

**Failure handling:**

| Failure | What happens |
|---|---|
| Download error or timeout | Up to three attempts with growing waits (2s, then 4s). Each failure is logged. |
| All attempts fail, earlier raw file exists | The latest raw file is used, `FALLBACK_TO_LAST_RAW` is logged at ERROR level, and `manifest.csv` records `status = fallback`. The freshness check then flags old data. |
| All attempts fail, no earlier file | `CRITICAL` event logged and the run stops. If FRED is the source that failed, the **manual file route** below can be used on Colab or a local run. |
| Source returns something that is not the expected CSV (for example an HTML error page) | Treated as a failed attempt. |
| Column names change (schema change) | `SCHEMA_CHANGE` logged at CRITICAL level and the run stops. No curated file is written. |
| Source has no usable rows after validation | `NO_USABLE_ROWS` logged and the run stops. |
| Row counts do not reconcile, or an integrity check fails | The run stops before the curated file is written. |

**Manual file route (FRED only):** if FRED stays unreachable, download the CSV by hand from the FRED page for `VIXCLS` (`Download > CSV`) and save it in `data/raw/` as `vix_fred_<YYYYMMDDTHHMMSS>_manual.csv`, using a timestamp later than any existing raw file. When the download fails, the pipeline's fallback treats it as the latest raw file, logs `FALLBACK_TO_LAST_RAW`, and validates it like any other download.

## 8. Data dictionary (`market_curated.csv`)

| Column | Type | Description |
|---|---|---|
| `date` | date (YYYY-MM-DD) | ASX trading date in Sydney time (Australia/Sydney). One row per date. The other columns are matched to this date. |
| `open` | number | Opening level as reported by Yahoo Finance. Yahoo can report the previous day's close in place of the opening level. Each run measures how often this happens and logs `OPEN_EQUALS_PREVIOUS_CLOSE` when it is more than 50% of days. |
| `high` | number | Highest level during the session, as reported by Yahoo Finance. Rows where high is below open, close or low are quarantined (`OHLC_INCONSISTENT`). |
| `low` | number | Lowest level during the session, as reported by Yahoo Finance. Rows where low is above open, close or high are quarantined (`OHLC_INCONSISTENT`). |
| `close` | number | Closing level for the session, as reported by Yahoo Finance. Not dividend-adjusted. |
| `volume` | integer | Volume as reported by Yahoo Finance. The source does not state the unit. Days with volume under 20% of the median are listed in the log (`LOW_VOLUME_OUTLIER`) and kept in the data. |
| `vix_close` | number | VIX (CBOE Volatility Index) close from FRED series `VIXCLS`, taken from the latest US market date strictly before `date`. Blank if no value exists within 7 days. |
| `vix_source_date` | date | The US market date `vix_close` came from. Always earlier than `date`. |
| `vix_is_filled` | True/False | True when no VIX value existed for the previous weekday before `date`, so the most recent earlier value was used. Otherwise False. |

## 9. How to audit a run

1. If the run was on GitHub Actions, open it in the **Actions** tab, confirm it finished green, and download the artefact. Then open `logs/quality_log_<run>.csv` and filter `level` for `WARNING`, `ERROR` and `CRITICAL`.
2. Check the `ROW_RECONCILIATION` lines. `balanced=True` means every row is accounted for.
3. Open `logs/quarantine_<run>.csv` to see every rejected row and why. Each row has `raw_row`, which is its line number in the raw file.
4. Check `data/raw/manifest.csv`. `status` should be `ok`. `fallback` means that run used old data.
5. Compare the SHA-256 in the manifest with the raw file to confirm it has not been edited.
6. Check `VIX_CARRIED_FORWARD` against the `vix_is_filled` column in the curated file.

## 10. References

* Yahoo Finance data accessed with the `yfinance` Python library (Aroussi, 2017 to present). <https://github.com/ranaroussi/yfinance>
* Federal Reserve Bank of St. Louis. *CBOE Volatility Index: VIX (VIXCLS)*. FRED. <https://fred.stlouisfed.org/series/VIXCLS>
