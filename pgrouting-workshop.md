---
layout: page
title: pgRouting Workshop
permalink: /pgrouting-workshop/
---

This long page will contain the workshop material for FOSS4G UK.

**Introduction and overview**

We'll be using the OSGeo Live desktop to import Ordnance Survey Open Roads data into a PostGIS database. Once loaded and configured the data will be built into a network topology for use with pgRouting.  With a working network we'll explore the different routing solutions with some use cases.

**Initial setup**

Have you got a DVD drive on your laptop?

* Yes: get one of the OSGeo Live DVDs at FOSS4G UK
* No: download either the [VM image](https://sourceforge.net/projects/osgeo-live/files/9.5/osgeo-live-vm-9.5.7z/download) for VirtualBox or the ISO ([32bit](https://sourceforge.net/projects/osgeo-live/files/9.5/osgeo-live-9.5-i386.iso/download) or [64bit](https://sourceforge.net/projects/osgeo-live/files/9.5/osgeo-live-9.5-amd64.iso/download)) for a USB drive.

**Previewing data**

Download the sample data from [Dropbox Link](#)

Open QGIS and drag shapefiles onto the canvas

**Loading data**

Open Processing toolbox (`Processing > Toolbox`) and search for `PostGIS`.  Find the `Import Vector into PostGIS database (available connections)` tool and open it.  Select the PostGIS connection `connex name` and choose the sotn_roadlink layer as input layer.  The output geometry type is `LineString` and the output CRS is `EPSG:27700`.  Set the schema to `schema name` and set the table name to something reasonable like `or_roadlinks`. I like my primary key to be named `gid` and geometry column to be called `geometry`.  Leave all other options as default but uncheck the `Promote to Multipart` option.  Note the OGR command in the box at the bottom - this could be copied into a batch file or shell script and reused.

**Preparing data**

Once data loaded to the database add some fields for pgRouting.

Populate fields with values suitable for network.

**Configuring data**

Build the network, check the network

**Routing!**

Using QGIS and PgAdmin explore some of the available routing algorithms

Djikstra

Driving Distance

Alphashape
