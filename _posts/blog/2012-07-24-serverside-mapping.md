---
layout: default
---
# Server-side mapping

We have several projects that involve processing large geospatial datasets 
(geo-data) and displaying them on maps. These projects present us with some 
interesting technical challenges involving the storage, transfer and processing 
of geo-data. This post outlines some of bigger challenges we have encountered 
and our corresponding solutions.


## The challenge

In the past we have used the [GMap](https://developers.google.com/maps/) and 
[OpenLayers](http://openlayers.org/) libraries and their equivalent Drupal 
modules on our mapping projects. They are effective solutions when you have 
a small or even moderately sized collection of entities containing some simple 
geodata (points, lines, polygons) which you want to present as vector overlays 
on a map. Unfortunately they tend to fall apart fast when you attempt them with 
larger datasets. There are two main reasons for this:

1. Geospatial data can be large. Particularly as we tend to encode it in 
   text-based formats such as WKT or GeoJSON when we are sending it to a web 
   browser. The larger the data, the longer it takes to transfer from server to 
   client.

2. The information being sent is raw data which means that the client needs to 
   parse and process the data before rendering it on the screen. The more data 
   there is, the longer this takes.

Making things worse, the geo-data is often sent at the beginning of the html 
document (via Drupal.settings or similar). Most browsers will wait until they 
have downloaded and parsed this data before they begin to render the rest of 
the page, further emphasizing the delay.

As a result of the above, it doesn't take much have a serious negative impact 
on your page load times and little more to actually crash your visitor's 
browser.


## Heavy lifting server-side

A good solution to these issues is to process and render the geo-data as image 
tiles on the server. Tiles can then be cached and served to the client when 
requested. This way the data is only rendered whenever it is changed instead of 
each time the page is loaded. Bandwidth is also reduced as the image tiles are 
relatively consistent in size regardless of the complexity or amount of data 
used to produce them.

There are several components involved in a server-side tile rendering pipeline.
They can be loosely categorised under storage, rendering, and caching.


## Storage

Geo-data can be stored in a variety of places and formats each with it's own 
advantages. Here are some that are common:

### [ESRI Shapefiles](http://en.wikipedia.org/wiki/Shapefile)

ESRI Shapefiles (commonly known as shapefiles) are a popular file format for 
storing and transfering geo-data. They are comprised of a .shp file and often 
bundled in a zip file with a collection of other files containing related 
information.

### [Well known text (WKT)](http://en.wikipedia.org/wiki/Well-known_text)



### [GeoJSON](http://www.geojson.org/)

GeoJSON is a relatively new format for encoding geospatial data. As it is plain 
JSON and easily parsible in Javascript, it makes for a convenient format to use 
when passing raw data to browser-based clients.

### PostGIS

PostGIS is a spatial database extension to PostgreSQL. The relational database 
gives you the ability to index, query, and manipulate your data with SQL and an 
extensive API of geospatial functions.

In Drupal it's common to store your data in fields attached to entities using 
the [Geofield](http://drupal.org/project/geofield) module. We often do this 
however the data is stored formatted as WKT column of type LONGTEXT which is 
pretty restrictive when compared to PostGIS.

As a result, we have developed [Sync 
PostGIS](http://drupal.org/project/sync_postgis) which allows for site 
developers to specify entity types with attached geofields to have their data 
mirrored in a PostGIS database. The source data in Drupal's main database is 
still kept and is treated as best-copy, but all changes (insert, update and 
delete) are reflected in the PostGIS version. This gives us the ability to 
utilize PostGIS's rich geospatial features on our Drupal-managed geo-data!

## Rendering

Once we have our raw geo-data stored somewhere we need a method to convert it 
into the images that we will display on our maps. Mapnik is an excellent tool 
for the job.

### [Mapnik](http://mapnik.org/)

Mapnik is an open source C++ library designed to generate map images from 
a variety of data sources and configurable style rules. Language bindings are 
available for Python and Javascript (Node.js) as well as an XML-based 
stylesheet format.

### [TileMill](http://mapbox.com/tilemill)

TileMill is a desktop application for creating web maps. It is developed by 
[Development Seed](http://developmentseed.org) to complement their MapBox 
service. Powered by Mapnik and Node.js it allows users to define style rules 
using a CSS-like language called CartoCSS. With each change, the rules and data 
sources are passed to Mapnik and a preview map is rendered giving immediate 
feedback.

TileMill's main output will render tiles and package them in the MBTiles 
format. However it can also be used to generate a Mapnik XML stylesheet which 
can be passed to Mapnik by other applications to render tiles.

MapBox has a great collection of resources to get you up and running with 
TileMill. I recommend starting with their [crash 
course](http://mapbox.com/tilemill/docs/crashcourse/introduction).


## Caching

So far, we have resolved the bandwidth issues discussed at the beginning of 
this post by rendering our data into tiles on the server with Mapnik. This has 
also alieviated the visitor's web browser of the strain of processing large 
amounts of raw geo-data. However generating tiles on the server is also 
a resource-intensive process; depending on the area and zoom levels you wish to 
cover, rendering a set of tiles at once can take anywhere from a few seconds to 
more than a week.

Obviously we don't want to be rendering tiles from scratch with every request. 
Instead it is much more efficient to cache the tiles somewhere after they have 
been rendered and serve requests directly from the cache, only falling back to 
rendering when a cached tile doesn't exist. There are lots of ways to cache 
tiles on your server, here are some methods that we use:

### [MBTiles] (http://mapbox.com/developers/mbtiles/)

MBTiles is a file-format specification pioneered by Development Seed, it is 
essentially a SQLite database containing a whole set of rendered map tiles. 
Known as tilesets, these files are portable and lightweight and can be 
generated by TileMill. They are great for caching base layers or layers 
comprised of data that doesn't change frequently. However they require your 
tiles to be rendered in advance, making them less useful for maps covering 
large areas and zoom levels or data sources are updated often.

### File system

Map tiles are just individual image files, usually 256x256 pixels in dimension 
and rendered in a compressed image format such as .png. In most cases storing 
them directly on your file system is an acceptable solution.

### Memcache

If you are expecting a lot of concurrent requests you may want to avoid the 
file system and cache your tiles in memory. Memcache or similar systems are 
made for this task.


## All together

There are a plenty of options available for tile servers including TileCache, 
TileStache, TileLite and TileStream. We have been using TileStache as it has an 
excellent balance of features and simplicity.

### [TileStache](http://tilestache.org/)

TileStache is a server application that handles requests for tiles and serves 
and caches tiles generated from Mapnik or other rendering providers. It's 
implemented in Python and designed to be extended with a solid plugin system. 

Out of the box it's features include:

- Rendering Mapnik maps
- Serving MBTiles tilesets
- Caching tiles to file system, MBTiles, Memcache or Amazon S3
- Composite 'layers' into single tilesets

The compositing feature in particular is very powerful. In TileStache's 
configuration you define a set of 'layers', each layer being a different 
tileset and effectively it's own map. You can then define composite layers 
which are new tilesets comprised of other layers layered on top of one another. 
This allows you to do things like combine a prerendered tileset stored in an 
MBTiles file with a tileset of features stored in PostGIS and serve them to 
your visitors browser as one flat set of tiles.

## Shifting constraints

The range of tools and techniques described above provides us with plenty of 
flexibility when we are working on our mapping projects. It is all achieved 
without wasting bandwith or bogging down our visitor's machines with redundant 
computation.

While before we had a strict upper-bound on the amount of geo-data we could 
manage and serve based on the limits of the network and our visitor's hardware. 
Now it is only a matter of how much data can we can fit into our maps without 
sacrificing their readability - not a horrible problem to have.
