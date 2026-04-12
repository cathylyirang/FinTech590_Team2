
================================================================
DATABASE SCHEMA: bilateral_edges.db
================================================================

TABLE: edges
Description: Aggregated bilateral trade between 50 European 
countries, sourced from UN Comtrade API (HS 2-digit annual data).

COLUMNS:
  reporter_iso3    TEXT     Reporter country ISO3 code (e.g. DEU)
  reporter_name    TEXT     Reporter country name (e.g. Germany)
  partner_iso3     TEXT     Partner country ISO3 code (e.g. FRA)
  partner_name     TEXT     Partner country name (e.g. France)
  period           TEXT     Year (2019-2023)
  flow_code        TEXT     M = Import, X = Export
  total_trade_usd  REAL     Sum of primary_value in USD
                            (CIF for imports, FOB for exports)

PRIMARY KEY: (reporter_iso3, partner_iso3, period, flow_code)

EDGE INTERPRETATION:
  Each row = one directed edge in a trade network
  - Source node: reporter_iso3
  - Target node: partner_iso3
  - Edge weight: total_trade_usd
  - Direction: reporter exports to (X) or imports from (M) partner

DATA SOURCE:
  - UN Comtrade API v1 (comtradeapi.un.org)
  - Endpoint: /data/v1/get/C/A/HS
  - Parameters: cmdCode=AG2 (all HS 2-digit), flowCode=M,X
  - Coverage: 50 European countries, 2019-2023

KNOWN MISSING VALUES:
  - total_trade_usd = 0 or NULL: Some country pairs have no
    reported bilateral trade for certain years. This may mean
    genuine zero trade or unreported data.
  - Some smaller countries (e.g. GIB, SMR, FRO, MCO, LIE, IMN)
    may have incomplete reporting across years.
  - partner_iso3 = 'UNK': Partner country not in the European
    country list (should not appear in this filtered dataset).
  - isReported = False in raw data means the record was estimated
    by UN from mirror data, not directly reported by the country.
