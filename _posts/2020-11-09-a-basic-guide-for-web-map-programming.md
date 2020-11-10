---
title: "A Basic Guide For Web Map Programming"
date: 2020-11-09T21:00:00-00:00
classes: wide
categories:
  - Maps
tags:
  - Programming
  - GIS
  - WGS84
  - Django
  - Python
---

## Getting started

When building a web application that needs to display mapping data, calculate distances or store coordinates you might eventually run into some confusion about how all these systems need to work together.

Here is a brief explanation/cheat sheet about some of the things you will need to know.

## Definitions of terms

Ellipsoid
:   In basic terms: An ellipsoid is a deformed (squashed/non perfect) sphere.

:   An ellipsoid is used in geographic information systems as a compromise to best portray the (non spherical) earth in a way that can be used to perform mathematical calculations.

Mercator Projection
:   The Mercator Projection is a projection (or design of map) that is used to display a spherical object (the Earth) onto a flat surface (a 2D map).

Web Mercator Projection (Pseudo-Mercator)
:   The Mercator Projection, but warped slightly for the purposes of web mapping.
:   Google Maps, Mapbox, OpenStreetMap etc. all use the Web Mercator Projection

WGS84
:   WGS stands for World Geodetic System.
:   (WGS)84 is the latest revision of this system (made in 1984).
:   In basic terms: WGS84 defines the coordinate system used to map the earth.

SRID
:   Spacial Reference System Identifier

:   In basic terms: A unique identifier used to define a spacial coordinate system.

EPSG
:   EPSG Geodetic Parameter Dataset is a registry/log of spacial reference systems, ellipsoids and datums


## Which codes are for what

EPSG:4326
:   The unique identifier used to reference WGS84

EPSG:3857
:   The identifier used to reference the Web Mercator Projection
:   **EPSG:3857 used to be known as: EPSG:900913 and EPSG:3785 (among other less common identifiers).**

> EPSG:4326 vs SRID:4326<br>For the purposes of what we need to know, these are the same thing.


## Making sense of the jargon
When storing geospatial information in a database to display on a web-based map, you will need to be aware of which reference system your coordinates are based on and the map will be displayed in.


Most coordinates used on the web look something like this: 51.464794, 0.258937. Most modern smartphones, GPS devices and sat-navs also show coordinates in this format.

These coordinates are typically based on a coordinate reference system that uses the WGS84 datum, AKA EPSG:4326.

## How does this relate to building web applications?

Imagine you have built an application in Django that makes use of geospatial functionality and has a database model like this:

<cite>models.py</cite>
```python
from django.db import models
from django.contrib.gis.db import models as geomodels


class Address(models.Model):

    coordinate = geomodels.PointField(geography=True, default='POINT(0.0 0.0)')

```
> This example is using 'geography=True' on its PointField. This is because geography is best suited for displaying locations over a large area. See the Django documentation [here](https://docs.djangoproject.com/en/dev/ref/contrib/gis/model-api/#geography) for more info.

> **Important:** geography=True forces the SRID to be 4326 for this model field

Fantastic! We now have a point field that understands SRID 4326 (WGS84) (the most common for coordinates).

But, how do we now display these SRID/EPSG:4326 coordinates on a Web Mercator Projection (EPSG:3785) mapping service such as Google Maps?

## Displaying EPSG:4326 coordinates on an EPSG:3857 map

Luckily for us, this is as simple as can be!

Google Maps, Mapbox, OpenStreetMap etc. all support adding 4326 coordinates directly, no conversion required. In fact, 4326 is the format that they expect, it is the default and doesn't need configuring at all.

Simply use your coordinates as they are, it does it all for you!

Even GeoJSON is 4326 by default.

## Using QGIS, ArcGIS etc.

When using applications like QGIS to model your geospatial data, you might be trying to overlay your coordinates over map tiles provided by Google Maps (or similar Web Mercator providers).

In these circumstances you will need to tell QGIS that you are using a different coordinate reference system for your map vs your coordinates.

You should define your coordinates as using EPSG:4326, and your map tiles as using EPSG:3857

## Other handy identifiers

<style>
    table{
        width: 100%
    }
</style>

| Name                          | EPSG Code             | Details                                               |
| -------------                 |-------------:         | -----                                                 |
| Web (Pseudo) Mercator         |3857                   | CRS: WGS84 (EPSG:4326)<br>Ellipsoid: WGS84 Ellipsoid (EPSG:7030)<br>Meridian: Greenwich (EPSG:8901)<br>Datum: World Geodetic System 1984 (EPSG:6326)                                                                                                     |
| British National Grid         |27700                  | CRS: OSGB 1936 (EPSG:4277)<br>Ellipsoid: Airy 1830 (EPSG:7001)<br>Meridian: Greenwich (EPSG:8901)<br>Datum: OSGB 1936 (EPSG:6277)                                  |
