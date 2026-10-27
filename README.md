# Spotify & YouTube: What the Catalog Actually Shows
PostgreSQL · Data quality · Window functions · Statistical analysis · Query plans

[Click Here to get Dataset](https://www.kaggle.com/datasets/sanjanchaudhari/spotify-dataset)

![Spotify Logo](https://github.com/najirh/najirh-Spotify-Data-Analysis-using-SQL/blob/main/spotify_logo.jpg)

## Overview
A SQL portfolio case study built from 20,594 Spotify/YouTube observations, covering 2,074 artists. I investigated album energy consistency, platform count differences and video engagement—and found that understanding the sample was more important than producing a dramatic correlation.

Main result: album danceability has only a weak relationship with energy variation (r = −0.079). More consequentially, 94.1% of artists have exactly ten observed tracks, and 63.1% of album-type releases have only one observed track. This extract should not be treated as a collection of complete artist catalogs or album tracklists.

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
[SQL analysis](sql/) · [Verified outputs](results/) · [Methodology](docs/METHODOLOGY.md) · [Performance design](docs/PERFORMANCE.md) · [Tableau specification](docs/DASHBOARD_SPEC.md)
The Spotify project in very simple terms
Think of the CSV as a big pile of paper cards. Some cards repeat the same song. Some songs have the same name but belong to different artists. Before counting anything, the SQL sorts the cards into sensible groups. It keeps values that agree and marks a value unknown when the source rows disagree.

An album proxy is like a folder labelled with the artist, album name, and release type. We calculate the highest energy minus the lowest energy among the tracks we actually saw. If we only saw one track, we do not pretend we know how the whole album varies. The file often contains only a small sample of each artist, so the results describe the sample—not the artist's entire music library.

A SQL JOIN is like matching two paper lists by a shared name. A window function makes a small ranking list separately for each artist. EXPLAIN ANALYZE asks PostgreSQL to run a query and show how it found the answer. pgAdmin displays both the SQL and the answer table so you can inspect each step.
The analytical brief
Use the extract to assess three questions:
1. Is there evidence that more danceable albums are more consistent in energy?
2. How do observed Spotify stream counts compare with selected YouTube video views?
3. Do official-video labels distinguish reach from engagement intensity?
The output is a reproducible analytical model, 18 analytical questions, an audit trail, a verified results graphic and dashboard-ready extracts. Findings describe the supplied snapshot; they do not establish causes or represent the full platforms.
Findings that survived the checks
Finding	Verified result	Interpretation
Album danceability and energy variation	r = −0.079 across 3,180 eligible album proxies	Too weak to use as a standalone catalog-selection rule
Adjustment for observed track count	Partial r = −0.081 after linear adjustment for log observed count	This particular adjustment leaves a weak association; it does not establish causality
Track count and energy range	r = 0.379 in the same cohort	A range statistic depends on how many tracks are observed; coverage must accompany it
Album coverage	5,527 / 8,765 (63.1%) album-type proxies have one observed track	Zero range frequently reflects sparse observation, not a uniformly energetic album or a confirmed single
Artist coverage	1,951 / 2,074 (94.1%) artists have exactly ten track proxies; none have more than ten	Consistent with a capped sample; upstream sampling cannot be confirmed from the file alone
Official videos: reach vs engagement	At ≥10,000 views, median views are 24.12M official vs 2.30M non-official; median likes per 100 views are 0.836 vs 0.980	Reach and engagement intensity tell different stories; these unadjusted groups are not a causal comparison
Platform counts with a denominator floor	Median stream / largest observed video-view ratio = 3.05 across 19,025 pairs with positive streams and ≥10,000 video views	A count ratio within this matched sample, not platform audience share


Source: SQL output sets in results/04_business_analysis.json, results/05_album_investigation.json and results/08_sensitivity.json. Source file: cleaned_dataset.csv; SHA-256 recorded in [source_manifest.json](results/source_manifest.json).
What changed under sensitivity analysis?
- Requiring at least five observed tracks leaves 305 album proxies and r = −0.102. Requiring ten leaves only 15 and changes the sign to +0.340. The original direction should not be described as universal or robust across coverage thresholds.
- Non-official videos retain a higher median like/view rate at 1,000, 10,000 and 100,000 minimum views. Audience, genre, artist, age and selection remain uncontrolled; this does not show that official production lowers engagement.
- The median platform ratio falls from 3.16 → 3.05 → 2.79 as minimum video views increase from 1,000 → 10,000 → 100,000. Ranking the biggest ratios without a floor would emphasize tiny or poorly matched video denominators.
- 572 track proxies have recorded zero streams. Their upstream meaning is unknown. The danceability vs log(1+streams) correlation remains weak with zeros included (0.047) and excluded (0.056).
Practical implication: use this file for exploratory comparisons and quality investigation. Before recommending release strategy, artist investment or platform budget allocation, obtain stable IDs, collection dates and fuller catalog coverage.
Reproducing—and correcting—the original story
The original title-only method is reproducible: 4,391 multi-row album-title groups, with r = −0.1267. Its 783 zero-range groups were already multi-row groups. 778 of those 783 contain one track title under multiple artist names, consistent with repeated collaboration credits rather than multiple distinct tracks. Across all 11,854 title-only groups, there are 8,246 zero ranges; mixing those denominators led to the misleading single-track explanation.

[The baseline audit](sql/09_baseline_audit.sql) preserves the original calculation so the correction can be checked. The revised −0.079 result uses a different key, release-type filter and complete-audio cohort; the difference cannot be attributed to the key correction alone.
The SQL decision that changed the project
Grouping by album title alone can merge unrelated releases. In this file, “Greatest Hits” appears under 21 artist names. Counting source rows can also mistake repeated observations for additional tracks.

I separated the analytical grains before computing metrics:
Analytical table	One row represents	Rule
track_fact	Artist + track-title proxy	Collapse consistent metrics; conflicting values become NULL
album_track	Artist + album + release type + track membership	Count each observed track once per release proxy
video_proxy	Artist + track + video title + channel proxy	Preserve video metrics separately from Spotify counters


The resulting 20,512 track proxies include 80 stream conflicts and 81 energy/danceability conflicts. These can reflect different versions sharing names; they are not automatically bad duplicates. Exact cleaned duplicate rows removed: 0. Missing IDs prevent definitive entity resolution, so conflicts remain visible and affected metrics are excluded rather than averaged away.

This query computes both audio measures from the same observed tracks:
SELECT artist, album, observed_tracks,
       avg_danceability, energy_range, energy_sd
FROM spotify_project.album_metrics
WHERE album_type = 'album'
  AND observed_tracks >= 2
  AND complete_audio_tracks = observed_tracks
ORDER BY energy_range DESC, artist, album;
An album with one observed track is excluded from the primary correlation, but retained in the coverage audit. A zero energy range can also occur among multiple equal-energy tracks; zero is not automatically an error.
Repository guide
File	Purpose
[01_schema.sql](sql/01_schema.sql)	Raw preservation and safe numeric/Boolean parsing
[02_model.sql](sql/02_model.sql)	Grain-aware tables, conflict handling, keys and analytical views
[03_quality.sql](sql/03_quality.sql)	Exclusions, title collisions, invalid fields and coverage audit
[04_business_analysis.sql](sql/04_business_analysis.sql)	Artist rankings, concentration, platform ratios, engagement and audio cohorts
[05_album_investigation.sql](sql/05_album_investigation.sql)	Zero-range diagnosis, raw/partial correlations and coverage sensitivity
[06_assertions.sql](sql/06_assertions.sql)	Stop-on-failure analytical invariants
[07_exports.sql](sql/07_exports.sql)	CSV extracts for Tableau and verification
[08_sensitivity.sql](sql/08_sensitivity.sql)	Sampling structure, zero streams and denominator thresholds
[09_baseline_audit.sql](sql/09_baseline_audit.sql)	Reproduce the draft and diagnose repeated collaboration credits
[10_pgadmin_performance.sql](sql/10_pgadmin_performance.sql)	Compare an unindexed and indexed temporary table with actual PostgreSQL plans


SQL techniques include CTEs, foreign keys, window rankings, conditional aggregation, ordered-set percentiles, subqueries, regression aggregates, residualization and execution-plan inspection. They serve specific questions rather than a checklist of syntax.
Run this project in pgAdmin 4
This project uses PostgreSQL SQL and pgAdmin 4. You do not need Python to run the database queries.
1. Install PostgreSQL and pgAdmin 4, create a practice database in pgAdmin, and open Tools → Query Tool.
2. Open sql/01_schema.sql in Query Tool and click Execute/Refresh. It creates a dedicated spotify_project schema, a text-only import staging table, and safe parsing functions.
3. In the Browser tree, open Schemas → spotify_project → Tables. Right-click raw_import, select Import/Export Data, choose Import, then select the supplied cleaned_dataset.csv. Set format to CSV, turn Header on, delimiter to comma, and encoding to UTF-8. The staging columns match the 24 columns in the supplied CSV, including energyliveness and most_playedon.
4. In Query Tool, execute the following load statement. It maps the source's original column spellings into the project's raw table while PostgreSQL generates source row IDs.
5. Execute sql/02_model.sql, then sql/06_assertions.sql. Run sql/03_quality.sql, sql/04_business_analysis.sql, sql/05_album_investigation.sql, sql/08_sensitivity.sql, and sql/09_baseline_audit.sql one at a time. Read each result grid in pgAdmin.
TRUNCATE TABLE spotify_project.raw RESTART IDENTITY;
INSERT INTO spotify_project.raw
 (artist,track,album,album_type,danceability,energy,loudness,speechiness,
  acousticness,instrumentalness,liveness,valence,tempo,duration_min,title,
  channel,views,likes,comments,licensed,official_video,stream,energy_liveness,
  most_played_on)
SELECT artist,track,album,album_type,danceability,energy,loudness,speechiness,
       acousticness,instrumentalness,liveness,valence,tempo,duration_min,title,
       channel,views,likes,comments,licensed,official_video,stream,
       energyliveness,most_playedon
FROM spotify_project.raw_import;
The included result files were calculated against the provided CSV and validated in PostgreSQL-compatible PGlite. For a fresh run in your own PostgreSQL/pgAdmin environment, compare your result grids with results/. Different PostgreSQL versions or input file revisions can produce small differences. Do not run the native benchmark until the model has been built.

To export a query result, right-click within pgAdmin's Data Output grid and choose Save Results to File. Select CSV if you want to build the optional Tableau views described in docs/DASHBOARD_SPEC.md.
SQL used in the analysis
The complete runnable SQL is in sql/. The key excerpts below show the setup, data model, analysis and plan comparison.
1. Define a raw landing table
The import table preserves source values as text. This lets the audit identify invalid values instead of failing the import or silently coercing them.
CREATE TABLE IF NOT EXISTS spotify_project.raw (
    source_row_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    artist text, track text, album text, album_type text,
    danceability text, energy text, loudness text, speechiness text,
    acousticness text, instrumentalness text, liveness text, valence text,
    tempo text, duration_min text, title text, channel text, views text,
    likes text, comments text, licensed text, official_video text,
    stream text, energy_liveness text, most_played_on text
);
2. Build an artist/track proxy without guessing conflicting values
Repeated rows may have the same artist and title. The query collapses a measure only if its non-null values agree. If two source rows disagree about a track's energy or streams, that measure becomes NULL and is counted for review.
CREATE TABLE spotify_project.track_fact AS
SELECT artist, track, count(*) AS observation_count,
       count(DISTINCT stream) > 1 AS stream_conflict,
       count(DISTINCT energy) > 1 OR count(DISTINCT danceability) > 1 AS audio_conflict,
       CASE WHEN count(DISTINCT stream) <= 1 THEN max(stream) END AS stream,
       CASE WHEN count(DISTINCT energy) <= 1 THEN max(energy) END AS energy,
       CASE WHEN count(DISTINCT danceability) <= 1 THEN max(danceability) END AS danceability
FROM spotify_project.observations
GROUP BY artist, track;
The full model also creates album membership and video proxy tables, adds keys and foreign-key checks, and builds views. Names are proxies because this CSV has no stable Spotify track, album, or YouTube video IDs.
3. Rank tracks within each artist
WITH ranked AS (
    SELECT artist, track, stream,
           dense_rank() OVER (PARTITION BY artist ORDER BY stream DESC) AS stream_rank
    FROM spotify_project.track_fact
    WHERE stream IS NOT NULL
)
SELECT artist, track, stream, stream_rank
FROM ranked
WHERE stream_rank <= 3
ORDER BY artist, stream_rank, track;
PARTITION BY artist starts a fresh ranking for each artist. DENSE_RANK keeps tied tracks at the same rank.
4. Calculate album-level variation on a consistent cohort
SELECT artist, album, album_type,
       count(*) AS observed_tracks,
       count(*) FILTER (
         WHERE energy IS NOT NULL AND danceability IS NOT NULL
       ) AS complete_audio_tracks,
       avg(danceability) FILTER (
         WHERE energy IS NOT NULL AND danceability IS NOT NULL
       ) AS avg_danceability,
       max(energy) FILTER (
         WHERE energy IS NOT NULL AND danceability IS NOT NULL
       ) - min(energy) FILTER (
         WHERE energy IS NOT NULL AND danceability IS NOT NULL
       ) AS energy_range
FROM spotify_project.album_track AS a
JOIN spotify_project.track_fact AS t USING (artist, track)
GROUP BY artist, album, album_type
HAVING count(*) >= 2
ORDER BY energy_range DESC;
The final analysis uses album_metrics and requires album type = album, at least two observed tracks and complete audio values. The excerpt shows how max-minus-min is calculated. An observed track count is not a verified full album tracklist.
5. Inspect an actual query plan in PostgreSQL
Pick an artist name from track_fact, then replace the example text and run this in pgAdmin's Query Tool:
EXPLAIN (ANALYZE, BUFFERS)
SELECT artist, track, stream
FROM spotify_project.track_fact
WHERE artist = 'Drake'
ORDER BY stream DESC NULLS LAST
LIMIT 10;
The table's primary key already begins with artist; adding another artist index may not help. Use the plan, result equivalence, table size and the real workload before adding an index. No speedup is claimed without a native PostgreSQL benchmark.
Query optimization without invented wins
The track table's (artist, track) primary key already supports artist lookups. An extra artist-only index may be redundant. Run sql/10_pgadmin_performance.sql in one pgAdmin Query Tool session. It makes a temporary unindexed copy, captures an EXPLAIN (ANALYZE, BUFFERS) plan, adds an artist index, runs the same query again, reports index size and checks that the returned rows match. Record your own PostgreSQL version and hardware with the plans. The included analytical results were not a native-server performance benchmark, so no speedup is claimed.
Run the index comparison in pgAdmin
After creating the model, run sql/10_pgadmin_performance.sql in the same Query Tool session. This is the code that tests the query plan:
EXPLAIN (ANALYZE, BUFFERS)
SELECT artist, track, stream FROM bench_tracks
WHERE artist = 'Drake'
ORDER BY stream DESC NULLS LAST LIMIT 10;

CREATE INDEX bench_artist_idx ON bench_tracks(artist);
ANALYZE bench_tracks;

EXPLAIN (ANALYZE, BUFFERS)
SELECT artist, track, stream FROM bench_tracks
WHERE artist = 'Drake'
ORDER BY stream DESC NULLS LAST LIMIT 10;
The full file also compares the result sets and reports the index size. The project does not claim an unmeasured speedup.
Boundaries and next step
Artist, release and video names are proxy identifiers. Collaborations and versions can cross proxy boundaries. Audio-field completeness does not mean complete release coverage. Neither platform counters nor video labels are experiments. The file contains no reliable time dimension, so growth, retention and release-age-adjusted analyses are outside scope.

The [Tableau build specification](docs/DASHBOARD_SPEC.md) defines three pages and their measures. CSV extracts and the results figure are included; a live Tableau dashboard is not included.
Data, attribution and licensing
- Dataset reference: Spotify dataset on Kaggle. The analyzed bytes are the user-supplied CSV recorded in the source manifest; equivalence to the current Kaggle download has not been verified.
- Starting-project inspiration: Najir H.'s Spotify SQL project. This extension adds explicit grain definitions, conflict-aware modeling, reproducible validation, sampling audits and sensitivity analyses.
- Project code: MIT License. Dataset rights remain with the original provider; this code license does not relicense the data. The raw dataset is excluded from the package.
