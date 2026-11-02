# tradeavail: UN Comtrade Data Availability, parsed, tested and visualised

Companion code for the paper *Reconstructing and Visualizing Six Decades of Global Trade-Data Availability: A Case Study on UN Comtrade Metadata with Order-of-Magnitude Charts*.

## 1. Problem and objective
The Comtrade **Data Availability** export describes *which* trade data exist (reporter x year x classification), not trade values. Its raw rows are left-shifted and ragged, so ordinary loaders misalign columns. Full statement: [`docs/PROBLEM_STATEMENT.md`](docs/PROBLEM_STATEMENT.md).

| RQ | Question |
|---|---|
| RQ1 | Can the export be reconstructed reliably? |
| RQ2 | How did classification regimes, volume and file size evolve 1962-2025? |
| RQ3 | How timely are releases, given only the *last* release date is recorded? |
| RQ4 | Can file size be predicted from record count, and is it stable over time? |

### Success metrics and results
| Metric | Target | Result |
|---|---|---|
| Rows reconstructed | 100% | **8,916 / 8,916** |
| Agreement with independent grammar | >= 99% | **100%**, 0 violations (the validator would have caught the original bug; see tests) |
| Manual audit, 60 random rows | 0 errors | **0 errors** (reviewed by the author; `docs/parser_audit_sample.csv`) |
| Size model, temporal holdout | R^2 >= 0.95 | **0.973** records only, **0.982** with year |
| Growth quantified | CAGR + 95% CI | **12.0% / yr (10.9-13.1%)** |
| Reproducibility | tests pass, one command | **14 tests**, two scripts |

## 2. The 90-second story
1. **Regimes.** SITC editions give way to HS editions H0-H6, each dominant for about five years.
![classification](docs/figures/classification_over_time.png)
2. **Scale.** Yearly records grew ~104x (2.0M to 210.6M), three orders of magnitude, so the OML/OMH charts show band crossings (1994, 2015) that a linear axis hides. They also mute growth *inside* a band, so choose the chart by the question.
![oml](docs/figures/oml.png)
3. **Why file size tracks records.** Bytes per record are ~13 until the 2000s, then rise (decade medians 14.8 in the 2010s, 18.4 in the 2020s), so the 0.97 correlation is mostly mechanical.
![bpr](docs/figures/bytes_per_record.png)
4. **Timeliness needs care.** The raw lag-vs-year correlation (-0.65) mostly reflects exposure; adjusted for the available window it is -0.12, and differences between editions are small and non-monotone.
![lag](docs/figures/lag_by_edition.png)

## 3. Data collection and sourcing
- **Source:** UN Comtrade Data Availability export, workbook `DataAvailability_9_21_2026_11_25_48.xlsx` (retrieved 21 Sep 2026), https://comtradeplus.un.org/. SHA-256 is in `DATASHEET.md` of the dataset release.
- **Integration:** single authoritative source, so no multi-table integration was needed. Joining external reference data (World Bank income groups, regions) is listed under next steps.

## 4. Cleaning and preparation (decisions and reasons)
| Issue | Decision | Why |
|---|---|---|
| Left-shifted ragged rows | Assign fields by value type (`parser.py`) | Position carries no information |
| Missing values (release metadata 38.9%, tariff-line ~52%) | **Flagged, not imputed** | Missingness is structural (all 1962-1987 records lack release data); imputing would invent values |
| Sizes in B/KB/MB/GB with thousands separators | Convert to MB, binary units | Comparable units. The original notebook missed `1,017.25 KB` and `671 B` (33 cells) |
| Duplicates | Dropped (none found) | |
| Outliers (bytes/record, Tukey fences per decade: 322 of 5,445, 5.9%) | **Flagged, retained** | They reflect a real post-2010 shift in per-record size |
| Feature engineering | `fileSize_MB`, `publication_lag_years`, exposure-adjusted lag ratio, bytes per record | Enable the analyses above |

## 5. Exploratory statistics (mean, SD, median, IQR)
| Variable | n | Mean | SD | Median | Q1-Q3 |
|---|---|---|---|---|---|
| File size (MB) | 8,916 | 5.82 | 27.00 | 0.72 | 0.22-2.36 |
| Total records | 5,445 | 541,771 | 1,825,555 | 117,078 | 46,325-279,331 |
| Tariff-line records | 4,309 | 460,792 | 963,739 | 136,649 | 46,463-395,889 |
| Tariff-line size (MB) | 4,307 | 7.69 | 16.68 | 2.30 | 0.74-6.42 |
| Lag (years, all) | 5,445 | 6.84 | 6.84 | 4.0 | 1-9 |

All variables are strongly right-skewed (SD >= mean), so medians describe a typical record better than means.

## 6. Statistical analysis and modelling (`tradeavail/stats.py`)
- **Growth:** log-linear fit 1988-2025, 12.0% per year (95% CI 10.9-13.1%), R^2 0.93.
- **H2 (bytes per record):** median 13.0 (SD 2.1) before 2010 vs 16.5 (SD 3.8) after; Mann-Whitney p ~ 1e-262, rank-biserial -0.54 (large).
- **H3 (lag by edition):** exposure-adjusted lag (lag / (2026 - year)), windows >= 5 years (n = 4,868); Kruskal-Wallis H = 135.6, p ~ 1e-27, epsilon^2 = 0.028 (small), non-monotone (H3 lowest 0.21, H5 highest 0.33). Kaplan-Meier is *not* used: every dated record has an observed value, so nothing is censored in the survival sense.
- **Model:** log10(file size) ~ log10(records), n = 5,445.

| Model | Evaluation | RMSE (log10) | R^2 | Median abs % error |
|---|---|---|---|---|
| Mean baseline | 5-fold CV | 0.712 | 0.00 | 81.1% |
| Records | 5-fold CV | 0.086 | 0.985 | 10.9% |
| Mean baseline | holdout (> 2010) | 0.894 | -0.50 | 78.2% |
| Records | holdout (> 2010) | 0.120 | 0.973 | 19.8% |
| Records + year | holdout (> 2010) | 0.099 | 0.982 | 14.5% |

## 7. Interactive dashboard
```bash
pip install -r requirements.txt
streamlit run app/dashboard.py
```
Filters for years, classification and reporters; tabs for classification regimes, volume and file size (with OML/OMH), lag, and a filtered-data download. A headless test (`tests/test_dashboard.py`) runs the app and applies a filter.

## 8. Limitations (what the data does *not* support)
- **Lag is to the latest release**, not first publication, and is exposure-biased; do not read it as timeliness trend.
- **Parser accuracy has no independent ground truth.** Checks: independent grammar (100% agreement), author-reviewed audit of 60 rows. A lone integer in the optional block would be assigned to `totalRecords` (never occurs in this export).
- **Statistical independence:** records cluster by reporter and year, so p-values are optimistic; read effect sizes and intervals. Random CV folds share reporters; the temporal holdout is more honest.
- **Metadata, not trade:** record growth reflects finer detail and more reporting, not necessarily more trade. The cause of the post-2010 per-record rise is unknown.
- No user study of OML/OMH; the assessment is analytical.
- 30 reporters tie at full coverage (64 records), so a "top-k reporters" ranking is arbitrary.

## 9. Recommendations and next steps
1. Have an independent annotator label ~200 rows to measure parser accuracy.
2. Obtain first-release dates (Comtrade release history) to measure true timeliness.
3. Join external reference data (income groups, regions) to test whether availability and timeliness differ by country group.
4. Snapshot the export monthly to track newly available reporter-years.
5. Deploy the dashboard (e.g. Streamlit Community Cloud) and run a small user study of OML/OMH versus log and linear charts.
6. For publishers: emit explicit empty cells so standard CSV tools parse the export correctly.

## 10. Reproduce
```bash
pip install -r requirements.txt
python scripts/run_pipeline.py --raw data/DataAvailability_9_21_2026_11_25_48.xlsx --out outputs   # clean CSV, summary.json, figures
python scripts/run_stats.py    --raw data/DataAvailability_9_21_2026_11_25_48.xlsx --out outputs   # tests, model, validation, audit sample
python -m pytest -q tests                                                                          # 14 tests
```
Place the raw workbook in `data/` (or a Latin-1 CSV conversion). `data/comtrade_availability_clean.csv` (the cleaned table) is included so the dashboard and regression tests run out of the box.

```
tradeavail/  parser.py cleaning.py analysis.py stats.py validate.py oml_omh.py
scripts/     run_pipeline.py run_stats.py
app/         dashboard.py
tests/       unit tests (synthetic rows), clean-dataset regression test, dashboard test
docs/        PROBLEM_STATEMENT.md, figures/, stats_summary.json, parser_audit_sample.csv
notebooks/   original exploratory notebook
```

## License
MIT (`LICENSE`). Comtrade data remain subject to UN terms of use.
