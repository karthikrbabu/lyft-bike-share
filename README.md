# Project #1 - Bike Share Analysis 
An analysis of Lyft Bay Wheels Bike share data in the Bay Area


#### Problem Statement

## Introduction

Using SQL and Google BigQuery, we take a dive into understanding trends of bike share riders. With the focus to increase ridership for Lyft Bay Wheels. For the customers using our service, we must learn what is the purpose of their ride, what locations see high ridership, and are we meeting the demand of the customers.

To answer these, and many other data-driven questions please take a read of the summary shared below, and a more detailed exposé in the subsequent jupyter notebook: "skrb_p1.ipynb"

**Data Sets:**
* `bigquery-public-data.san_francisco.bikeshare_stations`
* `bigquery-public-data.san_francisco.bikeshare_status`
* `bigquery-public-data.san_francisco.bikeshare_trips`


## Queries

### Part 1: Querying Data with BigQuery


#### Initial Quries

- What's the size of this dataset? (i.e., how many trips)

* 983,648 Trips

```sql
SELECT COUNT(*) 
FROM bigquery-public-data.san_francisco.bikeshare_trips  
```

- What is the earliest start time and latest end time for a trip?

* Earliest Started Trip: 2013-08-29 09:08:00 UTC
* Latest Ended Trip: 2016-08-31 23:48:00 UTC


```sql
SELECT MIN(start_date) as min_start_time, MAX(END_DATE) as max_end_time 
FROM bigquery-public-data.san_francisco.bikeshare_trips
```


- How many bikes are there?

* 700 unique bikes

```sql
SELECT COUNT(DISTINCT bike_number)
FROM bigquery-public-data.san_francisco.bikeshare_trips
```

#### Questions of my Own


- What was the longest, shortest, and average trip duration ?

* min: 60 (seconds)
* max: 17270400 (seconds) => 4797.3333333 (hours) => 199.88 days
* avg: 1018.9323467338004 (seconds) => 2.8303676298 (hours)

The numbers shown here are just an initial query, there is a high chance there are outliers. This will be examined in greater detail later on in the report.

```sql
SELECT  max(duration_sec) as max_duration, 
        min(duration_sec) as min_duration , 
        avg(duration_sec) as avg_duration

FROM bigquery-public-data.san_francisco.bikeshare_trips
```


- How many stations are there?

* 75 unique stations

```sql
SELECT COUNT (DISTINCT station_ID) 
FROM bigquery-public-data.san_francisco.bikeshare_status
```


-  What is the spread of subscriber types? [Customer, Subscriber]

* Customer: 136,809 people
* Subscriber:  846,839 people


```sql
 SELECT distinct subscriber_type, count(*) as total 
 FROM bigquery-public-data.san_francisco.bikeshare_trips 
 group by subscriber_type
```


### Part 2: Querying data from the BigQuery CLI

Re-running the above queries from the command line.


- What's the size of this dataset? (i.e., how many trips)
  
| number_of_trips |
|----|
| 983648 |


```shell
bq query --use_legacy_sql=false '
    SELECT count(*) as number_of_trips
    FROM `bigquery-public-data.san_francisco.bikeshare_trips`'

```

- What is the earliest start time and latest end time for a trip?

PST - Time Zone


|   min_start_time    |   max_start_time    |
|----|----|
| 2013-08-29  09:08:00 | 2016-08-31  23:32:00 |



```shell
bq query --use_legacy_sql=false '
SELECT min(start_date) as min_start_time, max(start_date) as max_start_time,
      FROM `bigquery-public-data.san_francisco.bikeshare_trips`'
```


- How many bikes are there?


| total_unique_bikes |
|----|
| 700 |



```shell
bq query --use_legacy_sql=false '
    SELECT count(DISTINCT bike_number) as total_unique_bikes
    FROM `bigquery-public-data.san_francisco.bikeshare_trips`'

```


#### New Query


- How many trips are in the morning vs in the afternoon?


| morning_trips | afternoon_trips |
|----|----|
|        404,009 |          576,667 |



Our base definition of a morning trip is any trip that has a start time and end time before 12:00, a trip is considered in the afternoon if its start timestamp is at 12:00 or after. 

In addition for trips that are started before 12:00 and end after 12:00, we take the difference respectively between 12:00 and the start time as well as between the end time and 12:00. Whichever difference is larger tells us that more time of that trip was spent in that portion of the day.

In addition we only select trips that start and end on the same day; trips that overlap days are slightly out of scope as they could be classified as night, full day, etc. as they may have a combination of morning and afternoon. Please note that we have a 24 hour time format, therefore 12:00 represents noon in PST.


```sql

  with START_TIMES as (
        SELECT 
            extract(time from start_date) as start_time, 
            extract(time from end_date) as end_time 
        from `bigquery-public-data.san_francisco.bikeshare_trips`
        where extract(DATE from start_date) = extract(DATE from end_date)

  SELECT 
      SUM(CASE WHEN (start_time < TIME(12, 00, 00) AND end_time < TIME(12, 00, 00)) OR 
              TIME_DIFF(TIME "12:00:00", start_time, MINUTE) > (TIME_DIFF(end_time, TIME "12:00:00", MINUTE))
              THEN 1 ELSE 0 END) morning_trips,
              
      SUM(CASE WHEN (start_time >= TIME(12, 00, 00)) OR 
              TIME_DIFF(TIME "12:00:00", start_time, MINUTE) <= (TIME_DIFF(end_time, TIME "12:00:00", MINUTE))
              THEN 1 ELSE 0 END) afternoon_trips,      
  from START_TIMES

```


#### Project Questions

This section identifies exploratory questions that will be useful for making recommendations to improve ridership.


- What is the spread of the data by landmark/city? 
  * As trends and ridership behavior could be very different for different places.

- What days of the week are most popular?

- What times of the day are most popular?

- What is the average duration of a trip (clean outliers)
  * What does this tell about the type of trip?

- What are the unique zip codes? Total count? (Home Zip Code of the User)

- What are popular zip codes that are using the service?
  * Maybe adjacent untapped zip codes could be a great opportunity

- What stations have an efficient use of their inventory?
  * i.e. what stations consistently have bikes that are being taken out vs. the max docking capacity they have.
  * Is the bike inventory managed incorrectly? Too many bikes in some places, not enough in others?

- What are the top 5 commuter trips?
  * How do we define a commuter trip? 


#### Some Answers...

Here we provide answers to some of the project questions shared above. Others are detailed in the jupyter notebook report.


- What is the spread of the data by landmark/city? 


|   landmark    | trip_counts |
|----|----|
| Redwood City  |        4,996 |
| San Jose      |       52,861 |
| Mountain View |       24,679 |
| San Francisco |      891,223 |
| Palo Alto     |        9,889 |


```sql
SELECT landmark, count(*) as trip_counts
FROM `bigquery-public-data.san_francisco.bikeshare_stations` AS stations
JOIN `bigquery-public-data.san_francisco.bikeshare_trips` AS trips 
ON stations.station_id = trips.start_station_id 
GROUP BY landmark

```


- What days of the week are most popular?

|day_of_week|total_count                  |
|-----------|-----------------------------|
|Tuesday    |184405                       |
|Wednesday  |180767                       |
|Thursday   |176908                       |
|Monday     |169937                       |
|Friday     |159977                       |
|Saturday   |60279                        |
|Sunday     |51375                        |


```sql
SELECT FORMAT_DATE('%A', DATE(start_date)) as day_of_week, COUNT(zip_code) as total_count
FROM `bigquery-public-data.san_francisco.bikeshare_trips` 
GROUP BY day_of_week
ORDER BY total_count DESC

```

From the above data we can see that weekdays are the high usage days, in fact on some days 3 times greater than the weekend. We can make the inference here that this is the case due to our customers who use Lyft bike share to get to their workplace.


- What times of the day are most popular?


|hour_of_day|trip_count|
|-----------|----------|
|0          |2929      |
|1          |1611      |
|2          |877       |
|3          |605       |
|4          |1398      |
|5          |5098      |
|6          |20519     |
|7          |67531     |
|8          |132464    |
|9          |96118     |
|10         |42782     |
|11         |40407     |
|12         |46950     |
|13         |43714     |
|14         |37852     |
|15         |47626     |
|16         |88755     |
|17         |126302    |
|18         |84569     |
|19         |41071     |
|20         |22747     |
|21         |15258     |
|22         |10270     |
|23         |6195      |

```sql
SELECT EXTRACT(HOUR from start_date) as hour_of_day, count(*) as trip_count
FROM `bigquery-public-data.san_francisco.bikeshare_trips`
GROUP BY hour_of_day
ORDER BY hour_of_day
```

We can see the clear rises, peaks, and falls in the flow of trips throughout the day. The data spikes around main commute times, a visualization for this is shown in the report.


- What are the unique zip codes? Total count? (Home Zip Code of the User)

| unique_zip_codes |
|----|
| 8829 |


```sql
SELECT COUNT(DISTINCT zip_code) as unique_zip_codes
FROM `bigquery-public-data.san_francisco.bikeshare_trips`
WHERE zip_code != 'nil'

```

We show the top 10 zip_codes. The full table will be used in our graphical analysis. 

* "Nil" = 15,385 unknown zip codes (NULL VALUES)


| zip_code | trip_count |
|----|----|
| 94107    |      106913 |
| 94105    |       61232 |
| 94133    |       46544 |
| 94103    |       38072 |
| 94111    |       33642 |
| 94102    |       30222 |
| 94109    |       19043 |
| 95112    |       15420 |
| 94158    |       13673 |
| 94611    |       13198 |


These zip codes are the home zip codes of the users, as mentioned in the schema, customers can choose to manually enter their zip code at kiosk's when checking out bikes, or have them implictly populated from the Lyft app. Because of this variance, the data is slightly unreliable. We see this with 15,385 null zip code entries. Considering that our total number of trips are 983,648 our null values are only 1.5% of this, therefore it is somewhat negligible in our analysis.

```sql
SELECT zip_code, COUNT(zip_code) as trip_count
FROM `bigquery-public-data.san_francisco.bikeshare_trips` 
WHERE zip_code != "nil" and zip_code IS NOT NULL and zip_code != ''
GROUP BY zip_code
ORDER BY trip_count DESC
LIMIT 10

```


- What is the relationship with missing zip code and user_type?

The result of this query tells us that there were 15,385 "nil" zip code entries, all of which were from users of the subscriber_type: "Customer". Customer's are defined as 24-hour or 3-day member, so it is somewhat reasonable to assume they are not high retention users, such as our subscribers who are annual or 30-day members.


```sql
SELECT zip_code, subscriber_type, COUNT(zip_code) as total_count
FROM `bigquery-public-data.san_francisco.bikeshare_trips` 
WHERE zip_code = "nil"
GROUP BY zip_code, subscriber_type
ORDER BY total_count DESC
```


- What is the average duration of a commuter trip (clean outliers)

As we calculated earlier the spread turned out to be
* min: 60 (seconds)
* max: 17270400 (seconds) => 4797.3333333 (hours) => 199.88 days
* avg: 1018.9323467338004 (seconds) => 2.8303676298 (hours)


Given that these values feel rather extreme, we will take a graphical look at this spread later. But for now we can try and deduce a reasonable range of which trips are valid.

An assumption that I am making is that trip duration is an indicator of the type of trip (commuter vs.recreational). As per duration a commuter trips on bike could look different for a variety of users, some examples:

* *duration < 10 minutes* : If you need a bike inbetween public transit
* *10 < duration < 60 minutes* : If you are travelling from your home to your workplace
* *duration > 60 minutes* : Most likely not a commuter trip


Pricing: https://www.lyft.com/bikes/bay-wheels/pricing#:~:text=Our%20Access%20Pass%20gives%20you,%243%20per%20additional%2015%20minutes
** NOTE: most trips longer than 30-45 minutes incur extra charges

** NOTE: A combination of analysis with duration and the itenerary of trip is necessary for a more accuracte representation of commuter trips. Below shares what could be considered "potential" commuter trips solely based off the trip durations.


- 360,929 trips
- 36.7% of total trips

```sql
SELECT COUNT(*) as trip_count
FROM `bigquery-public-data.san_francisco.bikeshare_trips`
WHERE duration_sec >= 600 AND duration_sec <= 3600
```

If we stretch our commuter trip duration range by 5 minutes on the lower bound, and 15 minutes on the upper bound, we see that we are able to capture nearly \~80% of all trips taken. This suggests that we should focus new programs to engage commuters as they are the main "type" of customer and implicit spokespersons for our service.

- 780,858 trips
- 79.4% of total trips

```sql
SELECT COUNT(*) as trip_count
FROM `bigquery-public-data.san_francisco.bikeshare_trips`
WHERE duration_sec >= 300 AND duration_sec <= 4500
```

- What are the top 5 commuter trips?

* Total Commuter Trips: 577,049
* 58.6% of Total Trips

We define a commuter trip as following:
* Trip thats duration is between 5 minutes and 75 minutes
* Trip that started between 5 AM and 11 AM 
* Trip that started between 3 PM and 7 PM

According to:
https://bikeleague.org/content/new-census-data-bike-commuting#:~:text=Commute%20time%3A%20The%20average%20bicycle,10%20and%2014%20minutes%20long.

The average commuter bike trip is 19.3 minutes, given the variance of this number we have chosen to keep a wider window for valid durations. Because we are grouping by start and end station, we get are able to aggregate on the most common trips over the timeline of our data. We have chosen not to filter on weekday vs. weekend as there very well may be people working on the weekends, and we do not have the data to support a clear distinction.


|start_station_name|end_station_name             |avg_trip_time (minutes)   |trip_count|
|------------------|-----------------------------|-----------------|----------|
|2nd at Townsend   |Harry Bridges Plaza (Ferry Building)|8.539|6311      |
|Harry Bridges Plaza (Ferry Building)|2nd at Townsend              |9.865|5844      |
|Embarcadero at Folsom|San Francisco Caltrain (Townsend at 4th)|10.315|5694      |
|San Francisco Caltrain (Townsend at 4th)|Harry Bridges Plaza (Ferry Building)|11.949|5457      |
|Harry Bridges Plaza (Ferry Building)|Embarcadero at Sansome       |12.177|5426      |


```sql

SELECT start_station_name, end_station_name, avg(duration_sec)/60 as avg_trip_time, count(*) as trip_count, SUM(count(*)) OVER() AS total_trips
FROM `bigquery-public-data.san_francisco.bikeshare_trips`
WHERE   
  (duration_sec >= 300 AND duration_sec <= 4500) AND
  
  (((EXTRACT(TIME FROM start_date)) >= TIME(5, 00, 00) AND (EXTRACT(TIME FROM start_date)) <= TIME(11, 00, 00))
  
  OR
    
  ((EXTRACT(TIME FROM start_date)) >= TIME(15, 00, 00) AND (EXTRACT(TIME FROM start_date)) <= TIME(19, 00, 00)))

GROUP BY start_station_name, end_station_name 
ORDER BY trip_count DESC

```


**Please move on to the Project 1 python notebook for an in-depth look at the Lyft Bay Wheels Data!**





