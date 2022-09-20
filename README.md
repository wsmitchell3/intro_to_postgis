# Introduction to PostGIS

by Bill Mitchell and Mike Dolbow

## Overview

This course is intended as an introduction to PostGIS for people who already have basic familiarity with SQL.
The first section is an introduction to why PostGIS may be right for your situation.
The second section will introduce basic spatial datatypes and operations, then build some solutions using these operations.
The final section will cover some more advanced considerations and capabilities to help PostGIS help you.


## Why PostGIS?

PostGIS solves a few different problems common to working with spatial data.
* One central source for data
* Multi-user access
* Role-based access controls (RBAC)
* Can keep data normalized (in the database table sense), making data easier to maintain

PostGIS also has several important features.
* Free, open source
* Under active development
* Supports vector, raster, and 3- or 4-dimensional data
* Lets you do most geospatial functions within the database
* Interoperates with many different spatial tools
* Can create GeoJSON and MVT formats (PostGIS 3+)
* Extensions
	* Foreign data wrapper extension `ogr_fdw` can make non-Postgres data (shapefile, CSV, and many more) query like another Postgres table
	* TIGER geocoder for geocoding
	* Routing extension `pgrouting` can find paths through a network (often roads)


## Spatial Datatypes and Queries

### Creating and Manipulating Spatial Datatypes

Points are the fundamental spatial datatype.  We can make them by specifying X and Y coordinates, and optionally Z and/or M coordinates.  We also specify a coordinate system (via spatial reference id, SRID) to avoid ambiguity.  Fun fact: the ST\_ prefix comes from an ISO standard for spatial/temporal SQL functionality.

```SQL
-- Make a point, specifying geographic coordinate system of WGS84 (srid:4326)
SELECT ST_SetSRID(ST_MakePoint(-93.210703, 44.882990), 4326);

-- Make a point, specifying projected coordinate system UTM15 (srid:26915)
SELECT ST_SetSRID(ST_MakePoint(482022, 4969010), 26915);


-- Make a point with Z and M values, specifying WGS84 but then transforming
-- to UTM15
SELECT ST_Transform(
    ST_SetSRID(
		    ST_MakePoint(-93.2, 44.9, 1234, 5678)
			  , 4326)
		, 26915);

-- Make a point with Z and M values, specifying WGS84 but then transforming
-- to UTM15, and verify the resulting values using subquery syntax
SELECT ST_X(pt), ST_Y(pt), ST_Z(pt), ST_M(pt), pt AS geom
FROM (SELECT ST_Transform(
    ST_SetSRID(
        ST_MakePoint(-93.2, 44.9, 1234, 5678)
        , 4326)
    , 26915) AS pt) AS my_point;
```

Another option we have for displaying spatial data is Extended Well-Known Text
(EWKT) format.
This format is similar to the Well-Known Text format, but includes the SRID.
It makes the data somewhat easier to read for a human.

``` SQL
SELECT ST_AsEWKT(ST_SetSRID(
  ST_MakePoint(-93.2, 44.9, 1234, 5678)
	, 4326)
);
```

Once we have points, we can build lines.

``` SQL
-- Make a line from two points (or lines, or combination of point/line)
SELECT ST_SetSRID(
	ST_MakeLine(
    ST_MakePoint(-93.5, 44.8), ST_MakePoint(-94, 45)
	)
	, 4326
);

-- ST_MakeLine works with two points, two lines, or a combination in any order.
-- It can also take well-known text input for either/both arguments
-- Here we will use two lines from well-known text
SELECT ST_SetSRID(
  ST_MakeLine(
	'LINESTRING(-95 45, -96 45, -96 46)', 'LINESTRING(-95 46, -93 48)'
	)
	, 4326
);

-- Make a line from the first polling place to the last (by polling place id).
SELECT ST_MakeLine(
    ARRAY(
		    SELECT geom FROM mock_polling_place
				ORDER BY id
				)
		);
```

Polygons can be made similarly to points and lines.
The `ST_MakePolygon` function takes either a closed `LINESTRING` or a closed
`LINESTRING` and one or more closed interior `LINESTRING`s (holes in the polygon).

```SQL
-- Create a triangle; this is the minimum number of points for a valid polygon.
SELECT ST_SetSRID(
	ST_MakePolygon(
		ST_GeomFromText('LINESTRING(-93 44, -93 45, -95 45, -93 44)'
			)
	)
	, 4326
);
```

The `ST_Length` and `ST_Area` functions take a line and a polygon, respectively, and return the length/area in map units.

Multipart geometries can sometimes cause issues, as some functions or tools
require simple geometries.
PostGIS requires consistent geometry types in its tables, so if your data
should allow multi-part geometries, you can cast everything to multi-part.
This is particularly useful when loading data into tables.
Rather than loading the geometry directly, load `ST_Multi(geometry)`.

```SQL
-- Convert a point to multipoint
SELECT ST_AsText(
  ST_Multi('POINT(-95 47)')
);
-- Yields 'MULTIPOINT(-95 47)'
```

Another frequent task is splitting multiparts into their components, as well
as the inverse task of building multiparts from several simple geometries.

```SQL
-- Find up to 15 multipart parcels, with their corresponding state_pin
SELECT state_pin
	,ST_NumGeometries(shape) 
FROM parcels
WHERE ST_NumGeometries(shape) > 1
LIMIT 15;

-- Convert multipolygon to polygon
-- We also use the `ROW_NUMBER() OVER ()` function to add row numbers
-- to our results.  This is useful for generating unique IDs in a query.
SELECT ROW_NUMBER() OVER () AS row_id
  , state_pin
	, (ST_Dump(shape)).path AS path
	, (ST_Dump(shape)).geom AS geom
FROM parcels
WHERE ST_NumGeometries(shape) > 1
LIMIT 15;

-- Rebuild multipoints from singles with shared attributes
SELECT facility_id, ST_Collect(geom) as geom
FROM mock_polling_place AS mpp
GROUP BY facility_id;
```



### Recap
So far we've seen how to make points, lines, and polygons, and to get a little information about the points within them.
Here are the functions we've come across:
* `ST_MakePoint`
* `ST_MakeLine`
* `ST_MakePolygon`
* `ST_SetSRID`
* `ST_Transform`
* `ST_AsEWKT`
* `ST_GeomFromText`
* `ST_X`, `ST_Y`, `ST_Z`, and `ST_M`
* `ST_Dump`
* `ST_Collect`
* `ST_Length`
* `ST_Area`

Other functions which are related but beyond scope:
* `ST_AddPoint`
* `ST_Boundary` (line to endpoints, or polygon to outer ring and holes)
* `ST_DumpPoints` (geometry to constituent points)

## Spatial Queries

Querying data based on spatial relationships is one of the most fundamental features of a spatial database.
Like with a regular database where you might want to relate things based on IDs or some shared property, we can do the same with spatial data.

Let us suppose we are working alongside election administrators to prepare for an upcoming election.  We have a number of precincts (non-overlapping polygons), parcels (and/or buildings, also polygons but potentially overlapping), and points identified which will be our fictional polling places.  Additionally, we have some facility contact and contract information stored as a separate, non-spatial table.

We can view non-spatial data in a spatial way by joining it with an appropriate spatial feature.  For instance, we can make our non-spatial facility dataset spatial by joining to the polling place points on the facility id.  This is like taking tabular data which exists at a county level and joining to the county spatial table on county name or county code.

Using a spatial database, we can maintain a single source of truth for county shapes or facility points, and combine them with other non-spatial data as needed.  We store the shapes only once, not once per table or "feature class".  The technical term for this kind of database architecture is _normalized_.

```SQL
-- Basic SQL join non-spatial Facility table to spatial Mock Polling Place
SELECT f.id, f.facility_name, f.facility_contact_name, f.contract_status,
  mpp.precinct_name, mpp.geom
FROM facilities AS f
INNER JOIN mock_polling_place AS mpp ON f.id = mpp.facility_id
;
```

Being able to attach tabular data to spatial data is useful, but what if we want to use or understand how two spatial entities relate to each other?  For this, we use operations such as _contains_, _touches_, _nearest_, and so on.

For point-in-polygon, we can use the `ST_Contains` function to query if _A_ contains _B_.

```SQL
-- Get parcels that contain a polling place
SELECT mpp.id, p.state_pin, p.anumber, p.st_pre_dir, p.st_name, p.st_pos_typ, p.st_pos_dir, p.postcomm, p.state_code, p.zip, p.zip4, p.shape
FROM parcels AS p
INNER JOIN mock_polling_place AS mpp ON ST_Contains(p.shape, mpp.geom)
;

-- Huh, no rows returned.  What are the coordinate systems?
-- Let's try again with both in UTM-15 N (26915)
SELECT mpp.id, p.state_pin, p.anumber, p.st_pre_dir, p.st_name, p.st_pos_typ, p.st_pos_dir, p.postcomm, p.state_code, p.zip, p.zip4, p.shape
FROM parcels AS p
INNER JOIN mock_polling_place AS mpp ON ST_Contains(p.shape, ST_Transform(mpp.geom, 26915))
;


-- Similarly, we can find the precincts where the polling place is not in the precinct
SELECT precinct, wkb_geometry
FROM precinct AS p
WHERE precinct LIKE 'Minneapolis%'
  AND precinctid NOT IN (  -- Get list of IDs to reject
	  SELECT precinctid
	  FROM precinct AS p2
		INNER JOIN mock_polling_place AS mpp
		  ON ST_Contains(p2.wkb_geometry, mpp.geom))  -- Coordinates both 4326
;

-- Advanced exercise: improve this query through better non-spatial indexing
```

To find the distance from each polling place the middle of its precinct, we can use the `ST_Centroid` and `ST_Distance` functions.  Sort descending to find the ones furthest from the center of the precinct.  Note that we need to project to get distances in meters.

```SQL
SELECT p.precinct, ST_Distance(ST_Centroid(
  ST_Transform(p.wkb_geometry, 26915)), ST_Transform(mpp.geom, 26915)) AS dist_m
FROM precinct AS p
INNER JOIN mock_polling_place AS mpp ON p.precinct = mpp.precinct_name
ORDER BY dist_m DESC;
```

Tired of reprojecting data for each query?  Noticing that it would take longer on bigger datasets?  It's time to do something about that.  There are several approaches:

1. Change the projection in the source table.  Any additions or updates to the data will need to be made in the new projection
2. Create a view to reproject.  The database reprojects each time, but additions and updates are handled seamlessly without your intervention and the data are always fresh.  Depending on how much RAM the server has for caching and how much data you have, this may or may not be sufficient.
3. Create a materialized view.  The database reprojects once and stores the result, which may become stale.  After additions, updates, or deletions, the materialized view will need to be refreshed to bring in the new data.  For seldom-changing datasets this can be done manually, or you can create database triggers (beyond scope) to automatically refresh.  [Good discussion of options on Stack Overflow](https://stackoverflow.com/questions/29437650/how-can-i-ensure-that-a-materialized-view-is-always-up-to-date).

We will create a materialized view.  The `WITH DATA` keyword tells the database not only to define how the view is created, but also to refresh its data now (this could be deferred).  

```SQL
-- I follow a convention that materialized views are prefixed with 'mvw_'
CREATE MATERIALIZED VIEW IF NOT EXISTS mvw_precinct AS
(SELECT ogc_fid, precinct, precinctid, county, countyid, congdist, mnsendist,
mnlegdist, ctycomdist, ST_Transform(wkb_geometry, 26915)
FROM precinct
WITH DATA
);

-- If the source data change, you can refresh the materialized view
REFRESH MATERIALIZED VIEW mvw_precinct;


-- Same story for the mock polling places
CREATE MATERIALIZED VIEW IF NOT EXISTS mvw_mock_polling_place AS
(SELECT precinct_name, facility_id, id, ST_Transform(geom, 26915)
FROM mock_polling_place
WITH DATA
);

-- If refresh is needed
REFRESH MATERIALIZED VIEW mvw_mock_polling_place;
```


Let's continue by finding the number of parcels which intersect each of the precincts, and order the results by the ward and precinct.  For this we'll use `ST_Intersects` so the parcel doesn't need to be fully within the precinct.

```SQL
SELECT pr.precinct, COUNT(pa.fid)
FROM precinct AS pr
INNER JOIN parcels AS pa ON ST_INTERSECTS(ST_Transform(pr.wkb_geometry, 26915), pa.shape)
WHERE pr.precinct LIKE 'Minneapolis%'
GROUP BY pr.precinct, pr.precinctid
ORDER BY pr.precinctid
;
```


SQL functions provide a way to define data processing which relies on inputs.
For instance, there is a 100' buffer zone around a polling place where certain laws and restrictions apply.
We want a streamlined way to provide a point (e.g. polling place point), then buffer the parcel (or building) that contains that point by 100'.
There are 3.28084 feet per meter (feet per map unit).

Here I've demonstrated some features of Postgres function definitions that are advanced but explained in the Postgres documentation (`STABLE`, `RETURNS NULL ON NULL INPUT`, `PARALLEL SAFE`).  These are here as points for further inquiry into database performance optimization.

```SQL
CREATE OR REPLACE FUNCTION bufferzone (pt GEOMETRY('Multipoint'))
RETURNS Geometry('Multipolygon')
AS 'SELECT ST_Buffer(pa.shape, 100/3.28084) FROM parcels AS pa WHERE ST_Contains(pa.shape, ST_Transform(pt, 26915))'
LANGUAGE SQL
STABLE
RETURNS NULL ON NULL INPUT
PARALLEL SAFE
;

-- Let's give that a try
SELECT 1 AS id, bufferzone(ST_SetSRID(ST_MakePoint(-93.290584, 44.935623), 4326));

-- We can also use this function to buffer all the polling places
SELECT precinct_name, id, bufferzone(geom) FROM mock_polling_place;

-- Wow, that seems like a thing we may want to use a bunch.  View time!
CREATE OR REPLACE VIEW vw_polling_place_buffers AS
  SELECT precinct_name, id, bufferzone(geom) FROM mock_polling_place;
```

Take a minute to reflect on what we have just set up.
We created a function that will convert our point to a buffer of the parcel underneath it.
Then, we made a view that runs that function for each polling place in our polling place table.
If during the planning process we need to move a polling place, we update the one polling place table and our buffers adjust automatically.


Next, let's find the closest polling place for each precinct centroid.  There is an operator, `<->`, which provides the 2D distance between two geometries.  
From [the documentation](https://postgis.net/docs/geometry_distance_knn.html), it uses an index-assisted (fast!) result only when in the ORDER BY clause.  
```SQL
-- Use the <-> operator to get the nearest five parcels to the centroid
-- for Minneapolis W-8 P-1
SELECT p.state_pin, p.shape
FROM parcels AS p
ORDER BY shape <->
  (SELECT ST_Centroid(ST_Transform(pr.wkb_geometry, 26915))
  FROM precinct AS pr
	WHERE pr.precinct = 'Minneapolis W-8 P-1')
LIMIT 5;


-- Adjust things a bit to get the polling locations closest to each of the
-- precinct centroids
SELECT pr.precinct,
  (SELECT precinct_name
    FROM mock_polling_place AS mpp
	ORDER BY mpp.geom <-> ST_Centroid(pr.wkb_geometry)
	LIMIT 1) AS closest_polling_place,
  ST_Centroid(ST_Transform(pr.wkb_geometry, 26915)) AS geom
FROM precinct AS pr
WHERE pr.precinct LIKE 'Minneapolis%'
ORDER BY pr.precinctid
;
```

If we want to know the things which are within a certain distance of a geometry we know, we use the `ST_DWithin` function.
```SQL
-- Search within 100 m of polling places
SELECT ROW_NUMBER() OVER() AS fid
  , mpp.precinct_name
	, pa.fid
	, pa.state_pin
	, pa.shape
FROM parcels AS pa,
  mock_polling_place AS mpp
WHERE ST_DWithin(pa.shape, ST_Transform(mpp.geom, 26915), 100)
;
```

## Advanced Considerations

Although this workshop is intended to give you a start with using PostGIS, there are a few advanced considerations that you should be aware of.
Many of these topics revolve around indexing or other database performance optimization.
While you may be able to get started without an understanding of indexing and these special cases, you may run into performance issues eventually.

### Spatial Indexing

The index generally uses the bounding box of a line or polygon as a rough approximation of the feature.  
When PostGIS runs a spatial relation function such as `ST_Overlaps`, `ST_Intersects`, or `ST_Touches`, the spatial index can quickly rule out feature pairs where the bounding boxes _don't_ overlap.
Once the number of rows has been limited by the spatial index, the more expensive geometry comparison is run on the (hopefully much smaller) subset of potentially-overlapping features.
Consequently, spatial indexes work most efficiently when comparing objects of similar sizes and when the bounding box is a good approximation of the feature.

For instance, if you wanted to find the parcels within 100 meters of US-169 in Minnesota, that may not work well: MN-169 has a bounding box that would include the entire Twin Cities metro, and runs from the southern border to well north of Duluth.
Because such a high percentage of the parcels in Minnesota would have an overlapping bounding box, many parcel polygons would need to be checked against the buffered road using the slow, complete check.

To address this problem, the feature with the big box should be made small, so the parcels and road are more comparable in size.  Using the `ST_Segmentize` and `ST_Subdivide` functions, we can chop up US-169 into 50 smaller pieces, and those pieces will be much better able to use the spatial index to speed up the querying.  `ST_Subdivide` also works with polygons, so you can take a big geometry (e.g. outline of Canada) and make it a bunch of smaller ones that will query much more easily, and then reconstitute it when needed.  Additionally, if you have multipart geometries that make a feature have a broad spatial extent, breaking it apart into individual features may also give the performance boost you need.

### Vacuum Analyze

Databases need periodic maintenance and care.  When there are transactions against a database table, eventually you will want to make the storage available for rows which have been deleted.  This action is accomplished with the `VACUUM` command.

Similarly, the database maintains statistics on the distribution of data within a given column.  If you are making a lot of changes (for instance, adding a bunch of new data), it can be helpful to run the `ANALYZE` command to refresh these statistics.  This can be combined with the above vacuuming with `VACUUM ANALYZE;`.


### More...

By this point, hopefully you're realizing there's a lot PostGIS can do.  There's a lot more not covered, including linear referencing, 3D operations, time-series and trajectories, and raster workflows.  PostGIS also ships with additional extensions such as the PostGIS TIGER Geocoder, fuzzy string matching, and an address standardizer.

For a full list of PostGIS functions, see the [PostGIS functions documentation page](https://postgis.net/docs/reference.html).  The [Postgres documentation](https://www.postgresql.org/docs/13/index.html) also has useful tips and tricks for the straight database administration and operation, with quick links to manual pages for different Postgres versions.

Both Azure and AWS offer managed database services which include recent versions of Postgres and PostGIS.  If you require Esri ArcSDE compatibility (such as for versioned databases), these options may not work for you.  PostGIS is also readily available in many different Postgres/PostGIS version combinations [via Docker images](https://registry.hub.docker.com/r/postgis/postgis/), which can be a great way to get a working system fast (if you know or are willing to learn Docker; it's neat, but that's a story for another day).

For several years, Crunchy Data has sponsored a PostGIS Day conference, which has been recorded and the talks are available on YouTube.  There is a [playlist for 2021 PostGIS Day](https://www.youtube.com/playlist?list=PLesw5jpZchudjKjwvFks-gAbz9ZZzysFm) and a [playlist for 2019 PostGIS Day](https://www.youtube.com/playlist?list=PLesw5jpZchudjfkx3vggVmJAOSBH7m4nj).  These feature a number of great talks both by PostGIS creators/developers as well as users with a variety of applications.
