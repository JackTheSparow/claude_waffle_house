# Data Dictionary — `waffle_houses.csv`

**Source file:** `waffle_houses.csv`
**Generated:** 2026-06-19
**Rows:** 2,006 location records (+1 header row)
**Columns:** 14
**Encoding/format:** UTF-8, comma-delimited, quoted fields where values contain commas

This document describes every column, then reports the results of a data-quality
review (missing values, duplicates, type problems, and inconsistencies that
could affect analysis).

---

## 1 · Column reference

| # | Column | Type | Description | Notes |
|---|--------|------|-------------|-------|
| 1 | `Store Code` | String (usually integer) | Unique location identifier. | All numeric (range **4–3442**) **except one**: `WH_Museum`. Acts as primary key. Not contiguous — gaps exist. |
| 2 | `Business Name` | String | Display name. | Pattern `Waffle House #<Store Code>` for all rows except the museum (`Waffle House Museum #WH_Museum`). |
| 3 | `Address` | String | Street address. | Mostly ALL-CAPS; the museum row is in Title Case. Free-text, not normalized. |
| 4 | `City` | String | City name. | ALL-CAPS for all rows except the museum (`Decatur`). |
| 5 | `State` | String (2-letter) | USPS state abbreviation. | 25 distinct states, all valid 2-letter uppercase codes. |
| 6 | `Postal Code` | String | 5-digit ZIP. | All 2,006 are exactly 5 digits. Keep as **string** to preserve leading zeros. |
| 7 | `Country` | String | Country code. | Constant `US` for every row — no analytic value. |
| 8 | `Latitude` | Float | Decimal latitude. | Range 25.10–41.78. **Precision is inconsistent** (1–13 decimal places). |
| 9 | `Longitude` | Float | Decimal longitude. | Range -112.34 to -75.34. Same precision inconsistency as latitude. |
| 10 | `Phone Number` | String | Phone number(s). | Format `(xxx) xxx-xxxx`. **228 rows contain multiple numbers** separated by `;`. One row uses dashes only. |
| 11 | `Website URL` | String | Location page on wafflehouse.com. | Contains a literal triple slash `///` (`https://locations.wafflehouse.com///...`). Consistent across rows. |
| 12 | `Operated By` | String (categorical) | Operator / ownership type. | 3 families: corporate, fully-owned subsidiary, franchise. 1 blank. See §3. |
| 13 | `Online Order Link` | String (URL) | Online ordering page. | Pattern `https://order.wafflehouse.com/menu/waffle-house-<code>`. 3 blanks. |
| 14 | `Formatted Business Hours` | String | Operating hours. | 1,998 are `Monday - Sunday\| 24 hours`. 8 rows differ (limited hours, closed, or blank). See §3. |

---

## 2 · Data-quality summary

| Check | Result |
|-------|--------|
| Missing / blank values | **6 blanks across 3 columns** (see below) |
| Duplicate Store Codes | **0** |
| Fully duplicate rows | **0** |
| Duplicate Address + City + State | **0** |
| Invalid state codes | 0 |
| Postal codes not 5 digits | 0 |
| Coordinates outside continental-US bounds | 0 |
| Constant (zero-information) columns | `Country` (always `US`) |

### Missing / blank values

| Column | Blanks | Where |
|--------|--------|-------|
| `Operated By` | 1 | The Waffle House Museum row (`WH_Museum`) |
| `Online Order Link` | 3 | Museum + 2 stores (#3442, and the closed/limited locations) |
| `Formatted Business Hours` | 2 | Museum + store #3442 |

All other 11 columns are 100% populated.

---

## 3 · Inconsistencies & issues that could affect analysis

### 🔴 1. The "Waffle House Museum" is not a restaurant
Store Code `WH_Museum` is the **single biggest data anomaly** and is the source
of almost every missing value and casing oddity:

- Non-numeric `Store Code` (`WH_Museum`) — will break any code that casts Store Code to integer.
- Only row where `Business Name` ≠ `Waffle House #<code>`.
- Only row with Title-Case `Address` / `City` (vs. ALL-CAPS everywhere else).
- Blank `Operated By`, `Online Order Link`, and `Formatted Business Hours`.
- Phone formatted with dashes (`770-326-7086`) instead of `(xxx) xxx-xxxx`.

**Recommendation:** Exclude this row from location/restaurant counts, or flag it
explicitly. It is a tourist attraction, not an operating store.

### 🟠 2. `Operated By` needs normalization before grouping
There are 15 distinct raw strings that collapse into 3 categories:

| Category | Count | Notes |
|----------|------:|-------|
| Corporate (`WAFFLE HOUSE, INC`) | 1,251 | |
| Fully-owned subsidiary | 627 | 4 named subsidiaries (East Coast, Mid South, Midwest, Ozark) |
| Franchise | 127 | 11 distinct franchise operators |
| Blank | 1 | The museum |

Formatting is inconsistent within the raw values:
- `FRANCHISE :` (space before colon) vs `FULLY OWNED SUBSIDIARY:` (no space).
- Mixed `INC` / `INC.` / `LLC.` suffixes.

**Recommendation:** Derive a clean `operator_type` column (Corporate / Subsidiary
/ Franchise) by prefix, rather than grouping on the raw string.

### 🟠 3. `Phone Number` — 228 rows hold multiple values
228 rows contain two numbers separated by `; ` (e.g. `(706) 956-4560; (706) 356-9921`).
Treating this column as a single phone number, or counting distinct values, will
mislead. One row (the museum) uses dash formatting.

**Recommendation:** Split on `;` and take the first number, or normalize all to digits.

### 🟡 4. Coordinate precision is uneven
Decimal places on `Latitude`/`Longitude` range from **1 to 13**. Six rows have
≤2 decimal places (≈1 km+ uncertainty), e.g.:

- #1684 Lawrenceville, GA `33.9`
- #2104 Jacksonville, FL `30.3`
- #2165 Hermitage, TN `36.2`

Values are valid and within US bounds — fine for mapping, but don't treat the
low-precision points as exact.

### 🟡 5. Business hours — 8 exceptions to "24 hours"
1,998 of 2,006 are open 24/7. The exceptions matter if you assume all stores are
always open:

| Store Code | City, State | Hours |
|------------|-------------|-------|
| 2473 | Xenia, OH | Closed |
| 2475 | Buchanan, GA | Closed |
| 2468 | Crawfordville, FL | 7:00am – 9:00pm |
| 2477 | North Myrtle Beach, SC | 7:00am – 9:00pm |
| 258  | Ft Lauderdale, FL | 7:00am – 9:00pm |
| 2488 | Commerce, GA | 7:00am – 2:00pm |
| 3442 | Baton Rouge, LA | *(blank)* |
| WH_Museum | Decatur, GA | *(blank)* |

Note the hours string carries a leading-space quirk (`|  7:00am`) and combines
the day-range and time with a `|` delimiter — parse, don't string-match.

### 🟡 6. Zero-information & cosmetic notes
- `Country` is `US` for all rows — safe to drop for analysis.
- `Website URL` contains a literal `///` — consistent, but clean if joining/parsing URLs.

---

## 4 · Recommended cleaning steps (before analysis)

1. **Decide how to handle `WH_Museum`** — exclude from store counts or flag it.
2. **Read `Postal Code` and `Store Code` as strings**, not integers (leading zeros / `WH_Museum`).
3. **Derive `operator_type`** (Corporate / Subsidiary / Franchise) from `Operated By`'s prefix.
4. **Split `Phone Number`** on `;` if you need a single canonical number.
5. **Treat the 8 non-24h rows explicitly** if any analysis assumes always-open.
6. **Drop `Country`** (constant) from analytic tables.
