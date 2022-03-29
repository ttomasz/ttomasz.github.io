---
layout: post
title:  "Import OpenStreetMap data to PostgreSQL with Imposm3"
date:   2022-03-30 01:25:36 +0200
author: tomasz
tags: OpenStreetMap
---

## Introduction

If you want to import OpenStreetMap data to PostgreSQL (+ PostGIS) database two popular tools are [osm2pgsql](https://osm2pgsql.org/) and [imposm3](https://imposm.org/docs/imposm3/latest/).

Both were designed to prepare data for rendering although some time ago osm2pgsql was upgraded with scripting capabilities that go beyond simple mapping files that specify what tables to create and what objects should be inserted there (filter by tags).

Imposm3 development was put on hold due to lack of funding while osm2pgsql is actively developed. This makes it questionable choice to use imposm3 but it does have some nice qualities that can make it better for some projects.

## Subjective evaluation:

__Pros:__
- __single file binary executable (not dependent on Java)__
- __automatic minutely data updates__
- faster than osm2pgsql
- nicer mapping file syntax than osm2pgsql
- can limit import of data to specified area (not just bbox)

__Cons:__
- __not maintained__
- __does not have scripting capabilities like osm2pgsql's Flex Output__
- supports only Linux

Setup is very simple. You download the executable, prepare config and run the executable to import the data to the database. Then you can run the executable with a different flag to automatically download and apply OSM data updates from replication log or even set it up as a service/daemon that will run in the background and keep your data updated. No dependencies, no need for external tools - like osmosis that is usually paired with osm2pgsql - just a nice single executable. I compared performance of both tools long time ago and probably osm2pgsql got faster since then but for me import of some specific data for Poland was about 45 minutes for osm2pgsql and 15 minutes for imposm3.

## Example usage

I'll list commands make an import of some data. This requires Linux based OS. I'll launch docker container with PostgreSQL database but you can use any other way to set up a database.

First let's make a working directory:
```bash
mkdir imposm-test
cd imposm-test
```

Then let's download the binary executable:
```bash
wget https://github.com/omniscale/imposm3/releases/download/v0.11.1/imposm-0.11.1-linux-x86-64.tar.gz
tar -xvf imposm-0.11.1-linux-x86-64.tar.gz
```

We'll download some small OSM extract as well as geojson file that will limit the import area:
```bash
wget http://download.geofabrik.de/europe/poland/opolskie-latest.osm.pbf
wget https://github.com/openstreetmap-polska/gugik2osm/raw/main/imposm3/poland.geojson
```

For this example we'll import buildings and roads. Full docs are in https://imposm.org/docs/imposm3/latest/mapping.html  
Create YAML file _mapping.yaml_ with your preferred text editor.
```yaml
tags:
  load_all: true
areas:
  area_tags: [building, landuse, leisure, natural, aeroway, amenity, shop, "building:part", boundary, historic, place, "area:highway", craft, office, public_transport, tourism, allotments, club, "demolished:building", "abandoned:building", healthcare, industrial, residential]
  linear_tags: [highway, barrier, route]
tables:
  buildings:
    type: polygon
    columns:
    - name: osm_id
      type: id
    - name: geometry
      type: geometry
    - key: building
      name: building
      type: string
    - key: building:levels
      name: levels
      type: string
    - key: roof:shape
      name: roof_shape
      type: string
    - key: building:flats
      name: flats_number
      type: string
    - key: building:levels:underground
      name: levels_underground
      type: string
    - key: height
      name: height_above_ground
      type: string
    mapping:
      building: [__any__]
  roads:
    type: linestring
    columns:
    - name: osm_id
      type: id
    - name: geometry
      type: geometry
    - name: tags
      type: hstore_tags
    filters:
      reject:
        area: ["yes"]
    mapping:
      highway: [__any__]
```

Create _config.json_ with a text editor.  
Here's an example, adjust paths and connections details accordingly:
```json
{
    "cachedir": "/home/tt/imposm-test/imposm_cache/",
    "connection": "postgis://postgres:1234@localhost:25434/gis",
    "limitto":  "/home/tt/imposm-test/poland.geojson",
    "mapping":  "/home/tt/imposm-test/mapping.yaml",
    "srid": 4326,
    "diffdir":  "/home/tt/imposm-test/imposm_diff/"
}
```

If you want to create temporary PostgreSQL database using docker run this command:
```bash
docker run --name "postgis25434" --shm-size=4g -e MAINTAINANCE_WORK_MEM=1024MB -p 25434:5432 -e POSTGRES_USER=postgres -e POSTGRES_PASS=1234 -e POSTGRES_DBNAME=gis -d -t kartoza/postgis
```

Now let's run data import:
```bash
./imposm-0.11.1-linux-x86-64/imposm import -config ./config.json -read ./opolskie-latest.osm.pbf -overwritecache -write -diff -deployproduction
```
All options are described in more details [in the docs](https://imposm.org/docs/imposm3/latest/tutorial.html#importing).

You can connect to the database and query the data using software like QGIS or just psql:
```bash
psql -h localhost -p 25434 -U postgres -d gis
```

To update the data run the following command:
```bash
./imposm-0.11.1-linux-x86-64/imposm run -config ./config.json
```
This will run continuously until you stop it with `CTR+C`.
