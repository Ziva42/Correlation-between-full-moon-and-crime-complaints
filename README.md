# Correlation between full moon and crime complaints

## Background

A while ago I was watching episode 7 of season 1 of the tv show "9-1-1", which was taking place on a full moon night.
During the episode, the following dialogue took place:

```
Abby: I do not buy into all that full moon BS.
Buck: Buy into? No, no, no, it is science. Every full moon, the freaks come out. Crime increases, emergency rooms are packed, animals and kids go nuts.
Abby: No, in my experience, anyone who is going to be an idiot on the full moon, is going to be an idiot on the half moon, or the quarter moon, or the new moon.
```

That moment I decided I'll try to settle the debate between Buck and Abby, and see which of them is right; Is there a correlation between full moon and rise in crime, or not?

I downloaded a [dataset of full moon dates](https://www.kaggle.com/datasets/lsind18/full-moon-calendar-1900-2050?select=full_moon.csv), and a [dataset of complaints recorded by the NYPD from 2006 to 2021 (inclusive)](https://data.cityofnewyork.us/Public-Safety/NYPD-Complaint-Data-Historic/qgea-i56i), and started my analysis, using SQL Server and Tableau.

So, who was right, Buck or Abby?
Well, not that I had any doubt in her, she is good at her job after all (and Buck was only 2 months on the job at that time!), but Abby was right - I didn't find any correlation between full moon and rise in crime.

[Link to the visualization in Tableau](https://public.tableau.com/views/FullMoon-Crime/Correlationbetweenfullmoonandcrimecomplaints?:language=en-US&publish=yes&:display_count=n&:origin=viz_share_link)

## Analysis Steps

* Checking to see if there are missing values in the dataset.
* Converting the Date column in the full moon dates dataset into date format.
* Writing an SQL query which compares between the number of complaints on full moon dates to the daily average number of complaints on non-full moon dates, per month.
* Writing an SQL query which compares between the number of complaints on full moon dates to the daily average number of complaints on non-full moon dates, per month and per crime type.
* Creating visualizations and making conclusions.

## SQL Queries

### 1. Comparison of the number of complaints on full moon dates to the daily average number of complaints on non-full moon dates, per month.

```sql
CREATE OR ALTER VIEW is_full_moon_greater_num_cmplnts AS
SELECT x.cmplnt_year,
x.cmplnt_month,
is_full_moon,
num_cmplnts,
AVG(num_cmplnts/num_days) AS avg_num_cmplnts,
AVG(num_cmplnts/num_days)-LAG(AVG(num_cmplnts/num_days),1,null) OVER (PARTITION BY x.cmplnt_year, x.cmplnt_month ORDER BY is_full_moon) AS diff_num_cmplnts,
CASE WHEN AVG(num_cmplnts/num_days)-LAG(AVG(num_cmplnts/num_days),1,null) OVER (PARTITION BY x.cmplnt_year, x.cmplnt_month ORDER BY is_full_moon)>0 THEN 'not_full_moon_greater_num_cmplnts'
	WHEN AVG(num_cmplnts/num_days)-LAG(AVG(num_cmplnts/num_days),1,null) OVER (PARTITION BY x.cmplnt_year, x.cmplnt_month ORDER BY is_full_moon)<0 THEN 'full_moon_greater_num_cmplnts'
	WHEN AVG(num_cmplnts/num_days)-LAG(AVG(num_cmplnts/num_days),1,null) OVER (PARTITION BY x.cmplnt_year, x.cmplnt_month ORDER BY is_full_moon)=0 THEN 'no_diff'
	END AS 'diff'
FROM
(SELECT c.cmplnt_year,
c.cmplnt_month,
is_full_moon,
COUNT(DISTINCT cmplnt_day) AS num_days,
COUNT(DISTINCT cmplnt_num) AS num_cmplnts
FROM
(SELECT cmplnt_num,
CMPLNT_FR_DT,
MONTH(CMPLNT_FR_DT) AS cmplnt_month,
YEAR(CMPLNT_FR_DT) AS cmplnt_year,
DAY(CMPLNT_FR_DT) AS cmplnt_day,
OFNS_DESC,
CASE WHEN CMPLNT_FR_DT=new_date THEN 'full_moon'
	ELSE 'not_full_moon'
	END AS 'is_full_moon'
FROM NYPD_Complaint_Data_Historic n
LEFT JOIN full_moon_fixed f
ON n.CMPLNT_FR_DT=f.new_date
WHERE DAY(CMPLNT_FR_DT)>'' AND YEAR(CMPLNT_FR_DT)>=2006) c
GROUP BY cmplnt_year, cmplnt_month, is_full_moon) x
GROUP BY cmplnt_year, cmplnt_month, is_full_moon, num_cmplnts
```

### 2. Comparison of the number of complaints on full moon dates to the daily average number of complaints on non-full moon dates, per month and per crime type.

```sql
CREATE VIEW ofns_full_moon AS
SELECT x.cmplnt_year,
x.cmplnt_month,
is_full_moon,
OFNS_DESC,
num_cmplnts,
AVG(num_cmplnts/num_days) AS avg_num_cmplnts,
AVG(num_cmplnts/num_days)-LAG(AVG(num_cmplnts/num_days),1,null) OVER (PARTITION BY x.cmplnt_year, x.cmplnt_month ORDER BY is_full_moon) AS diff_num_cmplnts,
CASE WHEN AVG(num_cmplnts/num_days)-LAG(AVG(num_cmplnts/num_days),1,null) OVER (PARTITION BY x.cmplnt_year, x.cmplnt_month, OFNS_DESC ORDER BY is_full_moon)>0 THEN 'not_full_moon_greater_num_cmplnts'
	WHEN AVG(num_cmplnts/num_days)-LAG(AVG(num_cmplnts/num_days),1,null) OVER (PARTITION BY x.cmplnt_year, x.cmplnt_month, OFNS_DESC ORDER BY is_full_moon)<0 THEN 'full_moon_greater_num_cmplnts'
	WHEN AVG(num_cmplnts/num_days)-LAG(AVG(num_cmplnts/num_days),1,null) OVER (PARTITION BY x.cmplnt_year, x.cmplnt_month, OFNS_DESC ORDER BY is_full_moon)=0 THEN 'no_diff'
	END AS 'diff'
FROM
(SELECT c.cmplnt_year,
c.cmplnt_month,
is_full_moon,
OFNS_DESC,
COUNT(DISTINCT cmplnt_day) AS num_days,
COUNT(DISTINCT cmplnt_num) AS num_cmplnts
FROM
(SELECT cmplnt_num,
CMPLNT_FR_DT,
MONTH(CMPLNT_FR_DT) AS cmplnt_month,
YEAR(CMPLNT_FR_DT) AS cmplnt_year,
DAY(CMPLNT_FR_DT) AS cmplnt_day,
OFNS_DESC,
CASE WHEN CMPLNT_FR_DT=new_date THEN 'full_moon'
	ELSE 'not_full_moon'
	END AS 'is_full_moon'
FROM NYPD_Complaint_Data_Historic n
LEFT JOIN full_moon_fixed f
ON n.CMPLNT_FR_DT=f.new_date
WHERE DAY(CMPLNT_FR_DT)>'' AND OFNS_DESC>'' AND YEAR(CMPLNT_FR_DT)>=2006) c
GROUP BY cmplnt_year, cmplnt_month, is_full_moon, OFNS_DESC) x
GROUP BY cmplnt_year, cmplnt_month, is_full_moon, OFNS_DESC, num_cmplnts
```
