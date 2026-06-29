# Project Briefing: Private Credit Dissertation Panel Data Pipeline

## What this project is

This is the data pipeline for a master's dissertation titled:
**"Private Credit in Emerging Markets: Global Drivers, Private Equity Linkages, and Implications for South Africa's Market Readiness"**

The empirical chapter replicates and extends the panel regression model from:
**Avalos, Doerr and Pinter (2025), "Global Drivers of Private Credit", BIS Quarterly Review, March 2025.**

The goal is to build a clean, merged, quarterly panel dataset across 44 countries from Q1 2010 to Q4 2024 (or Q4 2023 if 2024 data coverage is materially incomplete), run a panel OLS regression with country fixed effects and two-way clustered standard errors, and then add three extension variables beyond the BIS baseline.

---

## The regression model

### Equation

ln(PC)_ct = B * X_ct + controls_ct + country_FE_c + error_ct

- **c** = country
- **t** = quarter
- **ln(PC)** = natural log of total newly originated private credit deal value in USD, by borrower country and quarter (from PitchBook)
- **X_ct** = vector of main regressors (see below)
- **country_FE_c** = country fixed effects (absorb all time-invariant country-level differences)
- Standard errors are clustered two-ways: by country and by quarter

### Baseline regressors (BIS replication)

| Variable | Description | Source file | Frequency |
|---|---|---|---|
| `policy_rate` | End-of-quarter central bank policy rate | `bis_policy_rates.csv` | Quarterly |
| `fi_index` | IMF Financial Institutions index (sub-index of Financial Development Index) | `imf_fi_index.csv` | Annual |
| `cap_adequacy` | Regulatory capital to risk-weighted assets (% of RWA) | `imf_car.csv` | Annual (or quarterly where available) |
| `nfc_debt_gdp` | Non-financial corporate debt as % of GDP (loans + debt securities) | `imf_nfc_debt.csv` or `bis_total_credit.csv` | Annual |

### Extension regressors (beyond BIS)

| Variable | Description | Source file | Frequency |
|---|---|---|---|
| `pe_deal_activity` | Private equity deal value as % of GDP by borrower country | `pitchbook_pe_deals.csv` | Quarterly |
| `fx_volatility` | Rolling quarterly standard deviation of bilateral USD exchange rate | `imf_ifs_fx.csv` | Constructed from monthly data |
| `fdi_inflows` | Foreign direct investment inflows as % of GDP | `unctad_fdi.csv` or `wb_wdi.csv` | Annual |

### Control variables

| Variable | Description | Source file | Frequency |
|---|---|---|---|
| `log_population` | Natural log of total population | `wb_wdi.csv` | Annual |
| `log_gdp_per_capita` | Natural log of GDP per capita in USD | `wb_wdi.csv` | Annual |
| `gdp_growth` | Real GDP growth rate | `wb_wdi.csv` | Annual |
| `inflation` | CPI inflation rate | `imf_ifs_inflation.csv` | Quarterly preferred, else annual |

---

## Panel structure rules

- **Unit of observation**: one row per country per quarter (e.g. ZAF, 2015Q3)
- **Time identifier**: use a quarter string format like `2015Q3` or a date like `2015-09-30` consistently throughout
- **Country identifier**: ISO3 alpha code (e.g. ZAF, USA, GBR) throughout all files. This is the merge key.
- **Annual variables**: repeat the annual value across all four quarters of that year. This is consistent with the BIS paper methodology. Do not interpolate.
- **Quarterly variables**: use the value for that specific quarter
- **FX volatility**: compute as the standard deviation of monthly bilateral USD exchange rate observations within each quarter (requires 3 monthly observations per quarter-country cell)

---

## Key methodological decisions already made

- The BIS baseline uses a World Bank Bank Regulation and Supervision Survey index for bank regulatory stringency. This is excluded from our model. Reason: the survey is conducted every 4-5 years only, creating very sparse time variation. BIS footnote 10 confirms results are robust to its exclusion.
- In place of the World Bank survey index, we use the IMF capital adequacy ratio (CAR) from the Financial Soundness Indicators database. This substitution is supported by Irani et al. (2021), Gropp et al. (2019), and Bednarek et al. (2025), who link bank capital requirements to lending displacement toward non-bank lenders.
- The BIS paper sample runs Q1 2010 to Q4 2019. Our extension runs to Q4 2024 (or Q4 2023 depending on coverage). We report results on both windows.
- The dependent variable (private credit originations) comes from PitchBook's Debt and Lenders screener, filtered to direct lending deals, aggregated by borrower country and quarter. It has already been pulled and pivoted.

---

## File naming convention

All raw data files go in the `/data/raw/` folder. All cleaned files go in `/data/cleaned/`. The final merged panel goes in `/data/panel/`.

| File | Contents |
|---|---|
| `pitchbook_private_credit.csv` | Quarterly private credit originations by borrower country |
| `pitchbook_pe_deals.csv` | Quarterly PE deal activity by borrower country |
| `bis_policy_rates.csv` | End-of-quarter central bank policy rates |
| `imf_fi_index.csv` | IMF Financial Development Index, FI sub-index |
| `imf_car.csv` | IMF Financial Soundness Indicators, capital adequacy ratio |
| `imf_nfc_debt.csv` | NFC debt as % of GDP |
| `imf_ifs_inflation.csv` | CPI inflation |
| `imf_ifs_fx.csv` | Monthly bilateral exchange rates for FX volatility construction |
| `wb_wdi.csv` | World Bank WDI: population, GDP per capita, GDP growth, FDI |
| `unctad_fdi.csv` | FDI inflows by country (if not using WDI) |

---

## Country list

The panel covers 44 countries. The country list is in `country_list.xlsx` or `country_list.csv` in the working directory. Always filter to these countries using ISO3 codes. Do not add or remove countries without being told to.

If a source file uses country names rather than ISO3 codes, map them using the pycountry library or a manual mapping table. Flag any countries where the name mapping is ambiguous.

---

## Coverage and missing data rules

- If a country-quarter observation is missing a control variable value but has private credit data, flag it rather than silently dropping it. The decision on whether to drop or fill is a human judgment call.
- For annual variables with a one-year lag in publication (e.g. WDI), if the most recent year is missing for some countries, do not forward-fill from the prior year without flagging it explicitly.
- IMF CAR data is known to have patchy coverage for emerging market countries before 2015. Document the coverage rate (% of country-quarters with non-missing CAR) in a summary output.
- If 2024 annual data is missing for more than 20% of countries on any variable, flag this and ask before deciding whether to truncate the panel at Q4 2023.

---

## Current task

We are currently in the data cleaning phase. The first file being cleaned is the IMF Financial Development Index dataset (`imf_fi_index.csv`). This dataset covers over 180 countries from 1980 onward and contains nine sub-indices. We need only:

- The **FI index** column (financial institutions aggregate sub-index), not the full FD index or the FM (financial markets) sub-index
- The 44 countries in the country list
- Years 2010 to 2024 (or 2025/2026 if present)

The cleaned output should be in long format with columns: `iso3`, `year`, `fi_index`. It will later be merged into the panel and repeated across all four quarters of each year.

---

## Important notes for code quality

- Use pandas for all data manipulation
- Use linearmodels (Python) for the panel regression with fixed effects and clustered standard errors
- Print a summary of observations, countries, and date range after every cleaning step
- Print missing value counts by country and year for every variable after cleaning
- Save cleaned files to `/data/cleaned/` with descriptive filenames
- Keep all scripts modular: one script per data source, one script for the merge, one script for the regression
- Comment every non-obvious step
