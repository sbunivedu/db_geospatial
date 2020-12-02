# Geospatial Data

This case study explores geospatial data processing using PostgreSQL database.

## Install PostgreSQL database engine
To experiment with the examples you need access to a PostgreSQL installation or you can install it on your personal computer. I used [Postgres.app](https://postgresapp.com/) on a Mac.

## PostgreSQL commands
There are handful of commands you need to for the commandline:
* List databases: `\l`
* List tables: `\dt`
* Describe a table: `\d table_name`

Here is a complete reference to the commands: https://www.postgresqltutorial.com/psql-commands/

## Examples
We will follow the examples from [this tutorial](https://learnsql.com/blog/getting-started-with-postgis-your-first-steps-with-the-geography-data-type/) with bug fixes.

First we need to enable enable `postgis` if it is not already included:
```sql
CREATE EXTENSION postgis;
SELECT postgis_full_version();
```

Next, we create the `artwork` table with interesting data.
```sql
CREATE TABLE artwork (
  artwork_name TEXT,
  category VARCHAR(20),
  artist_name VARCHAR (50),
  showed_at_latitude FLOAT,
  showed_at_longitude FLOAT,
  where_is GEOGRAPHY
);


INSERT INTO artwork VALUES ('Giaconda','painting','Leonardo Da Vinci', 48.860547, 2.338513,NULL);
INSERT INTO artwork VALUES ('David','sculpture','Michelangelo Buonarroti', 43.776709, 11.258887,NULL);
INSERT INTO artwork VALUES ('Sunflowers','painting','Vincent Van Gogh', 48.149966, 11.570856,NULL);
INSERT INTO artwork VALUES ('Guernica',' painting','Pablo Picasso', 40.407561, -3.694042,NULL);
```

The next command populate the `where_is` with the coordinates stored in other columns. Note that this command in the tutorial has the longitude and latitude pair in the reverse order, which is incorrect.
```
UPDATE artwork SET where_is = ST_POINT(showed_at_longitude, showed_at_latitude);
```
Expected result:
```
SELECT * FROM artwork;
 artwork_name | category  |       artist_name       | showed_at_latitude | showed_at_longitude |                      where_is                      
--------------+-----------+-------------------------+--------------------+---------------------+----------------------------------------------------
 Giaconda     | painting  | Leonardo Da Vinci       |          48.860547 |            2.338513 | 0101000020E6100000B22AC24D46B50240E0D57267266E4840
 David        | sculpture | Michelangelo Buonarroti |          43.776709 |           11.258887 | 0101000020E610000017B83CD68C84264022C154336BE34540
 Sunflowers   | painting  | Vincent Van Gogh        |          48.149966 |           11.570856 | 0101000020E61000006473D53C4724274087C1FC1532134840
 Guernica     |  painting | Pablo Picasso           |          40.407561 |           -3.694042 | 0101000020E6100000496760E4658D0DC0021077F52A344440
(4 rows)
```
The geospatial data can be converted to KML format as shown in [artworks.kml](artworks.kml) so that it can be visualized on Google earth.

Next, we create the `museum` table with interesting data.
```sql
CREATE TABLE museum (
  museum_name VARCHAR(20),
  country VARCHAR(20),
  perimeter GEOGRAPHY(POLYGON)
);

INSERT INTO museum VALUES ('Munich Pinakotek','Deuschtland',ST_GeogFromText('POLYGON((11 48,11 49,12 49,12 48,11 48))'));
INSERT INTO museum VALUES ('Accademia Gallery','Italy',ST_GeogFromText('POLYGON((11 43,11 44,12 44,12 43,11 43))'));
INSERT INTO museum VALUES ('Musee du Louvre','France',ST_GeogFromText('POLYGON((2 48,2 49,3 49,3 48,2 48))'));
INSERT INTO museum VALUES ('Museo Reina Sofia','Spain',ST_GeogFromText('POLYGON((-4 40,-4 41,-3 41,-3 40,-4 40))'));
```
Expected result:
```
SELECT * FROM museum;
    museum_name    |   country   |                                                                                             perimeter                                                                                              
-------------------+-------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Munich Pinakotek  | Deuschtland | 0103000020E610000001000000050000000000000000002640000000000000484000000000000026400000000000804840000000000000284000000000008048400000000000002840000000000000484000000000000026400000000000004840
 Accademia Gallery | Italy       | 0103000020E610000001000000050000000000000000002640000000000080454000000000000026400000000000004640000000000000284000000000000046400000000000002840000000000080454000000000000026400000000000804540
 Musee du Louvre   | France      | 0103000020E610000001000000050000000000000000000040000000000000484000000000000000400000000000804840000000000000084000000000008048400000000000000840000000000000484000000000000000400000000000004840
 Museo Reina Sofia | Spain       | 0103000020E6100000010000000500000000000000000010C0000000000000444000000000000010C0000000000080444000000000000008C0000000000080444000000000000008C0000000000000444000000000000010C00000000000004440
(4 rows)
```

The spatial data can be converted to KML format as shown in [museums.kml](museums.kml) so that it can be visualized on Google earth.

Let's run some query that involve spatial calculation.

Find distance between pairs of artwork:
```sql
SELECT artwork1.artwork_name, artwork2.artwork_name, ST_DISTANCE(artwork1.where_is, artwork2.where_is)
FROM artwork artwork1, artwork artwork2;

SELECT artwork1.artwork_name, artwork2.artwork_name, ST_DISTANCE(artwork1.where_is, artwork2.where_is)
FROM artwork artwork1, artwork artwork2
WHERE artwork1.artwork_name < artwork2.artwork_name;

SELECT artwork1.artwork_name, artwork2.artwork_name, ST_DISTANCE(artwork1.where_is, artwork2.where_is) * 3.2808399
FROM artwork artwork1, artwork artwork2
WHERE artwork1.artwork_name < artwork2.artwork_name;
```

Sort museums by the geographical size:
```
SELECT museum_name, ST_AREA(perimeter)
FROM museum
ORDER BY 2 DESC;
```
Expected output:
```
museum_name    |      st_area      
-------------------+-------------------
Museo Reina Sofia | 9412788745.689148
Accademia Gallery | 8985387394.837158
Munich Pinakotek  | 8217589924.770569
Musee du Louvre   | 8217589924.770569
(4 rows)
```

Find which artwork is in which museum:
```
SELECT artwork_name, museum_name
FROM museum, artwork
WHERE ST_INTERSECTS(where_is, perimeter);
```
Expected output:
```
artwork_name |    museum_name    
--------------+-------------------
Sunflowers   | Munich Pinakotek
David        | Accademia Gallery
Giaconda     | Musee du Louvre
Guernica     | Museo Reina Sofia
(4 rows)
```

Create a convex hull covering all the artworks:
```sql
SELECT ST_AsText(
         ST_ConvexHull(
           ST_Collect(where_is::geometry))) as ploygon
FROM artwork;
```
Expected output:
```
ploygon                                                    
---------------------------------------------------------------------------------------------------------------
POLYGON((-3.694042 40.407561,2.338513 48.860547,11.570856 48.149966,11.258887 43.776709,-3.694042 40.407561))
(1 row)
```
The data can be converted to KML format as shown in [convex_hull.kml](convex_hull.kml) so that it can be visualized on Google earth.
