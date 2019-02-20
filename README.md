# West Nile Virus and St. Louis Encephalitis In California's Vulnerable Communities

West Nile Virus is pretty well known at this point but we've seen a reemergence here in California of St. Louis Encephalitis since about 2015 after a long period without detection.  In the web map linked in the data sources below we can see where WNV/SLE outbreaks may be occurring and with my project I hope to make an attempt at showing who may be affected the most by these viruses.  I would like to use CDC created layers for vulnerable census tracts in conjuction with surveillance data for WNV and SLE in California to provide finer grain detail as to what areas may be affected by these viruses instead of the more general detail we see with a point feaure displayed on a map with no other layers.

### Thoughts For Maps

1. Time series map with slider to display WNV & SLE Data.

2.  Individual maps for 2016, 2014, and 2010 showing detections overlayed with CDC layer for vulnerable census tracts.
    
3. Point in polygon analysis for these years?
     
     - Are we seeing more detections in tracts that were deemed vulnerable than in those that have not?
        - Ran a little bit of analysis for 2016 for SLE in PostGIS and seems like 60% of positives were in census tracts with a vulnerabiltiy value of .75 or higher
     - Did we see an increase in detections in vulnerable tracts for these years?
     - What's the cutoff for SVI index to determine most vulnerable?
     - Demographics of these census tracts that are seeing detections

4. Hexbin map for these years possibly?

### Data Sources

[CDC Social Vulnerability Index (SVI)](https://svi.cdc.gov/data-and-tools-download.html)
The CDC describes their Social Vulnerability index as a way to measure the ability of a community to withstand "external stresses on human health, such as natural or human-caused disasters, or disease outbreaks".  The Social Vulnerability Index has adopted "15 
U.S. census variables at the tract level" to aid in the identification of communities that would potentially need help in being able to prepare for, or recover from, a disaster which could dramtically afffect its public health and well-being.

[California Surveillance Gateway Maps](https://maps.calsurv.org/)

![California Surveillance Gateway Map](./images/csgMap.PNG)

### PostGIS Analysis For SLE Positives In Vulnerable Census Tracts For 2016

#### Add CDC 2016 GeoJSON to PostGIS

`ogr2ogr -f "PostgreSQL" PG:"host=localhost dbname=california user=postgres password=123" cdcSVI2016.geojson -nln data.cdc_svi_2016`


#### Add SLE GeoJSON to PostGIS

`ogr2ogr -f "PostgreSQL" PG:"host=localhost dbname=california user=postgres password=123" sle2015_2018_cleaned.geojson -nln data.sle_2015_2018`


#### Change date column in SLE table to date type from string

```sql
ALTER TABLE data.sle_2015_2018
ALTER COLUMN date TYPE date using to_date(date, 'MM/DD/YYYY');
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
