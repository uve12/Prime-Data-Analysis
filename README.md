# **Amazon Prime Video Analytics - QuickSight Dashboard**

![](https://github.com/uve12/Prime-Data-Analysis/blob/main/logo.png)

---


**Author:** Uve  
**Project Type:** Data Visualization & SQL Analytics  
**Tools Used:** AWS QuickSight, S3, Redshift, PostgreSQL/MySQL, Excel, PowerBi PowerQuery.

## **1. Project Overview**
This project aims to analyze Amazon Prime Video content using **AWS QuickSight**. The dataset includes movies and TV shows with attributes like title, genre, release year, director, actors, duration, and ratings.  

- The dashboard provides insights into:

- The most popular release years

- Genre trends over time

- Country-wise distribution of content

- Actor and director analysis

- Movie durations and rating distributions 

---

## **2. Database Setup & Table Creation**
Before analyzing the data, let's set up the **database schema**.

### **2.1 Create Database**
```sql
CREATE DATABASE amazon_prime_db;
```

### **2.2 Create Master Table (`amazon_prime_titles`)**
```sql
CREATE TABLE amazon_prime_titles (
    show_id SERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    type TEXT CHECK (type IN ('Movie', 'TV Show')),
    release_year INT,
    primary_genre TEXT,
    rating TEXT,
    duration_num INT,
    country TEXT,
    actor_name TEXT,
    director TEXT
);
```

### **2.3 Insert Sample Data**
```sql
INSERT INTO amazon_prime_titles (title, type, release_year, primary_genre, rating, duration_num, country, actor_name, director) 
VALUES
('Inception', 'Movie', 2010, 'Sci-Fi', 'PG-13', 148, 'USA', 'Leonardo DiCaprio, Joseph Gordon-Levitt', 'Christopher Nolan'),
('Stranger Things', 'TV Show', 2016, 'Drama', 'TV-14', NULL, 'USA', 'Winona Ryder, David Harbour', 'Duffer Brothers'),
('Avengers: Endgame', 'Movie', 2019, 'Action', 'PG-13', 181, 'USA', 'Robert Downey Jr., Chris Evans', 'Anthony Russo, Joe Russo');
```

---

## **3. Business Questions & Queries (Master Table)**
Here are **7 key questions** based on the `amazon_prime_titles` table.

### **1. What are the top 5 years with the highest number of movie releases?**
```sql
SELECT release_year, COUNT(*) AS num_releases
FROM amazon_prime_titles
WHERE type = 'Movie'
GROUP BY release_year
ORDER BY num_releases DESC
LIMIT 5;
```

### **2. Which country has produced the most content?**
```sql
SELECT country, COUNT(*) AS content_count
FROM amazon_prime_titles
GROUP BY country
ORDER BY content_count DESC
LIMIT 1;
```

### **3. How many shows/movies were added each year?**
```sql
SELECT release_year, COUNT(*) AS added_count
FROM amazon_prime_titles
WHERE release_year IS NOT NULL
GROUP BY release_year
ORDER BY release_year DESC;
```

### **4. What is the longest movie in terms of duration?**
```sql
SELECT title, duration_num
FROM amazon_prime_titles
WHERE type = 'Movie'
ORDER BY duration_num DESC
LIMIT 1;
```

### **5. Which movies have more than 3 actors?**
```sql
SELECT title, array_length(string_to_array(actor_name, ', '), 1) AS num_actors
FROM amazon_prime_titles
WHERE array_length(string_to_array(actor_name, ', '), 1) > 3;
```

### **6. Which director has directed the most content?**
```sql
SELECT director, COUNT(*) AS num_shows
FROM amazon_prime_titles
WHERE director IS NOT NULL
GROUP BY director
ORDER BY num_shows DESC
LIMIT 1;
```

### **7. What is the most common parental rating among movies?**
```sql
SELECT rating, COUNT(*) AS count
FROM amazon_prime_titles
WHERE type = 'Movie'
GROUP BY rating
ORDER BY count DESC
LIMIT 1;
```

---

## **4. Splitting into Two Tables (`shows` & `show_cast`)**
To optimize queries, we split the master table into two:  
- **`shows`** (contains show-related information)
- **`show_cast`** (contains actor & director information)

### **4.1 Create `shows` Table**
```sql
CREATE TABLE shows (
    show_id SERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    type TEXT CHECK (type IN ('Movie', 'TV Show')),
    release_year INT,
    primary_genre TEXT,
    rating TEXT,
    duration_num INT,
    country TEXT
);
```

### **4.2 Create `show_cast` Table**
```sql
CREATE TABLE show_cast (
    cast_id SERIAL PRIMARY KEY,
    show_id INT REFERENCES shows(show_id),
    actor_name TEXT,
    director TEXT
);
```

---

## **5. Business Questions & Queries (Joins & CTEs)**

### **5.1 Find top 5 actors appearing in the most shows**
```sql
SELECT unnest(string_to_array(c.actor_name, ', ')) AS actor, COUNT(*) AS num_shows
FROM show_cast c
JOIN shows s ON s.show_id = c.show_id
GROUP BY actor
ORDER BY num_shows DESC
LIMIT 5;
```

### **5.2 Find directors who have directed more than 3 movies**
```sql
SELECT c.director, COUNT(*) AS num_movies
FROM show_cast c
JOIN shows s ON s.show_id = c.show_id
WHERE s.type = 'Movie'
GROUP BY c.director
HAVING COUNT(*) > 3;
```

### **5.3 Use CTE to find average movie duration**
```sql
WITH avg_duration AS (
    SELECT AVG(duration_num) AS avg_dur FROM shows WHERE type = 'Movie'
)
SELECT title, duration_num 
FROM shows
WHERE duration_num > (SELECT avg_dur FROM avg_duration);
```

---

## **6. Window Function Queries**
### **6.1 Rank Movies by Duration**
```sql
SELECT title, duration_num,
       RANK() OVER (ORDER BY duration_num DESC) AS rank
FROM shows
WHERE type = 'Movie';
```

### **6.2 Find the cumulative count of shows per year**
```sql
SELECT release_year, COUNT(*) AS num_shows,
       SUM(COUNT(*)) OVER (ORDER BY release_year) AS cumulative_shows
FROM shows
GROUP BY release_year
ORDER BY release_year;
```

### **6.3 Find the top 3 longest movies for each genre**
```sql
SELECT title, primary_genre, duration_num,
       ROW_NUMBER() OVER (PARTITION BY primary_genre ORDER BY duration_num DESC) AS rank
FROM shows
WHERE type = 'Movie'
ORDER BY primary_genre, rank;
```

---

## **7. Final Summary & Insights**

#### **1. Most Common Movie Release Year**  
- The highest number of movies were released in **2018**.  
- Followed by **2017 and 2016** in terms of total titles.  

#### **2. Most Frequent Ratings**  
- The most common rating is **13+**, making it the dominant category.    

#### **3. Top Genres**  
- **Drama** is the most popular genre.  
- Followed by **Comedy, Action, Documentary, Horror, and Animation**.  

#### **4. Top Content-Producing Countries**  
- The highest number of titles are from:  
  - **United States**  
  - **India**  
  - **United Kingdom**    

#### **5. Average Movie Duration**  
- The average movie duration is **90-110 minutes**.  
- Some outliers exist, but most fall in this range.  

#### **6. Cast Member analysis**  
- Based on previous queries, some directors have significantly more titles than others.  
- Some actors appear in 5+ different shows

---

## **8. Conclusion**
- **AWS QuickSight** enabled interactive visualization of Amazon Prime Video data.
- **SQL queries** provided insights into release trends, top actors, and directors.
- **Splitting tables** optimized query performance.
- **CTEs & Window Functions** helped in advanced analytics.

---
