# Methodology

How the numbers in the story are calculated. All calculations are Tableau calculated fields in the workbook in `tableau/`.

## 1. Data model

| Source | Grain | Join key | Used for |
|---|---|---|---|
| Zillow Home Value Index (main table) | county × month, Jan 2000–Jul 2026 | FIPS, year, month | Mid-tier, bottom-tier and top-tier typical home values |
| `county_information.csv` | county | FIPS | Median household income, urban-rural classification, state |
| `economic_measures.csv` | month | year, month | Federal funds rate (context) |
| `mortgage_rates.csv` | month | year, month | 30-year fixed rate for the payment model |

**Data fix:** the rate files store two-digit years (`"00"`), so `Mortgage Year = [year] + 2000` aligns them with the main table.

## 2. Comparison windows

| Window | From | To |
|---|---|---|
| Pandemic boom | Jan 2020 | Jun 2022 |
| Aftermath / rate shock | Jun 2022 | Jun 2026 |

June 2022 is used as the split because it is just after the Fed's first hike (Mar 16, 2022) and roughly where county values peaked.

## 3. Anchor values (FIXED LOD)

Each county's value at each anchor month is pinned with a `FIXED` expression so it stays the same whatever else is in the view:

```
Jan 2020 Value = { FIXED [FIPS] : MAX(IF [year] = 2020 AND [month] = 1 THEN [allhomes_median_midtier] END) }
Jun 2022 Value = { FIXED [FIPS] : MAX(IF [year] = 2022 AND [month] = 6 THEN [allhomes_median_midtier] END) }
Jun 2026 Value = { FIXED [FIPS] : MAX(IF [year] = 2026 AND [month] = 6 THEN [allhomes_median_midtier] END) }

Pandemic Boom Growth = ([Jun 2022 Value] - [Jan 2020 Value]) / [Jan 2020 Value]
Post-2022 Growth     = ([Jun 2026 Value] - [Jun 2022 Value]) / [Jun 2022 Value]
```

## 4. Market segments

Counties are split at the **median county** on each axis (boom median ≈ 35.6%, post-2022 median ≈ 9.8%):

| | Post-2022 growth ≥ median | Post-2022 growth < median |
|---|---|---|
| **Boom growth ≥ median** | Resilient Growth | Boom & Correction |
| **Boom growth < median** | Emerging Momentum | Persistent Lag |

```
IF ISNULL([Pandemic Boom Growth]) OR ISNULL([Post-2022 Growth]) THEN NULL
ELSEIF [Pandemic Boom Growth] >= { FIXED : MEDIAN([Pandemic Boom Growth]) }
   AND [Post-2022 Growth]    >= { FIXED : MEDIAN([Post-2022 Growth]) } THEN "Resilient Growth"
...
```

**Null check:** the first version had no `ISNULL` branch, so 67 counties with no values fell through to "Persistent Lag". The segment counts didn't add up, which exposed the problem. Those counties are now excluded, and the dashboard footer says so.

## 5. Estimated monthly payment

Standard amortization for principal and interest (P&I) on a typical home, with 20% down and a 30-year fixed loan at that month's average rate:

```
Monthly Mortgage Rate = [us_frm_30yr] / 100 / 12
Estimated Loan Amount = [allhomes_median_midtier] * 0.80
Monthly payment (P&I) = Loan * (r * (1 + r)^360) / ((1 + r)^360 - 1)
```

Taxes, insurance and HOA fees are left out on purpose, so the model isolates the two variables in the dataset: price and rate.

## 6. Price vs. rate decomposition

The median payment rose from **$582 (Jan 2020)** to **$1,197 (Jun 2026)**, an increase of $615. How much of that comes from each driver depends on which one is changed first:

| Order | Rate share of increase |
|---|---|
| Change prices first, then rates | ≈ 36% |
| Change rates first, then prices | ≈ 54% |

Rather than pick the more favorable number, the dashboard **averages both orderings** (a two-factor Shapley split). That gives **≈ 55% prices ($337) and ≈ 45% rates ($278)**, and the method is stated on the dashboard.

## 7. Affordability

```
Payment-to-Income (Jun 2026) = (Monthly P&I × 12) / [median_hh_income]
```

States are compared against the **30% cost-burden line** (Congressional Research Service, 2024). The state value is the median across its counties. States with fewer than 10 counties are marked with `*`, and Connecticut has no income data in the source.

## 8. Tile map

The equal-area state grid comes from two hand-coded fields, `Tile X` and `Tile Y` (a `CASE` on state). The `Show Segment` parameter drives:

```
Share of counties in segment =
  COUNTD(IF [Market Segment] = [Show Segment] THEN [FIPS] END) /
  COUNTD(IF NOT ISNULL([Market Segment]) THEN [FIPS] END)
```
