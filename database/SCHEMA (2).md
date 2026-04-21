# Database Schema: european_trade.db

## Overview

Three tables linked by `iso3` (ISO 3166-1 alpha-3 country code) as the common key. The database captures European bilateral trade flows alongside country-level economic indicators for 50 European countries/territories, covering 2019–2023.

```
countries (50 rows)         — Reference: geography & identifiers
    │ iso3 (PK)
    │
    ├──→ nodes (205 rows)   — Country-year economic indicators
    │     iso3 (FK), year
    │
    └──→ edges (14,938 rows) — Bilateral trade flows
          reporter_iso3 (FK), partner_iso3 (FK), period, flow_code
```

---

## Table 1: countries (50 rows)

Master reference table for all 50 European countries/territories.

| Column      | Type | Constraints | Description                         |
|-------------|------|-------------|-------------------------------------|
| iso3        | TEXT | PK, UNIQUE  | ISO3 country code (e.g. DEU)        |
| countryname | TEXT |             | Full country name (e.g. Germany)    |
| capital     | TEXT |             | Capital city                        |
| lat         | REAL |             | Latitude of capital                 |
| lon         | REAL |             | Longitude of capital                |
| comtradenum | INT  | UNIQUE      | UN Comtrade numeric code (e.g. 276) |

**Missing values:** None — all 50 rows and all columns are complete.

---

## Table 2: nodes (205 rows)

Country-level economic indicators by year. Each row = one country × one year.

| Column          | Type | Constraints  | Description              |
|-----------------|------|--------------|--------------------------|
| iso3            | TEXT | FK→countries | ISO3 country code        |
| year            | INT  |              | Year (2019–2023)         |
| gdpusd          | REAL |              | GDP in current USD       |
| population      | REAL |              | Population count         |
| tradepctgdp     | REAL |              | Trade as % of GDP        |
| fdiinflowpctgdp | REAL |              | FDI inflow as % of GDP   |

**Missing values:** None within the 205 rows present.

**Missing countries (9):** The following countries appear in `countries` but have NO rows in `nodes` — World Bank indicators are unavailable for these small territories:

| iso3 | Country          |
|------|------------------|
| AND  | Andorra          |
| CHI  | Channel Islands  |
| FRO  | Faroe Islands    |
| GIB  | Gibraltar        |
| GRL  | Greenland        |
| IMN  | Isle of Man      |
| LIE  | Liechtenstein    |
| MCO  | Monaco           |
| SMR  | San Marino       |

**Year coverage:** 41 countries × 5 years (2019–2023) = 205 rows. All 41 countries present have complete 5-year coverage.

---

## Table 3: edges (14,938 rows)

Aggregated bilateral trade flows between European countries. Each row = one directed trade relationship (reporter → partner) for a given year and flow direction.

| Column          | Type | Constraints  | Description                                          |
|-----------------|------|--------------|------------------------------------------------------|
| reporter_iso3   | TEXT | FK→countries | Reporter country ISO3 code                           |
| reporter_name   | TEXT |              | Reporter country name                                |
| partner_iso3    | TEXT | FK→countries | Partner country ISO3 code                            |
| partner_name    | TEXT |              | Partner country name                                 |
| period          | TEXT |              | Year (2019–2023)                                     |
| flow_code       | TEXT |              | M = Import, X = Export                               |
| total_trade_usd | REAL |              | Total trade value in USD (CIF for imports, FOB for exports) |

**Edge interpretation for graph/network analysis:**
- Each row is a **directed edge**: reporter_iso3 → partner_iso3
- `flow_code = 'X'`: reporter EXPORTS TO partner (edge weight = FOB value)
- `flow_code = 'M'`: reporter IMPORTS FROM partner (edge weight = CIF value)
- `total_trade_usd` = edge weight

**Missing values:** None — all 14,938 rows and all 7 columns are complete.

**Year and flow coverage:**

| Year | Flow | Rows  | Reporter countries | Partner countries |
|------|------|-------|--------------------|-------------------|
| 2019 | M    | 1,532 | 38                 | 42                |
| 2019 | X    | 1,524 | 38                 | 42                |
| 2020 | M    | 1,533 | 38                 | 42                |
| 2020 | X    | 1,519 | 38                 | 42                |
| 2021 | M    | 1,540 | 38                 | 42                |
| 2021 | X    | 1,521 | 38                 | 42                |
| 2022 | M    | 1,431 | 36                 | 42                |
| 2022 | X    | 1,427 | 36                 | 42                |
| 2023 | M    | 1,468 | 36                 | 42                |
| 2023 | X    | 1,443 | 36                 | 42                |

**Country coverage notes:**
- 38 countries report as source in 2019–2021; drops to 36 in 2022–2023
- 42 countries appear as trade partners across all years
- 12 countries in `countries` never appear in `edges` as reporters (small territories with no reported trade data)

---

## Relationships

```
countries.iso3  ←──  nodes.iso3           (1:many — one country, many years)
countries.iso3  ←──  edges.reporter_iso3  (1:many — one country, many trade edges)
countries.iso3  ←──  edges.partner_iso3   (1:many — one country, many trade edges)
```

---

## Data Sources

| Table     | Source                              | Details                                                |
|-----------|-------------------------------------|--------------------------------------------------------|
| countries | European country reference list     | 50 countries/territories with capital coordinates      |
| nodes     | World Bank Open Data                | GDP (current USD), population, trade % of GDP, FDI inflow % of GDP |
| edges     | UN Comtrade API v1                  | Annual bilateral commodity trade, all HS codes aggregated to TOTAL  |

---

## Known Limitations

1. **9 countries missing from nodes**: Small territories (Andorra, Channel Islands, Faroe Islands, Gibraltar, Greenland, Isle of Man, Liechtenstein, Monaco, San Marino) lack World Bank economic indicators. These countries exist in `countries` and may appear in `edges` as partners, but have no GDP/population data.

2. **12 countries never report in edges**: Some countries in `countries` never appear as `reporter_iso3` in `edges`. They may appear as `partner_iso3` only (i.e., other countries report trading with them, but they don't report themselves).

3. **Reporter count drops in 2022–2023**: 38 reporters in 2019–2021 vs 36 in 2022–2023 — two countries stopped reporting or have delayed data submission.

4. **Trade values are aggregated**: Each edge row sums all HS commodity codes into a single `total_trade_usd`. Commodity-level breakdown is not included in this schema.

5. **Potential mirror asymmetry**: Country A's reported exports to B may not match Country B's reported imports from A, due to different valuation methods (FOB vs CIF), timing, and classification differences.
