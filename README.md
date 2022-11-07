# Google_Data_Analytics_Capstone

# Cyclistic

I work at Cyclistic, a bike-share program that features more than 5,800 bicycles and 600 docking stations. 

# Ask

## Identify the key business task 

Cyclistic's finance analysts has determined that **Subscribing members** are much more profitable than non-subscribing, **casual members**. Lily Moreno believes that the best way to gain subscribing members is to convert casual members rather than attracting new members altogether.

I am to use data to answer the following questions:

1. How do annual members and casual riders use Cyclistic bikes differently?
2. Why would casual riders buy Cyclistic annual memberships?
3. How can Cyclistic use digital media to influence casual riders to become members?

## Consider key stakeholders

My stakeholders are **Lily Moreno**, the director of marketing and my manager; and the **Cyclistic executive team**, who will decide whether to approve my recommended marketing program.

# Prepare

## Data Organization

Cyclistic collects their own data. They also scrub the data of any personally-identifiable information. I'm given access to the server where the raw data is stored. On this server are zipped CSV files organized by month. 

The data itself are individual rides taken by Cyclistic users. Each row records:

- Date and time ride starts and ends
- latitude and longitude of where ride starts and ends
- Name and ID of bike station where ride starts and ends
- What type of bike the user rode
- Whether the user was a subscriber or a casual member.

## Data Uses

We can use this data to compare between subscribers and casual members and see some things like:

- When do they prefer to ride?
- How long do they ride for?
- Does one group ride more or less during certain times of day/week/year?

## Data Limitations

However, this data does not provide much insight as to how Cyclistic users interact with our service digitally.

# Process

For this project I am using SQL. SQL is an industry standard tool for managing a large amount of data and, confession time, I've never used it for a project before, so now's the time to learn.

I downloaded the files from the previous twelve months and unzipped them. Everything is kept on an encrypted local drive. 

I am running a MySQL database. I imported the data into a single table using LOAD DATA INFILE queries. Duplicate rows are automatically discarded during this process. 

## Cleaning

\#Original dataset: tripdata_combined, 5,883,043 rows. 

\# Remove blank rows

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

\#Removed 5,727 rows. 

\# Calculate duration of ride. Stored as decimal number of hours.

```
CREATE TABLE tripdata_add_duration
SELECT *,
	(UNIX_TIMESTAMP(ended_at) - UNIX_TIMESTAMP(started_at))/(60*60) AS duration_hours_dec
FROM tripdata_cleaned_2

CREATE INDEX idx_trip_duration
ON tripdata_add_duration (duration_hours_dec)
```

\# This query finds all rows where ended_at is before or equal to started_at.

```
CREATE TABLE tripdata_cleaned_4
SELECT *
FROM tripdata_add_duration
EXCEPT
SELECT *
FROM tripdata_add_duration
WHERE duration_hours_dec <= 0
```

\#Removed 12,060 rows.

\# This query finds all tows where duration is longer than 7 hours.

```
CREATE TABLE tripdata_cleaned_5
SELECT *
FROM tripdata_cleaned_4
EXCEPT
SELECT *
FROM tripdata_cleaned_4
WHERE duration_hours_dec > 7
```

\#Removed 4,916 rows

# Analysis

## Transformation

\#Reformat dataset into tables to aggregate from.

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
```

```
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

## Analysis proper

Once I had my tables finalized I exported them into CSV files and began my analysis.

I used both SQL and Tableau to explore statistical patterns. I made the following observations:

Volume is defined as Rides per Hour
- Rider volume for both groups is highest in Summer, lowest in Winter.
	- July
		- Subscribers 282
		- Casual 178
	- January 
		- Subscribers 58
		- Casual 9
- Volume of Casual riders is almost always lower than Subscribers.
	- Members experience peaks at 
		- 8am with 280 rides/hour,
		- 12pm (265)
		- 5pm (483)
	- Casuals also peak at 5pm (209).
- By weekdays are the largest difference:
	- Members 
		- Peak on Tuesday 213
		- Steadily decline until Sunday at 161
	- Casuals are opposite. 
		- Monday 78 to Friday 93, 
		- Saturday 136
		- Sunday 115
	- Monday through Friday, there are more total members on average. But on Saturday/Sunday, casuals overtake members.

# Share

I believe that the data paints a clear picture of the differences between our Subscribing members and our non-subscribing, casual users. 

Subscribers appear to use our service for their daily commute to work. While casual users may also fit this use case, they appear to be more likely to use our service for leisure. 

![Story 2](https://github.com/cruhsaumnt/GoogleDataAnalyticsCapstone/blob/5c761b18a2c3c88fb51a06b926d6c394626e9503/Story%202.png)

The top graph shows average rider volume during different times of day. Notice that our subscribers, in orange, peak at 8am and 5pm, while casual members in blue have a more steady ebb and flow of volume with a slight peak at 5 pm.

The bottom graph compares volume by day of week. Subscribers are more active during the work week from Monday to Friday and dip during the weekend. Casual members, conversely, are not very active during the work week and peak on Saturday and Sunday. 

So it looks like people who need our bikes on a daily basis at 8 am and 5 pm are likely to subscribe. Maybe it's more economical for them to depend on us for their daily commute. But users who don't need us for their commute aren't as likely to subscribe. 

![Story 3](https://github.com/cruhsaumnt/GoogleDataAnalyticsCapstone/blob/5c761b18a2c3c88fb51a06b926d6c394626e9503/Story%203.png)

We just saw how many of each group use our service at different times. Now let's see how long they like to ride our bikes for. 

Subscribers, on the left side of this graph, take trips that are usually around 10 minutes long. You can also see that there's not a big difference between the shortest and the longest trip taken by subscribers on average: between 10 and 15 minutes.

Compare to casual users on the right. Their average trip is around 25 minutes long, but there is a much wider difference between their shortest and longest rides: anywhere from 15 and 35 minutes. There is also a bit more variation based on the day of week.

Subscribers look like they are on a mission! No time to stop and smell the roses, I need to get from A to B in fifteen minutes or less. Casual users are a lot more relaxed in comparison. Their shortest ride is longer than the subscriber's longest, and they don't seem to mind if they take the same amount of time each ride. 

# Act

## Conclusion

### Who are our users?

Folks who need a consistent way to get from A to B in fifteen minutes or less at 8 am and 5 pm every Monday through Friday are likely to subscribe to our service. 

On the other hand, users who like to take longer more leisurely bike rides on the weekend don't appear to have an incentive to subscribe. 

### Why aren't they subscribing? 

Perhaps our casual users ride for leisure less consistently and so they feel that it wouldn't be worth it to pay for a regular membership. 

### Recommendations

1. Since these users ride less frequently but take longer trips and more often on weekends, I would suggest offering a discounted rate for subscribers who ride on two or fewer days per week. 
2. It would also incentivize casual users to ensure that there are enough bikes and bike stations located by leisure areas such as downtown, bike paths, parks, hiking trails, etc.
3. We could also expand our insights by collecting additional data such as surveying casual users on how they like to use our service. 
