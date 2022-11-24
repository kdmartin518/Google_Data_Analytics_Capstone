
I downloaded the files from the previous twelve months and unzipped them. Everything is kept on an encrypted local drive. 

I am running a MySQL database. I imported the data into a single table using LOAD DATA INFILE queries. Duplicate rows are automatically discarded during this process. 

## Cleaning

Original dataset: tripdata_combined, 5,883,043 rows. 

Remove blank rows

```
CREATE TABLE tripdata_cleaned_1 
SELECT *
FROM tripdata_combined
EXCEPT
SELECT *
FROM tripdata_combined
WHERE 	(start_lat = 0 OR start_lng = 0) AND (start_station_name = '' OR start_station_id = '')

CREATE TABLE tripdata_cleaned_2 
SELECT *
FROM tripdata_cleaned_1
EXCEPT
SELECT *
FROM tripdata_cleaned_1
WHERE (end_lat = 0 OR end_lng = 0) AND (end_station_name = '' OR end_station_id = '')
```

Removed 5,727 rows. 

Calculate duration of ride. Stored as decimal number of hours.

```
CREATE TABLE tripdata_add_duration
SELECT *,
	(UNIX_TIMESTAMP(ended_at) - UNIX_TIMESTAMP(started_at))/(60*60) AS duration_hours_dec
FROM tripdata_cleaned_2

CREATE INDEX idx_trip_duration
ON tripdata_add_duration (duration_hours_dec)
```

This query finds all rows where ended_at is before or equal to started_at.

```
CREATE TABLE tripdata_cleaned_4
SELECT *
FROM tripdata_add_duration
EXCEPT
SELECT *
FROM tripdata_add_duration
WHERE duration_hours_dec <= 0
```

Removed 12,060 rows.

This query finds all tows where duration is longer than 7 hours.

```
CREATE TABLE tripdata_cleaned_5
SELECT *
FROM tripdata_cleaned_4
EXCEPT
SELECT *
FROM tripdata_cleaned_4
WHERE duration_hours_dec > 7
```

Removed 4,916 rows

# Transformation

Reformat dataset into tables to aggregate from.

```
CREATE TABLE working_copy
SELECT 	ride_id,
		started_at AS `date`,
		duration_hours_dec AS duration,
		member_casual,
		rideable_type
FROM tripdata_cleaned_5

CREATE TABLE `aggregate_average_durations_and_count`
SELECT `date`, `hour`, member_casual, rideable_type, AVG(duration) AS avg_duration, COUNT(*) as count
FROM (

	SELECT DATE(`date`) AS `date`, HOUR(`date`) AS `hour`, member_casual, rideable_type, duration
	FROM tripdata_by_ride_id 
	GROUP BY `date`, `hour`, member_casual, rideable_type, duration
	ORDER BY `date`, `hour`, member_casual, rideable_type, duration		
) t
GROUP BY `date`, `hour`, member_casual, rideable_type

CREATE TABLE latlng_by_ride_id
	SELECT 	ride_id, 
			started_at AS `date`,
			ROUND(start_lat,3) AS lat,
			ROUND(start_lng,3) AS lng,
			'start' AS `start_end`,
			member_casual
	FROM tripdata_cleaned_5
	UNION ALL
	SELECT 	ride_id,
			started_at AS `date`,
			ROUND(end_lat,3) AS lat,
			ROUND(end_lng,3) AS lng,
			'end' AS `start_end`,
			member_casual
	FROM tripdata_cleaned_5

CREATE TABLE aggregate_latlng_by_date_hour_truncated
SELECT `date`, `hour`, lat, lng, `start_end`, member_casual, COUNT(*) as count
FROM (
	SELECT DATE(`date`) AS `date`, HOUR(`date`) AS `hour`, ROUND(lat,1) AS lat, ROUND(lng,1) AS lng, start_end, member_casual
	FROM lat_lng_by_ride_id
) t
GROUP BY `date`, `hour`, lat, lng, `start_end`, member_casual
```

## Aggregation

```   
CREATE TABLE aggregate_avg_duration_and_count_by_month_weekday_hour
    SELECT MONTHNAME(`date`) AS `month`, WEEKDAY(`date`) AS `weekday`, `hour`, 
    		member_casual, 
			rideable_type, 			
			MIN(avg_duration) AS min_duration, 
			AVG(avg_duration) AS avg_duration, 
			MAX(avg_duration) AS max_duration,			
			MIN(count) AS min_count, 
			AVG(count) AS avg_count, 
			MAX(count) as max_count	
	FROM aggregate_average_durations_and_count
	GROUP BY `month`, `weekday`, `hour`, member_casual, rideable_type
    ORDER BY `month`, `weekday`, `hour`

CREATE TABLE aggregate_latlng_by_month
SELECT MONTHNAME(`date`) AS `month`, lat, lng, `start_end`, member_casual, AVG(`count`) 
FROM aggregate_latlng_by_date_hour_truncated
GROUP BY `month`, lat, lng, `start_end`, member_casual

CREATE TABLE aggregate_latlng_by_weekday
SELECT WEEKDAY(`date`) AS `weekday`, lat, lng, `start_end`, member_casual, AVG(`count`) 
FROM aggregate_latlng_by_date_hour_truncated
GROUP BY `weekday`, lat, lng, `start_end`, member_casual

CREATE TABLE aggregate_latlng_by_hour
SELECT `hour`, lat, lng, `start_end`, member_casual, AVG(`count`) 
FROM aggregate_latlng_by_date_hour_truncated
GROUP BY `hour`, lat, lng, `start_end`, member_casual
```
