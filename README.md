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
