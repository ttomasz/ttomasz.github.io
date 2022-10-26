---
layout: post
title:  "Web Maps - part 3"
date:   2022-10-26 22:31:15 +0200
author: tomasz
tags: OpenStreetMap webmaps
---

Continuing [part two](https://ttomasz.github.io/2022-05-17/web-maps-part-2) let's go through vector tiles generators.

## Vector Tiles

### Generators

There are many software projects that can generate vector tiles. Here are some of them.

#### OpenMapTiles

One of the first projects. It's a reference implementation for OpenMapTiles schema.

The projects has many components like Imposm3 fork to import data into PostgreSQL+PostGIS database, a lot of custom PL/PGSQL code to generate data in OpenMapTiles schema etc and everything is glued together using Docker.

They use PostGIS to generate the tiles.

Generating MBTILES for entire planet take a very long time but this project as one of the only ones supports incremental updates and generating tiles "on the fly".

[Project website](https://openmaptiles.org/)

[Github org](https://github.com/openmaptiles)

#### PostGIS

PostGIS can generate individual tiles using functions like:
- [ST_AsMVT](https://postgis.net/docs/ST_AsMVT.html)
- [ST_AsMVTGeom](https://postgis.net/docs/ST_AsMVTGeom.html)

Another useful function is [ST_TileEnvelope](https://postgis.net/docs/ST_TileEnvelope.html) which allows converting xyz coordinates to a geometry (rectangle).

Note that this function just encodes the data in vector tiles format. We define the schema.

#### Tippecanoe

CLI tool written in C++ that allows converting GeoJSON files to vector tiles (both mbtiles archive and individual files).

Has many useful options like clustering features or automatic max zoom level finder.

They provide many examples of usage in the docs.

The tool just converts the GeoJSON features which means that if we want specific schema we need to provide GeoJSON files already prepared.

It was originally created by Mapbox but now it's maintained by Felt.

[Github repo](https://github.com/felt/tippecanoe/)

#### Tilemaker

Written in C++ this tool allows generating vector tiles for entire planet in less than 24 hours which is significantly faster than OpenMapTiles.

It allows writing LUA scripts to perform data transforms which can be incredibly powerful to get more out of OpenStreetMap data.

Out of the box it contains OpenMapTiles compatible schema but it allows specifying custom ones too (JSON + LUA config). 

[Project website](https://tilemaker.org/)

[Github repo](https://github.com/systemed/tilemaker)

#### Planetiler

Currently the fastest generator. Allows processing entire planet in about 2 hours.

It provides OpenMapTiles compatible schema (some features like grouping bus stops are not available) out of the box.

Allows defining custom schemas with YAML config.

It can try to decrease MBTILES size by deduplicating ocean tiles.

Written in Java.

[Github repo](https://github.com/onthegomap/planetiler)
