# 🏅 Olympics SQL Solutions

> **A complete, well-documented README** containing SQL solutions and explanations for 8 analytical problems run against a 120-year Olympics dataset.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Datasets](#datasets)
3. [Prerequisites & Setup](#prerequisites--setup)
4. [Importing the CSV files into SQL Server](#importing-the-csv-files-into-sql-server)
5. [Database Schema & Notes](#database-schema--notes)
6. [SQL Solutions (Problems & Queries)](#sql-solutions-problems--queries)

   * [Problem 1 — Team with maximum gold medals](#problem-1-—-team-with-maximum-gold-medals)
   * [Problem 2 — Team silver summary + year of max silver](#problem-2-—-team-silver-summary--year-of-max-silver)
   * [Problem 3 — Top-only-gold player (never silver/bronze)](#problem-3-—-top-only-gold-player-never-silverbronze)
   * [Problem 4 — Yearly top gold winners (ties comma-separated)](#problem-4-—-yearly-top-gold-winners-ties-comma-separated)
   * [Problem 5 — India: first Gold / Silver / Bronze (year & event)](#problem-5-—-india-first-gold--silver--bronze-year--event)
   * [Problem 6 — Players with golds in both Summer & Winter](#problem-6-—-players-with-golds-in-both-summer--winter)
   * [Problem 7 — Players with Gold, Silver & Bronze in a single Olympics](#problem-7-—-players-with-gold-silver--bronze-in-a-single-olympics)
   * [Problem 8 — Players with golds in 3 consecutive Summer Olympics (same event) from 2000 onwards](#problem-8-—-players-with-golds-in-3-consecutive-summer-olympics-same-event-from-2000-onwards)
7. [Performance tips & indexing suggestions](#performance-tips--indexing-suggestions)
8. [How to reproduce & run the queries locally](#how-to-reproduce--run-the-queries-locally)
9. [Contributing & License](#contributing--license)
10. [Contact](#contact)

---

## Project Overview

This repo contains SQL solutions for common analytical questions run on a 120-year Olympics dataset. The solutions are written for **Microsoft SQL Server (T-SQL)** and assume two CSV files (provided in the zip): `athletes.csv` and `athlete_events.csv`.

* `athletes.csv` — master table of athletes (contains `id`, `name`, ...)
* `athlete_events.csv` — events data (contains `athlete_id` referencing `athletes.id`, `year`, `sport`, `event`, `medal`, `team`, `season`, etc.)

All queries in the **SQL Solutions** section are ready-to-use. Copy–paste into SSMS or Azure Data Studio after importing the CSVs.

---

## Datasets

* **athletes.csv** — columns expected (at minimum): `id`, `name`, `sex`, `height`, `weight`, `team`, `noc`, `dob` (if present)
* **athlete\_events.csv** — columns expected (at minimum): `athlete_id`, `year`, `city`, `sport`, `event`, `medal`, `team`, `season`

> *Note:* `athlete_id` in `athlete_events` maps to `id` in `athletes`.

---

## Prerequisites & Setup

* Microsoft SQL Server (2016+ recommended)
* SQL Server Management Studio (SSMS) or Azure Data Studio
* Permissions to create databases and import CSV files

Optional but helpful:

* PowerShell or the SQL Server Import and Export Wizard for bulk import.

---

## Importing the CSV files into SQL Server

### Option A — SQL Server Import and Export Wizard (GUI)

1. Open SSMS → connect to your instance.
2. Right-click a database (or create one) → Tasks → Import Data.
3. Choose Flat File Source → select `athletes.csv`, configure delimiter, preview, and create destination table `dbo.athletes`.
4. Repeat for `athlete_events.csv` → destination `dbo.athlete_events`.
5. After import, run quick `SELECT TOP 10 * FROM dbo.athlete_events` to validate.

### Option B — BULK INSERT (T-SQL)

```sql
-- Example (adjust path, columns, and format):
BULK INSERT dbo.athletes
FROM 'C:\path\to\athletes.csv'
WITH (
  FIRSTROW = 2,
  FIELDTERMINATOR = ',',
  ROWTERMINATOR = '\n',
  CODEPAGE = '65001'
);

BULK INSERT dbo.athlete_events
FROM 'C:\path\to\athlete_events.csv'
WITH (
  FIRSTROW = 2,
  FIELDTERMINATOR = ',',
  ROWTERMINATOR = '\n',
  CODEPAGE = '65001'
);
```

> If you have commas inside quoted values, ensure your CSV is well-formed or use the Import Wizard.

---

## Database Schema & Notes

Recommended minimal schema (types are suggestions):

```sql
CREATE TABLE dbo.athletes (
  id INT PRIMARY KEY,
  name NVARCHAR(255),
  sex CHAR(1),
  height INT NULL,
  weight INT NULL,
  team NVARCHAR(255),
  noc NVARCHAR(10)
);

CREATE TABLE dbo.athlete_events (
  athlete_id INT,
  year INT,
  city NVARCHAR(100),
  sport NVARCHAR(255),
  event NVARCHAR(255),
  medal NVARCHAR(10), -- e.g. 'Gold','Silver','Bronze','NA'
  team NVARCHAR(255),
  season NVARCHAR(10)
);
```

Make sure `athlete_events.athlete_id` values exist in `athletes.id`.

---

## SQL Solutions (Problems & Queries)

> All queries below assume `dbo.athletes` and `dbo.athlete_events` are loaded and that `athlete_events.athlete_id` maps to `athletes.id`.

---

### Problem 1 — Team with maximum gold medals

**Goal:** Which team has won the maximum gold medals over the years.

```sql
-- Problem 1
SELECT TOP 1
  ae.team,
  COUNT(DISTINCT ae.event) AS cnt
FROM dbo.athlete_events ae
INNER JOIN dbo.athletes a ON ae.athlete_id = a.id
WHERE ae.medal = 'Gold'
GROUP BY ae.team
ORDER BY cnt DESC;
```

*Notes:* This counts distinct events where golds were awarded. If you prefer to count total golds (including multiple medals in the same event across different athletes/teams), remove `DISTINCT`.

---

### Problem 2 — For each team: total silver medals and year of max silver

**Goal:** For each team print `team`, `total_silver_medals`, `year_of_max_silver`.

```sql
-- Problem 2
WITH cte AS (
  SELECT a.team,
         ae.year,
         COUNT(DISTINCT ae.event) AS silver_medals,
         RANK() OVER (PARTITION BY a.team ORDER BY COUNT(DISTINCT ae.event) DESC) AS rn
  FROM dbo.athlete_events ae
  INNER JOIN dbo.athletes a ON ae.athlete_id = a.id
  WHERE ae.medal = 'Silver'
  GROUP BY a.team, ae.year
)
SELECT team,
       SUM(silver_medals) AS total_silver_medals,
       MAX(CASE WHEN rn = 1 THEN year END) AS year_of_max_silver
FROM cte
GROUP BY team;
```

*Notes:* If multiple years are tied for max silver, `MAX(CASE WHEN rn = 1 THEN year END)` returns the latest of those. Adjust to `MIN()` if you prefer the earliest tied year.

---

### Problem 3 — Player with maximum golds among players who won only golds

**Goal:** Which player has won the maximum gold medals among players who have *only* golds (no silver/bronze)

```sql
-- Problem 3
WITH cte AS (
  SELECT a.name, ae.medal
  FROM dbo.athlete_events ae
  INNER JOIN dbo.athletes a ON ae.athlete_id = a.id
)
SELECT TOP 1
  name,
  COUNT(1) AS no_of_gold_medals
FROM cte
WHERE name NOT IN (SELECT DISTINCT name FROM cte WHERE medal IN ('Silver', 'Bronze'))
  AND medal = 'Gold'
GROUP BY name
ORDER BY no_of_gold_medals DESC;
```

*Notes:* This finds athletes who have *never* won any Silver or Bronze across the dataset, and among them picks the one with the most Golds.

---

### Problem 4 — In each year, player(s) with max golds (ties as comma separated names)

**Goal:** For each year print `year`, `no_of_golds`, and `players` (comma-separated if tie)

```sql
-- Problem 4
WITH cte AS (
  SELECT ae.year, a.name, COUNT(1) AS no_of_gold
  FROM dbo.athlete_events ae
  INNER JOIN dbo.athletes a ON ae.athlete_id = a.id
  WHERE ae.medal = 'Gold'
  GROUP BY ae.year, a.name
)
SELECT year, no_of_gold, STRING_AGG(name, ',') AS players
FROM (
  SELECT *,
         RANK() OVER (PARTITION BY year ORDER BY no_of_gold DESC) AS rn
  FROM cte
) t
WHERE rn = 1
GROUP BY year, no_of_gold
ORDER BY year;
```

*Notes:* `STRING_AGG` requires SQL Server 2017+. If you are on an earlier version, use `FOR XML PATH` trick to aggregate names.

---

### Problem 5 — India: first Gold, first Silver, first Bronze (year & sport/event)

**Goal:** For India, find the first year and event where each medal type was won.

```sql
-- Problem 5
SELECT DISTINCT medal, year, event
FROM (
  SELECT medal, year, event,
         RANK() OVER (PARTITION BY medal ORDER BY year) rn
  FROM dbo.athlete_events ae
  INNER JOIN dbo.athletes a ON ae.athlete_id = a.id
  WHERE ae.team = 'India'
    AND ae.medal != 'NA'
) A
WHERE rn = 1;
```

*Notes:* This returns one row per medal type (Gold/Silver/Bronze) with the earliest year. If multiple events in the same year tie, multiple rows may appear — alter `RANK()` to `ROW_NUMBER()` with additional tie-breakers if you want exactly one row per medal.

---

### Problem 6 — Players who won gold in both Summer & Winter Olympics

**Goal:** Find athletes who have won Gold medals in *both* seasons.

```sql
-- Problem 6
SELECT a.name
FROM dbo.athlete_events ae
INNER JOIN dbo.athletes a ON ae.athlete_id = a.id
WHERE ae.medal = 'Gold'
GROUP BY a.name
HAVING COUNT(DISTINCT ae.season) = 2;
```

*Notes:* This expects `season` to contain values like `Summer` and `Winter`.

---

### Problem 7 — Players who won Gold, Silver and Bronze in a single Olympics (same year)

**Goal:** Return player name and the year where they won all three medal types in that year.

```sql
-- Problem 7
SELECT year, name
FROM dbo.athlete_events ae
INNER JOIN dbo.athletes a ON ae.athlete_id = a.id
WHERE ae.medal != 'NA'
GROUP BY year, name
HAVING COUNT(DISTINCT ae.medal) = 3;
```

*Notes:* Useful check to ensure the row counts and medal labeling are consistent (`NA` typically means no medal).

---

### Problem 8 — Players who won golds in 3 consecutive Summer Olympics in same event (>=2000)

**Goal:** From 2000 onwards, find players who won Gold in the *same event* across three consecutive Summer Olympics (assume 2000, 2004, 2008, ...).

```sql
-- Problem 8
WITH cte AS (
  SELECT a.name, ae.year, ae.event
  FROM dbo.athlete_events ae
  INNER JOIN dbo.athletes a ON ae.athlete_id = a.id
  WHERE ae.year >= 2000
    AND ae.season = 'Summer'
    AND ae.medal = 'Gold'
  GROUP BY a.name, ae.year, ae.event
)
SELECT * FROM (
  SELECT *,
         LAG(year, 1) OVER (PARTITION BY name, event ORDER BY year) AS prev_year,
         LEAD(year, 1) OVER (PARTITION BY name, event ORDER BY year) AS next_year
  FROM cte
) A
WHERE year = prev_year + 4
  AND year = next_year - 4;
```

*Notes:* This checks that for a given `name` and `event`, a `year` has neighbors `year-4` and `year+4` (i.e., three consecutive Olympics). It will return the middle year row — you can adjust output to show the full triple.

---

## Performance tips & indexing suggestions

* Create indexes on join/filter columns:

```sql
CREATE INDEX IX_ae_athlete_id ON dbo.athlete_events(athlete_id);
CREATE INDEX IX_ae_medal ON dbo.athlete_events(medal);
CREATE INDEX IX_ae_team_year ON dbo.athlete_events(team, year);
```

* Use `COUNT(*)` vs `COUNT(DISTINCT ...)` depending on whether duplicates matter. `DISTINCT` is more expensive.
* If dataset is large and you do many analytics, consider creating summary tables (materialized view equivalents) grouped by year/team/medal.

---

## How to reproduce & run the queries locally

1. Import CSVs into a fresh database as described above.
2. Create recommended indexes.
3. Open SSMS/Azure Data Studio and run the SQL blocks from the *SQL Solutions* section.
4. If you need CSV outputs, wrap queries with `SELECT ...` and use `Results -> Save Results As...` in SSMS.
