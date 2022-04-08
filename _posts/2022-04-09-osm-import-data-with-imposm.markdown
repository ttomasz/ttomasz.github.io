---
layout: post
title:  "Import OpenStreetMap data to PostgreSQL with Imposm3"
date:   2022-04-09 01:57:23 +0200
author: tomasz
tags: OpenStreetMap
---

## Introduction

If you want to import OpenStreetMap data to PostgreSQL (+ PostGIS) database two popular tools are [osm2pgsql](https://osm2pgsql.org/) and [imposm3](https://imposm.org/docs/imposm3/latest/).

Both were designed to prepare data for rendering although some time ago `osm2pgsql` was upgraded with scripting capabilities that go beyond simple mapping files that specify what tables to create and what objects should be inserted there (filter by tags).

There is also [osmosis](https://github.com/openstreetmap/osmosis) if you want a - more or less - copy of OpenStreetMap database.

`imposm3` development was put on hold due to lack of funding while `osm2pgsql` is actively developed. This makes it questionable choice to use `imposm3` but it does have some nice qualities that can make it better for some projects.

In my opinion it's a very good tool that is useful when:
- you have a webapp and want to use OSM data but Overpass API is too slow/limited
- you want to use OSM data for spatial analysis and keep it updated

## Subjective evaluation:

__Pros:__
- __single file binary executable (not dependent on Java)__
- __automatic minutely data updates__
- faster than `osm2pgsql`
- nicer mapping file syntax than `osm2pgsql`
- can limit import of data to specified area (not just bbox)

__Cons:__
- __not maintained__
- __does not have scripting capabilities like `osm2pgsql`'s Flex Output__
- supports only Linux

Setup is very simple. You download the executable, prepare config and run the executable to import the data to the database. Then you can run the executable with a different flag/command to automatically download and apply OSM data updates from replication log or even set it up as a service/daemon that will run in the background and keep your data updated.

No dependencies, no need for external tools - like osmosis that is usually paired with `osm2pgsql` - just a nice single executable.

I compared performance of both tools long time ago and probably `osm2pgsql` got faster since then but for me import of some specific data for Poland was about 45 minutes for `osm2pgsql` and 15 minutes for `imposm3`.

## Example usage

I'll list commands make an import of some data. This requires Linux based OS ([WSL](https://docs.microsoft.com/en-us/windows/wsl/about) works well). I'll launch docker container with PostgreSQL database but you can use any other way to set up a database.

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
    - key: highway
      name: highway
      type: string
    - name: tags
      type: hstore_tags
    - name: geometry
      type: geometry
    filters:
      reject:
        area: ["yes"]
    mapping:
      highway: [__any__]
```

Create _config.json_ with a text editor.

Here's an example, adjust paths and connections details accordingly.

_srid_ means [EPSG](https://en.wikipedia.org/wiki/EPSG_Geodetic_Parameter_Dataset) code number (i.e. id of [CRS](https://en.wikipedia.org/wiki/Spatial_reference_system)). It can be _4326_ (geometry uses degrees as unit) or _3857_ (units are meters).

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

If you don't need the data to be extremely fresh you can set update interval to be 1 hour by adding these lines to _config.json_
```
    "replication_url": "https://planet.openstreetmap.org/replication/hour/",
    "replication_interval": "1h"
```

If you want to create temporary PostgreSQL database using docker run this command:

```bash
docker run --name "postgis25434" --shm-size=4g -e MAINTAINANCE_WORK_MEM=1024MB -p 25434:5432 -e POSTGRES_USER=postgres -e POSTGRES_PASS=1234 -e POSTGRES_DBNAME=gis -d -t kartoza/postgis
```

- _-p_ - sets port
- _-d_ - means container will be run in background

Now let's run data import:

```bash
./imposm-0.11.1-linux-x86-64/imposm import -config ./config.json -read ./opolskie-latest.osm.pbf -overwritecache -write -diff -deployproduction
```

- _-overwritecache_ - will clear _cachedir_ before import
- _-diff_ - will save additional data needed to apply data updates
- _-deployproduction_ - will move the data to _public_ schema, default is to save it to _import_ schema


__Note: When running import again existing data will be moved to _backup_ schema.__

All options are described in more details [in the docs](https://imposm.org/docs/imposm3/latest/tutorial.html#importing).

You can connect to the database and query the data using software like QGIS or just `psql` e.g.:

```bash
psql -h localhost -p 25434 -U postgres -d gis
```

To update the data run the following command:

```bash
./imposm-0.11.1-linux-x86-64/imposm run -config ./config.json
```

This will run continuously until you stop it with `CTR+C`.

## Making systemd service

Example of making systemd service on Ubuntu.

Let's create a script (you can name it: "imposm_run.sh") that will launch data updating process:

```bash
#!/bin/bash

<specify directory>/imposm-0.11.1-linux-x86-64/imposm run -config <specify directory>/config.json
```

Our service will start after PostgreSQL service starts. When using external database e.g. RDS that would not be desired. In such case remove `After=postgresql.service` from service file.

Create "imposm.service" file:

```
[Unit]
Description=IMPOSM3 service that keeps your copy of OpenStreetMap data updated.
After=network.target
After=postgresql.service
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=5
User=<specify user>
ExecStart=/bin/bash "<specify directory>/imposm_run.sh"

[Install]
WantedBy=multi-user.target
```

Copy service file to correct location, set its permissions, and reload service daemon:

```bash
sudo cp <specify directory>/imposm.service /etc/systemd/system/imposm.service
sudo chmod 600 /etc/systemd/system/imposm.service
sudo systemctl daemon-reload
```

Now we can use these commands to start and stop our service:

```bash
sudo service imposm start
sudo service imposm stop
```

## Tips for production use

Process that updates our database generates some data that stays on disk. If left without supervision this will grow in size indefinitely.

You can add these commands in `cron` to keep disk usage low.

```bash
# deleting osm replication files downloaded more than 7 days ago
find <specify directory>/imposm_diff/0* -mtime +7 -type f -delete
# remove empty directories
find <specify directory>/imposm_diff/0* -empty -type d -delete
# most of imposm logs are printed to system log by default
# you can clean it up with this command
journalctl --vacuum-time=10d
```

Additionally I would recommend to run `VACUUM ANALYZE` in Postgres periodically.

After service has been running for a long time shutting it down temporarily and _"defragmenting"_ data by running `VACUUM FULL` then `ANALYZE` may improve disk usage and performance.
