# 🎬 OTT Movie Streaming — ETL Data Engineering Pipeline

> **HCLTech Campus Drive Hackathon | 23 March 2026 | IMS Engineering College**

---

## 📋 Table of Contents

- [Problem Statement](#problem-statement)
- [Tech Stack](#tech-stack)
- [Dataset](#dataset)
- [Architecture](#architecture)
- [Data Cleansing Rules](#data-cleansing-rules)
- [Star Schema](#star-schema)
- [IICS Mappings](#iics-mappings)
- [KPI Outputs](#kpi-outputs)
- [Pipeline Flow](#pipeline-flow)
- [How to Run](#how-to-run)
- [Team](#team)

---

## 🎯 Problem Statement

An OTT Movie Streaming platform generates high-volume user watch event data daily. This data is **siloed, inconsistent and unanalysed**. The platform requires a robust data engineering pipeline to:

- Cleanse and integrate raw watch event data
- Load into a structured data warehouse
- Generate KPI outputs for business analytics

**Business Goals:**
- Identify top performing movie categories
- Measure user engagement scores
- Analyse city-wise viewership trends
- Track category popularity month over month

---

## 🛠 Tech Stack

| Tool | Purpose |
|---|---|
| **Informatica IICS** | ETL pipeline — ingestion, cleansing, transformation |
| **SQL Server / Oracle** | Target database — staging + DWH tables |
| **SQL / PL/SQL** | Business logic, window functions, KPI queries |

---

## 📂 Dataset

Four CSV source files provided:

| File | Records | Status |
|---|---|---|
| `Watch_Events_Clean.csv` | 400 | Clean — direct load |
| `Watch_Events_Dirty.csv` | 418 | Requires cleansing |
| `Movie_Master.csv` | 150 | Clean — reference data |
| `User_Master.csv` | 120 | Clean — reference data |

### Watch Events Schema
```
EVENT_ID         INT       Unique event identifier
USER_ID          INT       User identifier
MOVIE_ID         INT       Movie identifier
EVENT_TYPE       VARCHAR   PLAY/PAUSE/REPLAY/COMPLETE/RATING
EVENT_TIMESTAMP  DATETIME  Date and time of event
WATCH_MINUTES    INT       Minutes watched
USER_RATING      INT       Rating 1-5 (RATING events only)
```

### Movie Master Schema
```
MOVIE_ID          INT       Unique movie identifier
MOVIE_NAME        VARCHAR   Movie title
CATEGORY          VARCHAR   Genre (Action, Comedy, Drama etc.)
DURATION_MINUTES  INT       Total movie duration
```

### User Master Schema
```
USER_ID            INT       Unique user identifier
USER_NAME          VARCHAR   Display name
AGE                INT       User age
CITY               VARCHAR   User city
SUBSCRIPTION_TYPE  VARCHAR   Basic/Standard/Premium
```

---

## 🏗 Architecture

We followed the **Medallion Architecture** — Bronze → Silver → Gold.

```
┌─────────────────────────────────────────────────────┐
│  BRONZE LAYER — Raw Staging                         │
│  STG_WATCH_EVENTS_CLEAN                             │
│  STG_WATCH_EVENTS_DIRTY                             │
│  STG_MOVIE_MASTER                                   │
│  STG_USER_MASTER                                    │
└─────────────────────┬───────────────────────────────┘
                      │ IICS Cleansing (7 rules)
┌─────────────────────▼───────────────────────────────┐
│  SILVER LAYER — Cleansed Data                       │
│  TGT_CLEAN_WATCH_EVENTS                             │
│  TGT_REJECTED_WATCH_EVENTS                          │
│  TGT_DUPLICATE_WATCH_EVENTS                         │
└─────────────────────┬───────────────────────────────┘
                      │ IICS Transformation
┌─────────────────────▼───────────────────────────────┐
│  GOLD LAYER — Star Schema + KPIs                    │
│  FACT_WATCH_HISTORY                                 │
│  MOVIE_MASTER (DIM)                                 │
│  USER_MASTER (DIM)                                  │
│  KPI Output Tables                                  │
└─────────────────────────────────────────────────────┘
```

---

## 🧹 Data Cleansing Rules

Applied on `Watch_Events_Dirty.csv` using IICS Expression + Filter + Router:

| Rule | Condition | Action |
|---|---|---|
| 1 | Invalid EVENT_TYPE (UNKNOWN, null, 0) | Reject |
| 2 | Invalid or null EVENT_TIMESTAMP | Reject |
| 3 | Missing USER_ID or MOVIE_ID | Reject |
| 4 | Negative WATCH_MINUTES | Convert: ABS(WATCH_MINUTES) |
| 5 | WATCH_MINUTES > movie DURATION_MINUTES | Cap to movie duration |
| 6 | USER_RATING outside 1-5 | Reject |
| 7 | Duplicate: USER_ID + MOVIE_ID + EVENT_TIMESTAMP | Mark as duplicate |

### Dirty Data Analysis
```
Total rows:              418
Null USER_ID:            35  → rejected
Null MOVIE_ID:           19  → rejected
Invalid EVENT_TYPE:      25  → rejected (value: UNKNOWN)
Negative WATCH_MINUTES:  18  → converted to positive
Bad USER_RATING (> 5):   14  → rejected (value: 6)
Duplicates:              18  → duplicate table
```

### IICS Expression — Key Ports
```
IS_DUPLICATE         IIF(USER_ID = v_prev_user_id AND MOVIE_ID = v_prev_movie_id
                         AND EVENT_TIMESTAMP = v_prev_timestamp, 'Y', 'N')

IS_VALID_EVENT_TYPE  IIF(ISNULL(EVENT_TYPE) OR EVENT_TYPE = '0'
                         OR UPPER(EVENT_TYPE) = 'UNKNOWN'
                         OR INSTR('PLAY,PAUSE,REPLAY,COMPLETE,RATING',
                         UPPER(EVENT_TYPE)) = 0, 'N', 'Y')

IS_REJECT            IIF(IS_VALID_EVENT_TYPE='N' OR IS_VALID_TIMESTAMP='N'
                         OR IS_VALID_IDS='N' OR IS_RATING_VALID='N', 'Y', 'N')

CLEANED_WATCH_MIN    IIF(ISNULL(WATCH_MINUTES), 0,
                         IIF(WATCH_MINUTES < 0, ABS(WATCH_MINUTES),
                             IIF(ISNULL(LKP_DURATION), WATCH_MINUTES,
                                 IIF(WATCH_MINUTES > LKP_DURATION,
                                     LKP_DURATION, WATCH_MINUTES))))

REJECT_REASON        IIF(IS_VALID_EVENT_TYPE='N','Invalid EVENT_TYPE | ','')
                     || IIF(IS_VALID_TIMESTAMP='N','Invalid TIMESTAMP | ','')
                     || IIF(IS_VALID_IDS='N','Missing USER_ID or MOVIE_ID | ','')
                     || IIF(IS_RATING_VALID='N','USER_RATING outside 1-5 | ','')
```

---

## ⭐ Star Schema

```
                    ┌─────────────────┐
                    │   MOVIE_MASTER  │
                    │  movie_id (PK)  │
                    │  movie_name     │
                    │  category       │
                    │  duration_mins  │
                    └────────┬────────┘
                             │
┌──────────────┐    ┌────────▼──────────────────────┐
│  USER_MASTER │    │      FACT_WATCH_HISTORY        │
│  user_id(PK) │    │  event_id (PK)                 │
│  user_name   ├────►  user_id (FK)                  │
│  age         │    │  movie_id (FK)                 │
│  city        │    │  event_type                    │
│  sub_type    │    │  event_timestamp               │
└──────────────┘    │  watch_minutes                 │
                    │  user_rating                   │
                    │  month_num (derived)            │
                    │  engagement_score (derived)     │
                    │  completion_pct (derived)       │
                    │  completed_flag (derived)       │
                    └───────────────────────────────-─┘
```

### Derived Fields
```sql
month_num        = TO_CHAR(EVENT_TIMESTAMP, 'MM')
engagement_score = WATCH_MINUTES * IIF(ISNULL(USER_RATING), 1, USER_RATING)
completion_pct   = (WATCH_MINUTES / DURATION_MINUTES) * 100
completed_flag   = IIF(completion_pct >= 90, 1, 0)
```

---

## 🔄 IICS Mappings

| Mapping | Source | Target | Transformations |
|---|---|---|---|
| `m_ingest_watch_clean` | Watch_Events_Clean.csv | STG_WATCH_EVENTS_CLEAN | Source → Target |
| `m_ingest_watch_dirty` | Watch_Events_Dirty.csv | STG_WATCH_EVENTS_DIRTY | Source → Target |
| `m_ingest_movie_master` | Movie_Master.csv | STG_MOVIE_MASTER | Source → Target |
| `m_ingest_user_master` | User_Master.csv | STG_USER_MASTER | Source → Target |
| `m_cleanse_dirty_events` | STG_WATCH_EVENTS_DIRTY | TGT_CLEAN / TGT_REJECTED / TGT_DUPLICATE | Sorter → Lookup → Expression → Router |
| `m_load_fact_watch_history` | STG_WATCH_EVENTS_CLEAN | FACT_WATCH_HISTORY | Joiner → Expression → Target |
| `M_Category_TREND` | FACT + MOVIE_MASTER | TGT_CATEGORY_TREND | Joiner → Expression → Aggregator → Sorter → Rank → Target |
| `M_User_Engagement` | FACT + USER_MASTER | TGT_USER_ENGAGEMENT | Joiner → Expression → Aggregator → Target |
| `M_Category_Popularity` | FACT + MOVIE_MASTER | TGT_CATEGORY_POPULARITY | Joiner → Expression → Aggregator → Sorter → Target |
| `M_Watch_Behavior` | FACT + USER + MOVIE | TGT_USER_WATCH_BEHAVIOR | Joiner x2 → Expression → Aggregator → Target |
| `M_City_Viewership` | FACT + USER_MASTER | TGT_CITY_WISE_VIEWERSHIP | Joiner → Expression → Aggregator → Target |

---

## 📊 KPI Outputs

### Task 1 — TGT_CATEGORY_POPULARITY.csv
Category and month-wise popularity sorted by highest watch time.
```
MONTH | CATEGORY | WATCH_TIME | ENGAGEMENT_SCORE
01    | Drama    | 2129       | 8133
01    | Doc...   | 1657       | 6271
```

### Task 2 — TGT_USER_ENGAGEMENT_SCORE.csv
Total engagement score per user per month. `Engagement = WATCH_MINUTES × USER_RATING (null = 1)`
```
MONTH | USER_ID | USER_NAME | ENGAGEMENT_SCORE
01    | 1044    | User_1044 | 233
```

### Task 3 — TGT_CATEGORY_TREND.csv
Top 3 categories per month by total watch time.
```
MONTH | CATEGORY | NO_OF_USERS | WATCH_TIME
01    | Drama    | 34          | 2129
01    | Doc...   | 36          | 1657
01    | Romance  | 26          | 1535
02    | Drama    | 32          | 1751
02    | Comedy   | 23          | 1586
02    | Action   | 21          | 1400
```

### Task 4 — TGT_USER_WATCH_BEHAVIOR.csv
Watch time per user per category per month.
```
MONTH | CATEGORY | USER_ID | USER_NAME | WATCH_TIME
01    | Comedy   | 1033    | User_1033 | 633
```

### Task 5 — TGT_CITY_WISE_VIEWERSHIP.csv
City and month-wise viewership summary.
```
MONTH | CITY | NO_OF_USERS | WATCH_TIME
01    | Pune | 18          | 2100
```

---

## 🚀 Pipeline Flow

```
START
  │
  ▼
Truncate all staging tables
  │
  ├──► Load Watch_Events_Clean  ──► STG_WATCH_EVENTS_CLEAN
  ├──► Load Watch_Events_Dirty  ──► STG_WATCH_EVENTS_DIRTY
  ├──► Load Movie_Master        ──► STG_MOVIE_MASTER
  └──► Load User_Master         ──► STG_USER_MASTER
                │
                ▼
        Cleanse dirty events
                │
        ┌───────┼───────┐
        ▼       ▼       ▼
      CLEAN  REJECTED  DUPLICATE
        │
        ▼
  FACT_WATCH_HISTORY
  (Joiner + Expression + derived fields)
        │
        ▼
  ┌─────┼─────┬─────┬─────┐
  ▼     ▼     ▼     ▼     ▼
KPI1  KPI2  KPI3  KPI4  KPI5
  │
  ▼
END
```

---

## ▶️ How to Run

### Prerequisites
```
1. Informatica IICS account with Secure Agent running
2. SQL Server or Oracle DB with connections configured
3. All 4 CSV files available in flat file connection path
```

### Steps
```
1. Create all staging tables in DB
   (or use Create at Runtime in IICS Target)

2. Run ingestion mappings (Bronze layer):
   mt_ingest_watch_clean
   mt_ingest_watch_dirty
   mt_ingest_movie_master
   mt_ingest_user_master

3. Run cleansing mapping (Silver layer):
   mt_cleanse_dirty_events

4. Run FACT mapping (Gold layer):
   mt_load_fact_watch_history

5. Run KPI mappings:
   mt_category_popularity
   mt_user_engagement_score
   mt_category_trend
   mt_watch_behavior
   mt_city_wise_viewership
```

### Expected Row Counts
```
STG_WATCH_EVENTS_CLEAN     → 400 rows
STG_WATCH_EVENTS_DIRTY     → 418 rows
STG_MOVIE_MASTER           → 150 rows
STG_USER_MASTER            → 120 rows
TGT_CLEAN_WATCH_EVENTS     → ~289 rows
TGT_REJECTED_WATCH_EVENTS  → ~111 rows
TGT_DUPLICATE_WATCH_EVENTS → ~18 rows
FACT_WATCH_HISTORY         → 400 rows
TGT_CATEGORY_TREND         → 6 rows (top 3 x 2 months)
```

---

## 👥 Team

| Member | Role |
|---|---|
| Member 1 | IICS ingestion mappings + Taskflow |
| Member 2 | IICS cleansing mapping + data quality |
| Member 3 | IICS FACT + KPI mappings |
| Member 4 | SQL Server tables + PL/SQL + documentation |

---

## 📌 Key Design Decisions

**Why Star Schema over flat table?**
Dimension data like movie name and category would repeat with every watch event. Star schema stores it once in dimension tables referenced via foreign keys — eliminating redundancy and improving query performance.

**Why Joiner over Lookup for KPIs?**
KPI mappings need CATEGORY as a full column flowing into Aggregator for GROUP BY. Joiner combines two active data streams making all columns available downstream. Lookup is more appropriate for fetching a single reference value.

**Why Medallion Architecture?**
Bronze preserves raw data for reprocessing. Silver cleanses without destroying original. Gold is optimised for analytics. Each layer has a clear purpose and can be rerun independently if issues arise.

**Why variable ports for duplicate detection?**
IICS does not support LAG() window functions natively. Variable ports simulate the previous row comparison by storing the current row's composite key and comparing it on the next row after sorting.

---

*Built with IICS + SQL Server | HCLTech Campus Hackathon 2026*
