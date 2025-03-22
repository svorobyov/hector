# TODO

Please visit the https://github.com/svorobyov/hector/blob/main/hector-test3.ipynb Colab notebook containing detailed answers
to all the questions data frames, graph plots, Python code, etc.


## Task 1 

From what date is the oldest data point in the data set? 

The oldest is from 2001-01-01 00:00:00 as given by the query:

```sql
SELECT *
FROM `alva-coding-test.chicago_crime.crime`
ORDER BY `Date`
LIMIT 1000;  -- first row corresponds to oldest crime on 2001-01-01 00:00:00
```

Actually there are 195 simultaneous crimes on that date:
```sql
SELECT COUNT(*)
FROM `alva-coding-test.chicago_crime.crime`
WHERE `Date` = '2001-01-01 00:00:00';
```

## Task 2

Which year had the highest amount of crimes?

Year 2002 had 486825 crimes, followed by year 2001 with 485920 crimes.

```sql
SELECT
    EXTRACT(YEAR FROM `Date`) AS crime_year,
    COUNT(*) AS n_crimes
FROM `alva-coding-test.chicago_crime.crime`
GROUP BY crime_year
ORDER BY n_crimes DESC LIMIT 2;
```

## Task 3

Let's define "Arrest Rate" as the share of crimes that led to an arrest.

What year had the highest arrest rate? What is the overall trend in number of crimes per year?

Arrest rate per year is computed by the query, where the salient line is 
`AVG(CAST(arrest AS INT64)) AS ARREST_RATE`:

```sql
SELECT
    EXTRACT(YEAR FROM `Date`) AS crime_year,
    COUNT(*) AS n_crimes,
    COUNT(CASE WHEN `arrest` = TRUE THEN 1 END) AS ARRESTED_COUNT,
    COUNT(CASE WHEN `arrest` = FALSE THEN 1 END) AS NON_ARRESTED_COUNT,
    AVG(CAST(arrest AS INT64)) AS ARREST_RATE
FROM `alva-coding-test.chicago_crime.crime`
GROUP BY crime_year
ORDER BY crime_year;
```

If we want to se the same data but ordered by instead of year, we replace the last line by:

```sql
...ORDER BY ARREST_RATE DESC;
```

Corresponding data frames and plots are given 
in the https://github.com/svorobyov/hector/blob/main/hector-test3.ipynb Colab notebook.
The overall trend of arrest rate seems **declining**.


## Task 4

What were the five most common crimes in 2020? Which of those crimes had the highest and lowest arrest rate?

The answer is given by the query:
```sql
SELECT
    `primary_type`,
    COUNT(*) AS CRIME_COUNT,
    COUNT(CASE WHEN `arrest` = TRUE THEN 1 END) AS ARRESTED_COUNT,
    COUNT(CASE WHEN `arrest` = FALSE THEN 1 END) AS NON_ARRESTED_COUNT,
    AVG(CAST(arrest AS INT64)) AS ARREST_RATE
FROM `alva-coding-test.chicago_crime.crime`
WHERE EXTRACT(YEAR FROM `Date`) = 2020
GROUP BY `primary_type`
ORDER BY CRIME_COUNT DESC LIMIT 5;
```

Top 5 by number of crimes:
`BATTERY`, `THEFT`, `CRIMINAL DAMAGE`, `DECEPTIVE PRACTICE`, `ASSAULT` (in decreasing order).

Top 5 by arrest rate:
`BATTERY`, `ASSAULT`, `THEFT`, `CRIMINAL DAMAGE`, `DECEPTIVE PRACTICE` (in decreasing order).

Corresponding data frames and plots are given 
in the https://github.com/svorobyov/hector/blob/main/hector-test3.ipynb Colab notebook.

## Task 5

Investigate which year that had the highest number of crimes leading to an arrest. 
What year was it, and how many arrests were made during that year?

The answer is 2005 given by the query:

```sql
SELECT
    EXTRACT(YEAR FROM `Date`) AS crime_year,
    COUNT(*) AS n_crimes,
    COUNT(CASE WHEN `arrest` = TRUE THEN 1 END) AS ARRESTED_COUNT,
    COUNT(CASE WHEN `arrest` = FALSE THEN 1 END) AS NON_ARRESTED_COUNT, 
    AVG(CAST(arrest AS INT64)) AS ARREST_RATE
FROM `alva-coding-test.chicago_crime.crime`
GROUP BY crime_year
ORDER BY ARREST_RATE DESC;
```

Corresponding data frames and plots are given 
in the https://github.com/svorobyov/hector/blob/main/hector-test3.ipynb Colab notebook.

## Task 6

How has the arrest rate looked like over time? 
- Plot the trend of the arrest rate.
- Between which years can you see the biggest change in "Arrest Rate"?
- Can you point at specific reasons to why the Arrest Rate dropped between those years? Comment on your conclusions.

```sql
-- Biggest changes in arrest rate, look at first and last in the result table lines
WITH
  SUMMARY2 AS(
    SELECT
        EXTRACT(YEAR FROM `Date`) AS crime_year,
        COUNT(*) AS n_crimes,
        COUNT(CASE WHEN `arrest` = TRUE THEN 1 END) AS ARRESTED_COUNT,
        COUNT(CASE WHEN `arrest` = FALSE THEN 1 END) AS NON_ARRESTED_COUNT,
        AVG(CAST(arrest AS INT64)) AS ARREST_RATE
    FROM `alva-coding-test.chicago_crime.crime`
    GROUP BY crime_year
    ORDER BY crime_year
  )
SELECT
    *,
    (ARREST_RATE - LAG(ARREST_RATE, 1, NULL) OVER (ORDER BY crime_year)) / LAG(ARREST_RATE, 1, NULL) OVER (ORDER BY crime_year) AS DIFF_PERCENT 
FROM SUMMARY2 ORDER BY DIFF_PERCENT DESC;
```

The largest increase is in line 0 year 2024, the largest decrease is in line 22 year 2016.

The biggest drops in year 2016 and 2020. Because of pandemic, many educational 
institutions and public areas were partially or fully closed in many jurisdictions, 
and many events were cancelled or postponed during 2020 and 2021.
Interstingly, homicides increased dramatically in Chicago in 2016. 
In 2015, 480 Chicago residents were killed. The next year, 
754 were killed 274 additional homicide victims, tragically producing 
an extraordinary 58% increase in a single year. But homicide is only a small 
fraction of all crimes, however one of the most important. Possibly, the police 
turned all their attention to homicides thus all other crimes were left neglected 
and consequently overall arrest rate dropped.

Corresponding data frames and plots are given 
in the https://github.com/svorobyov/hector/blob/main/hector-test3.ipynb Colab notebook.


## Task 7

* What was the arrest rate for thefts during 2017 and 2018?
* How much did the arrest rate decrease between those two years?
* Perform a t-test to determine if the decrease in arrest rate is significant at a 95% confidence level. 

Comment on your conclusions.

The following query selects reuiested data for years 2017 and 2018:

```sql
WITH
  SUMMARY4 AS (
    SELECT
      EXTRACT(YEAR FROM `Date`) AS crime_year,
      AVG(CAST(arrest AS INT64)) AS ARREST_RATE_OVERALL,
      AVG(CASE WHEN `arrest` = TRUE AND `primary_type` = 'THEFT' THEN 1 ELSE 0 END) AS ARREST_RATE_THEFT, 
    FROM `alva-coding-test.chicago_crime.crime`
    WHERE (EXTRACT(YEAR FROM `Date`) BETWEEN 2016 AND 2018)
    GROUP BY crime_year
    ORDER BY crime_year
  )
SELECT
  *,
  (ARREST_RATE_OVERALL - LAG(ARREST_RATE_OVERALL, 1, NULL) OVER (ORDER BY crime_year)) / LAG(ARREST_RATE_OVERALL, 1, NULL) OVER (ORDER BY crime_year) AS DIFF_ARREST_RATE_OVERALL,
  (ARREST_RATE_THEFT - LAG(ARREST_RATE_THEFT, 1, NULL) OVER (ORDER BY crime_year)) / LAG(ARREST_RATE_THEFT, 1, NULL) OVER (ORDER BY crime_year) AS DIFF_ARREST_RATE_THEFT,
  FROM SUMMARY4
  ORDER BY crime_year;
```

The data about `ARREST_RATE_THEFT` and `ARREST_RATE_OVERALL` for year 2017 and 2018 is fed to a 
Python/`scipy.stats` `t-test` function which produces:

```sql
t-statistic: 1.3680406263808815
p-value: 0.30472375549456165
```

Please see the the https://github.com/svorobyov/hector/blob/main/hector-test3.ipynb Colab notebook
for Python code and plots.

### Conclusions

1. The null hypothesis: there is no difference in arrest rate change for thefts and all crimes altogether in the years 2017 and 2018.

2. The relatively high p-value of 0.3... (much larger than the typical significance level of 0.05) shows that we fail to reject the null hypothesis.

3. It's very likely that the observed difference is just due to random chance, and not a real effect.
