# OTT Movie Streaming ETL – Complete IICS Solution Guide

---

## 1. PROJECT OVERVIEW

| Item | Detail |
|---|---|
| Tool | Informatica Intelligent Cloud Services (IICS) – Cloud Data Integration (CDI) |
| Database | Oracle |
| Source Files | Watch_Events_Clean.csv, Watch_Events_Dirty.csv, Movie_Master.csv, User_Master.csv |
| Flow Type | Batch ETL via Mappings + Taskflow |

---

## 2. ORACLE TABLE CREATION SCRIPTS

### Staging Tables (Step 1 – Load CSV → DB)

```sql
-- Staging: Clean Watch Events
CREATE TABLE STG_WATCH_EVENTS_CLEAN (
    EVENT_ID        NUMBER,
    USER_ID         NUMBER,
    MOVIE_ID        NUMBER,
    EVENT_TYPE      VARCHAR2(20),
    EVENT_TIMESTAMP TIMESTAMP,
    WATCH_MINUTES   NUMBER,
    USER_RATING     NUMBER
);

-- Staging: Dirty Watch Events
CREATE TABLE STG_WATCH_EVENTS_DIRTY (
    EVENT_ID        NUMBER,
    USER_ID         NUMBER,
    MOVIE_ID        NUMBER,
    EVENT_TYPE      VARCHAR2(20),
    EVENT_TIMESTAMP TIMESTAMP,
    WATCH_MINUTES   NUMBER,
    USER_RATING     NUMBER
);

-- Staging: Movie Master
CREATE TABLE STG_MOVIE_MASTER (
    MOVIE_ID          NUMBER PRIMARY KEY,
    MOVIE_NAME        VARCHAR2(200),
    CATEGORY          VARCHAR2(50),
    DURATION_MINUTES  NUMBER
);

-- Staging: User Master
CREATE TABLE STG_USER_MASTER (
    USER_ID            NUMBER PRIMARY KEY,
    USER_NAME          VARCHAR2(100),
    AGE                NUMBER,
    CITY               VARCHAR2(100),
    SUBSCRIPTION_TYPE  VARCHAR2(20)
);
```

### Cleansing Output Tables (Step 3)

```sql
-- Clean output from dirty file
CREATE TABLE TGT_CLEAN_WATCH_EVENTS (
    EVENT_ID        NUMBER,
    USER_ID         NUMBER,
    MOVIE_ID        NUMBER,
    EVENT_TYPE      VARCHAR2(20),
    EVENT_TIMESTAMP TIMESTAMP,
    WATCH_MINUTES   NUMBER,
    USER_RATING     NUMBER
);

-- Rejected records with reason
CREATE TABLE TGT_REJECTED_WATCH_EVENTS (
    EVENT_ID        NUMBER,
    USER_ID         NUMBER,
    MOVIE_ID        NUMBER,
    EVENT_TYPE      VARCHAR2(20),
    EVENT_TIMESTAMP TIMESTAMP,
    WATCH_MINUTES   NUMBER,
    USER_RATING     NUMBER,
    REJECT_REASON   VARCHAR2(300)
);

-- Duplicate records
CREATE TABLE TGT_DUPLICATE_WATCH_EVENTS (
    EVENT_ID        NUMBER,
    USER_ID         NUMBER,
    MOVIE_ID        NUMBER,
    EVENT_TYPE      VARCHAR2(20),
    EVENT_TIMESTAMP TIMESTAMP,
    WATCH_MINUTES   NUMBER,
    USER_RATING     NUMBER
);
```

### Fact Table (Step 5)

```sql
CREATE TABLE FACT_WATCH_HISTORY (
    EVENT_ID          NUMBER,
    USER_ID           NUMBER,
    MOVIE_ID          NUMBER,
    EVENT_TYPE        VARCHAR2(20),
    EVENT_TIMESTAMP   TIMESTAMP,
    WATCH_MINUTES     NUMBER,
    USER_RATING       NUMBER,
    COMPLETION_PCT    NUMBER(6,2),   -- derived
    IS_COMPLETED      VARCHAR2(1),   -- derived flag
    ENGAGEMENT_SCORE  NUMBER,        -- derived
    SOURCE            VARCHAR2(10)   -- 'CLEAN' or 'DIRTY'
);
```

### Final Target Tables (Tasks 1–6)

```sql
-- Task 1
CREATE TABLE TGT_CATEGORY_POPULARITY (
    MONTH            VARCHAR2(2),
    CATEGORY         VARCHAR2(50),
    WATCH_TIME       NUMBER,
    ENGAGEMENT_SCORE NUMBER
);

-- Task 2
CREATE TABLE TGT_USER_ENGAGEMENT_SCORE (
    MONTH            VARCHAR2(2),
    USER_ID          NUMBER,
    USER_NAME        VARCHAR2(100),
    ENGAGEMENT_SCORE NUMBER
);

-- Task 3
CREATE TABLE TGT_CATEGORY_TREND (
    MONTH        VARCHAR2(2),
    CATEGORY     VARCHAR2(50),
    NO_OF_USERS  NUMBER,
    WATCH_TIME   NUMBER
);

-- Task 4
CREATE TABLE TGT_USER_WATCH_BEHAVIOR (
    MONTH       VARCHAR2(2),
    CATEGORY    VARCHAR2(50),
    USER_ID     NUMBER,
    USER_NAME   VARCHAR2(100),
    WATCH_TIME  NUMBER,
    AVG_RATING  NUMBER(5,2)
);

-- Task 6
CREATE TABLE TGT_CITY_WISE_VIEWERSHIP (
    MONTH           VARCHAR2(2),
    CITY            VARCHAR2(100),
    NO_OF_USERS     NUMBER,
    TOTAL_WATCH_TIME NUMBER,
    TOTAL_EVENTS    NUMBER
);
```

---

## 3. IICS ARCHITECTURE & MAPPING PLAN

```
TASKFLOW: TF_OTT_MASTER_FLOW
│
├── MT_Load_CSVs_to_Staging         (Mapping Task 1 – 4 mappings)
├── MT_Cleanse_Dirty_Events         (Mapping Task 2)
├── MT_Build_Fact_Watch_History     (Mapping Task 3)
├── MT_Category_Popularity          (Mapping Task 4 → Task 1 output)
├── MT_User_Engagement_Score        (Mapping Task 5 → Task 2 output)
├── MT_Category_Trend               (Mapping Task 6 → Task 3 output)
├── MT_User_Watch_Behavior          (Mapping Task 7 → Task 4 output)
└── MT_City_Wise_Viewership         (Mapping Task 8 → Task 6 output)
```

---

## 4. DETAILED MAPPING DESIGNS

---

### MAPPING 1: m_Load_CSVs_to_Staging
**Purpose:** Load all 4 CSV flat files into Oracle staging tables.
**Repeat 4 times (one per CSV). Show once as template.**

#### Transformations Used:
| # | Transformation | Why Used |
|---|---|---|
| 1 | **Source** (Flat File) | Read CSV data using Flat File connection in IICS |
| 2 | **Target** (Oracle via JDBC) | Write records into Oracle staging table |

#### Configuration:
- **Source:** Flat File connection, delimiter = comma, First row as header = Yes
- **Target:** Insert mode, Truncate target before load = Yes

#### Field Mapping (example: Watch_Events_Clean → STG_WATCH_EVENTS_CLEAN):
| Source Field | → | Target Field |
|---|---|---|
| EVENT_ID | → | EVENT_ID |
| USER_ID | → | USER_ID |
| MOVIE_ID | → | MOVIE_ID |
| EVENT_TYPE | → | EVENT_TYPE |
| EVENT_TIMESTAMP | → | EVENT_TIMESTAMP |
| WATCH_MINUTES | → | WATCH_MINUTES |
| USER_RATING | → | USER_RATING |

> Repeat same structure for Movie_Master and User_Master CSVs.

---

### MAPPING 2: m_Cleanse_Dirty_Watch_Events
**Source:** STG_WATCH_EVENTS_DIRTY
**Targets:** TGT_CLEAN_WATCH_EVENTS, TGT_REJECTED_WATCH_EVENTS, TGT_DUPLICATE_WATCH_EVENTS

#### Transformations Used:
| # | Transformation | Why Used |
|---|---|---|
| 1 | **Source** | Read from STG_WATCH_EVENTS_DIRTY |
| 2 | **Sorter** | Sort by USER_ID, MOVIE_ID, EVENT_TIMESTAMP to identify duplicates |
| 3 | **Expression** | Apply cleansing logic – fix negatives, cap watch minutes, build reject reason, detect duplicates |
| 4 | **Lookup** | Lookup STG_MOVIE_MASTER to get DURATION_MINUTES for capping |
| 5 | **Router** | Route records to 3 groups: Duplicate / Rejected / Clean |
| 6 | **Target ×3** | Write to each of the 3 output tables |

#### Expression Transformation – Extra Fields & Config:

| Output Port Name | Data Type | Expression / Config | Purpose |
|---|---|---|---|
| `v_prev_user_id` | Number | Variable port: `USER_ID` (carry previous row) | Duplicate detection |
| `v_prev_movie_id` | Number | Variable port: `MOVIE_ID` | Duplicate detection |
| `v_prev_timestamp` | String | Variable port: `TO_CHAR(EVENT_TIMESTAMP,'YYYY-MM-DD HH24:MI:SS')` | Duplicate detection |
| `IS_DUPLICATE` | String(1) | `IIF(USER_ID = v_prev_user_id AND MOVIE_ID = v_prev_movie_id AND TO_CHAR(EVENT_TIMESTAMP,'YYYY-MM-DD HH24:MI:SS') = v_prev_timestamp, 'Y', 'N')` | Flag duplicate rows |
| `CLEANED_WATCH_MIN` | Number | `IIF(WATCH_MINUTES < 0, ABS(WATCH_MINUTES), IIF(WATCH_MINUTES > LKP_DURATION, LKP_DURATION, WATCH_MINUTES))` | Fix negative & cap to duration |
| `CLEANED_RATING` | Number | `IIF(ISNULL(USER_RATING) OR USER_RATING < 1 OR USER_RATING > 5, NULL, USER_RATING)` | Null out invalid ratings |
| `IS_VALID_EVENT_TYPE` | String(1) | `IIF(INSTR('PLAY,PAUSE,REPLAY,COMPLETE,RATING', EVENT_TYPE) > 0, 'Y', 'N')` | Validate event type |
| `IS_VALID_TIMESTAMP` | String(1) | `IIF(ISNULL(EVENT_TIMESTAMP), 'N', 'Y')` | Check null timestamp |
| `IS_VALID_IDS` | String(1) | `IIF(ISNULL(USER_ID) OR ISNULL(MOVIE_ID), 'N', 'Y')` | Check missing IDs |
| `IS_RATING_VALID` | String(1) | `IIF(EVENT_TYPE = 'RATING' AND (ISNULL(USER_RATING) OR USER_RATING < 1 OR USER_RATING > 5), 'N', 'Y')` | Rating out of 1-5 |
| `REJECT_REASON` | String(300) | `IIF(IS_VALID_EVENT_TYPE='N','Invalid EVENT_TYPE\|', '') \|\| IIF(IS_VALID_TIMESTAMP='N','Invalid EVENT_TIMESTAMP\|', '') \|\| IIF(IS_VALID_IDS='N','Missing USER_ID or MOVIE_ID\|', '') \|\| IIF(IS_RATING_VALID='N','USER_RATING outside 1-5\|','')` | Combined rejection message |
| `IS_REJECT` | String(1) | `IIF(IS_VALID_EVENT_TYPE='N' OR IS_VALID_TIMESTAMP='N' OR IS_VALID_IDS='N' OR IS_RATING_VALID='N', 'Y', 'N')` | Overall reject flag |

#### Lookup Transformation Config:
- **Lookup Table:** STG_MOVIE_MASTER
- **Lookup Condition:** `MOVIE_ID = MOVIE_ID`
- **Return Port:** `LKP_DURATION` ← DURATION_MINUTES
- **No Match Policy:** Return NULL (handle in Expression)

#### Router Transformation Groups:
| Group Name | Condition |
|---|---|
| `GRP_DUPLICATE` | `IS_DUPLICATE = 'Y'` |
| `GRP_REJECTED` | `IS_DUPLICATE = 'N' AND IS_REJECT = 'Y'` |
| `GRP_CLEAN` | Default (all remaining) |

#### Target Field Mappings:

**→ TGT_DUPLICATE_WATCH_EVENTS** (GRP_DUPLICATE):
All original fields passed through as-is.

**→ TGT_REJECTED_WATCH_EVENTS** (GRP_REJECTED):
| Expression Port | → | Target Column |
|---|---|---|
| EVENT_ID | → | EVENT_ID |
| USER_ID | → | USER_ID |
| MOVIE_ID | → | MOVIE_ID |
| EVENT_TYPE | → | EVENT_TYPE |
| EVENT_TIMESTAMP | → | EVENT_TIMESTAMP |
| WATCH_MINUTES | → | WATCH_MINUTES |
| USER_RATING | → | USER_RATING |
| REJECT_REASON | → | REJECT_REASON |

**→ TGT_CLEAN_WATCH_EVENTS** (GRP_CLEAN):
| Expression Port | → | Target Column |
|---|---|---|
| EVENT_ID | → | EVENT_ID |
| USER_ID | → | USER_ID |
| MOVIE_ID | → | MOVIE_ID |
| EVENT_TYPE | → | EVENT_TYPE |
| EVENT_TIMESTAMP | → | EVENT_TIMESTAMP |
| CLEANED_WATCH_MIN | → | WATCH_MINUTES |
| CLEANED_RATING | → | USER_RATING |

---

### MAPPING 3: m_Build_Fact_Watch_History
**Purpose:** Merge clean events from both files, enrich with derived fields, load to FACT table.

#### Transformations Used:
| # | Transformation | Why Used |
|---|---|---|
| 1 | **Source ×2** | Read STG_WATCH_EVENTS_CLEAN and TGT_CLEAN_WATCH_EVENTS (dirty-cleaned) |
| 2 | **Expression ×2** | Add SOURCE tag ('CLEAN' or 'DIRTY') before union |
| 3 | **Union** | Merge both clean streams into one pipeline |
| 4 | **Lookup** | Get DURATION_MINUTES from STG_MOVIE_MASTER |
| 5 | **Expression** | Compute derived fields: COMPLETION_PCT, IS_COMPLETED, ENGAGEMENT_SCORE |
| 6 | **Target** | Write to FACT_WATCH_HISTORY |

#### Expression (pre-Union) – Add Source Tag:
| Port | Expression |
|---|---|
| `SOURCE` | `'CLEAN'` (for clean file stream) / `'DIRTY'` (for dirty-cleaned stream) |

#### Expression (post-Union + Lookup) – Derived Fields:
| Output Port | Expression | Purpose |
|---|---|---|
| `COMPLETION_PCT` | `IIF(LKP_DURATION > 0, (WATCH_MINUTES * 100.0) / LKP_DURATION, 0)` | % of movie watched |
| `IS_COMPLETED` | `IIF(COMPLETION_PCT >= 90, 'Y', 'N')` | Completion flag |
| `EFF_RATING` | `IIF(ISNULL(USER_RATING), 1, USER_RATING)` | Null rating treated as 1 |
| `ENGAGEMENT_SCORE` | `WATCH_MINUTES * EFF_RATING` | Engagement metric |

#### Target Field Mapping → FACT_WATCH_HISTORY:
| Expression Port | → | Target Column |
|---|---|---|
| EVENT_ID | → | EVENT_ID |
| USER_ID | → | USER_ID |
| MOVIE_ID | → | MOVIE_ID |
| EVENT_TYPE | → | EVENT_TYPE |
| EVENT_TIMESTAMP | → | EVENT_TIMESTAMP |
| WATCH_MINUTES | → | WATCH_MINUTES |
| USER_RATING | → | USER_RATING |
| COMPLETION_PCT | → | COMPLETION_PCT |
| IS_COMPLETED | → | IS_COMPLETED |
| ENGAGEMENT_SCORE | → | ENGAGEMENT_SCORE |
| SOURCE | → | SOURCE |

---

### MAPPING 4: m_Category_Popularity
**Target:** TGT_CATEGORY_POPULARITY (Task 1)
**Source:** FACT_WATCH_HISTORY joined with STG_MOVIE_MASTER

#### Transformations:
| # | Transformation | Why Used |
|---|---|---|
| 1 | **Source** | Read FACT_WATCH_HISTORY |
| 2 | **Joiner** | Join with STG_MOVIE_MASTER on MOVIE_ID to get CATEGORY |
| 3 | **Expression** | Extract MONTH from EVENT_TIMESTAMP |
| 4 | **Aggregator** | Group by MONTH + CATEGORY, sum WATCH_MINUTES and ENGAGEMENT_SCORE |
| 5 | **Sorter** | Sort by MONTH ASC, WATCH_TIME DESC (highest watch time first) |
| 6 | **Target** | Write to TGT_CATEGORY_POPULARITY |

#### Expression Ports:
| Port | Expression |
|---|---|
| `MONTH` | `TO_CHAR(EVENT_TIMESTAMP, 'MM')` |

#### Aggregator Config:
| Port | Type | Expression |
|---|---|---|
| MONTH | Group By | — |
| CATEGORY | Group By | — |
| WATCH_TIME | Output | `SUM(WATCH_MINUTES)` |
| ENGAGEMENT_SCORE | Output | `SUM(ENGAGEMENT_SCORE)` |

#### Sorter Config:
| Column | Order |
|---|---|
| MONTH | Ascending |
| WATCH_TIME | Descending |

#### Target Field Mapping:
| Aggregator Port | → | Target Column |
|---|---|---|
| MONTH | → | MONTH |
| CATEGORY | → | CATEGORY |
| WATCH_TIME | → | WATCH_TIME |
| ENGAGEMENT_SCORE | → | ENGAGEMENT_SCORE |

---

### MAPPING 5: m_User_Engagement_Score
**Target:** TGT_USER_ENGAGEMENT_SCORE (Task 2)

#### Transformations:
| # | Transformation | Why Used |
|---|---|---|
| 1 | **Source** | Read FACT_WATCH_HISTORY |
| 2 | **Lookup** | Lookup STG_USER_MASTER to get USER_NAME |
| 3 | **Expression** | Extract MONTH, compute EFF_RATING (null=1), compute per-record engagement |
| 4 | **Aggregator** | Group by MONTH + USER_ID, sum engagement score |
| 5 | **Target** | Write to TGT_USER_ENGAGEMENT_SCORE |

#### Expression Ports:
| Port | Expression |
|---|---|
| `MONTH` | `TO_CHAR(EVENT_TIMESTAMP, 'MM')` |
| `EFF_RATING` | `IIF(ISNULL(USER_RATING), 1, USER_RATING)` |
| `ROW_ENGAGEMENT` | `WATCH_MINUTES * EFF_RATING` |

#### Aggregator Config:
| Port | Type | Expression |
|---|---|---|
| MONTH | Group By | — |
| USER_ID | Group By | — |
| USER_NAME | Group By | — (from Lookup) |
| ENGAGEMENT_SCORE | Output | `SUM(ROW_ENGAGEMENT)` |

#### Target Field Mapping:
| Port | → | Target Column |
|---|---|---|
| MONTH | → | MONTH |
| USER_ID | → | USER_ID |
| USER_NAME | → | USER_NAME |
| ENGAGEMENT_SCORE | → | ENGAGEMENT_SCORE |

---

### MAPPING 6: m_Category_Trend
**Target:** TGT_CATEGORY_TREND (Task 3)
**Logic:** Top 3 categories per month by total watch time

#### Transformations:
| # | Transformation | Why Used |
|---|---|---|
| 1 | **Source** | Read FACT_WATCH_HISTORY |
| 2 | **Joiner** | Join STG_MOVIE_MASTER on MOVIE_ID to get CATEGORY |
| 3 | **Expression** | Extract MONTH |
| 4 | **Aggregator** | Group by MONTH + CATEGORY → SUM(WATCH_MINUTES), COUNT(DISTINCT USER_ID) |
| 5 | **Rank** | Rank by WATCH_TIME within each MONTH partition, keep Top 3 |
| 6 | **Filter** | RANKINDEX <= 3 |
| 7 | **Target** | Write to TGT_CATEGORY_TREND |

#### Aggregator Config:
| Port | Type | Expression |
|---|---|---|
| MONTH | Group By | — |
| CATEGORY | Group By | — |
| WATCH_TIME | Output | `SUM(WATCH_MINUTES)` |
| NO_OF_USERS | Output | `COUNT(DISTINCT USER_ID)` |

#### Rank Transformation Config:
- **Rank By:** WATCH_TIME (Descending)
- **Group By:** MONTH
- **Top/Bottom:** Top
- **Number of Ranks:** 3
- **Rank Port:** RANKINDEX

#### Target Field Mapping:
| Port | → | Target Column |
|---|---|---|
| MONTH | → | MONTH |
| CATEGORY | → | CATEGORY |
| NO_OF_USERS | → | NO_OF_USERS |
| WATCH_TIME | → | WATCH_TIME |

---

### MAPPING 7: m_User_Watch_Behavior
**Target:** TGT_USER_WATCH_BEHAVIOR (Task 4)
**Logic:** Per month, per user, per category – total watch time and average rating

#### Transformations:
| # | Transformation | Why Used |
|---|---|---|
| 1 | **Source** | Read FACT_WATCH_HISTORY (filter EVENT_TYPE = 'RATING') |
| 2 | **Filter** | Keep only RATING events for avg rating computation |
| 3 | **Joiner** | Join STG_MOVIE_MASTER to get CATEGORY |
| 4 | **Lookup** | Lookup STG_USER_MASTER for USER_NAME |
| 5 | **Expression** | Extract MONTH |
| 6 | **Aggregator** | Group by MONTH + CATEGORY + USER_ID → SUM watch time, AVG rating |
| 7 | **Target** | Write to TGT_USER_WATCH_BEHAVIOR |

#### Aggregator Config:
| Port | Type | Expression |
|---|---|---|
| MONTH | Group By | — |
| CATEGORY | Group By | — |
| USER_ID | Group By | — |
| USER_NAME | Group By | — |
| WATCH_TIME | Output | `SUM(WATCH_MINUTES)` |
| AVG_RATING | Output | `AVG(USER_RATING)` |

#### Target Field Mapping:
| Port | → | Target Column |
|---|---|---|
| MONTH | → | MONTH |
| CATEGORY | → | CATEGORY |
| USER_ID | → | USER_ID |
| USER_NAME | → | USER_NAME |
| WATCH_TIME | → | WATCH_TIME |
| AVG_RATING | → | AVG_RATING |

---

### MAPPING 8: m_City_Wise_Viewership
**Target:** TGT_CITY_WISE_VIEWERSHIP (Task 6)

#### Transformations:
| # | Transformation | Why Used |
|---|---|---|
| 1 | **Source** | Read FACT_WATCH_HISTORY |
| 2 | **Lookup** | Lookup STG_USER_MASTER on USER_ID → get CITY |
| 3 | **Expression** | Extract MONTH |
| 4 | **Aggregator** | Group by MONTH + CITY → count users, sum watch time, count events |
| 5 | **Target** | Write to TGT_CITY_WISE_VIEWERSHIP |

#### Aggregator Config:
| Port | Type | Expression |
|---|---|---|
| MONTH | Group By | — |
| CITY | Group By | — |
| NO_OF_USERS | Output | `COUNT(DISTINCT USER_ID)` |
| TOTAL_WATCH_TIME | Output | `SUM(WATCH_MINUTES)` |
| TOTAL_EVENTS | Output | `COUNT(EVENT_ID)` |

#### Target Field Mapping:
| Port | → | Target Column |
|---|---|---|
| MONTH | → | MONTH |
| CITY | → | CITY |
| NO_OF_USERS | → | NO_OF_USERS |
| TOTAL_WATCH_TIME | → | TOTAL_WATCH_TIME |
| TOTAL_EVENTS | → | TOTAL_EVENTS |

---

## 5. IICS TASKFLOW DESIGN

**Taskflow Name:** TF_OTT_Master_Flow

```
[Start]
   ↓
[MT_Load_Watch_Events_Clean]  ──┐
[MT_Load_Watch_Events_Dirty]    ├──→ (Run in parallel)
[MT_Load_Movie_Master]          |
[MT_Load_User_Master]       ────┘
   ↓
[MT_Cleanse_Dirty_Events]
   ↓
[MT_Build_Fact_Watch_History]
   ↓
[MT_Category_Popularity]    ──┐
[MT_User_Engagement_Score]    ├──→ (Run in parallel)
[MT_Category_Trend]           |
[MT_User_Watch_Behavior]      |
[MT_City_Wise_Viewership]  ───┘
   ↓
[End]
```

**Note:** In IICS Taskflow, use "And" gateway after parallel branches to wait for all tasks before proceeding.

---

## 6. TRANSFORMATION SUMMARY – WHY EACH IS USED

| Transformation | IICS Name | Reason Used |
|---|---|---|
| Source | Source | Reads flat file or Oracle table; required start of every mapping |
| Expression | Expression | Computes derived fields, cleans data, applies business logic inline |
| Lookup | Lookup | Enriches pipeline data from reference/master tables without a full join |
| Joiner | Joiner | Joins two active streams (e.g., fact + dimension) on a key |
| Filter | Filter | Drops rows not meeting condition (e.g., non-RATING events) |
| Router | Router | Splits one stream into multiple groups (clean/reject/duplicate) |
| Aggregator | Aggregator | Performs GROUP BY with SUM, COUNT, AVG across records |
| Sorter | Sorter | Orders data before writing or ranking; needed for duplicate detection via variable ports |
| Rank | Rank | Selects top-N records within a partition (Top 3 categories per month) |
| Union | Union | Merges two or more streams of same schema into one |
| Target | Target | Writes data to Oracle table (Insert/Update/Upsert mode) |

---

## 7. IICS CONNECTIONS NEEDED

| Connection Name | Type | Used For |
|---|---|---|
| FF_OTT_Source | Flat File | Read 4 CSV source files |
| JDBC_Oracle_OTT | JDBC (Oracle) | Read/Write staging, fact, target Oracle tables |

---

## 8. QUICK CHECKLIST

- [ ] Create all Oracle tables (DDL above)
- [ ] Configure FF and JDBC connections in IICS Admin Console
- [ ] Build m_Load_CSVs_to_Staging (4 mappings)
- [ ] Build m_Cleanse_Dirty_Watch_Events with Router → 3 targets
- [ ] Build m_Build_Fact_Watch_History with Union + Expression
- [ ] Build Mappings 4–8 for all 5 KPI outputs
- [ ] Create Taskflow TF_OTT_Master_Flow with correct sequencing
- [ ] Schedule Taskflow via IICS Scheduler (monthly or on-demand)
