---
layout: post
title: Tile Map Service for a CarPC
---

Second part of blog on open source CarPC, focused on running local TMS
server to provide satellite imagery layer in navigation GUI.
Here was [first part](/2017/12/15/Raster-navigation.html) on open source
raster navigation suite.

## Definition of TMS

Tile Map Service is a primitive way to serve raster map data over HTTP.
Most public map services like Google Maps use it. A very short description
of the protocol:

* Earth is projected on a cylinder using
  [Mercator projection](https://en.wikipedia.org/wiki/Mercator_projection).
* Cylinder is unrolled into a square image. As usually with Mercator, areas
  south of 82°S and north of 82°N are lost.
* Map at zoom level **0** is a single
  <span style="text-decoration:underline;">tile</span>:
  a square 256x256 pixel image of the whole Earth.
* Map at zoom level **N** is four times bigger than map at zoom level **(N-1)**,
  meaning that each tile of level **(N-1)** is divided into four 256x256 tiles.
* Each tile within a zoom level Z can be addressed by its X and Y coordinates.

A better understanding of how TMS works might be required to fully
grok this blog, so you may start with 
[Wikipedia article on TMS](https://en.wikipedia.org/wiki/Tiled_web_map)
and follow links from there.

### So many nuances

There is no standard on how to encode X, Y and zoom into an URL.
For example, [OpenStreetMap](http://osm.org) own tiles have this template:
```
http://tile.openstreetmap.org/{zoom}/{x}/{y}.png
```
Which probably matches filesystem layout of the server.
[Google](http://maps.google.com) encodes these variables as URI parameters.
It will also require its internal API version as a parameter and a special cookie:
```
http://khm{n}.google.com/kh/v={ver}&x={x}&y={y}&z={zoom}&s={cookie}
```
[Bing](https://www.bing.com/maps)
encodes all three variables into a single string, that will define a filename:
```
http://a0.ortho.tiles.virtualearth.net/tiles/a{hash}.jpeg?g={ver}
```

There is no standard of what kind of file will be served in response.
It could be JPEG, PNG or GIF.
Server shall define that in Content-Type header.

More devil is in the detail.
There is no general agreement on where the tile (0,0) is.
There are at least three ways of defining tile (0,0):
* Leftmost and uppermost tile. This "standard" definitely was created by
  people whose background is computer graphics.
* Leftmost and downmost tile. That's what mathematicians prefer.
* Tile to the right and up to the very center of the mosaic, where
  geographical coordinates 0°N, 0°E are. True geographer was here!

The worst thing is that what a TMS assumes the Earth to be.
The basic assumption is that Earth is *a sphere* and we project it
on a cylinder, that touches equator.
In this case X = f(Longitude) would be a basic linear function,
and Y = f(Latitude) would be rather a simple function:
![y=f(Lat) image][f1]{: .container }
This assumption is what most popular services Google and Bing do.
However, there are services that take an assumption of the Earth shape
that is closer to reality - *an ellipsoid*.
Russian TMS services [Yandex](http://yandex.ru/maps/) and [ScanEx](http://kosmosnimki.ru/)
provide better satellite imagery of Russia and adjacent countries than global companies do,
but these two services use ellipsoid model of the Earth:
![y=f(Lat) image][f2]{: .container }
where R is Earth's radius and E its eccentricity.
This yields in less longitudal distortion of images, barely noticable, but in more
headache in TMS layer implementation.

## TMS and JOSM

Java OpenStreetMap editor can work with any TMS natively, if that TMS assumes
the same projection and location of (0,0) tile as Google does, and has a
straightforward way to insert zoom level and X,Y coordinates into URL
template.
JOSM can also do ellipsoid TMS if explicitely told to use ScanEx data,
I added this back in 2011.

Thus, if we got a TMS mosaic on local filesystem, with help of a web
server we can serve it directly to JOSM processing only URIs, but not
processing any map data.
Since we control the web server, we can adjust the URI, matching actual
layout of mosaic cache, for example rewrite zoom/x/y to zoom/y/x using
simple nginx rewrite rules.
We can recalculate the position of (0,0) tile with help of perl embedded in nginx.
This all gives us some flexibility.
We can reuse unmodified tile caches created by different tools that
download maps from the Internet, for example 
[SAS Planet software tool](https://www.openhub.net/p/sasplanet).
This is an important feature, since it allows our precious "open-source DIY navigation"
share map data with people who use other software.

## Downloading, caching and managing TMS data

Let's list the goals I want to achieve here:

* Download areas of arbitrary shape in a scripted manner
* Filter out areas of arbitrary shape out of downloaded data
* Mix several downloaded areas into one cache
* Rapidly copy data from desktop/server to the CarPC
* Keep versioning of downloaded data
* If online, be able to fetch from the Internet and store data in the cache

### Storing

For every service I use, I'm going to create a directory to store the tile
cache in zoom/x/y file hierarchy:
```
>ls /maps/tms/
bing google yandex
>ls /maps/tms/bing/
10      12      14      16      2       4       6       8
11      13      15      17      3       5       7       9
```

As you already noticed, with TMS number of files per zoom grows exponentially.
As an example, for my last journey I have had 23 millions tiles, 360 Gbytes of data.
Copying 23 million files from my desktop to the CarPC over SSH, NFS or SAMBA will take days.
Not to mention that CarPC runs slow Intel Atom processor.
This is a real problem in TMS management, you may read
[a discussion of SAS Planet users forum](http://www.sasgis.org/forum/viewtopic.php?f=3&t=540&start=0)
about this.

Here ZFS comes to the rescue!
Every TMS cache directory is a separate ZFS filesystem:
```
>zfs list -r zroot/maps/tms
NAME                           USED  AVAIL  REFER  MOUNTPOINT
zroot/maps/tms                 386G  11,0T   140K  /maps/tms
zroot/maps/tms/bing            359G  11,0T   358G  /maps/tms/bing
zroot/maps/tms/google         27,1G  11,0T  27,1G  /maps/tms/google
```

ZFS filesystems can be sent via **zfs send/receive**, which works at block level and doesn't
depend on number of files.
Speed is limited by 1 Gbit/s network card of the CarPC.

Whenever I download some area, I take snapshot of the filesystem, and record what does
each snapshot contain:

```
>zfs list -t snapshot -r zroot/maps/tms
NAME                            USED  AVAIL  REFER  MOUNTPOINT
zroot/maps/tms/bing@22.07.2015  1,48G      -   221G  -
zroot/maps/tms/bing@17.06.2017  11,6K      -   358G  -
>cat /maps/tms/bing/Notes 
22.07.2015 - Mongolia/2015.poly
17.06.2017 - Mongolia/2017.poly
```

If CarPC already got older snapshot, I can update to new one with incremental **zfs send**.
Now updating TMS data on CarPC takes only a few minutes.

### Downloading

First, I've written a library in Perl that supports projecting,
addressing and URL generation for every TMS service I use.
With help of
[HTTP::Async](http://search.cpan.org/~kaoru/HTTP-Async/)
module I created a function that will fetch a rectangular boundary box with given
geographic coordinates to given zoom level at a decent speed, paralleling several
HTTP connections.

However, areas of interest usually are not rectangular, but of an arbitrary shape.
Fetching minimal rectangle that would have my oddly shaped area inscribed is
a waste of time and disk space.
So, next with help of
[Math::Polygon](http://search.cpan.org/~markov/Math-Polygon/)
I've written a more useful function, that would fetch minimum tiles required
to cover an arbitrary shaped polygon at given zoom level.

Although I travel to different places, distant from each other, I'm storing
all data that originated from a given TMS service in the same directory.
Sometimes I need to filter out and delete or extract some area out of my
collection. This calls for yet another script similar to previous one,
but basicly working with existing cache rather than fetching from Internet.

All this pile of ugly code
[lives here](https://github.com/glebius/map-scripts/tree/master/tms).

To define a polygon of course I don't type out geographic coordinates out
of my head.
I use JOSM with base OSM layer, and click around the area of interest:

[ ![JOSM with a polygon defined](/assets/posts/TMS/osm-poly.png) ](/assets/posts/TMS/osm-poly.png)

Then current layer is saved and converted to a .poly file that
[Math::Polygon](http://search.cpan.org/~markov/Math-Polygon/),
and thus my downloading script understand:
```
> osm2poly.pl 2017.poly.osm > 2017.poly
> fetch-poly.pl bing /maps/tms/bing 1 17 2017.poly
```

### How deep should we zoom when downloading?

In the code snippet above I specified downloading from zoom level 1 to 17.
Why?
At zoom level 17 the whole Earth is an image cut into 2<sup>17</sup>x2<sup>17</sup>
tiles, which means 33554432x33554432 pixels.
Since equator length is roughly 40,075,000 meters, at zoom level 17 one pixel
represents roughly 1.2 meters of actual Earth surface.
This allows to clearly identify edges of a forest, small creeks, individual
trees, changes in type of vegetation and of course roads, even single tracks.
Higher zoom levels are called sub-meter resolution and allow to identify
cars and humans.
Usually sub-meter imagery is available only in densely populated areas.
We aren't interested in them.
In areas where sub-meter resolution is not available zoom levels 18 and 19 still
exist, but they are just image at level 17 resized, not bringing any new information.

### Caching

During a journey I may find myself in an area that wasn't downloaded but
has mobile internet.
In this case I'd like to fetch tiles from the Internet, but cache
them aggresively.
To achieve that, I will access any TMS via local nginx, that can be configured
to first try local storage, and in case of no data there, it will proxy me
to the original resource, of course caching the result.
So, files pre-downloaded at home by the script and files downloaded on my way
by JOSM itself would share the file cache hierarchy.

## Few conclusions

* TMS requires some amount of projection knowledge from UI and little from server.
* TMS is fast to serve, since no image processing is done neither by server,
  nor by the navigation UI.
* Fast to serve doesn't come for free. There is a lot of data duplication
  in the storage.
  Any zoom layer N consumes roughly 1/4 of space consumed by (N+1)
  while storing the same data resized.
* Even if you got plenty of space, TMS is a filesystem hog.
  Its enormous number of files makes it hard to handle.
  But ZFS mitigates the problem.
* TMS allows comparatively easy way to download from public services which are
  not supposed to be used for bulk download.
* Since most of publicly available satellite imagery is distributed via TMS,
  we have little choice left.

[f1]: http://latex.codecogs.com/gif.latex?%5Cbg_black%20y%20%3D%20log%28%5Cfrac%7B1%20&plus;%20sin%28Lat%29%7D%7B1%20-%20sin%28Lat%29%7D%29%5Ccdot%20f%28Zoom%29
[f2]: http://latex.codecogs.com/gif.latex?%5Cbg_black%20y%20%3D%20R%5Ccdot%20log%28%5Cfrac%7Btan%28%5Cpi/4%20&plus;%20Lat/2%29%7D%7B%28tan%28%5Cpi%20/4%20&plus;%20asin%28E%5Ccdot%20sin%28Lat%29%29/2%29%29%5E%7BE%7D%7D%29%5Ccdot%20f%28Zoom%29
