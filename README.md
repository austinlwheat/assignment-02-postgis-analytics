# Assignment 02: PostGIS Analytics

**Due: Oct 18, 2021 by 11:59pm ET**

## Submission Instructions

1. Fork this repository to your GitHub account.

2. Write a query to answer each of the questions below. Your queries should produce results in the format specified. Write your query in a SQL file corresponding to the question number (e.g. a file named _query06.sql_ for the answer to question #6). Each SQL file should contain a single `SELECT` query (though it may include other queries before the select if you need to do things like create indexes or update columns). Some questions include a request for you to discuss your methods. Update this README file with your answers in the appropriate place.

3. There are several datasets that are prescribed for you to use in this assignment. Your datasets should be named:
  * septa_bus_stops ([SEPTA GTFS](http://www3.septa.org/developer/))
  * septa_bus_shapes ([SEPTA GTFS](http://www3.septa.org/developer/))
  * septa_rail_stops ([SEPTA GTFS](http://www3.septa.org/developer/))
  * phl_pwd_parcels ([OpenDataPhilly](https://opendataphilly.org/dataset/pwd-stormwater-billing-parcels))
  * census_block_groups ([OpenDataPhilly](https://opendataphilly.org/dataset/census-block-groups))
  * census_population ([Census Explorer](https://data.census.gov/cedsci/table?t=Populations%20and%20People&g=0500000US42101%241500000&y=2010&d=DEC%20Summary%20File%201&tid=DECENNIALSF12010.P1))

4. Submit a pull request with your answers. You can continue to push changes to your repository up until the due date, and those changes will be visible in your pull request.

**Note, I take logic for solving problems into account when grading. When in doubt, write your thinking for solving the problem even if you aren't able to get a full response.**

## Questions

1. Which bus stop has the largest population within 800 meters? As a rough estimation, consider any block group that intersects the buffer as being part of the 800 meter buffer.
 ```sql
  (
    create index septa_bus_stops__the_geom__32129__idx
    on septa_bus_stops
    using GiST (ST_Transform(the_geom, 32129));

UPDATE septa_bus_stops
    set the_geom = st_setsrid(st_makepoint(stop_lon, stop_lat), 4326)

with septa_bus_stop_block_groups as (
    select
        s.stop_id,
        '1500000US' || bg.geoid10 as geo_id
    from septa_bus_stops as s
    join census_block_groups as bg
        on ST_DWithin(
            ST_Transform(s.the_geom, 32129),
            ST_Transform(bg.the_geom, 32129),
            800
        )
),

septa_bus_stop_surrounding_population as (
    select
        stop_id,
        sum(total) as estimated_pop_800m
    from septa_bus_stop_block_groups as s
    join census_population as p on s.geo_id = p.id
    group by stop_id
)

select
    stop_name,
    estimated_pop_800m,
    the_geom
from septa_bus_stop_surrounding_population
join septa_bus_stops using (stop_id)
order by estimated_pop_800m desc
limit 1  
  )
  ```
2. Which bus stop has the smallest population within 800 meters?

  **The queries to #1 & #2 should generate relations with a single row, with the following structure:**

  ```sql
  (
     create index septa_bus_stops__the_geom__32129__idx
    on septa_bus_stops
    using GiST (ST_Transform(the_geom, 32129));

UPDATE septa_bus_stops
    set the_geom = st_setsrid(st_makepoint(stop_lon, stop_lat), 4326)

with septa_bus_stop_block_groups as (
    select
        s.stop_id,
        '1500000US' || bg.geoid10 as geo_id
    from septa_bus_stops as s
    join census_block_groups as bg
        on ST_DWithin(
            ST_Transform(s.the_geom, 32129),
            ST_Transform(bg.the_geom, 32129),
            800
        )
),

septa_bus_stop_surrounding_population as (
    select
        stop_id,
        sum(total) as estimated_pop_800m
    from septa_bus_stop_block_groups as s
    join census_population as p on s.geo_id = p.id
    group by stop_id
)

select
    stop_name,
    estimated_pop_800m,
    the_geom
from septa_bus_stop_surrounding_population
join septa_bus_stops using (stop_id)
order by estimated_pop_800m asc
limit 1
  )
  ```

3. Using the Philadelphia Water Department Stormwater Billing Parcels dataset, pair each parcel with its closest bus stop. The final result should give the parcel address, bus stop name, and distance apart in meters. Order by distance (largest on top).

  **Structure:**
  ```sql
  (
      -- create a geometry column on buses
alter table septa_bus_stops
    add column if not exists geometry geometry(Point, 4326);

update septa_bus_stops
  set geometry = ST_SetSRID(ST_MakePoint(stop_lon, stop_lat), 4326);
  
-- indexes
create index if not exists septa_bus_stops__the_geom__32129__idx
    on septa_bus_stops
    using GiST (ST_Transform(geometry, 32129));

create index if not exists phl_pwd_parcels_geometry_32129_idx
    on phl_pwd_parcels
    using GiST (ST_Transform(geometry, 32129));

-- query & cross join
SELECT 
    p.address,
    cl.stop_name,
    cl.distance_m
from phl_pwd_parcels as p
cross join lateral(
    select 
        s.stop_name,
        ST_Distance(
            ST_Transform(p.geometry, 32129),
            ST_Transform(s.geometry, 32129)
        )as distance_m
    from septa_bus_stops as s
    order by distance_m desc
    limit 1
) as cl;
  )
  ```

4. Using the _shapes.txt_ file from GTFS bus feed, find the **two** routes with the longest trips. In the final query, give the `trip_headsign` that corresponds to the `shape_id` of this route and the length of the trip.

  **Structure:**
  ```sql
  (
      --Min in Sequence
WITH min_t as (
select shape_id, min(CAST(shape_pt_sequence as INT)) min_station
from septa_bus_shapes
GROUP BY shape_id
),

--Max in Sequence
max_t as (
select shape_id, max(CAST(shape_pt_sequence as INT)) max_station
from septa_bus_shapes
GROUP BY shape_id
),

-- Lat/Long
septa_origin as (
SELECT shape_id, shape_pt_lon, shape_pt_lat, CAST(shape_pt_sequence as INT) sequence_int
FROM septa_bus_shapes
),

--Makeline, First in Sequence
first as(
SELECT ST_Makeline(ST_setsrid(ST_MakePoint(o.shape_pt_lon, o.shape_pt_lat), 4326) ORDER BY sequence_int)::geometry geom, o.shape_id
FROM septa_origin o
GROUP BY o.shape_id),

--Makeline, Last in Sequence
last as(
SELECT ST_Makeline(ST_setsrid(ST_MakePoint(o.shape_pt_lon, o.shape_pt_lat),4326) ORDER BY sequence_int)::geometry geom, o.shape_id
FROM septa_origin o
GROUP BY o.shape_id)

--Return 2 Longest Trips
SELECT *, (ST_Length(geom)) FROM first
ORDER BY st_length desc
LIMIT 2
  )
  ```

5. Rate neighborhoods by their bus stop accessibility for wheelchairs. Use Azavea's neighborhood dataset from OpenDataPhilly along with an appropriate dataset from the Septa GTFS bus feed. Use the [GTFS documentation](https://gtfs.org/reference/static/) for help. Use some creativity in the metric you devise in rating neighborhoods. Describe your accessibility metric:

  I don't believe this was a successful attempt. My goal here was to count the # of stops where wheelchair_access = 1 (meaning that stop is wheelchair accessible) in each neighborhood. The neighborhoods were ranking based on the number of wheelchair accessible stops therewithin.

```sql
  (
     with stopz as(SELECT stop_id, wheelchair_boarding, ST_SetSRID(ST_MAKEPOINT(stop_lon, stop_lat), 4326) geom
FROM septa_bus_stops),

wc_access as(SELECT s.stop_id , s.wheelchair_boarding, s.geom, n.mapname, n.geom
FROM stopz AS s
JOIN neighborhoods_philadelphia as n
ON ST_Contains(
	n.geom,
	s.geom))
	
SELECT mapname, count(stop_id)
FROM wc_access
WHERE 'wheelchair_boarding' = '1'
GROUP BY mapname
  )
  ```
6. What are the _top five_ neighborhoods according to your accessibility metric?
 ```sql
  (
      with stopz as(SELECT stop_id, wheelchair_boarding, ST_SetSRID(ST_MAKEPOINT(stop_lon, stop_lat), 4326) geom
FROM septa_bus_stops),

wc_access as(SELECT s.stop_id , s.wheelchair_boarding, s.geom, n.mapname, n.geom
FROM stopz AS s
JOIN neighborhoods_philadelphia as n
ON ST_Contains(
	n.geom,
	s.geom))
	
SELECT mapname, count(stop_id)
FROM wc_access
WHERE 'wheelchair_boarding' = '1'
GROUP BY mapname
ORDER BY 'wheelchair_boarding' ASC
Limit 5
  )
  ```
7. What are the _bottom five_ neighborhoods according to your accessibility metric?
 ```sql
  (
      with stopz as(SELECT stop_id, wheelchair_boarding, ST_SetSRID(ST_MAKEPOINT(stop_lon, stop_lat), 4326) geom
FROM septa_bus_stops),

wc_access as(SELECT s.stop_id , s.wheelchair_boarding, s.geom, n.mapname, n.geom
FROM stopz AS s
JOIN neighborhoods_philadelphia as n
ON ST_Contains(
	n.geom,
	s.geom))
	
SELECT mapname, count(stop_id)
FROM wc_access
WHERE 'wheelchair_boarding' = '1'
GROUP BY mapname
ORDER BY 'wheelchair_boarding' DESC
Limit 5
  )
  ```

8. With a query, find out how many census block groups Penn's main campus fully contains. Discuss which dataset you chose for defining Penn's campus.

  **Structure (should be a single value):**
  ```sql
  (
    with count_block_groups as(
SELECT bg.geoid10, bg.geom, penn.geom
FROM upenn_campus as penn
JOIN census_block_groups AS bg
ON ST_Contains(
	bg.geom,
	penn.geom))

SELECT COUNT(geoid10) FROM count_block_groups;  
  )
  ```

9. With a query involving PWD parcels and census block groups, find the `geo_id` of the block group that contains Meyerson Hall. ST_MakePoint() and functions like that are not allowed.

  **Structure (should be a single value):**
  ```sql
  (
     SELECT c.geoid10 as geo_id
from phl_pwd_parcels as p
join census_block_groups as c
on ST_Contains(c.geom, p.geometry)
where address like '230 S 34TH ST'

  )
  ```

10. You're tasked with giving more contextual information to rail stops to fill the `stop_desc` field in a GTFS feed. Using any of the data sets above, PostGIS functions (e.g., `ST_Distance`, `ST_Azimuth`, etc.), and PostgreSQL string functions, build a description (alias as `stop_desc`) for each stop. Feel free to supplement with other datasets (must provide link to data used so it's reproducible), and other methods of describing the relationships. PostgreSQL's `CASE` statements may be helpful for some operations.

  **NOTE: For this one I downloaded the Septa Regional Rail Line data from: https://septaopendata-septa.opendata.arcgis.com/datasets/septa-regional-rail-lines/explore**
  ```sql
  (
      --Add geom column to the stops table
alter table septa_rail_stops
add column geom_32129 geometry(Point, 32129);
update septa_rail_stops
set geom_32129 = ST_Transform(ST_SetSRID(ST_MakePoint(stop_lon, stop_lat), 4326), 32129);

--Add route_name column to the stops table
alter table septa_rail_stops
add column route_name varchar (50);

--JOIN
with stops_lines as (SELECT s.stop_name, s.geom_32129,s.route_name, l.geom
FROM septa_rail_stops as s
JOIN rail_lines as l
ON ST_Intersects (s.geom_32129, l.geom))

--fill in route_name column in stops table
update septa_rail_stops
set route_name = stop_lines.route_name
FROM stop_lines;
  )
  ```
