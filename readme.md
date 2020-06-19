create table
rental_permits
(address text,
permit text,
subtype text,
status text,
owner text,
issued text,
expired text,
description text,
stipulations text)


> It is important to note that not all of these are active rentals, and therefore, we cannot ensure the accuracy of the information provided for any CR with a status of anything other than “Certified” or “Under Review”.

Washtenaw County geocode server, doesn't seem to work: https://webmapssecure.ewashtenaw.org/arcgisshared/rest/services/WashtenawComposite/GeocodeServer

```
ogr2ogr -f GeoJSON permits.geojson pg:'user=matth password= host=localhost port=5432 dbname=matth' rental_permits -sql "select * from rental_permits where clean_status is not null and clean_status != 'owner_occupied'"
```

Permits to tippecanoe: `-r1` keeps all features

```
tippecanoe --output-to-directory=permit_tiles permits.geojson --include=address --include=owner --include=issued --include=expired --include=description --include=stipulations --include=clean_status --read-parallel -f --no-tile-compression --drop-densest-as-needed -r1
```

## Fixing geocodes


```
select distinct address, 'Ann Arbor' as city, 'MI' as state
from rental_permits
where geom is null;

select distinct address, 'Ann Arbor' as city, 'MI' as state
from rental_permits
where address ilike '%brookfield%'
or address ilike '%braeburn%'
or address ilike '%arrowwood%';

-- Bring it back into postgres
drop table rental_geocodes_1;
create table
rental_geocodes_1 (address text, lat text, lng text, quality text);

update rental_geocodes_1
set address = replace(address, ', Ann Arbor, MI', '');

update rental_permits
set lat = g.lat,
lng = g.lng,
quality = g.quality
from rental_geocodes_1 g
where rental_permits.address = g.address;
```



## Address points:

```
create table ann_arbor_parcel_address
as
select address,
(regexp_matches(address, '\d+'))[1],
st_centroid(wkb_geometry) as geom
from ann_arbor_parcels;
```

```
ogr2ogr -f GeoJSON ann_arbor_parcel_address.geojson pg:'user=matth password= host=localhost port=5432 dbname=matth' ann_arbor_parcel_address -sql "select * from ann_arbor_parcel_address"
```

## Address labels:
address labels to tippecanoe: `-r1` keeps all features. `-Z17` only starts generating at z17:

```
tippecanoe --output-to-directory=address_labels ann_arbor_parcel_address.geojson --include=num --read-parallel -f --no-tile-compression --drop-densest-as-needed -r1 -Z15 -z16
```

```
tippecanoe --output-to-directory=damnit ann_arbor_parcel_address.geojson --read-parallel -f --no-tile-compression
```