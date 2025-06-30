# My-Spotify-data
This case study talks about my personal listening history ,marquee campaigns and streaming History. 

## PROBLEM STATEMENT
So, I requested for my personal Spotify data and I want to see how Marquee campaigns influence my listening behavior and artist popularity over time, as measured by streaming activity and my search interest across my two devices.

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
 2)searchTime:
 3) searchQuery:

  ## DATA CLEANING PROCESS
 - Data Extraction using Numpy (Python library) from JSON to csv
    
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
## BUSINESS QUESTIONS
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

- What are the most common search queries?
- What is the total listening time (in minutes) for each segment?
- Which artist had the highest total listening time among those who ran a Marquee campaign?
- Which listener type is most influenced by marquee ads?
- Is there a pattern between marquee impressions → search → stream?

