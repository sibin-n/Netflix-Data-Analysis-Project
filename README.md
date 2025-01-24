# Netflix Data Analysis Project Using SQL(PostgreSQL)
![Image](https://github.com/user-attachments/assets/f8f36c63-a4f3-437e-8a9a-63f92ce4622f)
## Problem Statement
Streaming platforms like Netflix host extensive catalogs of movies and TV shows but often face challenges in maximizing user engagement and retention. With a growing library of content, understanding patterns in viewer preferences and content characteristics is crucial for delivering personalized recommendations and curating engaging experiences. How can we analyze this dataset to extract actionable insights about the content catalog and its alignment with audience preferences?

## Objective
To explore Netflix's content library and uncover insights into the distribution, diversity, and characteristics of movies and TV shows.

## Dataset

Dataset is collected from Kaggle:

- **Dataset Link:** [Netflix Dataset](https://docs.google.com/spreadsheets/d/15aYozX751NvKL08zy7H0WyUZp7RLucV4271zrq0Q-nE/edit?gid=0#gid=0)

## About
The analysis focuses on understanding user behavior to ensure that the content aligns with what users truly want, helping to minimize churn.
### Common User Behavior
- Browsing Behavior
- Time & Mood Dependency
- Social Behavior
- Multi Tasking
- Decision Making Patterns
- Viewing Habit
- Exploring New Content

## Schema

```sql
CREATE TABLE netflix
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
SELECT * FROM netflix;

SELECT show_id, COUNT(*) FROM netflix
GROUP BY show_id
HAVING COUNT(*)>1;                       -- No Duplicate Value has been found -- 

SELECT COUNT(*) FROM netflix
WHERE show_id IS NULL OR show_id=''; -- show_id has no null values

SELECT COUNT(*) FROM netflix
WHERE type IS NULL OR type='';       -- type has no null values

SELECT COUNT(*) FROM netflix
WHERE title IS NULL OR title='';     -- title has no null values

SELECT COUNT(*) FROM netflix
WHERE director IS NULL OR director=''; -- Director have null Values(2634)--

SELECT COUNT(*) FROM netflix
WHERE casts IS NULL OR casts='';       --  Casts have null values(825) --

SELECT COUNT(*) FROM netflix
WHERE country IS NULL OR country='';  --  Country have null values(831) --

SELECT COUNT(*) FROM netflix
WHERE date_added IS NULL OR date_added='';  -- Date_added have null values(10) --

SELECT COUNT(*) FROM netflix
WHERE release_year IS NULL;               -- Release year have no null values

SELECT COUNT(*) FROM netflix
WHERE rating IS NULL OR rating='';       -- rating have null value(4) --

SELECT COUNT(*) FROM netflix
WHERE duration IS NULL OR duration='';  -- duration have null value(3) --

SELECT COUNT(*) FROM netflix
WHERE listed_in IS NULL OR listed_in='';  -- listed_in have no null values

SELECT COUNT(*) FROM netflix
WHERE description IS NULL OR description=''; -- description has no null values


-- Replacing Director, Cast, Country with Unknown and rest is keep as null(null % below 5) --

UPDATE netflix
SET director = CASE WHEN director IS NULL OR director='' THEN 'Unknown' ELSE director END,
casts = CASE WHEN casts IS NULL OR casts='' THEN 'Unknown' ELSE casts END,
country = CASE WHEN country IS NULL OR country='' THEN 'Unknown' ELSE country END;


-- 1. Unique values in show_id
SELECT DISTINCT show_id FROM netflix;

-- 2. Unique values in type
SELECT DISTINCT type FROM netflix;

-- 3. Unique values in title
SELECT DISTINCT title FROM netflix;

-- 4. Unique values in director
SELECT DISTINCT director FROM netflix;

-- 5. Unique values in casts
SELECT DISTINCT casts FROM netflix;

-- 6. Unique values in country
SELECT DISTINCT country FROM netflix;

-- 7. Unique values in date_added
SELECT DISTINCT date_added FROM netflix;

-- 8. Unique values in release_year
SELECT DISTINCT release_year FROM netflix
ORDER BY release_year;

-- 9. Unique values in rating
SELECT DISTINCT rating FROM netflix;

SELECT COUNT(rating) FROM netflix
WHERE rating LIKE '%min%';     -- Need to Update --

-- 10. Unique values in duration
SELECT duration FROM netflix
WHERE type='Movie'AND duration NOT LIKE '%min';

SELECT duration FROM netflix
WHERE type='Tv Show'AND duration NOT LIKE '%season%';

-- 11. Unique values in listed_in
SELECT DISTINCT listed_in FROM netflix;

-- 12. Unique values in description
SELECT DISTINCT description FROM netflix;


-- Changed data error in rating
UPDATE netflix               
SET rating= CASE WHEN rating LIKE '%min%' THEN 'Unknown' ELSE rating END;


-- Updating char to date
UPDATE netflix
SET date_added = TO_CHAR(TO_DATE(date_added, 'Month DD, YYYY'), 'YYYY-MM-DD')
WHERE LENGTH(date_added)>10;

UPDATE netflix
SET date_added= TO_DATE(date_added,'YYYY-MM-DD'); 

ALTER TABLE netflix
ALTER COLUMN date_added TYPE DATE
USING TO_DATE(date_added, 'YYYY-MM-DD');
```

## Analysis

### Current distribution of Movies vs. TV Shows in Netflix's catalog

```sql
SELECT type, COUNT(*) total FROM netflix  -- Count of diff Content
GROUP BY type;
```
![Image](https://github.com/user-attachments/assets/fa3acbcc-0803-4c41-941e-61d903dc40ea)
#### Insight : More than 50% of content is movie.

### 2. Yearly Trend

```sql
SELECT EXTRACT(YEAR FROM date_added) year_added, COUNT(CASE WHEN type='Movie' THEN 1 END) Movies,
COUNT(CASE WHEN type='TV Show' THEN 1 END) TV_Shows FROM netflix
GROUP BY EXTRACT(YEAR FROM date_added)
ORDER BY year_added;
```
![Image](https://github.com/user-attachments/assets/bbc8bd2a-29d2-4280-afe1-d2b979859776)
#### Insight : Before COVID-19, the average content growth rate was 212% for movies and 96% for TV shows. Post-COVID, growth slowed to 177% for movies and 80% for TV shows.	

### 3. Content Released & Added Within a Year

```sql
SELECT EXTRACT(YEAR FROM date_added) year_added, COUNT(CASE WHEN type='Movie' THEN 1 END) Movies,
COUNT(CASE WHEN type='TV Show' THEN 1 END) TV_Shows
FROM netflix
WHERE release_year>=2008 AND EXTRACT(YEAR FROM date_added)=release_year
GROUP BY EXTRACT(YEAR FROM date_added)
ORDER BY year_added;                            --- Movies added in same year--

SELECT release_year, COUNT(CASE WHEN type = 'Movie' THEN 1 END) AS Movies,
    COUNT(CASE WHEN type = 'TV Show' THEN 1 END) AS TV_Shows
	FROM netflix
	WHERE release_year>=2008
	GROUP BY release_year
	ORDER BY release_year;   -- total actual release in a year --
```
![Image](https://github.com/user-attachments/assets/fa754187-b739-4712-8448-5886573e920a)

### 4. Content Distribution Across Countries

```sql
WITH countries_and_content AS (
    SELECT 
        TRIM(UNNEST(STRING_TO_ARRAY(country, ','))) AS Country, 
        type 
    FROM netflix
)
SELECT 
    Country, 
    COUNT(CASE WHEN type = 'Movie' THEN 1 END) AS Movies,
    COUNT(CASE WHEN type = 'TV Show' THEN 1 END) AS TV_Shows
FROM countries_and_content
GROUP BY Country
ORDER BY Country;
```
![Image](https://github.com/user-attachments/assets/23d04fdf-2904-43bd-bf97-244559c45744)
#### Insight: Netflix delivers a substantial share of its content to the United States, with 35% of movies and 31% of TV shows. However, limited availability of regional content could lead to viewer churn, as audiences tend to prefer content from their own country.


### 5.  Rating Across Content Type

```sql
SELECT rating, COUNT(CASE WHEN type='Movie' THEN 1 END) Movies,
COUNT(CASE WHEN type='TV Show' THEN 1 END) TV_Shows FROM netflix
GROUP BY rating;
```
![Image](https://github.com/user-attachments/assets/119d242d-618c-4c58-8dc4-9da9015ae166)
#### Insight: Mature audience-rated content dominates Netflix's library globally, as most countries feature this rating prominently. So the country with low content can release new Mature audience-rated regional content of their own which help to gain the market of each country.

### 6. Genre for Movies & TV Show

```sql
WITH genre_and_content AS (
    SELECT 
        TRIM(UNNEST(STRING_TO_ARRAY(listed_in, ','))) AS genre, 
        type 
    FROM netflix
)
SELECT 
    genre, 
    COUNT(CASE WHEN type = 'Movie' THEN 1 END) AS Movies,
    COUNT(CASE WHEN type = 'TV Show' THEN 1 END) AS TV_Shows
FROM genre_and_content
GROUP BY genre;
```
![Image](https://github.com/user-attachments/assets/475e4879-59f7-49f2-b459-568fac9b9378)
#### Insight : The top three genres for both movies and TV shows are International, Dramas, and Comedies. Notably, over 50% of movie content falls within these genres.


### 7. Actors and Director with most content in netflix

```sql
SELECT director, COUNT(CASE WHEN type = 'Movie' THEN 1 END) AS Movies,
    COUNT(CASE WHEN type = 'TV Show' THEN 1 END) AS TV_Shows FROM netflix
GROUP BY director
ORDER BY movies DESC
OFFSET 1;          -- Directors


SELECT TRIM(UNNEST(STRING_TO_ARRAY(casts,','))) artist,
COUNT(CASE WHEN type = 'Movie' THEN 1 END) AS Movies,
    COUNT(CASE WHEN type = 'TV Show' THEN 1 END) AS TV_Shows
FROM netflix
GROUP BY artist;    --- Artist --
```
![Image](https://github.com/user-attachments/assets/7a4a7c69-c9ce-4378-bbe0-72698234dc52)

### 8. Length of content in Movies & TV Show

```sql
SELECT duration, COUNT(*) total FROM netflix
WHERE type='Movie'
GROUP BY duration
ORDER BY total DESC;                                         -- Movie Duration ---

SELECT duration, COUNT(*) total FROM netflix
WHERE type='TV Show'
GROUP BY duration
ORDER BY total DESC;                                      -- Web Series Duration -- 
```
![Image](https://github.com/user-attachments/assets/2bfab8d3-53c9-4671-837e-a948c071407f)
#### Insight : Most movies have a runtime of around 90 minutes, making the 1-2 hour range the most appealing choice for viewers seeking concise entertainment.
**As the number of seasons increases, the volume of content decreases.Shorter seasons are ideal for moderately active viewers, while longer-running series with more seasons are better suited for engaging highly active audiences.**


## Suggestions

- Releasing movies on Netflix soon after their initial release helps create an early buzz and engage viewers effectively. The gap between a movie's release year and its addition to Netflix is generally longer compared to TV shows.

- Currently, Netflix offers content across 20 different genres for both movies and TV shows, with international, drama, and comedies ranking as the top 3 in each category. Expanding and promoting diverse genres can help engage audiences seeking variety in their viewing preferences.

- Featuring the top 20 actors and directors in their respective fields can enhance viewer engagement, as audiences are more likely to watch movies associated with renowned names in the industry.

- The United States has the highest content volume on Netflix, but providing regional content tailored to top user countries can be highly beneficial. Since the most-watched genre is international, viewers enjoy exploring films from other cultures. Focusing on creating mature dramas and comedies can be a great strategy, as mature content currently dominates Netflix's library. Additionally, movies with a runtime of 1-2 hours are increasingly popular, making them a perfect fit for viewers looking for a quick but engaging watch.

