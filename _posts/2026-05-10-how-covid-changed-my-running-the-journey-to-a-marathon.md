---
layout: post
title: COVID Changed My Running + The Journey to a Marathon
type: Essay
description: COVID changed all of us, in more than one way. One of mine was how my fitness and running took a backseat for many years. Now, I am digging into my data as I prepare for first marathon.
tags: [Data Analysis]
---
I recently signed up for my first marathon — the Marine Corps Marathon in Washington DC this October. Despite being a runner for the better part of the last 15 years, a marathon has always been something I'd "get to eventually" or "do someday." That hesitation was probably less about whether I could do it and more about the fact that if I were going to do it, I'd want to do it properly, which meant diligent training.

The problem was I knew my running fitness had been slipping for years. What I didn't know was how much — until I looked at my own data.

---


"We'll be back on Monday" — that's what most of us thought in March 2020 as we left the office for what turned out to be the last time for over a year, and in some cases, forever.

In the early days of COVID, working from home felt like a gift. Suddenly I had time — to read, to cook, to pick up new habits, to run. More distance and faster than I ever had in my life.

On a monthly basis I was hitting over 150 miles, close to 40 miles a week.

![Chart description](/assets/img/How COVID Changed My Running/Overall_Monthly Distance_Runs.png)

As the pandemic wore on though, the motivation quietly eroded. The lack of community set in, the noise of isolation grew louder, and it became easy to put things off. I had more time than ever — I could always just run tomorrow. It became too much of a good thing, and my running paid the price.

Not only was my overall mileage dropping, but the runs themselves were getting shorter. By July 2022, only 7% of my runs were more than 4 miles — meanwhile, just a couple of years earlier almost every run had been longer than that.

![Chart description](/assets/img/How COVID Changed My Running/Percentage_Of_Runs_By_Distance.png)

Running longer distances is proven to build both speed and endurance. By cutting my runs short for years, I was building neither — and the data makes the cost clear.

**Between 2020 and 2025, my average pace per mile slowed by roughly 60 seconds** — a slow, quiet attrition over five years.

![Chart description](/assets/img/How COVID Changed My Running/Pace_Distribution_Comparison.png)


Perhaps the most humbling number: 2025 was my slowest year on record. My average pace for runs over 4 miles exceeded 8 minutes per mile for the first time.

![Chart description](/assets/img/How COVID Changed My Running/Average_4_Mile Pace.png)

---


But that's not where the story ends.

**Look at 2026 in each of those charts and something is changing:**

- Monthly mileage is higher now than at any point since 2020
- Nearly 70% of my runs are more than 4 miles
- My pace distribution is shifting back to the left — I am getting faster

I probably won't be back to 2020 fitness by race day in October, but perhaps seeing what I was once capable of is the motivation I needed to start training. In any case, looking honestly at your own data is a worthwhile and occasionally humbling exercise — what data do you have that could tell you something about yourself?

---

## The Technicals 

Strava has a feature that lets you download a full archive of your data, including all the FIT/GPX route files. They also offer API access, but I opted for the bulk archive; 11 years of running, all in one download.

The archive comes as a zip folder. The most important file inside is a CSV named `activities`. Each row is a single activity, with roughly 100 columns covering everything from distance and moving time to heart rate and elevation. My file had around 3,000 activities — not too large for Excel, but Excel isn't the right tool for iterating through analysis and building transformations. My preference is to use SQL for the heavy lifting and Excel for charting.

Building a full database for a 3,000-row file would be overkill. Fortunately, DuckDB solves this cleanly — it's a Python library that lets you run SQL directly against CSV files, no database setup or separate software required. Combined with Jupyter notebooks, it's an easy and effective workflow: each query lives in its own cell, easy to iterate and adjust.

*Setup (Jupyter Cell 1):**

```python
import duckdb
%load_ext sql
conn = duckdb.connect()
%sql conn --alias duckdb
%config SqlMagic.feedback = False
%config SqlMagic.displaycon = False
%config SqlMagic.displaylimit = 0
%config SqlMagic.named_parameters = "disabled"
```

**Register the CSV as a queryable view (Cell 2):**

```sql
CREATE OR REPLACE VIEW activities AS
SELECT * FROM read_csv_auto('Running Data/export_30532952_april_all/activities.csv')
```

Before any real analysis could begin, I needed to sort out four things with the raw data:

1. The only date field was a full datetime — no separate month or day columns
2. Distance was in kilometres, not miles
3. Pace wasn't stored — I had to derive it from moving time divided by distance
4. With ~100 columns and spaces in the column names, I needed to simplify the view considerably

I handled all four in a single SQL script that creates what I called the `core_run_data` view — a clean, derived table with only the columns I needed, plus calculated fields for pace and mileage. Every subsequent query called this view rather than the raw CSV.

**Building the core table:**

```sql
CREATE OR REPLACE VIEW core_run_data AS
(
    SELECT
        "Activity ID"           AS activity_id,
        "Activity Type"         AS activity_type,

        YEAR((strptime("Activity Date", '%b %d, %Y, %I:%M:%S %p')
            AT TIME ZONE 'UTC'
            AT TIME ZONE 'America/New_York')::DATE)         AS run_year,

        LAST_DAY((strptime("Activity Date", '%b %d, %Y, %I:%M:%S %p')
            AT TIME ZONE 'UTC'
            AT TIME ZONE 'America/New_York')::DATE)         AS run_month,

        (strptime("Activity Date", '%b %d, %Y, %I:%M:%S %p')
            AT TIME ZONE 'UTC'
            AT TIME ZONE 'America/New_York')::DATE          AS run_date,

        ROUND("Distance" * 0.621371, 2)                     AS dist_mi,

        CASE
            WHEN ROUND("Distance" * 0.621371, 2) < 4  THEN '< 4.00 miles'
            WHEN ROUND("Distance" * 0.621371, 2) >= 4 THEN '4.00+ miles'
        END                                                  AS dist_category,

        printf('%d:%02d',
            FLOOR("Moving Time" / ("Distance" * 0.621371) / 60)::INT,
            ("Moving Time" / ("Distance" * 0.621371))::INT % 60
        )                                                    AS pace_per_mile,

        ROUND("Moving Time" / ("Distance" * 0.621371), 1)   AS pace_sec_raw

    FROM activities
    WHERE "Activity Type" = 'Run'
      AND ROUND("Distance" * 0.621371, 2) > 1
)
```

From there, building the charts was a matter of writing focused queries against `core_run_data`. Here are the three that powered the analysis above:

**Percentage of runs over 4 miles by month:**

```sql
SELECT
    run_month,
    "< 4.00 miles_cnt",
    "4.00+ miles_cnt",
    COALESCE("< 4.00 miles_cnt", 0) + COALESCE("4.00+ miles_cnt", 0) AS total_runs,
    ROUND(
        "4.00+ miles_cnt" / NULLIF(COALESCE("< 4.00 miles_cnt", 0) + COALESCE("4.00+ miles_cnt", 0), 0)
    , 2) AS pct_over_four,
    "4.00+ miles_avg_pace"
FROM
    (
    PIVOT (
        SELECT run_month, dist_category, activity_id, pace_sec_raw
        FROM core_run_data
        WHERE run_month > '2017-12-31'
    )
    ON dist_category
    USING count(activity_id) AS cnt, ROUND(AVG(pace_sec_raw) / 60, 1) AS avg_pace
    GROUP BY run_month
    ) t1
ORDER BY run_month
```

**Pace distribution by year:**

```sql
SELECT
    run_year,
    pace_min,
    count(*) AS cnt_run
FROM (
    SELECT
        run_year,
        ROUND(pace_sec_raw / 60.0, 1) AS pace_min
    FROM core_run_data
    WHERE pace_sec_raw BETWEEN 300 AND 900
) t1
GROUP BY run_year, pace_min
ORDER BY run_year
```

**Total distance by month:**

```sql
SELECT
    run_month,
    ROUND(SUM(dist_mi), 2) AS total_miles
FROM core_run_data
GROUP BY run_month
```

The query outputs were small enough that Excel handled the charting without any issues. Sometimes simple is best.

There are plenty of great resources online for getting started with DuckDB — I'd recommend it to anyone who wants to run SQL analysis without the overhead of a full database setup. I'll definitely be using it again.
