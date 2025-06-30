# My-Spotify-data
This case study talks about my personal listening history ,marquee campaigns and streaming History. 

## PROBLEM STATEMENT
So, I requested for my personal Spotify data and I want to see how Marquee campaigns influence my listening behavior and artist popularity over time, as measured by streaming activity and my search interest across my two devices.

## OBJECTIVES
- Evaluate marquee campaign impact
- Track artist popularity trends
- Explore search-to-stream platforms
- Identify most streamed artists
- Identify optimal engagement windows
- Provide insights for personal music discovery

## ABOUT DATASET
I will be working with three tables namely; 
1.	Marquees table
2.	Streaming history table
3.	Search queries table

## LANDING PAGE
- Marquees table
1) artistName:
2) Segment:

- Streaming history table
 1) endTime:
 2) artistName:
 3) TrackName :
 4) msPlayed :
  
- Search queries table
 1) platform:
 2) searchTime:
 3) searchQuery:

 ![Database relationship](https://github.com/Ifeoma28/My-Spotify-data/blob/3b41c7a67a367a80cdd10cb9407dee3abcb10b46/spotify%20relationship%20table.png)

  ## DATA CLEANING PROCESS
 - Data Extraction using Numpy (Python library) from JSON to csv
![python extraction](https://github.com/Ifeoma28/My-Spotify-data/blob/3d39ca1b72602a798579bd6bf949388efe72f99a/pythonsportify_convert.png)

- Data cleaning in Excel by checking for duplicates and spelling errors
 - MSSQL for querying the dataset.
 -  Below are the cleaning techniques performed using sql
 ```
  -- data cleaning process
-- firstly change the nvarchar column in search queries table to datetime

-- removing the UTC
--SELECT searchTime,
--REPLACE(searchTime, '[UTC]', '') AS search_time
--FROM search_queries;

--SELECT searchTime,
--REPLACE(searchTime, '[UTC]', '') AS search_time,
--CAST(REPLACE(searchTime, '[UTC]', '') AS datetime2) AS converted_datetime
--FROM search_queries;

--UPDATE search_queries
--SET searchTime = CAST(REPLACE(searchTime, '[UTC]', '') AS datetime2)
--WHERE searchTime IS NOT NULL;
-- I CHANGED THE SEARCHTIME COLUMN TO A DATETIME2 COLUMN
 ```
## TOOLS USED
- python for data extraction
- excel for cleaning csv file
- MSSQL for querying
- Power BI for data viz

## QUESTIONS USED FOR ANALYSIS
- First of all, evaluate the Marquees table,Which listener segment is most commonly targeted by Marquee?
```
SELECT segment,COUNT(*) AS frequency
FROM Marquee
GROUP BY segment
ORDER BY frequency DESC;
```
- Which artists have the most marquee campaigns? (Appear most often)
 ```
SELECT artistName,COUNT(*) AS frequency
FROM Marquee
GROUP BY artistName
ORDER BY frequency DESC;
```

- What’s the distribution of segments per artist?
```
SELECT artistName,segment,COUNT(*) AS segment_count
FROM Marquee
GROUP BY artistName,segment
ORDER BY artistName,segment_count DESC;
```
- Which artist had the most Marquee reach across segments? (which artists are mostly frequently targeted toward ‘light’ or ‘previously active’ listeners ?
```
SELECT artistName,segment,COUNT(*) AS segment_count
FROM Marquee
GROUP BY artistName,segment
HAVING COUNT(*) >= 2
ORDER BY artistName,segment_count DESC;
```
- For each artist in the Marquee table, calculate total streaming activity from the first time you streamed them (estimated campaign) to the last time?
```
--For each artist in the Marquee table, calculate total streaming activity from the first
--time you streamed them(estimated campaign) to the last time ?
-- the date i first streamed the artist around the time of their appearance
-- in my marquee table is when i saw the campaign.


-- first let us get an estimated earliest stream date per artist and last stream date
WITH streams AS (
SELECT artistName,MIN(endTime) AS earliest_streaming_date,
MAX(endTime) AS last_stream_date
FROM StreamingHistory_music 
WHERE artistName  IN (SELECT artistName FROM marquee)
GROUP BY artistName
),
-- then we want to calculate the total streaming time
streaming_totals AS (
SELECT s.artistName,s.earliest_streaming_date,s.last_stream_date,
COUNT(*) AS stream_count,
ROUND(SUM(st.msplayed)/60000.0,2) AS total_minutes
FROM streams s
JOIN StreamingHistory_music st ON s.artistName = st.artistName
WHERE endTime BETWEEN s.earliest_streaming_date AND s.last_stream_date
GROUP BY s.artistName,s.earliest_streaming_date,s.last_stream_date
)
SELECT *
FROM streaming_totals
ORDER BY total_minutes DESC;
```
- What is the conversion rate from marquee impression to streaming?
```
-- first of all, conversion happens when an artist from the marquee table also appears in your streaming history table
WITH marquee_segment AS (
SELECT DISTINCT artistName,segment
FROM Marquee
),
StreamedArtists AS (
SELECT  artistName
FROM StreamingHistory_music
),
Converted_segment AS (
SELECT ms.segment,COUNT(ms.artistName) AS total_campaigns,
COUNT(CASE WHEN sa.artistName IS NOT NULL THEN 1 END) AS converted_campaigns
FROM marquee_segment ms
LEFT JOIN StreamedArtists sa ON ms.artistName = sa.artistName
GROUP BY ms.segment
)
SELECT *,ROUND(100.0*converted_campaigns/total_campaigns,2) AS conversion_rate_percent
FROM Converted_segment;
```
![stream by segment](https://github.com/Ifeoma28/My-Spotify-data/blob/9310470c952d9a133ebcb64742fc599a9dc785cf/stream%20count%20by%20segment.png)

- Those marquee campaigns that led me to stream, which segments convert best?
 ```
WITH streams AS (
SELECT m.artistName,MIN(endTime) AS estimated_streaming_date,
MAX(endTime) AS last_stream_date,m.segment
FROM StreamingHistory_music sm
JOIN Marquee m ON sm.artistName = m.artistName
GROUP BY m.artistName,m.segment
),
-- then we want to calculate the total streaming time
streaming_totals AS (
SELECT s.artistName,s.estimated_streaming_date,s.last_stream_date,
COUNT(*) AS stream_count,
ROUND(SUM(st.msplayed)/60000.0,2) AS total_minutes,s.segment
FROM streams s
JOIN StreamingHistory_music st ON s.artistName = st.artistName
WHERE endTime BETWEEN s.estimated_streaming_date AND s.last_stream_date
GROUP BY s.artistName,s.estimated_streaming_date,s.last_stream_date,s.segment
)
SELECT *
FROM streaming_totals
ORDER BY total_minutes DESC;
```

- Do super listeners convert less?
Looking at the chart below, they convert best 
![super listeners](https://github.com/Ifeoma28/My-Spotify-data/blob/0e89413b896fa860148ec48d69baf127630eff14/conversion%20rate%20per%20segment.png)

 
- Are light listeners turning into moderate ones over time?
 ```
WITH light_listeners AS (
SELECT DISTINCT artistName
FROM Marquee
WHERE segment = 'light listeners'
),
streamTotals AS (
SELECT sm.artistName,SUM(sm.msPlayed) AS total_ms_played
FROM StreamingHistory_music sm
JOIN light_listeners ll ON sm.artistName = ll.artistName
GROUP BY sm.artistName
),
Engagementlevel AS (
SELECT artistName,total_ms_played,
CASE 
	WHEN total_ms_played < 600000 THEN 'Still light'-- less than 10mins since 600,000 milliseconds make 10 mins
	WHEN total_ms_played BETWEEN 600000 AND 1800000 THEN 'Moderate' -- between 10 and 30 mins
	WHEN total_ms_played  > 1800000 THEN 'Super listener' -- more than 30 mins
	ELSE 'no streams'
	END AS engagement_status
FROM streamTotals
),
Total_count AS (
SELECT COUNT(*)  AS total_artistes FROM light_listeners
)
SELECT engagement_status,COUNT(*) AS artistes_count,ROUND(100.0 * COUNT(*)/total_artistes,2) AS artistes_count_percent
FROM Engagementlevel
CROSS JOIN Total_count
GROUP BY engagement_status,total_artistes;
```

- Are marquee impressions more effective on light listeners vs. moderate/super listeners?

- What is the top-streamed song or segment that account for the highest total number of streams over time?
```
SELECT s.artistName,trackName,COUNT(*) AS stream_count,segment,
ROUND(SUM(msplayed)/60000.0,2) AS total_minutes
FROM StreamingHistory_music s
JOIN Marquee m ON s.artistName = m.artistName
GROUP BY s.artistName,trackName,msPlayed,segment
ORDER BY stream_count DESC;
```
- Track artist popularity trends
Are certain artists gaining or losing popularity over time? (Light listeners that became super listeners)?
Below are the list of light listeners that have been listened to for more than 30 mins which I classified as super listeners.
![light listeners convert](https://github.com/Ifeoma28/My-Spotify-data/blob/1d66d7809fe251a83da8a3d1ff5e629a99e0df78/light%20listeners.png)


- What are the most common search queries?
![search queries](https://github.com/Ifeoma28/My-Spotify-data/blob/6e28e8e64b225a63d0b711d87077026e6ffe9f39/search%20queries.png)

- What is the total listening time (in minutes) for each segment?
- Which artist had the highest total listening time among those who ran a Marquee campaign?
- Which listener type is most influenced by marquee ads?
- Is there a pattern between marquee impressions → search → stream?
- Most engaged days of the week (How does my engagement differ across time (e.g., weekdays vs weekends))?
```
SELECT 
    DATENAME(WEEKDAY, endTime) AS DayOfWeek,
    COUNT(*) AS TotalStreams,
    SUM(msPlayed) / 60000.0 AS TotalListeningMinutes
FROM StreamingHistory
GROUP BY DATENAME(WEEKDAY, endTime)
ORDER BY TotalListeningMinutes DESC;
```
- most engaged listening times during the day ?
```
SELECT 
    DATEPART(HOUR, endTime) AS Hour_of_day,
    COUNT(*) AS TotalStreams,
    SUM(msPlayed) / 60000.0 AS TotalListeningMinutes
FROM StreamingHistory_music
GROUP BY DATEPART(HOUR, endTime)
ORDER BY TotalListeningMinutes DESC;
```

## ANALYSIS
- Conversion vs. Engagement
1) Super Listeners had the highest conversion rate after being targeted ( they responded best to Marquee campaigns).

2) But Previously Active Listeners delivered the highest total listening time, showing deeper post-click engagement.

3) Infinity had the most reach and were  targeted twice towards previously active listeners.

- Artist Engagement Depth

1) Dunsin Oyekan was the artist I spent the most time listening to (565 minutes).

2) Tems had the highest number of streams.

3) “Burning” was the top song by listening time followedby "Olorun Agbaye by Nathaniel Bassey", while “Born in the Wild” had the highest stream count.

4) I can see that Dunsin Oyekan, Taylor Swift, Tems, Benson Boone and Nathaniel Bassey kept my interest the longest (Top 5) based on the total minutes i listened to them.

- Listener Lifecycle Movement, this shows Artist popularity over time
1) Artists like Ariana Grande and Lawrence Oyor moved from light listeners to super listeners, showing organic growth in engagement.

2) Only 9 out of 305 one-time streamed artists were ever targeted in a campaign.

-  Search Behavior Missed by Campaigns

1) More of my searched artists were streamed than those targeted via campaigns.
2) Number of searched artistes that were streamed are more than the number of searched artists in the marquee campaign.

- Top 10 listening days
![Top 10 listening days](https://github.com/Ifeoma28/My-Spotify-data/blob/23a5a679ef66f621478e20889b6a3ea973bf9479/Top%2010%20listening%20days.png)


- Top 2 Peak Listening Hours:
1) 7 PM – 8 PM (19th hour) – Primary engagement window
![peak listening hours](https://github.com/Ifeoma28/My-Spotify-data/blob/d1c3e8e9aa295f5d08db6e45608e60a6aeae2bdc/peak%20listening%20hours.png)

3) 9 AM – 10 AM (9th hour) – Secondary engagement window

## INSIGHTS

➤  Based on the conversion rate for segments, Quick converters aren’t always the most valuable — strategy should balance activation and depth (listening times)

➤ Artist Engagement Depth Shows how time spent and frequency of streams reflect different forms of engagement.

➤  Targeting lightly engaged listeners could improve long-term retention due to their growth over time.

➤ Based on my search queries, Organic interest is not well-aligned with paid targeting — missed opportunity to retarget based on search intent.

➤ My peak listening hours reveals how campaigns, playlists, or artist recommendations could be more effective when aligned with daily mood shifts.

## CONCLUSION
- This project taught me how to transform raw engagement data into actionable strategy.
  
-  I discovered that segment behavior varies not just by volume but by depth  and also that user intent (search) is often a better predictor of future engagement than ad impressions. Focused storytelling and segment-level analysis brought the ‘why’ behind the numbers to life.
  
- I identified that my most engaged listening happens in the evening by 7pm or 9:00am in the morning, especially on Wednesdays.
  
- I also found that my organic search behavior drives most of my new artist discovery — not Spotify's campaigns. By combining streaming behavior with search activity, I could recommend better timing and artist targeting for campaigns, and even improve my own listening experience.
