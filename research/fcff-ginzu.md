# FCFF simple Ginzu (`fcffsimpleginzu.xlsx`), taken apart

Research for #3.

- **Source:** https://pages.stern.nyu.edu/~adamodar/pc/fcffsimpleginzu.xlsx, linked from the spreadsheets page https://pages.stern.nyu.edu/~adamodar/New_Home_Page/spreadsh.htm (section `#ginzumodels`). Companion guide: https://pages.stern.nyu.edu/~adamodar/pptfiles/SpreadsheetGuides/fcffsimpleginzuguide.pptx (not read).
- **Version read:** HTTP `Last-Modified: Sun, 12 Apr 2026 16:20:41 GMT`, sha256 `d6ffb67d965dc22463e4d013636befab3da431e7fc608936a82612fa73198e72`. The ERP sheet says "Updated January 1, 2026".
- **Method:** I loaded the workbook twice with openpyxl, once for formulas and once for cached values (`data_only=True`). Every formula and number below comes from that dump.
- **Example company:** Almarai (Saudi Arabia, Food Processing), valuation date 2026-02-01. The file does not say what units the numbers are in. The output is **$7.19/share against a price of 72.28**, which is very likely a unit mismatch in the template's sample data (share count vs. financials). It is still the right golden fixture, because it is exactly what Excel computes.
- **Fixture:** `research/fcff-ginzu-fixture.json` holds the inputs, every row for years 1-10 plus the terminal year, the bridge to equity and the option-sheet values. A separate Python reimplementation (formulas below, all switches set to No) gives value/share = 7.18784027006211, which matches the cell to within 1e-9.

## Sheets

The main path runs through `Input sheet` → `Valuation output`. Supporting sheets:

- `Cost of capital worksheet`: feeds B35.
- `Option value`: dilution-adjusted Black-Scholes.
- `Operating lease converter` and `R& D converter`: only used when their switches are on.
- `Synthetic rating`: rating → spread table.
- `Country equity risk premiums`: B1 = mature-market ERP, 4.23%.
- `Industry Averages(US)` / `(Global)` and `Input Stat Distributioons`: benchmark lookups only.
- `Failure Rate worksheet`: helper for choosing the probability of failure.
- `Diagnostics`, `Stories to Numbers`, `Valuation as picture`: presentation only.
- `Trailing 12 month Worskheet`, `Answer keys`.

Sheet names keep Damodaran's typos ("Distributioons", "Worskheet") because these are the actual names.

The workbook needs **iterative calculation** turned on (Input C3/C4). In practice the circularity comes from `Option value!B17`, where the adjusted S depends on the option value.

## 1. Inputs (`Input sheet`)

| Cell | Name | Meaning | Example |
|---|---|---|---|
| B3 | Valuation date | | 2026-02-01 |
| B4 | Company | | Almarai |
| B7 | Country of incorporation | ERP lookup if chosen | Saudi Arabia |
| B8 / B9 | Industry US / Global | benchmark + unlevered beta lookups | Food Processing |
| B11 / C11 | Revenues LTM / last 10K | | 21765.4 / 20979.5 |
| D11 | Years since last 10K | annualises the growth shown at I25 | 0.5 |
| B12 / C12 | EBIT LTM / 10K | | 3060.9 / 3064.1 |
| B13 / C13 | Interest expense | feeds cost of debt | 493.4 / 562.7 |
| B14 / C14 | Book equity | | 10667.8 / 13111.1 |
| B15 / C15 | Book debt | | 45063 / 44544 |
| B16 | Capitalise R&D? | Yes/No | No |
| B17 | Operating leases to convert? | Yes/No | No |
| B18 / C18 | Cash & marketable securities | | 19000 / 13663 |
| B19 / C19 | Cross holdings / non-operating assets | | 21119 / 21644 |
| B20 / C20 | Minority interests | | 1558 / 1539 |
| B21 | Shares outstanding | | 4315 |
| B22 | Price | | 72.28 |
| B23 | Effective tax rate | | 0.175 |
| B24 | Marginal tax rate | | 0.25 |
| B26 | Revenue growth, year 1 | | 0.05 |
| B27 | Operating margin, year 1 | default `=I26` (current margin) | 0.140631 |
| B28 | CAGR, years 2-5 | default `=B26` | 0.05 |
| B29 | Target pre-tax operating margin | default `=B27` | 0.140631 |
| B30 | Year of margin convergence | | 5 |
| B31 | Sales/capital, years 1-5 | default `=K27` (global industry) | 1.708538 |
| B32 | Sales/capital, years 6-10 | default `=B31` | 1.708538 |
| B34 | Risk-free rate | | 0.0458 |
| B35 | Initial cost of capital | `='Cost of capital worksheet'!B13` (do not type over it) | 0.0705502 |
| B37 | Employee options? | Yes/No | No |
| B38 | Number of options | | 7.72 |
| B39 | Average strike price | | 1.29 |
| B40 | Average maturity, years | | 7 |
| B41 | Stock price standard deviation | | 0.45 |

The benchmark block (I25:N31) is display-only:

- I25 annualised growth: `(B11/C11)^(1/D11)-1` = 0.076324.
- I26 margin = 0.140631.
- I27 sales/capital `= B11 / IC0` = 0.592565.
- I29 ROIC = 0.06875.

## 2. Override switches ("Default assumptions", A43:B72)

Every switch defaults to `"No"`. The value next to each switch is only used when the switch is `"Yes"`.

| Switch | Value cell(s) | Default behaviour → override |
|---|---|---|
| B45 terminal cost of capital | B46 (=B35) | WACC_T = rf + mature ERP → B46 |
| B48 terminal ROIC | B49 (0.15) | ROIC_T = WACC_T → B49 |
| B51 failure probability | B52 (0.12), B53 basis `B`/`V` ("V"), B54 proceeds % (0.5) | p = 0 → B52 |
| B56 growth lag | B57 (1; options 0-3) | reinvestment for year t is driven by revenue growth in year t+1 → lag of B57 years |
| B59 tax-rate convergence | — | effective rate moves to marginal rate by the terminal year → stays at effective rate |
| B61 NOL | B62 (=474.8+256.6) | NOL = 0 → B62 |
| B64 risk-free rate after year 10 | B65 (0.02) | rf stays as today's → B65 (used in g and WACC_T) |
| B67 perpetual growth | B68 (-0.05) | g = rf → B68 |
| B70 trapped cash | B71 (140000), B72 (0.15) | cash taken at face value → cash − B71·(B24 − B72) |

Quirk: B27 checks `="YES"` while the other switches check `="Yes"`. Excel string comparison ignores case, so both work. A port must also compare case-insensitively.

## 3. Calculation chain (`Valuation output`)

Columns: C..L = years 1..10, M = terminal year, B = base year. Notation below: t = year, gT = M2, WACC_T = M12.

**Revenue growth (row 2)**
- g1 = B26.
- g2..g5 = B28.
- t = 6..10: g_t = g5 − (g5 − gT)·(t−5)/5.
- gT = B68 if B67 = Yes, else B65 if B64 = Yes, else B34.

**Revenue (row 3)**
- R0 = B11.
- R_t = R_{t−1}(1+g_t).
- Terminal: R_T = R10(1+gT).

**Margin (row 4)**
- m1 = B27.
- For t > B30: m_t = B29.
- Otherwise: m_t = B29 − (B29 − m1)/B30 · (B30 − t).
- m_T = m10.

**EBIT (row 5)**
- B5 = B12, plus lease adjustment `'Operating lease converter'!F32` if B17 = Yes, plus R&D adjustment `'R& D converter'!D39` if B16 = Yes.
- EBIT_t = m_t · R_t.

**Tax rate (row 6)**
- t1..5 = B23.
- t6..10 step linearly by (tT − t5)/5.
- tT = B23 if B59 = Yes, else B24.

**EBIT(1−t) (row 7)**
- Base year: EBIT(1−t) if EBIT > 0, else EBIT.
- Years 1-10, with the NOL shield from the prior year:
  - if EBIT ≤ 0: EBIT
  - else if EBIT < NOL_{t−1}: EBIT (no tax)
  - else: EBIT − (EBIT − NOL_{t−1})·tax_t
- Terminal: EBIT_T(1 − tT), with no NOL.

**NOL (row 10)**
- NOL0 = B62 if B61 = Yes, else 0.
- NOL_t = NOL_{t−1} − EBIT_t if EBIT_t < 0 or NOL_{t−1} > EBIT_t, else 0.

**Reinvestment (row 8)**, default lag of 1:
- Reinv_t = (R_{t+1} − R_t)/S2C_t. Year 10 uses R_T.
- With the lag override: lag 0 uses (R_t − R_{t−1}); lag 2/3 shifts forward. The tail years use R·g extrapolation (see K8/L8).
- S2C_t (row 38) is B31 for years 1-5 and B32 for years 6-10.

**Terminal reinvestment (M8)**
- M8 = gT/ROIC_T · EBIT_T(1−t) if gT > 0, else 0.
- ROIC_T (M40) = B49 if B48 = Yes, else L12 (≈ WACC_T).

**FCFF (row 9)**: FCFF_t = EBIT(1−t)_t − Reinv_t.

**Cost of capital (row 12)**
- Years 1-5 = B35.
- Years 6-10 step linearly to WACC_T.
- WACC_T (M12) = B46 if B45 = Yes; else (B65 if B64 = Yes, else B34) + `'Country equity risk premiums'!B1`.

**Discount factor (row 13)**: DF_t = Π_{k≤t} 1/(1+WACC_k). The discount rate changes year by year, so the factor is a cumulative product rather than a single exponent.

**PV (row 14)**: PV_t = FCFF_t · DF_t.

**Invested capital and ROIC (rows 39-40)**
- IC0 = B14 + B15 − B18, plus lease debt (F33) and the R&D asset (D35) when those switches are on.
- IC_t = IC_{t−1} + Reinv_t.
- ROIC_t = EBIT(1−t)_t / IC_{t−1}. This is display only.

**Terminal value and value of operating assets**
- B16 = TCF = FCFF_T.
- B18 = TV = TCF/(WACC_T − gT).
- B19 = TV · DF10.
- B20 = ΣPV_t.
- B21 = B19 + B20.

**Failure probability**
- B22: p = B52 if B51 = Yes, else 0.
- B23 proceeds: (B14 + B15)·B54 if B53 = "B", otherwise B21·B54.
- B24 operating assets = B21(1−p) + B23·p.

**Bridge from operating assets to equity**
- B25 debt = B15, plus lease debt `'Operating lease converter'!C28` if B17 = Yes.
- B26 minority interests = B20.
- B27 cash = B18, or B18 − B71·(B24 − B72) if trapped cash is on.
- B28 non-operating assets = B19.
- B29 equity = B24 − B25 − B26 + B27 + B28.
- B30 options = `'Option value'!B29` if B37 = Yes, else 0.
- B31 = B29 − B30.
- **B33 value/share = B31/B21.**
- B35 = price / value.

### Options (`Option value`): dilution-adjusted Black-Scholes

Inputs:
- S = price, K = B39, T = B40, σ = B41.
- Dividend yield y = 0 (B8 is hard-coded).
- r = B34, n = options, N = shares.

Formulas:
- S* = (S·N + V_opt·n)/(N + n). This is circular.
- d1 = [ln(S*/K) + (r − y + σ²/2)T]/(σ√T).
- d2 = d1 − σ√T.
- V_opt = e^{−yT}·S*·N(d1) − K·e^{−rT}·N(d2).
- Total = V_opt · n.

With the example inputs: S* = 72.2783, d1 = 4.2460, d2 = 3.0554, V_opt = 71.3424, total = 550.76. The total is not used in the example because B37 = No.

### Cost of capital (`Cost of capital worksheet`, B13)

B11 selects the method: `I will input` (B12), `Detailed` (E62), `Industry Average` (B67), or the histogram method (B72). The example uses **Detailed**.

**Unlevered beta (B23)**
- Taken from the US industry table, column 7 (`Single Business(US)`), or a multi-business or global variant.
- The example gives about 0.4694. This is derived from the levered beta; the cached value of B23 itself was blank.

**Equity risk premium (B27)**
- Comes from the operating-regions calculator: revenue-weighted ERP = 0.0558081 (K32).

**Cost of debt (B37)**
- rf plus the spread for the actual rating `A1/A+` from `Synthetic rating!G42:H56` = 0.052803, so the spread is 0.7003%.
- The method can instead be direct input or a synthetic rating.

**Market value of debt (C53)**
- I·[1−(1+kd)^−M]/kd + D/(1+kd)^M with M = B32 = 3, giving 39953.64.

**Levered beta (C57)**
- βu·(1 + (1−t)·D/E), with t = the marginal rate, giving 0.514525.

**Weights (row 61)**
- E = shares × price = 311888.2.
- Weight of equity = 0.886444; weight of debt = 0.113556.

**Component costs (row 62)**
- ke = rf + βL·ERP = 0.0745147.
- kd(1−t) = 0.0396023.
- WACC (E62) = **0.0705502**.

## 4. Example numbers (golden values)

| Year | g | Revenue | Margin | EBIT | Tax | EBIT(1−t) | Reinv | FCFF | WACC | DF | PV |
|---|---|---|---|---|---|---|---|---|---|---|---|
| base | | 21765.4 | 0.140631 | 3060.9 | 0.175 | 2525.2425 | | | | | |
| 1 | 0.05 | 22853.67 | 0.140631 | 3213.945 | 0.175 | 2651.5046 | 668.8079 | 1982.6967 | 0.070550 | 0.934099 | 1852.0353 |
| 2 | 0.05 | 23996.3535 | " | 3374.6423 | 0.175 | 2784.0799 | 702.2483 | 2081.8316 | 0.070550 | 0.872541 | 1816.4839 |
| 3 | 0.05 | 25196.1712 | " | 3543.3744 | 0.175 | 2923.2838 | 737.3607 | 2185.9231 | 0.070550 | 0.815040 | 1781.6149 |
| 4 | 0.05 | 26455.9797 | " | 3720.5431 | 0.175 | 3069.4480 | 774.2288 | 2295.2193 | 0.070550 | 0.761328 | 1747.4152 |
| 5 | 0.05 | 27778.7787 | " | 3906.5702 | 0.175 | 3222.9204 | 799.2828 | 2423.6377 | 0.070550 | 0.711156 | 1723.5845 |
| 6 | 0.04916 | 29144.3835 | " | 4098.6172 | 0.19 | 3319.8800 | 824.2467 | 2495.6332 | 0.074060 | 0.662119 | 1652.4071 |
| 7 | 0.04832 | 30552.6401 | " | 4296.6624 | 0.205 | 3415.8466 | 849.0532 | 2566.7934 | 0.077570 | 0.614456 | 1577.1816 |
| 8 | 0.04748 | 32003.2794 | " | 4500.6679 | 0.22 | 3510.5210 | 873.6319 | 2636.8891 | 0.081080 | 0.568372 | 1498.7348 |
| 9 | 0.04664 | 33495.9124 | " | 4710.5791 | 0.235 | 3603.5930 | 897.9098 | 2705.6832 | 0.084590 | 0.524043 | 1417.8956 |
| 10 | 0.0458 | 35030.0252 | " | 4926.3236 | 0.25 | 3694.7427 | 939.0341 | 2755.7086 | 0.0881 | 0.481613 | 1327.1860 |
| T | 0.0458 | 36634.4003 | " | 5151.9492 | 0.25 | 3863.9619 | 2008.7339 | 1855.2280 | 0.0881 | | |

Terminal and equity bridge (cell: value):

- B18 TV: 43858.8187
- B19 PV(TV): 21122.9918
- B20 ΣPV: 16394.5389
- B21: 37517.5308
- B22 p: 0
- B23 proceeds: 18758.7654 (computed, not used because p = 0)
- B24 operating assets: 37517.5308
- B25 debt: 45063
- B26 minority interests: 1558
- B27 cash: 19000
- B28 non-operating assets: 21119
- B29 equity: 31015.5308
- B30 options: 0
- B31 common equity: 31015.5308
- B32 shares: 4315
- **B33 value/share: 7.187840270062114**
- B35 price/value: 10.0559

Implied quantities:

- IC0 = 36730.8, IC10 = 44796.6042.
- ROIC: base 0.06875, year 10 0.084244, terminal 0.0881.
- Margin is flat at 0.140631 in every year because the defaults set target margin = year-1 margin = current margin.

Full-precision values for every row are in the fixture JSON.

## Gotchas for a port

- The fixture only exercises the all-"No" path. Each switch needs its own fixture: change the value in Excel/LibreOffice and record the cached outputs.
- Growth and WACC in years 6-10 interpolate linearly toward the terminal value in 5 steps. The tax rate does the same, starting from year 5's rate.
- Terminal ROIC defaults to **L12** (year-10 WACC), which equals WACC_T because the year-10 rate has fully converged.
- Reinvestment is *forward-looking*: reinvestment in year t funds revenue growth in year t+1.
- The NOL check uses the *previous* year's NOL balance.
- Failure proceeds tied to "V" use the pre-failure B21 value.
- Trapped cash is only subtracted when B70 is exactly `"YES"` in uppercase in the formula. Excel comparison is case-insensitive, so "Yes" works too.
