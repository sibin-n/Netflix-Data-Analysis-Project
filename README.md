# Netflix Data Analysis Project Using SQL

## Overview
This analysis explores Netflix’s content strategy, focusing on content distribution, genre and rating preferences, and regional trends for Movies and TV Shows. Independently crafted questions guide the project to uncover audience trends, regional preferences, and insights for optimizing engagement and content reach.

## Objective
Goal is to provide insights on optimizing content release strategies and improving audience engagement across different countries.

## Dataset

Dataset is collected from Kaggle:

- **Dataset Link:** [Netflix Dataset](https://www.kaggle.com/datasets/octopusteam/full-netflix-dataset)

## Schema

```sql
CREATE TABLE netflix_data
(
	show_id	VARCHAR(10)PRIMARY KEY,
	type VARCHAR(7),
	title VARCHAR(150),
	director VARCHAR(250),
	casts VARCHAR(800),
	country	VARCHAR(150),
	date_added	VARCHAR(20),
	release_year INT,
	rating	VARCHAR(10),
	duration VARCHAR(10),
	listed_in VARCHAR(100),
	description VARCHAR(300)
);
```

## Data Cleaning

```sql
SELECT show_id, COUNT(*) FROM netflix_data
GROUP BY show_id
HAVING COUNT(*)>1; -- No Duplicate Rows are Present


-- Checking For Null Values

SELECT COUNT(*) FROM netflix_data
WHERE show_id IS NULL OR show_id=''; -- show_id has no null values

SELECT COUNT(*) FROM netflix_data
WHERE type IS NULL OR type='';       -- type has no null values

SELECT COUNT(*) FROM netflix_data
WHERE title IS NULL OR title='';     -- title has no null values

SELECT COUNT(*) FROM netflix_data
WHERE director IS NULL OR director=''; -- Director have null Values(2634)

SELECT COUNT(*) FROM netflix_data
WHERE casts IS NULL OR casts='';       --  Casts have null values(825)

SELECT COUNT(*) FROM netflix_data
WHERE country IS NULL OR country='';  --  Country have null values(831)

SELECT COUNT(*) FROM netflix_data
WHERE date_added IS NULL OR date_added='';  -- Date_added have null values(10)

SELECT COUNT(*) FROM netflix_data
WHERE release_year IS NULL;               -- Release year have no null values

SELECT COUNT(*) FROM netflix_data
WHERE rating IS NULL OR rating='';       -- rating have null value(4)

SELECT COUNT(*) FROM netflix_data
WHERE duration IS NULL OR duration='';  -- duration have null value(3)

SELECT COUNT(*) FROM netflix_data
WHERE listed_in IS NULL OR listed_in='';  -- listed_in have no null values

SELECT COUNT(*) FROM netflix_data
WHERE description IS NULL OR description=''; -- description has no null values


-- Replacing Director, Cast, Country with Unknown and rest is keep as null(null % below 5) --

UPDATE netflix_data
SET director = CASE WHEN director IS NULL THEN 'Unknown' ELSE director END,
casts = CASE WHEN casts IS NULL THEN 'Unknown' ELSE casts END,
country = CASE WHEN country IS NULL THEN 'Unknown' ELSE country END;


-- Wrong Value changed from column rating --

SELECT COUNT(rating) FROM netflix_data
WHERE rating LIKE '%min%';

UPDATE netflix_data                 
SET rating= CASE WHEN rating LIKE '%min%' THEN NULL ELSE rating END;


-- Formatting Date Column --

UPDATE netflix_data
SET date_added = TO_CHAR(TO_DATE(date_added, 'Month DD, YYYY'), 'YYYY-MM-DD');

UPDATE netflix_data
SET date_added = TO_DATE(date_added,'YYYY-MM-DD');


-- Inserted a new column with value year from date_added --

ALTER TABLE netflix_data
ADD COLUMN date_added_year INT;

UPDATE netflix_data
SET date_added_year= CAST(LEFT(date_added, 4) AS INT);

SELECT date_added,date_added_year FROM netflix_data;
```

## Analysis

### 1. What is the current distribution of Movies vs. TV Shows in Netflix's catalog?

```sql
SELECT type, COUNT(*) Total_Content FROM netflix_data
GROUP BY type;
```

### 2. Which ratings are most common for Movies and TV Shows, and do certain ratings dominate specific content types?

```sql
WITH content_rating AS (
SELECT type,rating, COUNT(*) Total_Count
FROM netflix_data
GROUP BY type,rating
ORDER BY Total_Count DESC)

SELECT r.type, r.rating, r.total_count FROM (
SELECT *,
           RANK() OVER (PARTITION BY type ORDER BY Total_Count DESC) AS ranking
    FROM content_rating
) r
WHERE ranking=1;
```

### 3. Which years saw the highest volume of content added, and how does this break down between Movies and TV Shows?

```sql
-- For Movies
SELECT date_added_year, COUNT(*)Total_Content FROM netflix_data
WHERE type ='Movie'
GROUP BY date_added_year
ORDER BY Total_Content DESC
LIMIT 10;

-- For TV Shows
SELECT date_added_year, COUNT(*)Total_Content FROM netflix_data
WHERE type ='TV Show'
GROUP BY date_added_year
ORDER BY Total_Content DESC
LIMIT 10;
```

### 4. Is there a trend in Netflix adding "fresh" Movies & TV Show those released within the same year as their original release date?

```sql
WITH release_year AS (
SELECT date_added_year,
COUNT(CASE WHEN date_added_year=release_year AND type='Movie' THEN 1 ELSE NULL END) Movies,
COUNT(CASE WHEN date_added_year=release_year AND type='TV Show' THEN 1 ELSE NULL END) TV_Shows,
COUNT(CASE WHEN type='Movie' THEN 1 ELSE NULL END) Total_Movies,
COUNT(CASE WHEN type='TV Show' THEN 1 ELSE NULL END) Total_TV_Show
FROM netflix_data
GROUP BY date_added_year
ORDER BY Movies DESC
LIMIT 6)

SELECT date_added_year, Movies,
ROUND((Movies::numeric/Total_Movies)*100,2) Achievement_percentage_of_movies,
TV_Shows,
ROUND((TV_Shows::numeric/Total_TV_Show)*100,2) Achievement_percentage_of_tv_shows
FROM release_year;
```

### 5.  How are Movies and TV Shows distributed across each country, and are there notable differences in content availability by region?

```sql
WITH content_across_countries AS (
SELECT type, country,
       UNNEST(string_to_array(country, ',')) AS available_country
FROM netflix_data)

SELECT TRIM(available_country) Country, 
COUNT(CASE WHEN type='Movie' THEN 1 ELSE NULL END) Total_movie_content,
COUNT(CASE WHEN type='TV Show' THEN 1 ELSE NULL END) Total_tvshow_content
FROM content_across_countries
GROUP BY TRIM(available_country)
ORDER BY Total_movie_content DESC;
```

### 6. What are the most popular ratings for Movies and TV Shows in different countries?

```sql
-- Rating Across Countries -- 
WITH content_across_country AS (
SELECT rating, country,
       UNNEST(string_to_array(country, ',')) AS available_country
FROM netflix_data),

content_across_countries_with_rating as (
SELECT TRIM(available_country) Country, rating, COUNT(*) Total_Content
FROM content_across_country
WHERE available_country <>''
GROUP BY TRIM(available_country), rating
ORDER BY TRIM(available_country))

SELECT r.country, r.rating, r.total_content FROM 
 (SELECT * ,
 RANK() OVER (PARTITION BY country ORDER BY total_content DESC) ranking
 FROM content_across_countries_with_rating)r
WHERE ranking =1
ORDER BY total_content DESC;

-- Most rating across countries by types --
WITH content_across_country AS (
SELECT country, type, rating,
       UNNEST(string_to_array(country,',')) AS available_country
FROM netflix_data),

Content_across_countries_type AS (
SELECT TRIM(available_country) Country, type, rating, COUNT(*) Total_Content
FROM content_across_country
WHERE available_country <>''
GROUP BY TRIM(available_country), type, rating
ORDER BY TRIM(available_country))

SELECT q.Country, q.type, q.rating, q.total_content FROM 
 (SELECT * ,
 RANK() OVER (PARTITION BY country, type ORDER BY total_content DESC) ranking
 FROM Content_across_countries_type)q
WHERE ranking =1
ORDER BY total_content DESC, country asc;
```

### 7. Are there specific genre preferences within each country, and are any genres particularly popular?

```sql
WITH country_with_genre AS (
SELECT listed_in,
       UNNEST(string_to_array(country,',')) AS available_country
FROM netflix_data),

Countries_and_genre AS (
SELECT TRIM(available_country) Country,listed_in, COUNT(*) Total_Content
FROM country_with_genre
WHERE available_country <>''
GROUP BY TRIM(available_country),listed_in
ORDER BY TRIM(available_country))

SELECT q.Country, q.listed_in, q.total_content FROM 
 (SELECT * ,
 RANK() OVER (PARTITION BY country ORDER BY total_content DESC) ranking
 FROM Countries_and_genre)q
WHERE ranking =1
ORDER BY total_content DESC, country asc;
```

### 8. How does the popularity of various genres differ between Movies and TV Shows across countries?

```sql
WITH country_with_genre AS (
SELECT listed_in,type,
       UNNEST(string_to_array(country,',')) AS available_country
FROM netflix_data),

Countries_and_genre AS (
SELECT TRIM(available_country) Country,type, listed_in, COUNT(*) Total_Content
FROM country_with_genre
WHERE available_country <>''
GROUP BY TRIM(available_country),type, listed_in
ORDER BY TRIM(available_country))

SELECT q.Country, q.type, q.listed_in, q.total_content FROM 
 (SELECT * ,
 RANK() OVER (PARTITION BY country,type ORDER BY total_content DESC) ranking
 FROM Countries_and_genre)q
WHERE ranking =1
ORDER BY total_content DESC, country asc;
```

### 9. Does Netflix prioritize "fresh" content releases by genre in each country to maximize audience reach?

```sql
WITH Countries_with_genre AS (
SELECT 
TRIM(UNNEST(string_to_array(country, ','))) AS available_country,
release_year, date_added_year, listed_in
FROM netflix_data),

Total_content AS (
SELECT available_country,release_year, listed_in, COUNT(*) Total_content
FROM Countries_with_genre
GROUP BY available_country,release_year, listed_in),

same_year_content AS (
SELECT available_country AS Country, release_year, listed_in, COUNT(*) AS Total_content_with_same_year
FROM countries_with_genre
WHERE release_year = date_added_year
GROUP BY Country, release_year, listed_in),

content_with_same_year_and_total AS (
SELECT 
    t.available_country,
    t.release_year,
    t.listed_in,
    t.Total_content,
    Total_content_with_same_year
FROM Total_content t
JOIN Same_year_content s
    ON t.available_country = s.Country 
   AND t.release_year = s.release_year 
   AND t.listed_in = s.listed_in
ORDER BY t.Total_content DESC),

genre_in_countries AS (
SELECT r.available_country,
    r.release_year,
    r.listed_in,
    r.Total_content,
	r.total_content_with_same_year
	FROM (
SELECT *,
RANK() OVER (PARTITION BY available_country, release_year ORDER BY total_content DESC) Ranking
FROM Content_with_same_year_and_total
WHERE available_country <>''
ORDER BY total_content Desc)r
WHERE ranking=1)

SELECT *,
ROUND((total_content_with_same_year::numeric/total_content)*100,2) achievement_rate
FROM genre_in_countries;
```

### 10. Does Netflix prioritize "fresh" content releases by rating in each country based on local popularity?

```sql
WITH countries_fresh_genre AS (
SELECT UNNEST(string_to_array(country,',')) AS available_country,
release_year, type, rating FROM netflix_data
WHERE date_added_year=release_year),

country_with_genre AS (
SELECT TRIM(available_country) country, release_year, type, rating, COUNT(rating) Total_Content
FROM countries_fresh_genre
WHERE TRIM(available_country) <>''
GROUP BY country, release_year, type, rating)

SELECT q.Country, q.release_year, type, q.rating, q.total_content FROM 
 (SELECT * ,
 RANK() OVER (PARTITION BY country, release_year,type ORDER BY total_content DESC) ranking
 FROM country_with_genre)q
WHERE ranking =1
ORDER BY total_content DESC, country asc;
```

### 11. Do audiences show specific intrest in genre preferences, and are there particular rating preferences within those genres?

```sql
WITH Rating_across_genre AS (
SELECT  listed_in,rating,COUNT(*) Total_Content
FROM netflix_data
GROUP BY listed_in,rating
ORDER BY total_content DESC)

SELECT r.listed_in, r.rating,r.total_content FROM 
(SELECT *,
RANK() OVER (PARTITION BY listed_in ORDER BY total_content DESC) ranking
FROM Rating_across_genre)r
WHERE ranking=1
ORDER BY r.total_content DESC;
```

### 12. Are there actors who frequently appear in certain genres and ratings, and does their presence correlate with increased audience reach?

```sql
WITH artist_with_genre AS (
SELECT 
    listed_in, 
    rating,
    UNNEST(STRING_TO_ARRAY(casts, ',')) AS Artist
FROM netflix_data)

SELECT listed_in, TRIM(Artist) Artists, COUNT (*) Total_Content
FROM artist_with_genre
WHERE artist<>'Unknown'
GROUP BY listed_in, TRIM(Artist)
ORDER BY total_content DESC
LIMIT 10;
```

## Findings
- Most Netflix content for Movies and TV Shows is rated for mature audiences, with **60%** falling under the TV-MA rating. The remaining ratings are significantly lower by comparison. By expanding content in other rating categories, Netflix could potentially attract a broader audience and enhance viewership diversity.

- The average yearly growth rate of Netflix content was around **176% for Movies** and **193% for TV Shows**. However, in **2020 and 2021**, this growth slowed, likely due to the **COVID-19 pandemic**, bringing the average growth rate down to **128% for Movies** and **163% for TV Shows**.

- Netflix has been increasing its addition of “fresh” content (Movies & TV Shows released and added within the same year) compared to previous years. However, when comparing total Movies with their original release dates, Netflix still struggles to consistently deliver the content as fresh as possible, with an average freshness rating of **-4.618% from 2016-2021**. For TV Shows, there has been an average growth rate of **5.5%**, showing a steady but moderate increase in timely additions.

- Netflix delivers a significant portion of its content to the United States, with **45%** of Movie content and **35%** of TV Show content available for U.S. viewers. This represents a notable disparity compared to other countries, highlighting a primary focus on the U.S. market.

- Mature audience-rated content dominates Netflix's library globally, as most countries feature this rating prominently. However, certain countries stand out with a higher prevalence of other-rated content, surpassing mature audience ratings for both Movies and TV Shows.

- In the United States, Netflix content leans heavily toward documentaries, while in India, popular genres include drama, international movies, or a combination of both. Japan's top genre is anime, reflecting varied preferences across countries. Although there’s minimal difference in genre distribution between Movies and TV Shows for most regions, some countries do show distinct preferences, with TV Show content differing slightly from Movies.

- A few artists have appeared in specific genres more than 10 times, showcasing their strong association with those genres and their popularity within them.


## Suggestions

- Expand content in underrepresented ratings to attract broader demographics, particularly in regions where mature content is less dominant.

- Aim to release Movies and TV Shows closer to their original release dates to capitalize on their initial buzz and increase audience engagement.

- Promote frequently featured artists in specific genres to attract loyal fanbases and boost viewership.
