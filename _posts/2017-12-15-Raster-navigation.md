---
layout: post
title: CarPC raster navigation, the UNIX way
---

A part of a long story how back in 2009 I had built a FreeBSD based
CarPC running only open source software, featuring all required
functionality for my offroad travels.
I am still using this setup, at least once per year.

This part will be focused on raster navigation GUI.
If you know what is raster map and why one may need it,
jump right to [the UNIX way of doing it](#true-unix-way).

### What is a raster map and how it differs from vector map?

Vector map is a set of nodes and ways, it is a graph.
Most mapping devices work with vector data, as it allows to build routes,
properly zoom, rotate map and consumes little space in device memory.
A raster map is basically an image, that is georeferenced.
Georeferencing means defined relation between pixel coordinates
and real latitude and longitude coordinates for several pixels in the
image (at least 3), and also requires a known projection of the image,
since images are flat, and Earth is not.

Detalization of raster maps is measured in meters per pixel.
For example, if you see parked cars in Google satellite images,
then satellite imagery you are looking at features at least
2-3 meters/pixel resolution.

There are two most common types of raster map data:
* **Tiled**. Each zoom level is cut into small rectangular squares, called
  tiles. To fill a certain canvas with map image, several tiles need
  to be merged. Every next zoom level will require 4x more tiles.
  You are probably familiar with this approach as Google satellite
  maps use it.
* **Continuous**. The whole map of a region is a single file, an image of
  enormous size, which would consume many gigabytes of memory to be
  displayed, and won't fit into any display. A certain canvas is filled
  by cutting out a piece from the image.

I will go into more detail about both types in future blog posts.

### When does raster navigation kick in?

Raster navigation becomes necessary when you go truly exploring to a places,
where no vector maps are available, or their detalization is far below
acceptable level.
When you are seeking for roads through taiga, abandoned for decades, for
mountain passes known only to local shepherds and other routes like that.
You probably won't find such a place in the USA, but in Russia, Kazakhstan,
Mongolia and likely many other countries of Asia there are plenty of such
terrain.

### Windows way

Since 1996 there exists an application that is considered a golden standard
for raster navigation.
It is called [OziExplorer](http://www.oziexplorer.com).
Any attempt to write another tool, either intentionally or not mimics
OziExplorer.
Back when I was building my CarPC, there was no Android version
of OziExplorer, neither a good Android device to run as a CarPC.

### UNIX way?

There existed several Linux applications, that tried to achieve same goals
as OziExplorer, for example QLandkarteGT, Viking, etc.
I tried all of them and neither satisfied me.

### true UNIX way

As [defined by Eric Raymond](https://en.wikipedia.org/wiki/Unix_philosophy#Eric_Raymond%E2%80%99s_17_Unix_Rules)
the proper way to do things in UNIX is to have a set of small self-sufficient
applications connected by well defined interfaces.
A raster navigation suite will consist of the following components:
1. GUI application that displays map layers and position cursor
2. Daemon or a library that processes georeferenced data in various formats
   and serves it to the GUI as simple images
3. Daemon that does logging of GPS data, independent of the GUI
4. Daemon that communicates with GPS device and feeds location to the GUI
   and the logger 

#### 1. GUI

Suprisingly for main application I had taken an editor of vector maps, the
[Java OpenStreetMap Editor](https://josm.openstreetmap.de).
Unusual use case, definitely not foreseen by JOSM developers :)
The editor has ability to display raster maps, as a background layers,
to assist with composing the vector OSM map.
It can download these layers from the Internet, be it a *tiled* map,
or *continuous* map.

What if I don't load the OSM vector data at all?
I will load just a raster layer and then I will put on top a layer with
position cursor and current track.
That would create a raster navigation tool!

Looks like I wasn't the first one to come with this idea, because
there already existed
[LiveGPS plugin](https://wiki.openstreetmap.org/wiki/JOSM/Plugins/LiveGPS),
that drawed the position cursor in JOSM.
The plugin appeared to be abandoned, and required some care to get
running.
As I already had committer access to OpenStreetMap subversion, the
hacking began.
Eventually I adopted LiveGPS, added more features in it and took burden
of maintainance, as I seem to be its major (the only) user :)

[ ![JOSM running with LiveGPS](/assets/posts/Raster-navigation/josm-livegps.png) ](/assets/posts/Raster-navigation/josm-livegps.png)

A final dash for the GUI was
[a small plugin](https://svn.openstreetmap.org/applications/editors/josm/plugins/touchscreenhelper/)
that changed left mouse button behavior in JOSM, making it easy to drag
map on a touchscreen.

#### 2. Geo image processing

To some extent JOSM supports both the
[WMS](http://www.opengeospatial.org/standards/wms) protocol used
by professional GIS applications,
as well as the
[TMS](https://wiki.osgeo.org/wiki/Tile_Map_Service_Specification)
protocol, that is used by consumer grade map services, e.g. Google Maps,
Bing Maps, Yandex Maps, etc.
Both protocols are running on top of HTTP.
So, the second component of our suite will be a web application,
running on localhost.

Since complexity and longevity of the topic is out of the scope of
this post, later I will write two separate blog posts:
on running TMS service with
[NGINX](http://www.nginx.org)
rewrite rules, and on using
[mapserver](http://www.nginx.org) to run WMS serving different kinds
of map data.

#### 3. GPX trace logging

My first attempt was to run the *gpxlogger* application from the
[GPSD](http://catb.org/gpsd)
suite and feed its standard output to a file.
Soon I understood that I need more features out of it and started
adding them.
As Eric and rest of GPSD team weren't eager to add them, I forked
the program to my own ***gpxloggerd*** project, and
[hosted it on GitHub](https://github.com/glebius/gpxloggerd).

Always after my travels, I work a lot with my own traces to contribute
data to the OpenStreetMap.
So I wanted them to be as nice and clean as possible.
That's why gpxloggerd has features for suppression of logging
when vehicle stands still, and starting new log file after such
a delay.
Another useful feature is suppression of logging when vehicle
moves in a straight line.
This allows for very nice trace detalization on curves and turns,
without data bloat in the file overall.

![sample trace](/assets/posts/Raster-navigation/trace.png)

Now, suppressing my modesty, I will note that my GPS traces are
the best looking traces I ever used as OSM contributor.

#### 4. Communicating with GPS device

That was the easiest part.
The
[GPSD](http://catb.org/gpsd)
supports many devices, and feeds GPX fix data to multiple clients
via TCP server.
