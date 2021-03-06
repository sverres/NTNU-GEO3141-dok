# PostGIS topology exploration

---


## Test case

- AR5-data from Gjøvik municipality

## Simple features data

- ESRI Shape file

## PostGIS import

- import by means of *PostGIS 2.0 Shapefile and DBF loader exporter*.
- table: *ar5sf* in public schema

## Create PostGIS topology

### Install topology functions in database

```SQL
-- enable topology in PostGIS database
CREATE EXTENSION postgis_topology;
```

### Create topology schema

```SQL
-- create topology schema
SELECT topology.CreateTopology('ar5topo', 25832);
```


### From existing simpe features-geometry, populate topology-tables in topology schema

A new table, *ar5*, function as a integrator table for the topology information

```SQL
-- create a new table in public schema
CREATE TABLE ar5(gid serial primary key, artype smallint);

-- add a topogeometry column to it
-- integer AddTopoGeometryColumn(varchar topology_name, varchar schema_name, varchar table_name, varchar column_name, varchar feature_type);
SELECT topology.AddTopoGeometryColumn('ar5topo', 'public', 'ar5', 'topo', 'MULTIPOLYGON') AS new_layer_id;

-- use new layer id in populating the new topogeometry column
-- we add the topogeoms to the new layer with 0 tolerance
INSERT INTO ar5(artype, topo)
SELECT artype,  topology.toTopoGeom(geom, 'ar5topo', 1)
FROM ar5sf;
```

### Verify and validate

```SQL
-- use to verify what has happened --
SELECT *
FROM topology.TopologySummary('ar5topo');

-- validate topology
SELECT * FROM  topology.ValidateTopology('ar5topo');
```

### Use information in topology tables to extract outer borders

```SQL
-- extract outer edges - unordered, without sequence-info (useless)
CREATE TABLE leftnull AS
SELECT left_face, right_face, edge_id, geom
FROM ar5topo.edge_data
WHERE left_face = 0;

-- create a geometry-less table with sequence number and id's for the edges
CREATE TABLE ar5ring AS
	SELECT * FROM  topology.GetRingEdges('ar5topo', 90);

-- extract outer edges - with sequence-info
CREATE TABLE ar5ringgeom AS
  SELECT public.ar5ring.sequence, ar5topo.edge_data.geom
  FROM public.ar5ring, ar5topo.edge_data
  WHERE left_face = 0
    AND (public.ar5ring.edge = ar5topo.edge_data.edge_id)

-- join linestrings
create table ar5singleline as
SELECT ST_COLLECT(geom order by sequence) as geom
FROM ar5ringgeom
```

### Unfortunately, this linestring is not closed. Polygon creation is then more difficult, but hopefully possible.

```SQL
-- check - answer is 'f'
SELECT ST_ISCLOSED(geom)
FROM ar5singleline
```


## PostGIS topology and some other functions used

Function |Task
-- |--
[CreateTopology](http://postgis.net/docs/manual-dev/CreateTopology.html) | Creates a new topology schema and registers this new schema in the topology.topology table
[AddTopoGeometryColumn](http://postgis.net/docs/manual-dev/AddTopoGeometryColumn.html) |Adds a topogeometry column to an existing table, registers this new column as a layer in topology.layer and returns the new layer_id
[toTopoGeom](http://postgis.net/docs/manual-dev/toTopoGeom.html) |Converts a simple Geometry into a topo geometry
[ValidateTopology](http://postgis.net/docs/manual-dev/ValidateTopology.html) |Returns a set of validatetopology_returntype objects detailing issues with topology
[GetRingEdges](http://postgis.net/docs/manual-dev/GetRingEdges.html) |Returns the ordered set of signed edge identifiers met by walking on an a given edge side
[ST_Collect](http://postgis.net/docs/manual-dev/ST_Collect.html) |Return a specified ST_Geometry value from a collection of other geometries
[ST_IsClosed](http://postgis.net/docs/manual-dev/ST_IsClosed.html) |Returns TRUE if the LINESTRING's start and end points are coincident

## Visuals

*AR5 polygons:*

![AR5 polygons](../images/ar5-polygon.png)


*A closer look on faces, edges and nodes - geometry from ar5topo-schema tables:*

![face-edge-node](../images/face-edge-node.png)


*Outer border extracted by means of topology information:*

![ar5-outer-polygon](../images/ar5-ytre-polygon.png)
