### PostGIS Analysis For SLEV/WNV Positives In Vulnerable Census Tracts For 2016

#### Add CDC 2016 GeoJSON to PostGIS

`ogr2ogr -f "PostgreSQL" PG:"host=localhost dbname=california user=postgres password=123" cdcSVI2016.geojson -nln data.cdc_svi_2016`


#### Add SLE GeoJSON to PostGIS

`ogr2ogr -f "PostgreSQL" PG:"host=localhost dbname=california user=postgres password=123" sle2015_2018_cleaned.geojson -nln data.sle_2015_2018`


#### Change date column in SLE table to date type from string

```sql
ALTER TABLE data.sle_2015_2018
ALTER COLUMN date TYPE date using to_date(date, 'MM/DD/YYYY');
```

#### Change date field unix timestamps

```
extract(EPOCH from data.sle_2015_2018.date) as date_timestamp
```

#### Reproject point geometry in SLE table to  CA Albers

```sql
alter table data.sle_2015_2018
alter column wkb_geometry
type geometry(Point, 3310)
using st_transform(wkb_geometry, 3310);
```


#### Reproject polygon/multipolygon geometry in CDC 2016 table to CA Albers

```sql
alter table data.cdc_svi_2016
alter column wkb_geometry
type geometry(Geometry, 3310)
using st_transform(wkb_geometry, 3310);
```

#### Rename tables to reflect geometry projection

```sql
ALTER TABLE data.cdc_svi_2016
RENAME TO cdc_svi_2016_3310;
```

```sql
ALTER TABLE data.sle2015_2018
RENAME TO sle_2015_2018_3310;
```


#### Points within CDC SVI polygons with overall vulnerability value of >= .75 

```sql
 SELECT year_2016.city,
    year_2016.county,
    year_2016.spectype,
    year_2016.wkb_geometry
   FROM ( SELECT sle_2015_2018_3310.ogc_fid,
            sle_2015_2018_3310.date,
            sle_2015_2018_3310.city,
            sle_2015_2018_3310.county,
            sle_2015_2018_3310.virus,
            sle_2015_2018_3310.spectype,
            sle_2015_2018_3310.wkb_geometry
           FROM data.sle_2015_2018_3310
          WHERE date_part('year'::text, sle_2015_2018_3310.date) = '2016'::double precision) year_2016
     JOIN data.cdc_svi_2016_3310 ON st_dwithin(year_2016.wkb_geometry, cdc_svi_2016_3310.wkb_geometry, 0::double precision)
  WHERE cdc_svi_2016_3310.rpl_themes >= 0.75::double precision;
```

#### Export the above view to a GeoJSON with GDAL

`ogr2ogr -t_srs EPSG:4326 -s_srs EPSG:3310 -f GeoJSON sle_2016_cdc_svi_gt75.geojson "PG:host=localhost dbname=california user=postgres password=123" -sql "SELECT * FROM data.sle_2016_cdc_svi"`

#### Working with the WNV data was similar to the SLE commands above with one exception.
#### Since the date column was a list of strings (e.g. `8-19-2003, 9-2-2003`) a new `date_array` column was created to work with instead of manipulating the original column.
```sql
-- Add a data_array column to hold a copy of the date column to manipulate as an array type
ALTER TABLE data.wnv_2003_2018_3310
ADD COLUMN date_array varchar;

-- Copy over date to date_array
UPDATE data.wnv_2003_2018_3310 
SET date_array = date;

-- Change tpe of the date_array column
alter table data.wnv_2003_2018_3310 alter date_array drop default;
alter table data.wnv_2003_2018_3310 alter date_array type text[] using array [date];
alter table data.wnv_2003_2018_3310 alter date_array set default '{}';
```

#### Table was then queried similar to the SLEV table to get WNV points in vulnerable census tracts for a given year.
#### The `where` clause in the subquery was changed to be able to access the first element of the entry in the `date_array` column to pick out a particular year. 
```sql
 SELECT year_2010.city,
    year_2010.spectype,
    year_2010.wkb_geometry
   FROM ( SELECT wnv_2003_2018_3310.ogc_fid,
            wnv_2003_2018_3310.date,
            wnv_2003_2018_3310.city,
            wnv_2003_2018_3310.virus,
            wnv_2003_2018_3310.spectype,
            wnv_2003_2018_3310.wkb_geometry
           FROM data.wnv_2003_2018_3310
          WHERE wnv_2003_2018_3310.date[1] ~~ '%2010%'::text) year_2010
     JOIN data.cdc_svi_2010_3310 ON st_dwithin(year_2010.wkb_geometry, cdc_svi_2010_3310.wkb_geometry, 0::double precision)
```

#### Counts of detections by city for 2016 from sle table

```sql
 SELECT count(*) AS thecount,
    city_counts.city,
    city_counts.spectype,
    city_counts.county
   FROM ( SELECT sle_2015_2018_3310.ogc_fid,
            sle_2015_2018_3310.date,
            sle_2015_2018_3310.city,
            sle_2015_2018_3310.county,
            sle_2015_2018_3310.virus,
            sle_2015_2018_3310.spectype,
            sle_2015_2018_3310.wkb_geometry
           FROM data.sle_2015_2018_3310
          WHERE date_part('year'::text, sle_2015_2018_3310.date) = '2016'::double precision) city_counts
  GROUP BY city_counts.city, city_counts.county, city_counts.spectype
  ORDER BY (count(*)) DESC;
```

