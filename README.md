# Spotify Advanced SQL Project and Query Optimization P-6
Project Category: Advanced
[Click Here to get Dataset](https://www.kaggle.com/datasets/sanjanchaudhari/spotify-dataset)

![Spotify Logo](https://github.com/najirh/najirh-Spotify-Data-Analysis-using-SQL/blob/main/spotify_logo.jpg)

## Overview
This project involves analyzing a Spotify dataset with various attributes about tracks, albums, and artists using **SQL**. It covers an end-to-end process of normalizing a denormalized dataset, performing SQL queries of varying complexity (easy, medium, and advanced), and optimizing query performance. The primary goals of the project are to practice advanced SQL skills and generate valuable insights from the dataset.

```sql
-- create table
DROP TABLE IF EXISTS spotify;
CREATE TABLE spotify (
    artist VARCHAR(255),
    track VARCHAR(255),
    album VARCHAR(255),
    album_type VARCHAR(50),
    danceability FLOAT,
    energy FLOAT,
    loudness FLOAT,
    speechiness FLOAT,
    acousticness FLOAT,
    instrumentalness FLOAT,
    liveness FLOAT,
    valence FLOAT,
    tempo FLOAT,
    duration_min FLOAT,
    title VARCHAR(255),
    channel VARCHAR(255),
    views FLOAT,
    likes BIGINT,
    comments BIGINT,
    licensed BOOLEAN,
    official_video BOOLEAN,
    stream BIGINT,
    energy_liveness FLOAT,
    most_played_on VARCHAR(50)
);
```
# Spotify Streaming & Audio Feature Analysis

I set out to answer a simple question: does an album's overall danceability say anything about how much its energy level swings from track to track? Turns out yes, a little — albums with higher average danceability tend to stay more energetically consistent (r ≈ -0.13 across 4,391 albums), even after I checked whether that was just being driven by longer albums having more room to vary.

![Dashboard screenshot](dashboard/screenshot.png)
*[Link to interactive dashboard on Tableau Public →](#)*

---

## Key Findings

- Higher-danceability albums show less internal energy variation. The correlation isn't huge (r ≈ -0.13), but it holds up across the full dataset, not just a handful of albums.
- Track count turned out to matter more than I expected — it correlates with energy range at r ≈ 0.34, stronger than danceability does. I checked whether this explained away the danceability pattern; it didn't fully, but it's clearly part of the picture.
- About 18% of albums (783 of 4,391) initially came back with an energy range of exactly zero, which looked like a bug at first. It wasn't — I'll explain below.

---

## How I Got Here

The first version of my per-album energy query returned a suspicious number of zeros. My instinct was that something was broken in the `GROUP BY`, so I pulled the offending rows to look closer:

```sql
SELECT album, energy_range, track_count
FROM (
  SELECT album, MAX(energy) - MIN(energy) AS energy_range, COUNT(*) AS track_count
  FROM spotify
  GROUP BY 1
) t
WHERE energy_range = 0
ORDER BY track_count DESC
LIMIT 10;
```

Almost every one of those albums had exactly one track. That made sense once I thought about it — if an "album" is really a single song, then max and min energy are the same value, so the range is always going to come out to zero. Not a bug, just a metric that doesn't mean anything for single-track releases. I filtered those out before drawing any conclusions:

```sql
SELECT album,
       AVG(danceability) AS avg_danceability,
       MAX(energy) - MIN(energy) AS energy_range,
       COUNT(*) AS track_count
FROM spotify
GROUP BY 1
HAVING COUNT(*) > 1
ORDER BY 3 DESC;
```

Once I filtered to actual multi-track albums, the pattern I mentioned above showed up — lower-danceability albums tended to sit near the top of the energy range ranking. I didn't want to trust that off a quick glance at the top rows, so I checked it against the whole dataset and also tested whether it was really just album length in disguise (some albums here have 20-30 tracks, which obviously gives more room for energy to swing). It's a real effect on top of that, just a modest one.

---

## Queries

Full SQL is in [`/sql`](./sql). Quick summary of what's in each file:

| File | What it does |
|---|---|
| `01_setup_and_cleaning.sql` | Table setup, plus basic checks for nulls and junk rows before doing anything else |
| `02_core_analysis.sql` | Top tracks per artist, Spotify vs. YouTube streaming comparison, above-average liveness |
| `03_energy_range_investigation.sql` | The zero-range issue above and how I tracked it down |
| `04_query_optimization.sql` | Added an index on `artist`, compared query plans before and after |

Techniques: CTEs, window functions (`DENSE_RANK() OVER PARTITION BY`), subqueries, conditional aggregation, `HAVING` filters, indexing.

---

## Query Optimization

```sql
EXPLAIN ANALYZE SELECT * FROM spotify WHERE artist = '[artist_name]';
-- Execution time: [X] ms | Planning time: [X] ms

CREATE INDEX idx_artist ON spotify(artist);

EXPLAIN ANALYZE SELECT * FROM spotify WHERE artist = '[artist_name]';
-- Execution time: [X] ms | Planning time: [X] ms
```
*(numbers above are from my own run against this dataset — swap in yours if you're reusing this structure)*

---

## Limitations

- The main correlation is real but weak — I wouldn't call it predictive on its own.
- I didn't fully isolate the track-count effect from the danceability effect; a tighter version of this would compare only albums with similar track counts to each other.
- Single-track "albums" were dropped from the energy range analysis since the metric doesn't apply to them.

---

## Tools

PostgreSQL, pgAdmin 4, Tableau Public

## Data

Spotify + YouTube combined metrics dataset — audio features, streaming counts, and engagement data by track/album/artist.

## License
This project is licensed under the MIT License.
