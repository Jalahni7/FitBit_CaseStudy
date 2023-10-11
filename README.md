# FitBit_CaseStudy
## This is my capstone project for the Google Data Analytics Professional Certificate. You can find the Tableau dashboard for this case study [here](https://public.tableau.com/views/CapstoneProjectFitBitUserStudy/FitBitUserStudy?:language=en-US&publish=yes&:display_count=n&:origin=viz_share_link).

## Ask:
Analyze the FitBit dataset to gain insight into how consumers use non-Bellabeat smart devices and identify growth opportunities. Determine how Bellabeat might improve one of its products.

### Stakeholders
* Urška Sršen: Bellabeat’s co founder and Chief Creative Officer 
* Sando Mur: Mathematician and Bellabeat cofounder; key member of the Bellabeat executive team 
* Bellabeat marketing analytics team 

### Guiding Questions:
1.     What does a typical day look like for FitBit users?

## Prepare:
The data is stored [here](https://www.kaggle.com/datasets/arashnic/fitbit) and will be imported into BigQuery for my analysis. 
### ROCCC Analysis
* Reliable – Low: The data was collected from 30 consenting FitBit users. The small sample size might cause limitations and biases in the analysis. 
* Original – Low: The data was collected through Amazon Mechanical Turk, a third party.
* Comprehensive - High: A large amount of data was collected across datasets including daily activity intensity, calories used, daily steps taken, daily sleep time and weight record (across both narrow and wide data formats).
* Current – Medium: The data was collected about seven years ago at the point of my analysis, but I consider the data relatively current as advancement sin wearable fitness products and consumer habits might not have dramatically changed during that time
* Cited – High: The data source is well documented.
###Data Selection
The data is overwhelming and in some cases, redundant since both narrow and wide data formats are shared for some of the datasets. My first filtering task will be to filter out the wide datasets and focus on the narrow. 
Furthermore, I will focus on the following datasets to answer my initial questions through my analysis: 
* daily_activity.csv
* sleep_day.csv
* weight_log_info.csv

## Process:
I will be using Excel and BigQuery to clean and filter the data and Tableau to visualize the data. Before uploading my datasets into BigQuery, I used Excel to pre-clean the data. This step was necessary as the Datetime format in the files included AM/PM which could not be parsed by BigQuery. To troubleshoot this problem, I used Excel to reformat the Datetime into 24-hour time, which allowed me to preserve the reliability of the data.

My next cleaning step was to verify the consistency of the data using a `COUNT(DISTINCT (Id))` function to ensure all selected datasets had data from all 30 users. This step was taken for each of my datasets, and this is what was discovered:
* daily_activity.csv = 33 unique user ids
* sleep_day.csv = 24 unique user ids
* weight_log_info.csv = 8 unique user ids

```
/*Cleaning - query number of unique ids*/
SELECT COUNT(DISTINCT(Id)) 
FROM `bellabeat-company-project.Fitness_Tracker.daily_activity`
LIMIT 1000;
```

The final step in my cleaning process is to verify that the Id numbers are consistent across each table. Having the number of rows in each table, I decided to use an `INNER JOIN`. However, I found this process to be resource intensive and Big Query was unable to execute it. As a workaround, I tried using an `IN` clause instead, but had the same outcome. Luckily, I was able to revert to using an `INNER JOIN` which required me to INNER JOIN to join two tables at a time. Had the dataset been more complexed, this would have been a challenge for me, so this is an area for me to improve my troubleshooting.
```
/*Cleaning - query tables for consistent ids, activity joined on calories table*/
SELECT
  DISTINCT(active.Id)
FROM `bellabeat-company-project.Fitness_Tracker.daily_activity` active
INNER JOIN `bellabeat-company-project.Fitness_Tracker.daily_calories` calorie
  ON (
    active.Id = calorie.Id
  )
GROUP BY active.Id
ORDER BY active.Id;
```
```
/*Cleaning - query tables for consistent ids, all tables inner joined*/
SELECT
  DISTINCT(active.Id)
FROM `bellabeat-company-project.Fitness_Tracker.daily_activity` active
INNER JOIN `bellabeat-company-project.Fitness_Tracker.daily_calories` calorie
  ON (
    active.Id = calorie.Id
  )
INNER JOIN `bellabeat-company-project.Fitness_Tracker.daily_steps` steps
  ON (
    active.Id = steps.Id
  )
 INNER JOIN `bellabeat-company-project.Fitness_Tracker.heartrate_seconds` heartrate
  ON (
    active.Id = heartrate.Id
  )
 INNER JOIN `bellabeat-company-project.Fitness_Tracker.sleep_day` sleep
  ON (
    active.Id = sleep.Id
  )
 INNER JOIN `bellabeat-company-project.Fitness_Tracker.weight_log_info` weight
  ON (
    active.Id = weight.Id
  )
GROUP BY active.Id
ORDER BY active.Id
LIMIT 100;
```
```
/*Cleaning - query tables for consistent ids,using IN clause*/
SELECT DISTINCT(active.Id)
FROM `bellabeat-company-project.Fitness_Tracker.daily_activity` active
WHERE 
  active.Id IN (
    SELECT Id FROM `bellabeat-company-project.Fitness_Tracker.daily_calories`
  )
  AND active.Id IN (
    SELECT Id FROM `bellabeat-company-project.Fitness_Tracker.daily_steps`
  )
  AND active.Id IN (
    SELECT Id FROM `bellabeat-company-project.Fitness_Tracker.heartrate_seconds`
  )
  AND active.Id IN (
    SELECT Id FROM `bellabeat-company-project.Fitness_Tracker.sleep_day`
  )
  AND active.Id IN (
    SELECT Id FROM `bellabeat-company-project.Fitness_Tracker.weight_log_info`
  )
ORDER BY active.Id
LIMIT 100;
```

## Analyze:
First, I wanted to understand how the sleep feature was used. To do this, I executed the following queries:
```
/*Understanding the timeline for the sleep_day data*/
SELECT 
  MIN(SleepDay) AS start_date,
  MAX(SleepDay) AS end_date
FROM `bellabeat-company-project.Fitness_Tracker.sleep_day` 
LIMIT 1000;
```
```
/*Understanding how the sleep product is used*/
WITH sleep_data AS (
  SELECT 
    DISTINCT(Id) AS unique_id,
    SUM(TotalSleepRecords) AS total_sleep_records,
    SUM(TotalMinutesAsleep) AS total_minutes_asleep
  FROM `bellabeat-company-project.Fitness_Tracker.sleep_day`
  WHERE 
    Id IS NOT NULL
  GROUP BY Id
)
SELECT 
  unique_id,
  total_sleep_records,
  (total_sleep_records / 30) AS daily_sleeps_recorded,
  total_minutes_asleep
FROM sleep_data
WHERE (total_sleep_records/30) >= 0.5
ORDER BY total_sleep_records DESC
LIMIT 1000;
```
My findings from the sleep dataset indicated that only 15 of the 24 users who submitted sleep records, used the sleep feature on their device. Furthermore, only half of the total 30 users in the study used the sleep feature at least every other day.
 
Next, I wanted to look at how users tracked their weight across the study. Here’s my query and what I found:
```
/*Weight log analysis*/
SELECT *
FROM `bellabeat-company-project.Fitness_Tracker.weight_log_info`;
```
```
/*Understanding the changes in weight*/
SELECT 
  DISTINCT(Id),
  ROUND(MIN(WeightPounds),2) AS min_weight,
  ROUND(MAX(WeightPounds),2) AS max_weight,
FROM `bellabeat-company-project.Fitness_Tracker.weight_log_info` 
  WHERE Id IS NOT NULL
GROUP BY Id
LIMIT 1000;
```
Having queried the data, I’ve decided to no longer consider the weight_log_info.csv dataset in my analysis. Since there are only 8 unique Ids in the data sets, I was curious to understand what sort of change in weight users recorded. Weight change seemed to be minimalistic and consequently challenging to derive any meaningful insights from. 
 
Finally, I will be analyzing the most comprehensive table, Daily_Activity.csv. and perform joins to assess the relationships between my datasets. My first step was to aggregate the data and attribute it to unique user ids.
```
/*Daily Activity analysis*/
SELECT * 
FROM `bellabeat-company-project.Fitness_Tracker.daily_activity` 
LIMIT 1000;
```
This query made the dataset more digestible and allowed me to further analyze the data set's relationship to sleep records. The Syntax is below:
```
/*Aggregating the data according to unique user ids*/
SELECT 
  DISTINCT(Id),
  SUM(TotalSteps) AS sum_steps,
  ROUND(SUM(TotalDistance),2) AS sum_distance,
  ROUND(SUM(VeryActiveDistance),2) AS very_active_distance,
  ROUND(SUM(ModeratelyActiveDistance),2) AS moderately_active_distance,
  ROUND(SUM(LightActiveDistance),2) AS light_active_distance,
  SUM(VeryActiveMinutes) AS very_active_minutes,
  SUM(FairlyActiveMinutes) AS fairly_active_minutes,
  SUM(LightlyActiveMinutes) AS lightly_active_minutes,
  SUM(SedentaryMinutes) AS resting_minutes,
  SUM(Calories) AS total_calories,
FROM `bellabeat-company-project.Fitness_Tracker.daily_activity` 
GROUP BY 1
ORDER BY sum_steps DESC
LIMIT 50;
```
This table gives us more insight into the relationship between the tenacity of daily activity and sleep time. A noticeable trend exists as the activity increases, the number of sleep records increase as well.

## Share: 
To visually represent my findings, I created a dashboard using Tableau [link here](https://public.tableau.com/views/CapstoneProjectFitBitUserStudy/FitBitUserStudy?:language=en-US&publish=yes&:display_count=n&:origin=viz_share_link). Here are some of my findings:
* Users log lightly active minutes the most. 
* The number of steps a user takes and the has no significant relationship with the user’s weight.
* The sleep feature is commonly used in at least half of users.
* There’s a positive trend between total steps and the number of minutes slept by users.

## Act:
After analyzing the FitBit Fitness Tracker data, I came up with some recommendations for Bellabeat marketing strategy based on trends in smart device usage.
* Most participants are lightly active. Bellabeat should tie its product features like sleep, weight, and steps to how customers are already using the product. For example, integrating more lightly active forms exercise to increase step count throughout the day.
* To help users better track their weight, Bellabeat might consider educating customers on the importance of diet and exercise and how their product can help.
* Simplify the weight entry process to boost use of the feature.
* Showing the benefit of entering information will provide more accurate results for the user. 
* Design promotional campaigns focused on features that users are consistently engaging with like Step count, distance tracking, calories, and sleep. 

