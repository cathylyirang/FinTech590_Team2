# Global Country Network Dataset — Project Roadmap
**Scope:** Trade flows + Migration flows (multi-relational graph)  
**Timeline:** 7 weeks  
**Deliverables:** Cleaned node/edge tables, documented schema, data pipeline, example queries & visualizations

---

## Overview

This project constructs a graph-based dataset of bilateral economic relationships between countries. Each country is a **node**; each bilateral relationship (trade or migration) is a **directed, weighted edge**. The final dataset must support both relational queries (SQL-style) and graph-based analysis (NetworkX, Neo4j, etc.).

---

## Data Sources

| Layer | Source | URL | Format | Frequency |
|---|---|---|---|---|
| Trade flows | CEPII BACI | http://www.cepii.fr/CEPII/en/bdd_modele/bdd_modele_item.asp?id=37 | CSV | Annual |
| Migration stocks | UN DESA Bilateral Matrix | https://www.un.org/development/desa/pd/data/international-migration-flows | CSV/XLSX | Every 5 years |
| Country metadata | World Bank WDI | https://databank.worldbank.org/source/world-development-indicators | CSV | Annual |
| Country codes reference | ISO 3166 / CEPII GeoDist | http://www.cepii.fr/CEPII/en/bdd_modele/bdd_modele_item.asp?id=6 | CSV | Static |

---

## Schema

### Node Table — `countries`

| Column | Type | Description |
|---|---|---|
| `country_id` | VARCHAR(3) | ISO 3166-1 alpha-3 code (primary key) |
| `country_name` | TEXT | Standard English name |
| `iso2` | VARCHAR(2) | ISO 3166-1 alpha-2 code |
| `region` | TEXT | World Bank region classification |
| `income_group` | TEXT | World Bank income group |
| `year` | INT | Reference year for metadata |

### Edge Table — `edges`

| Column | Type | Description |
|---|---|---|
| `source_id` | VARCHAR(3) | Exporting / origin country (ISO3) |
| `target_id` | VARCHAR(3) | Importing / destination country (ISO3) |
| `relationship_type` | TEXT | `"trade"` or `"migration"` |
| `year` | INT | Reference year |
| `value` | FLOAT | USD (trade) or person count (migration) |
| `unit` | TEXT | `"USD"` or `"persons"` |
| `value_normalized` | FLOAT | Value normalized by source country GDP |
| `data_quality` | TEXT | `"reported"`, `"estimated"`, `"missing"` |
| `source_dataset` | TEXT | e.g. `"BACI_2020"`, `"UNDESA_2020"` |

> **Note on directionality:** All edges are directed. For trade: `source → target` means source *exports to* target. For migration: `source → target` means migrants *move from* source *to* target.

---

## Week-by-Week Plan

### Week 1 — Environment Setup & Country Reference Table
**Goal:** Establish the project foundation before touching any real data.

- [ ] Initialize git repository with a clear folder structure (see below)
- [ ] Set up Python environment (`pandas`, `networkx`, `sqlalchemy`, `matplotlib`, `jupyter`)
- [ ] Download ISO 3166 country code list
- [ ] Download CEPII GeoDist for supplementary country metadata
- [ ] Download World Bank country classification file (region, income group)
- [ ] **Build the master country reference table** — merge ISO codes, World Bank classifications into a single `countries.csv`. This table is the backbone of everything else; all edge data will be joined against it.
- [ ] Document any country code conflicts or ambiguities (e.g., Kosovo, Taiwan, historical states)

**Output:** `data/reference/countries.csv`, `docs/country_reference_notes.md`

---

### Week 2 — Trade Data: Acquisition, Cleaning & Edge Construction
**Goal:** Get BACI trade data fully cleaned and into the final edge format in a single week. BACI is well-structured enough that acquisition and finalization can be done together.

- [ ] Register and download BACI dataset (select years: **2010, 2015, 2018, 2019, 2020** to capture pre/post-COVID and a time series)
- [ ] Understand BACI's structure: it reports trade in USD thousands, by HS product code, by reporter/partner pair
- [ ] Aggregate to country-pair level (sum across all HS codes) to create total bilateral trade edges
- [ ] Join BACI country codes (they use a custom numeric scheme) to ISO3 via CEPII's concordance file
- [ ] Flag and log any country codes that don't map cleanly (dissolved states, territories, etc.)
- [ ] Write and run a data quality report: how many country pairs have data? What share of world trade is covered?
- [ ] Handle asymmetric reporting: BACI already harmonizes importer/exporter-reported values — document this in the schema notes
- [ ] Normalize trade values by source country GDP (download GDP from World Bank WDI, join by ISO3 + year)
- [ ] Assign `data_quality` flags: `"reported"` for standard BACI entries, `"missing"` for country pairs with no bilateral record
- [ ] Decide and document a **missing data policy**: are zero-value pairs included as edges with value=0, or omitted? (Recommended: include with value=0 and flag as `"missing"` for transparency)
- [ ] Produce the final `trade_edges.csv` matching the edge schema above
- [ ] Write unit tests: check for self-loops (source=target), check ISO3 codes are all valid, check no nulls in required fields

**Output:** `data/final/trade_edges.csv`, `docs/trade_data_notes.md`, `tests/test_trade_edges.py`

---

### Week 3 — Migration Data Acquisition & Cleaning
**Goal:** Integrate the UN DESA bilateral migration dataset.

- [ ] Download UN DESA International Migrant Stock by origin and destination (available for 1990, 1995, 2000, 2005, 2010, 2015, 2020)
- [ ] Note and document that this is **migrant stock** (people living abroad at a point in time), not annual flow — this distinction must appear in schema documentation
- [ ] Map UN DESA country names/codes to ISO3 (this requires manual work; UN uses its own country names in some cases — budget time for a ~30-entry manual crosswalk)
- [ ] Handle historical country changes: Yugoslavia, USSR, Czechoslovakia appear in older vintages — document the policy for handling them (recommended: exclude pre-dissolution data or aggregate to successor states)
- [ ] Select comparable years with trade data: use **2010, 2015, 2020**
- [ ] Normalize by source country population (download from World Bank WDI)

**Output:** `data/processed/migration_edges_raw.csv`, `docs/migration_data_notes.md`

---

### Week 4 — Migration Edge Finalization & Schema Integration
**Goal:** Merge both edge types into a unified schema.

- [ ] Apply same `data_quality` flagging logic as trade data
- [ ] Produce final `migration_edges.csv` matching the edge schema
- [ ] **Merge trade and migration edges** into a single `edges.csv` with `relationship_type` column distinguishing them
- [ ] Validate the combined edge table: consistent ISO3 codes across both types, no schema violations, correct units per type
- [ ] Confirm the node table covers all countries appearing in edge tables (no orphan edges)
- [ ] Write unit tests for the combined edge table

**Output:** `data/final/edges.csv`, `data/final/countries.csv`, `tests/test_combined_edges.py`

---

### Week 5 — Database Setup & Relational Queries
**Goal:** Load data into a queryable store and demonstrate relational use cases.

- [ ] Load `countries.csv` and `edges.csv` into SQLite (lightweight, no server required, easy to share)
- [ ] Write and document at least **5 example SQL queries**, such as:
  - Top 10 export partners of a given country in a given year
  - Countries with the highest in-degree (most import sources) for trade
  - Country pairs that appear in both trade and migration edges
  - Countries whose top trade partner changed between 2010 and 2020
  - Migration corridors with the highest stock relative to origin population
- [ ] Export query results as CSVs for use in visualizations

**Output:** `data/final/network.db`, `queries/example_queries.sql`, `queries/query_results/`

---

### Week 6 — Graph Analysis & Visualizations
**Goal:** Demonstrate graph-native use cases and produce deliverable visualizations.

- [ ] Load edge data into NetworkX as a directed, weighted multigraph
- [ ] Compute and export **network statistics** for each year and edge type:
  - Degree centrality (in-degree and out-degree)
  - Betweenness centrality
  - PageRank (useful for trade networks)
- [ ] Run **Louvain community detection** on the trade network — do regional trade blocs emerge?
- [ ] Produce at least **3 visualizations**:
  1. Global trade network map for a single year (node size = GDP, edge weight = trade value) — use `matplotlib` + `geopandas` or `pyvis`
  2. Time series chart: how top trade partners of a selected country evolved 2010–2020
  3. Comparison plot: trade centrality vs. migration centrality by country (scatter plot)
- [ ] Save all visualizations as PNG/SVG

**Output:** `analysis/graph_stats.csv`, `visualizations/`

---

### Week 7 — Documentation, Final Review & Packaging
**Goal:** Deliver a clean, reproducible, fully documented package.

- [ ] Write `README.md` covering: project overview, how to reproduce the pipeline, folder structure, schema reference, known limitations
- [ ] Write `docs/data_sources.md`: full citation for each source, access date, version/edition used
- [ ] Write `docs/schema.md`: full column definitions, edge type definitions, data quality flag definitions, decisions made on missing data and asymmetric reporting
- [ ] Write `docs/limitations.md`: what the dataset does NOT capture (informal trade, undocumented migration, intra-year changes, etc.)
- [ ] Final end-to-end pipeline test: run all scripts from scratch on a clean environment and confirm outputs match
- [ ] Package final deliverables into a single zip / repository release

**Output:** Complete documented repository, `README.md`, `docs/`

---

## Folder Structure

```
project/
├── data/
│   ├── raw/              # Original downloaded files, never modified
│   ├── processed/        # Intermediate cleaned files
│   ├── reference/        # Country codes, crosswalks
│   └── final/            # Final node and edge tables, database
├── scripts/
│   ├── 01_build_country_reference.py
│   ├── 02_clean_trade.py
│   ├── 03_clean_migration.py
│   ├── 04_merge_edges.py
│   ├── 05_load_database.py
│   └── 06_graph_analysis.py
├── queries/
│   ├── example_queries.sql
│   └── query_results/
├── analysis/
├── visualizations/
├── tests/
├── docs/
│   ├── data_sources.md
│   ├── schema.md
│   ├── limitations.md
│   ├── trade_data_notes.md
│   ├── migration_data_notes.md
│   └── country_reference_notes.md
└── README.md
```

---

## Key Technical Decisions to Document

These are choices the developer must make explicitly and document — they're not just implementation details, they're part of what makes the dataset meaningful and reproducible.

**Missing bilateral data:** Include zero-value edges with `data_quality = "missing"` rather than silently dropping pairs. This preserves the distinction between "no relationship" and "no data."

**Asymmetric reporting:** BACI already harmonizes this for trade. For migration, UN DESA provides a single origin-destination matrix so there is no asymmetry issue.

**Country definitions over time:** Use ISO3 codes as of 2020 as the canonical reference. Countries that dissolved before 2010 (e.g., Yugoslavia, USSR) are excluded. South Sudan (independent 2011) is included from 2015 onward only.

**Normalization:** Always retain raw values alongside normalized values. Never overwrite.

**Time window:** 2010, 2015, 2020 for both layers — chosen for alignment between annual (trade) and quinquennial (migration) sources.

---

## Risks & Mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| BACI download requires registration / delayed access | Low | Register in Week 1; have UN Comtrade as backup |
| UN DESA country name crosswalk takes longer than expected | Medium | Budget 2 extra days in Week 4; start the manual mapping early |
| GDP/population data missing for small territories | Medium | Use World Bank data; flag gaps; use regional averages as fallback if needed |
| Community detection produces uninterpretable clusters | Low | This is exploratory — document results as-is; don't over-interpret |

---

*Last updated: February 2026 — revised to 7-week timeline*
