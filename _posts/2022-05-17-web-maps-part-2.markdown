---
layout: post
title:  "Web Maps - part 2"
date:   2022-05-17 23:12:07 +0200
author: tomasz
tags: OpenStreetMap webmaps
---

Continuing [part one](https://ttomasz.github.io/2022-05-05/web-maps-part-1) let's go through vector tiles formats and schemas based on OpenStreetMap data.

## Vector tiles

[Part one](https://ttomasz.github.io/2022-05-05/web-maps-part-1) and Mapbox docs [1](https://docs.mapbox.com/data/tilesets/guides/vector-tiles-introduction/) [2](https://docs.mapbox.com/data/tilesets/guides/vector-tiles-standards/) explain briefly what Vector tiles are. Now onto how they are stored.

### Formats

While one can generate tiles dynamically on request by user oftentimes they are pre-generated for the entire dataset. This is viable for vector tiles but not raster tiles. With raster tiles maps often go to zoom levels of 18 or 19 which contain so many tiles that it's not economically justified to render them all. With vector tiles they are generated usually for zoom levels up to 14 or 15 because we can scale geometries for other zoom levels.

There are a few ways that pre-generated vector tiles are packaged before distribution.

#### PBF files

Each tile can be saved as PBF file. Other solution package these files in a single database/archive but these files can be saved directly on disk and served by any web server. This solution is however not the most performant since we end up with millions of small files.

#### MBTILES

This is the most popular solution. PBF files are saved in SQLite database and identified by zoom, x, y attributes. It's much more convenient since we have a single file containing all the data.

You need a server that will query the db for individual tile and serve it back to web server and user.

Some servers provide ability to render vector tiles to raster images and serve them using WMTS/XYZ service to clients that do not support vector tiles as source.

#### PMTILES

[Github repo with a spec](https://github.com/protomaps/PMTiles)

[Protomaps blog post explaining how the format works](https://protomaps.com/blog/dynamic-maps-static-storage/)

Cloud native format designed by Brandon Liu (Protomaps).

Tiles are saved into a single file - similarly to MBTILES. This file is internally divided in blocks which allows [HTTP Range Requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests) so you can serve it from S3 or with regular webserver. No need for additional server software. The client however needs to be able to calculate which part of th file to download to get tiles it wants. Author prepared plugins to popular JavaScript libraries for web maps that allow using this format as data source.

With this format one can create fully static websites serving maps even for an entire planet.

#### COMTiles

[Github repo](https://github.com/mactrem/com-tiles)

Cloud Optimized Map Tiles. Project similar to PMTILES.

### Schemas

To use any data you need to know its structure - what columns or attributes are available and what values in them mean. This structure is called a schema.

A style will be tied to a schema as it specifies for example that a point with attribute _"amenity"_ and value _"pub"_ will be rendered as icon _"pub.png"_ with text from attribute _"name"_ underneath.

Raw data e.g. OpenStreetMap is processed and transformed to fit into a chosen schema.

#### Mapbox

The first schemas, developed by Mapbox and tied to their services. You can read what kind of features are available and what attributes they have [in the Mapbox docs](https://docs.mapbox.com/data/tilesets/reference/)

Only available as a service from Mapbox.

#### OpenMapTiles

[Project](https://openmaptiles.org) maintained by Maptiler. It provides an open schema and set of tools to generate and serve vector tiles. It also provides some styles compatible with the schema.

Schema is described [under this link](https://openmaptiles.org/schema/).

It is the most popular schema at the moment. Alternative tools to generate tiles usually aim to be able to produce tiles in this format.

License requires attribution similarly to OpenStreetMap.

#### Shortbread (Geofabrik)

Schema recently created by Geofabrik. Right now this schema requires a special branch of Tilemaker to generate.

[Read docs](https://shortbread.geofabrik.de/) to see how to set it up.

Goal of this schema is to create simple background maps. It doesn't have POI layer. Nice addition to docs is OSM tags mapping for each feature layer.

#### Protomaps

Recently released. Created for Protomaps Mantle service.

[Docs](https://protomaps.com/docs/map-layers-tags) contain description what OSM tags map to each layer.

#### Tilezen (Mapzen)

Created for Mapzen vector tile service. Mapzen commercial activity was shut down but all components are open source.

Tilezen primarily sources from OpenStreetMap, but includes a variety of other open data which is quite unique.

[Docs URL](https://tilezen.readthedocs.io/en/latest/)
