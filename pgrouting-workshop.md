---
layout: page
title: pgRouting Workshop
permalink: /pgrouting-workshop/
---

**Introduction and overview**

We'll be using the OSGeo Live desktop to import Ordnance Survey Open Roads data into a PostGIS database. Once loaded and configured the data will be built into a network topology for use with pgRouting.  With a working network we'll explore the different routing solutions with some use cases.

**Initial setup**

Have you got a DVD drive on your laptop?

* Yes: get one of the OSGeo Live DVDs at FOSS4G UK
* No: download either the [VM image](https://sourceforge.net/projects/osgeo-live/files/9.5/osgeo-live-vm-9.5.7z/download) for VirtualBox or the ISO ([32bit](https://sourceforge.net/projects/osgeo-live/files/9.5/osgeo-live-9.5-i386.iso/download) or [64bit](https://sourceforge.net/projects/osgeo-live/files/9.5/osgeo-live-9.5-amd64.iso/download)) for a USB drive.

Get the data: [Dropbox Link](#)

***

**Step 1: View the data in QGIS**

Open QGIS (Start > Geospatial > Desktop GIS > QGIS)

Browse to data folder

Drag shapefiles onto canvas.

***

**Step 2: Check the PostGIS database**

Open pgAdmin (Start > Geospatial > Databases > pgAdmin III)

Check connections: `local` (user / user)

Check database: `pgrouting`

Check schema: `public`

Check extensions: `pgRouting` and `PostGIS`

***

**Step 3: Load data from QGIS to PostGIS**

Open or switch back to QGIS.

Set the following: 

-	Database connection name: `pgrouting`
-	Input layer: `sotn_road`
-	Output geometry type: `linestring`
-	Output CRS: `EPSG:27700`
-	Schema name: `public`
-	Table name: leave blank to use existing name
-	Primary key: `gid`
-	Geometry column name: `geometry`
-	Uncheck `Promote to Multipart` (pgRouting likes LINESTRING and not MULTILINESTRING)
 
Note the OGR command in the box at the bottom - this could be copied into a batch file or shell script and reused.

Click the Add PostGIS layer button (blue elephant) and connect to the pgRouting database.  Select the `sotn_road` layer and click Add.  Note that it is in EPSG:27700.  The layer will be added to the QGIS canvas and will match the shapefile version already there.  Use the identify tool to select a link and check the attributes.

***

**Step 4: Add the pgRouting fields**

Open or switch to PgAdminIII.

Navigate to the tables in the `public` schema of the `pgRouting` database.  Click the SQL button on the top menu bar to open a SQL editor window.  We’ll be using this to update our sotn_road table with the fields and values that pgRouting needs.

This section is a straightforward copy and paste exercise but we’ll go through it step by step.

4.1 First, add the columns required to the `sotn_road` table:

    ALTER TABLE public.sotn_road
      ADD COLUMN source integer,
      ADD COLUMN target integer,
      ADD COLUMN speed_km integer,
      ADD COLUMN cost_len double precision,
      ADD COLUMN rcost_len double precision,
      ADD COLUMN cost_time double precision,
      ADD COLUMN rcost_time double precision,
      ADD COLUMN x1 double precision,
      ADD COLUMN y1 double precision,
      ADD COLUMN x2 double precision,
      ADD COLUMN y2 double precision,
      ADD COLUMN to_cost double precision,
      ADD COLUMN rule text,
      ADD COLUMN isolated integer;

4.2 Create the required indices on the source and target fields for the fast finding of the start and end of the route.

    CREATE INDEX sotn_road_source_idx ON public.sotn_road USING btree(source);
    CREATE INDEX sotn_road_target_idx ON public.sotn_road USING btree(target);

4.3 Populate the line end coordinate fields.

    UPDATE public.sotn_road
      SET x1 = st_x(st_startpoint(geometry)),
        y1 = st_y(st_startpoint(geometry)),
        x2 = st_x(st_endpoint(geometry)),
        y2 = st_y(st_endpoint(geometry));

4.4 Use the length of the road link as the distance cost.

    UPDATE public.sotn_road
      SET cost_len = ST_Length(geometry),
      rcost_len = ST_Length(geometry);

4.5 Set the average speed depending on road class and the nature of the road using the class and formofway fields.  I have set the "Not Classified" links to have a speed of 1km/h which increases the cost of traversing that link. Most "Not Classified" are paths and private roads and so I have elected to make them less desirable to travel on. Adjust the speeds here as you see fit and note that I have used kilometres per hour and not miles per hour.

    UPDATE public.sotn_road SET speed_km = 
      CASE WHEN class = 'A Road' AND formofway = 'Roundabout' THEN 20
      WHEN class = 'A Road' AND formofway = 'Collapsed Dual Carriageway' THEN 60
      WHEN class = 'A Road' AND formofway = 'Dual Carriageway' THEN 60
      WHEN class = 'A Road' AND formofway = 'Single Carriageway' THEN 55
      WHEN class = 'A Road' AND formofway = 'Slip Road' THEN 55
      WHEN class = 'B Road' AND formofway = 'Single Carriageway' THEN 50
      WHEN class = 'B Road' AND formofway = 'Collapsed Dual Carriageway' THEN 55
      WHEN class = 'B Road' AND formofway = 'Slip Road' THEN 50
      WHEN class = 'B Road' AND formofway = 'Roundabout' THEN 20
      WHEN class = 'B Road' AND formofway = 'Dual Carriageway' THEN 55
      WHEN class = 'Motorway' AND formofway = 'Collapsed Dual Carriageway' THEN 70
      WHEN class = 'Motorway' AND formofway = 'Dual Carriageway' THEN 70
      WHEN class = 'Motorway' AND formofway = 'Roundabout' THEN 20
      WHEN class = 'Motorway' AND formofway = 'Slip Road' THEN 60
      WHEN class = 'Motorway' AND formofway = 'Single Carriageway' THEN 60
      WHEN class = 'Not Classified' AND formofway = 'Roundabout' THEN 1
      WHEN class = 'Not Classified' AND formofway = 'Single Carriageway' THEN 1
      WHEN class = 'Not Classified' AND formofway = 'Slip Road' THEN 1
      WHEN class = 'Not Classified' AND formofway = 'Dual Carriageway' THEN 1
      WHEN class = 'Not Classified' AND formofway = 'Collapsed Dual Carriageway' THEN 1
      WHEN class = 'Unclassified' AND formofway = 'Single Carriageway' THEN 30
      WHEN class = 'Unclassified' AND formofway = 'Dual Carriageway' THEN 40
      WHEN class = 'Unclassified' AND formofway = 'Roundabout' THEN 20
      WHEN class = 'Unclassified' AND formofway = 'Slip Road' THEN 30
      WHEN class = 'Unclassified' AND formofway = 'Collapsed Dual Carriageway' THEN 40
      ELSE 1 END;

4.6 Then use the speed and length to calulate road link travel time.

    UPDATE public.sotn_road
      SET cost_time = ST_Length(geometry)/1000.0/speed_km::numeric*3600.0,
      rcost_time = ST_Length(geometry)/1000.0/speed_km::numeric*3600.0;

It’s worth noting here that OS Open Roads has been designed as a high level road network for quick and dirty routing and as such does not have the extra detail required for turn restrictions and one way streets.  ITN and the new Highways layer have got the road routinginformation (RRI) detail.

***

**Step 5: Building the network**

Now that we have added all the fields and populated them with some reasonable values we can build our network topology.  This will take about a minute.

    SELECT public.pgr_createTopology('public.sotn_road', 0.001, 'geometry', 'gid', 'source', 'target');

It is always a good idea to analyse your network graph as it will highlight potential errors.  There will be some isolated segments, dead ends, potential gaps, intersections and ring geometries.

    SELECT public.pgr_analyzegraph('public.sotn_road', 0.001, 'geometry', 'gid', 'source', 'target');

Before we use the network it’s a good idea to clean up the table after all the additions and changes to it.

    VACUUM ANALYZE VERBOSE public.sotn_road;

Now we are ready to route!

***

**Step 6: Routing in QGIS**

Start or switch to QGIS.

Add the PgRouting Layer plugin through `Plugins > Install and Manage Plugins`

You will need to go to Settings and check the experimental box and then back to the `All` tab to search for “pgRouting”. Check the box next to the plugin and click Install.

The plugin should appear in a docked panel in QGIS.  If it doesn’t, right click on an empty space on the menu bar and check the box in the context menu to add it.

To configure the plugin we need to:

- set the database connection – `pgrouting `
- name the network table – `sotn_road`
- set the unique identifier – `gid` 
- set the source and target fields – `source` and `target`
- set the cost fields – `cost_len` and `rcost_len` (or `cost_time` and `rcost_time`)

Once configured, we can set the routing function to **djikstra** and use the pickers to select the start and end points for our first route.

Try changing the cost fields from distance (`cost_len` / `rcost_len`) to time (`cost_time` / `rcost_time`) and see if the route changes.

Select the **driving distance** function and pick a start node and then a distance (metres) or time (seconds) and see the rash of red nodes that indicate the nodes reachable within the specified cost.

Select the **alphashape** function and use the same start node and cost set above and see the lovely pink result.  The alphashape is like a shrink-wrapped convex hull around the set of points.

***

**Step 7: Routing with SQL in PgAdminIII**

Open or switch back to PgAdminIII.

Navigate to the `pgrouting` database and open a SQL editor window.
