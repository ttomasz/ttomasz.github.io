---
layout: post
title:  "Web Maps - part 1"
date:   2022-05-05 23:25:41 +0200
author: tomasz
tags: OpenStreetMap webmaps
---

Web maps have evolved greatly in the recent years. Vector tiles is particularly hot technology that has seen a lot of development recently. I wanted to provide general intro into concepts related to web maps and describe some of the recent developments in vector tile tech.

This is part one.

# Introduction

To create a web map you need some data with information about location. It can be either vector or raster. For vector features you also need information on how to render them (what style to use). There are many formats used and in this post only some of them will be mentioned.

If you are unfamiliar with [GIS](https://en.wikipedia.org/wiki/Geographic_information_system) this is is a little introduction into the domain.

_Note: These are simplified explanations. If you studied geodesy please don't be mad._

Spatial data contains information about location represented using numbers (coordinates). Most of us are probably familiar with longitude and latitude used by GPS.

Most uses are covered by three basic geometry types:
- point
- line (also called linestring)
- polygon (area)

There are also multi-points/lines/polygons, collections and other types but this is beyond the scope of this post.

Aforementioned longitude and latitude are expressed in degrees. This is not very convenient when trying to calculate distance or area. While calculating distance between two points is easy enough using [Haversine formula](https://en.wikipedia.org/wiki/Haversine_formula), anything more complicated can be pretty cumbersome. Additionally if you try plotting the geometries using degrees on a flat surface it's going to be pretty distorted and won't make for a very good map.

Fortunately, thanks to the power of complicated math we can transform our spatial data from system based on 3d globe and degrees to a system using nice flat cartesian plane using meters as units. Of course there is a catch. You can't perfectly flatten 3d globe. This means that these transformations are not perfect and will always have some distortions. Either shape or area of the features won't be quite right and it's a matter of choosing the right compromise for given scenario. These systems are called [Coordinate/Spatial Reference Systems](https://en.wikipedia.org/wiki/Spatial_reference_system) in short CRS or SRS.

# Coordinate Reference Systems (CRS)

Coordinate Reference Systems are identified using [EPSG](https://en.wikipedia.org/wiki/EPSG_Geodetic_Parameter_Dataset) codes. They also have their own names given by creators or legislators but using EPSG codes is generally easier.

In this context you need to know only about two CRS-es:
- EPSG:4326 aka [WGS84](https://en.wikipedia.org/wiki/World_Geodetic_System#WGS84)
- EPSG:3857 aka [WebMercator](https://en.wikipedia.org/wiki/Web_Mercator_projection)

EPSG:4326 is the one with latitude and longitude expressed in degrees.

EPSG:3857 was created by Google for web maps and is de facto standard for general purpose web maps. Units are meters. It's good for navigation but the compromise it makes is that areas further away from equator are stretched. For example Greenland with the real life area of 2,166,086 km2 looks to be the size of Africa that has area of 30,370,000 km2.

Mapbox is trying very interesting idea of dynamically switching projections as you zoom in or out to mitigate this. See their docs: https://docs.mapbox.com/mapbox-gl-js/guides/projections/

# Data Sources

To display a map you need some data. Spatial data is usually either in a raster format or a vector format.

Vector data contains geometry (points/lines/polygons) which can be styled by client to render it on a map.

Raster data is images that have information about their location. It can be for example satellite imagery or a map rendered by a server that's ready to display.

Some formats/protocols commonly used are described below.

## GeoJSON

It's a superset of JSON with a slightly more rigid structure and a set of rules that describe encoding geometries. For processing large datasets it can be used in ["line separated" flavour](https://datatracker.ietf.org/doc/html/rfc8142) just [like JSON](https://datatracker.ietf.org/doc/html/rfc7464) although this is not used in web maps.

For small datasets it can be a good source of vector data.

## OGC Services

Open Geospatial Consortium publishes standards for spatial data and services.

Standards most relevant here - and most used - are [WMS](#WMS) and WMTS. You may also encounter WFS. There are new versions of the standards being worked on but I won't be describing them here since their adoption is only beginning.

Many institutions that want to share spatial data provide it using services using OGC standards/protocols. These can be used for web maps.

### WMS

It describes a service (API) that returns map or part of map as an image. In the request client provides parameters such as: width and height of the map, coordinate reference system, image format, and area that interests them.

This protocol is pretty simple but not very scalable. If the clients are smart they will request images of constant sizes which they can stitch together to show on a map. This allows caching which is crucial for performance but not all clients may do that.

Performance makes it a poor choice as the main service for a global map such as OpenStreetMap or Google Maps.

Bonus feature of this standard is that it has API for user clicking on a map which provides information about objects in that location.

[Wiki article](https://en.wikipedia.org/wiki/Web_Map_Service)

### WMTS

This is optimized version of WMS. It divides the world virtually into predefined tiles. Clients can request specific tiles which can be cached. This makes it much faster.

The problem is that it was made to support all projections which makes it more complicated than a competing standard for serving raster tiles (images): [XYZ](https://en.wikipedia.org/wiki/Tiled_web_map).

Not many javascript libraries used for web maps support this protocol.

[Wiki article](https://en.wikipedia.org/wiki/Web_Map_Tile_Service)

### WFS

Similar to WMS but provides vector data for given area (usually referred to as bounding box or envelope).

Standard defines GML (spatial XML) as the default response data type. Other data types are subject to servers capabilities.

Not many javascript libraries used for web maps support this protocol.

[Wiki article](https://en.wikipedia.org/wiki/Web_Feature_Service)

## Raster tiles

Raster tiles are parts of the map served as 256x256 or 512x512 pixel images. The client library stitches them together and shows as continuous map.
They use `XYZ` scheme mentioned in the `WMTS` section. This method of distributing maps is highly scalable and widely supported.

[OpenStreetMap](https://osm.org) uses this to share their map with the world.

XYZ tiles use very simple addressing scheme where each tile is identified by three values: x, y and z (zoom). The url will be usually in the form of: `https://some.domain/{zoom}/{x}/{y}.png`. Using `EPSG:3857` projection is pretty much assumed. There is a deeper explanation how the x, y, and z values are calculated [on OSM Wiki](https://wiki.openstreetmap.org/wiki/Slippy_map_tilenames).

Map is rendered server side using software like [Mapnik](https://wiki.openstreetmap.org/wiki/Mapnik).

The main disadvantage of raster tiles is that each style of the map needs to be rendered separately.

## Vector tiles

Vector tiles also use `XYZ` scheme but instead of images they contain vector features with their geometry and attributes.

Mapbox codified open standard called [Mapbox Vector Tiles](https://github.com/mapbox/vector-tile-spec) and created first open source library which supported it [Mapbox GL JS](https://github.com/mapbox/mapbox-gl-js). They have changed license of the library recently which resulted in a fork called [MapLibre GL JS](https://github.com/maplibre/maplibre-gl-js) which will be described later.

Vector tiles assume simplification of complex geometries - like administrative boundaries or large lakes - which works well for most maps but in some use cases may not be suitable.

With vector tiles features are styled client side which means that you can have one source of data for multiple styles.

Mapbox created [style language definition](https://docs.mapbox.com/mapbox-gl-js/style-spec/) which is great as it allows defining styles as JSON files instead of having to code rules using regular programming language. There is one open source editor: [Maputnik](https://maputnik.github.io/) that allows creating and editing styles in a visual editor. Unfortunately development of Maputnik slowed down, publicly deployed version (1.7.0) contains errors that sometimes prevent it from working at all and the 1.8.0 version seems to be ready but it's unreleased and one would need to build and release it from source code.

---

End of part one.
