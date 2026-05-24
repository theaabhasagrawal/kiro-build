# Homeowner Data — Analysis & Master Schema Design

> Goal: build a single homeowner-centric view that lets you target & pitch services
> (custom website, PDF booklet, etc.) to vacation-rental homeowners.
>
> All numbers below come from the SAMPLE files (most CSVs capped at 1,000 rows,
> XLSX MasterData capped at 1,000 rows). In production these patterns will hold
> but absolute counts will scale up.

---

## 1. The big picture in one sentence

Your data covers vacation rentals from **one parent group running multiple brands**
(TUI / Interhome / Novasol on one side, Belvilla / DanCenter / Park on the other),
plus **externally scraped Google Business data** for some of those rentals. The
files split cleanly by brand family, and homeowner contact info is highly skewed
toward the DanCenter side.

```
                          ┌─────────────────────────┐
                          │  PARENT GROUP (Awaze?)  │
                          └────────────┬────────────┘
                                       │
        ┌──────────────────────────────┴──────────────────────────────┐
        │                                                             │
   TUI / Interhome / Novasol                              Belvilla / DanCenter / Parks
   (Company codes: TUI, TMO)                              (Company codes: BVAG, PRK, DC)
   Code format: BRE01215-P                                Code format: NL-4493-361
                                                          (sub-units: NL-4493-361-01)
```

---

## 2. File-by-file inventory (what's actually in each file)

| # | File | Rows (sample) | What it is | Brand coverage | Has homeowner ID? | Has homeowner contact? |
|---|---|---:|---|---|---|---|
| 1 | `Accommodation_Basic_SAMPLE.csv` | 1,000 | Property master record (location, contract, status, sales agent) | Mixed: BVAG 81% / PRK 11% / TUI 8% | Yes — `Homeowner` (numeric) | No |
| 2 | `Accommodation_Images_SAMPLE.csv` | 1,000 | One row per property containing a JSON blob of all photo URLs + tags + charm scores | Belvilla format mostly | No (joins via `PropertyCode`) | No |
| 3 | `Accommodation_Feature_Types_SAMPLE.csv` | 413 | **Reference** — multilingual feature dictionary (Castle/Villa/Pool/Pets…) | – | – | – |
| 4 | `Accommodation_Layout_SAMPLE.csv` | 20 | **Reference** — multilingual layout dictionary (Floors/Rooms/Beds…) | – | – | – |
| 5 | `Accommodation_Status_SAMPLE.csv` | 18 | **Reference** — multilingual status code dictionary | – | – | – |
| 6 | `BQ Results Jan 29 2026_SAMPLE.csv` | 1,000 | OTA listing snapshot — Booking.com / Airbnb / HomeAway / Expedia IDs per property + minimal owner info | TUI / Interhome / Novasol only | Yes — `OWNER_ID` | Partial — country, city, zip, website, language |
| 7 | `Bookings Base_SAMPLE.csv` | 1,000 (174 real) | Booking-level facts: dates, revenue, costs, guest origin, cancellations | Belvilla + OYO US | Yes — `HOMEOWNER_ID` (5% fill) | No |
| 8 | `DC_Agency_Info_SAMPLE.csv` | 1,000 | **NOT homeowners — these are travel agencies / B2B partners** with commission rates | DanCenter side | No | (Has agency contact, irrelevant for HO outreach) |
| 9 | **`HO Info Database_SAMPLE.csv`** | **1,000** | **Homeowner master record — full contact details** | DanCenter only (78% DK, 13% SE, 5% NO) | **Yes — `HomeownerNumber` (D-prefix)** | **YES — phone 99.9%, email 93%, address 99%, name 93%** |
| 10 | `Inventory_Flow_SAMPLE.csv` | 1,000 | Daily/period inventory snapshot — open vs blocked nights, GBV predictions, account manager, tenure | Belvilla mostly | Indirect (via accommodation) | No |
| 11 | `Tableau_Accommodation_SAMPLE.csv` | 1,000 | **Property master + rich owner profile** (151 cols) — photos quality, OTA presence, owner type/language/website/PPP | TUI / Interhome / Novasol | **Yes — `OWNER_ID`** | **Partial — language, country, city, zip, website, ambassador, PPP program** |
| 12 | `Tableau_Booking_SAMPLE.csv` | 1,000 | Booking-level dim with channel, partner, customer, NPS, complaint flags | Belvilla mostly | No (only guest `CUSTOMER_ID`) | No |
| 13 | `Tableau_Features_SAMPLE.csv` | 1,000 | Long-format property features (1 row per property × feature) | Belvilla mostly | No | No |
| 14 | `Interhome Leads_SAMPLE.xlsx` | 23 sheets, 1,000 in MasterData | **External Google Maps scrape** of Interhome listings — Place ID, rating, reviews, phone, email, website | Interhome | No (extractable from URL) | **YES — phone 94%, email 52%, website 64%** |
| 15 | `Novasol Leads_SAMPLE.xlsx` | 18 sheets, 1,000 in MasterData | **External Google Maps scrape** of Novasol listings | Novasol | No (extractable from URL) | **YES — phone 88%, email 91%, website 96%** |

### Lookup files demystified

The three reference CSVs (`Accommodation_Feature_Types`, `_Layout`, `_Status`) each
have a JSON `Description` field with translations in 9 languages
(NL, NO, FR, DE, EN, SV, IT, ES, DA). Use these to render any feature/status code in
the homeowner's preferred language for the booklet/website. Example: status code `CR`
maps to "Contract rejected" / "Vertrag abgelehnt" / "Contrato rechazado" etc.

---

## 3. The relationship map (how everything joins)

```
                                      ┌───────────────────────────────┐
                                      │       HOMEOWNER (master)      │
                                      │   homeowner_id (numeric)      │
                                      └────────────┬──────────────────┘
                                                   │ 1
                                                   │
                                                   │ N
                                      ┌────────────▼──────────────────┐
                              ┌───────│   ACCOMMODATION (property)    │───────┐
                              │       │   PropertyCode = ACCO_CODE    │       │
                              │       │   ACCOMMODATION_ID            │       │
                              │       └────────────┬──────────────────┘       │
                              │                    │ 1                        │
                              │                    │                          │
                              │                    │ N                        │
                              │       ┌────────────▼──────────────────┐       │
                              │       │     HOUSE / RENTAL UNIT       │       │
                              │       │  HOUSE_ID = ACCO_CODE + '-NN' │       │
                              │       └────────────┬──────────────────┘       │
                              │                    │                          │
                              │                    │ N                        │
                              │       ┌────────────▼──────────────────┐       │
                              │       │           BOOKING             │       │
                              │       │         BOOKING_ID            │       │
                              │       └───────────────────────────────┘       │
                              │                                               │
                  ┌───────────▼──────────┐                          ┌─────────▼────────┐
                  │  Photos (JSON blob)  │                          │     Features     │
                  │    Images table      │                          │  Features table  │
                  └──────────────────────┘                          └──────────────────┘

  External, joined by URL slug or fuzzy match on Name / Lat-Long / Address:
  ┌─────────────────────────────────────────────────────────────────────────────────────┐
  │  Interhome Leads xlsx  →  Property Link contains code like 'at6642.240.1'           │
  │  Novasol Leads  xlsx   →  Property Link contains code like 'asa500'                 │
  └─────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.1 ID / join-key cookbook

| Concept | Found in (column) | Format example | Notes |
|---|---|---|---|
| **Property** | `Accommodation_Basic.PropertyCode`, `Accommodation_Images.PropertyCode`, `BQ Results.ACCOMMODATION_CODE`, `Tableau_Accommodation.ACCOMMODATION_CODE`, `Inventory_Flow.ACCOMMODATION_ID`, `Bookings Base.ACCOMMODATION_CODE`, `Tableau_Booking.ACCOMMODATION_CODE`, `Tableau_Features.ACCOMMODATION_CODE` | `BRE01215-P` (TUI) / `NL-4493-361` (Belvilla) | All these are **the same key** despite different column names. Format depends on brand. |
| **Sub-unit / rental unit** | `Inventory_Flow.HOUSE_ID`, `Bookings Base.HOUSE_ID` | `NL-4493-361-01` | = `ACCOMMODATION_CODE + "-" + 2-digit suffix`. ~1.5 sub-units per property in the sample. |
| **Homeowner (numeric)** | `Accommodation_Basic.Homeowner`, `BQ Results.OWNER_ID`, `Tableau_Accommodation.OWNER_ID`, `Bookings Base.HOMEOWNER_ID` | `1046096`, `951674` | Same ID space across these four. **Caveat:** `55` is a sentinel/system value, not a real homeowner. |
| **Homeowner (DanCenter)** | `HO Info Database.HomeownerNumber` | `D1069326` | Strip the `D` prefix → numeric ID joins back to the others. `Table` column == `DC` for all rows. |
| **Caretaker** | `Accommodation_Basic.Caretaker`, `Tableau_Accommodation.CARETAKER_ID` | numeric | Often equals the homeowner (homeowner is their own caretaker), sometimes a separate person. |
| **Owner company** | `Tableau_Accommodation.OWNER_COMPANY_ID` / `Inventory_Flow.LARS_COMPANY` / `Accommodation_Basic.Company` | `TUI`, `BVAG`, `PRK`, `DC`, `TMO` | Tells you which brand-family a property belongs to. |
| **OTA listing IDs** | `BOOKINGCOM_ID`, `AIRBNB_ID`, `HOMEAWAY_ID`, `EXPEDIA_ID` (BQ Results & Tableau_Accommodation) | `4716027,471602702,14013089` | Comma-separated triples. Useful to know if the homeowner is **already** on competing platforms. |
| **Google Place ID** | `Interhome Leads.Place ID`, `Novasol Leads.Place ID` | `ChIJPw5dGUTtnEcRxkWDtQN_GHM` | Lets you re-pull Google reviews / business profile any time. |

### 3.2 Joining the Lead xlsx files to the internal data

The xlsx files have **no internal ID column**, but the brand property code is
embedded in `Property Link`:

- **Interhome:** `…/holiday-house-chalet-lechtraum-at6642.240.1/?partnerid=…`
  → extract the regex `[a-z]{2}\d{4,5}\.\d+\.\d+` (e.g. `at6642.240.1`).
  This is the Interhome website slug. To map to your internal `PropertyCode`
  (e.g. `OSB02086-CYB`), you'll likely need either:
  (a) a TUI mapping table you don't have here, OR
  (b) fuzzy match on Lat/Long + Name (recommended fallback).
- **Novasol:** `…/holidayhome/weisspriach-asa500?…`
  → extract the trailing `[a-z]{3}\d+` token (e.g. `asa500`). Same caveat — needs
  mapping to internal Novasol code or Lat/Long fuzzy match.

For a rapid first pass, **Lat/Long + zipcode** match within ~50m is usually accurate
enough since these are physical addresses scraped from the brand sites.

---

## 4. Proposed master `HOMEOWNER` schema

This is the table you should build for outreach — **one row per homeowner**, with
every fact you'd want to see during a sales call or to pick targets.

### 4.1 Identity & contact (the core of the outreach view)

| Field | Source | Notes / fill-rate (DC sample) |
|---|---|---|
| `homeowner_id` | numeric, the join key | union of all `OWNER_ID` / `Homeowner` / `HomeownerNumber-stripped` |
| `homeowner_id_dc` | `HO Info Database.HomeownerNumber` | `D1069326` form, only DanCenter |
| `brand_family` | derived | `TUI` (Interhome/Novasol) vs `Belvilla` vs `DanCenter` vs `Park` |
| `full_name` | `HO Info Database.FullName` | 93% |
| `company_name` | `HO Info Database.CompanyName` | 4% — most are individuals |
| `email` | `HO Info Database.Email` ⊕ Lead xlsx Email | 93% (DC), 52–91% (xlsx leads) |
| `phone` | `HO Info Database.Telephone` ⊕ Lead xlsx Phone | 99.9% (DC), 88–94% (xlsx leads) |
| `address_line` | `HO Info Database.Address` | 99% |
| `city` | `HO Info Database.City` ⊕ `Tableau_Accommodation.OWNER_CITY` | 71% (DC) |
| `zipcode` | `HO Info Database.Zipcode` ⊕ `OWNER_ZIPCODE` | 99.8% (DC) |
| `country` | `HO Info Database.Country` ⊕ `OWNER_COUNTRY_CODE` | 100% |
| `language` | `HO Info Database.Language` ⊕ `OWNER_LANGUAGE_CODE` | 100% — **drives which language to pitch & build the booklet in** |
| `do_not_disturb` | `HO Info Database.DoNotDisturb` ⊕ `OWNER_DND` | 100% — **filter these OUT** |
| `existing_website` | `HO Info Database.HomeownerWebsite` ⊕ `OWNER_WEBSITE` ⊕ Lead xlsx Website | 0.1% (DC), ~64% (Interhome xlsx) — **the missing-website segment is your highest-fit target** |

### 4.2 Owner profile / segmentation signals

| Field | Source | Why it matters for pitching |
|---|---|---|
| `owner_type` | `Tableau_Accommodation.OWNER_TYPE`, `HO Info Database.Type` | Individual vs Local Office vs Partner — different pitch & price point |
| `homeowner_creation_date` | `BQ Results.HOMEOWNER_CREATION_DATE`, `Tableau_Accommodation.HOMEOWNER_CREATION_DATE` | Tenure with the platform |
| `last_login_date` | `OWNER_LAST_LOGIN_DATE`, `HO Info Database.LastLoginDate` | **Active / dormant / churned** indicator |
| `is_ambassador` | `OWNER_AMBASSADOR`, `HO Info Database.Ambassador` | Engaged power-users — likely to refer |
| `is_in_ppp` | `OWNER_PREFFERRED_PARTNER_PROGRAM` | Already on premium program |
| `ppp_account_manager` | `OWNER_PPP_ACCOUNT_MANAGER` | Who they currently deal with |
| `marketing_source` | `OWNER_MKT_SOURCE_LEAD`, `HO Info Database.Origin` / `OriginDetail` | How they originally arrived — useful for tailored messaging |

### 4.3 Portfolio facts (aggregated up from accommodations)

| Field | Aggregation | From |
|---|---|---|
| `n_properties` | count distinct `ACCOMMODATION_CODE` | property files |
| `n_rental_units` | count distinct `HOUSE_ID` | `Inventory_Flow`, `Bookings Base` |
| `property_countries` | distinct list | `Accommodation_Basic.Country` |
| `property_cities` | distinct list | `Accommodation_Basic.City` |
| `total_persons_capacity` | sum `ACCOMMODATION_NUMBER_OF_PERSONS` | `Tableau_Accommodation`, `BQ Results` |
| `total_bedrooms` | sum `ACCOMMODATION_NUMBER_OF_BEDROOMS` | `Tableau_Accommodation` |
| `total_size_m2` | sum `ACCOMMODATION_SIZE` | `Tableau_Accommodation` |
| `accommodation_types` | distinct list (Apartment / Villa / Chalet / Castle / …) | via `Accommodation_Feature_Types` lookup |
| `has_pool / has_pet_friendly / has_seafront / etc` | bool flags from `Tableau_Features` | – |

### 4.4 Performance signals (what they're earning, how good they are)

| Field | Aggregation | From |
|---|---|---|
| `gbv_lifetime` | sum | `Inventory_Flow.GBV_LIFETIME` |
| `gbv_last_12m` | sum | `Inventory_Flow.GBV_BOOK_DATE_THIS_PERIOD` |
| `n_bookings_lifetime` | count | `Bookings Base` (BOOKING_ID notnull) |
| `n_bookings_last_12m` | count by date | `Bookings Base` |
| `avg_booking_value` | mean | `Bookings Base.REVENUE_GROSS_BOOKING_VALUE` |
| `avg_stay_duration_nights` | mean | `Tableau_Booking.STAY_DURATION` |
| `n_complaints` | sum | `Tableau_Accommodation.ACCOMMODATION_NUMBER_OF_COMPLAINTS`, `Tableau_Booking.INDICATOR_COMPLAINT` |
| `n_damages_sos` | sum | `Tableau_Accommodation.ACCOMMODATION_NUMBER_OF_DAMAGES / _SOS` |
| `avg_review_score` | mean | `Tableau_Accommodation.ACCOMMODATION_REVIEW_SCORE` |
| `avg_nps_score` | mean | `Tableau_Booking.NPS_SCORE` |
| `tenure_months` | from | `Inventory_Flow.TENURE_SINCE_FIRST_LIVE_MONTHS` |
| `traffic_light_status` | mode | `Tableau_Accommodation.HOMEOWNER_TRAFFIC_LIGHT` (likely red/yellow/green) |

### 4.5 Distribution / OTA presence (the most pitch-relevant signal)

| Field | Source | Why |
|---|---|---|
| `is_on_bookingcom` | bool from `BOOKINGCOM_ID notnull` | 27% of TUI sample is on Booking.com |
| `is_on_airbnb` | `AIRBNB_ID notnull` | 14% on Airbnb |
| `is_on_homeaway` | `HOMEAWAY_ID notnull` | 77% |
| `is_on_expedia` | `EXPEDIA_ID notnull` | 0% in sample |
| `n_ota_channels` | count of the above | low number → high pitch fit (they aren't multi-listed) |
| `google_place_id` | `Interhome/Novasol Leads.Place ID` | use to pull live Google reviews |
| `google_rating` | xlsx | **0–5 score** — pitch the booklet harder to <4.0 owners |
| `google_review_count` | xlsx | engagement proxy |
| `has_own_website` | xlsx Website notnull | **Your primary segmentation axis** |

### 4.6 Photos & content quality (justifies the website pitch)

| Field | Source | Why |
|---|---|---|
| `n_photos` | `Tableau_Accommodation.ACCOMMODATION_NUMBER_OF_PHOTOS` | low → "you need a photoshoot" |
| `photo_check_score` | `ACCOMMODATION_PHOTO_CHECK_SCORE` | quality gate |
| `photo_charm_score` | `ACCOMMODATION_PHOTO_CHARM_SCORE` | aesthetic |
| `photographed_by` | `ACCOMMODATION_PHOTOGRAPHY_BY` | DIY vs pro |
| `has_image_blob` | derived from `Accommodation_Images.Media` (parsing the JSON) | extract photo URLs for the booklet |

---

## 5. Data-quality issues you must handle

1. **Sentinel homeowner_id `55`** appears across files for many properties. Filter
   it out — it's almost certainly a system/internal account.
2. **Bookings Base has 826/1000 "phantom" rows** with no `BOOKING_ID` but full
   property metadata. Always filter `BOOKING_ID.notnull()` before doing booking
   analytics; for property-only fields use other property tables instead.
3. **`HO Info Database.LastLoginDate` is a placeholder `1900-01-01`** for nearly
   every row in the sample. In production this likely contains real dates; verify
   before using it as a "dormant homeowner" signal.
4. **`HO Info Database` covers DanCenter only** (DK 78%, SE 13%, NO 5%). For
   Belvilla / Parks / TUI homeowners, contact details live partially in
   `Tableau_Accommodation` (city/zip/website/language) — there is **no equivalent
   master with phone & email** in the files you've shared. This is the biggest
   gap.
5. **Owner email/phone for ~80% of your TUI homeowners is only in the Lead xlsx
   files**, which were obtained via Google scraping. Treat them as "best effort"
   contact data and verify before outreach.
6. **`DC_Agency_Info` is misleadingly named** — it's travel agencies, not
   homeowners. Don't pipe it into the homeowner master.
7. **Property-code overlap across CSVs is small in the sample** because each
   sample randomly drew different rows. In production with full data the same
   `PropertyCode` will join cleanly across all files.
8. **Lead xlsx country sheets duplicate MasterData rows** for the same country.
   Use only `MasterData` (or rebuild `MasterData` by concatenating country sheets
   if you trust those more) — don't double-count.
9. **`Accommodation_Images.Media` is a stringified Python repr** (with `array(...)`
   and `dtype=object`), not pure JSON. You'll need to use `ast.literal_eval` or a
   regex to parse it.
10. **HOUSE_ID vs ACCOMMODATION_CODE level matters.** When aggregating booking
    revenue to the homeowner, dedupe at HOUSE_ID level first (a single booking is
    on one HOUSE_ID), then group up to ACCOMMODATION_CODE, then to homeowner.

---

## 6. Open questions I need from you before building the pipeline

1. **Is this data from Awaze (Interhome/Novasol/Belvilla/James-Villas/HomeAway-EU)
   or is it actually TUI Group / another parent?** This affects how we name
   `brand_family` and is useful pitch context.
2. **Do you have a code-mapping table** between the Interhome/Novasol website
   slugs (`at6642.240.1`, `asa500`) and the internal `PropertyCode`
   (`OSB02086-CYB`)? If yes — sharing it will make the xlsx → CSV join trivial.
   If no — I'll fall back to Lat/Long fuzzy matching.
3. **Is there a `HO Info Database`-equivalent for Belvilla / Park / TUI
   homeowners?** Or for those brands, do we only have what's in
   `Tableau_Accommodation` (city/zip/language but no phone/email)? This is the
   single biggest factor for how usable the pipeline is for non-DanCenter
   homeowners.
4. **What's the legal status of the Lead xlsx scrape data?** Email outreach using
   scraped business profiles is fine in most EU jurisdictions for B2B but check
   your country. Phone outreach is more restrictive.
5. **Do you want one homeowner = one row, or one (homeowner, brand_family) = one
   row?** A homeowner could in theory list under multiple brands; the schema
   above assumes one row per `homeowner_id`.
6. **Sample size:** all CSVs are capped at 1,000 rows. What's the **real**
   homeowner count when you run this on full data? (For sizing your outreach
   funnel.)
7. **What's `LastLoginDate = 1900-01-01`?** Placeholder, missing, or genuine? If
   placeholder, do you have a better "owner activity" signal we can fall back to
   (last contract action, last booking, last login on real system)?
8. **`Type` column in `HO Info Database` has values `Local Office` / `Partner`
   only filled on 0.2% of rows.** What is `Type` for the other 99.8%? Should the
   default be `Individual`?

---

## 7. Suggested target segments for your outreach (based on the data you have)

| Priority | Segment | Definition (filter rule) | Approximate volume from sample | Pitch angle |
|---|---|---|---:|---|
| **🟢 1** | **Active DanCenter homeowner without a website** | `has_own_website == False AND last_login < 6 months AND brand_family == 'DanCenter'` | ~99% of the 1,000 HO Info rows have no website | Custom website + booklet; we already have phone + email |
| 🟢 2 | High-revenue, low-OTA-presence | `gbv_last_12m > €X AND n_ota_channels <= 1` | – | Help them list on more channels + a website to reduce platform dependency |
| 🟡 3 | Low Google rating | `google_rating < 4.0 AND google_review_count > 5` | – | Booklet & website to control narrative |
| 🟡 4 | Many photos missing / low photo score | `n_photos < 10 OR photo_charm_score < threshold` | – | Photoshoot + website + booklet bundle |
| 🟡 5 | Large portfolio (5+ properties) | `n_properties >= 5` | 174 owners → 5.7 props avg in TUI sample | Account-style pitch — multi-property dashboard, branded site |
| 🔴 6 | New homeowners | `homeowner_creation_date < 12 months` | – | Onboarding bundle |
| ❌ Skip | DND owners | `do_not_disturb == True` | – | Don't contact |
| ❌ Skip | `homeowner_id == 55` | sentinel | – | System account |

---

## 8. Recommended next step

Tell me:

1. **Yes/no on the open questions in §6** (especially #2 and #3 — they decide how
   much homeowner contact data we actually have for non-DanCenter brands).
2. **Which segment(s) from §7 you want to start with**, and I'll build:
   - a Python pipeline (`build_homeowner_master.py`) that ingests all 15 files,
     normalises the IDs, joins everything into the schema in §4, and outputs:
     - `homeowners_master.csv` (one row per homeowner, ready for outreach)
     - `homeowners_targets_<segment>.csv` (filtered to your priority segment)
     - a small Markdown report with counts per segment, brand, country.

Once we agree on §6 answers, the implementation is a few hundred lines of
pandas — ~2 hours of work, tops.
