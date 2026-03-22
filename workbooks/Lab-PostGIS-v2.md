# Lab - PostGIS

**Dataset:** `aquafarms` — PSA registry data. This dataset contains aquafarm categories (e.g., Fishpond, Fish Cage, Fish Pen, Oyster Farm, Seaweed Farm) along with their counts (`Total_Aqua`) and area (`Area_Aquaf`) across Region I, Region II, and Region III.

**Tools:** PostgreSQL 16 / PostGIS 3, pgAdmin 4, QGIS (DB Manager), Docker Compose, `ogr2ogr` / `shp2pgsql`.

**Problem framing:** A colleague asks: "Which provinces have the highest concentration of aquafarms?". In ArcGIS Online, this would require a join on a hosted feature layer, then a summary statistics tool, then a new hosted layer. In PostGIS, it is one query.

## 1. Exercises for Builders
**Objective:**
*   Deploy a fresh PostGIS instance using the provided `docker-compose.yml`.
*   Bulk-load the aquafarm dataset using `ogr2ogr`. 
*   Verify the table exists with `\dt` and `SELECT count(*)`. 
*   Create a spatial index on the geometry column (`wkb_geometry`) to speed up future spatial queries.
*   Take a `pg_dump` backup.

## 2. Exercises for Managers
**Objective:**
*   **Schema & Role Creation:** Using pgAdmin connected to the Builders' instance, create a schema named `restricted` and a schema named `public_data` [8]. Create three roles: `afcd_reader` (SELECT on public_data), `afcd_editor` (INSERT/UPDATE on restricted), and `afcd_admin` (all). 
*   **Access Verification:** Assign roles to test users and verify that the `reader` role cannot see the `restricted` schema.
*   **Row-Level Security (RLS) Policy:** Apply Row-Level Security to the restricted schema. Create a policy where users can only edit or view records that match their assigned region (e.g., users assigned to Region III can only see records where `Region = 'Region III (Central Luzon)'`).

## 3. Exercises for Users
**Objective:**
*   **Database Connection:** Connect QGIS to the PostGIS database and load the `aquafarms` layer. 
*   **Baseline Task:** Open DB Manager. Write a query that sums the `Total_Aqua` for each province, returning the top provinces by count, and includes the `Province` and `Region` names. Export the result as a styled QGIS map of aquafarm density by province.
*   **Stretch Task:** Modify the query to compute the total aquafarm area per province (using `ST_Area` and `ST_Transform`) and add a column for average farm size. 
*   **Error Analysis Variant:** If a naive area calculation using `ST_Area(wkb_geometry)` returns extremely small, almost-zero decimal numbers instead of realistic square meters, diagnose the problem. **Common pitfall:** Your layer is likely in a geographic Coordinate Reference System (CRS) measured in degrees rather than meters. Check the CRS of your table using `SELECT ST_SRID(wkb_geometry) FROM restricted.aquafarms LIMIT 1;`. To fix it, you must dynamically project the geometry to a metric CRS in your query using the `ST_Transform` function (e.g., `ST_Area(ST_Transform(wkb_geometry, 32651))`) before calculating the area.
* Connect QGIS to a live PostGIS database; query layers with `ST_Intersects` and `ST_Buffer`; interpret spatial query results; export a filtered layer to `GeoPackage`.

# Code Reference Guide

Below are the exact SQL codes for the exercises, configured specifically for your `aquafarms` table.

## 1. Exercises for Builders

### Verification and Spatial Index.
After loading the dataset using `ogr2ogr`, Builders must verify the table and create a GIST spatial index on the geometry column (`wkb_geometry`).

```sql
-- Verify the table loaded correctly
SELECT count(*) FROM aquafarms;

-- Create a GIST spatial index on the geometry column
CREATE INDEX aquafarms_geom_idx 
ON aquafarms 
USING GIST (wkb_geometry);

### Take a `pgdump` backup

**Step 1: Find your exact container name.** Run this command in PowerShell:
```sql
docker ps

Look at the column on the far right called **NAME**. Your container will likely be named something like `postgres`, `postgis`, or  `ri-rii-riii-aquafarm-pgadmin-1`.

**Step 2: Create the backup inside the container.** Run the pg_dump command using the exact name you found in Step 1. We will use the `-f` flag to save the file inside the container's temporary folder first, which avoids the PowerShell < and > errors entirely:

```sql
docker exec -t ri-rii-riii-aquafarm-db-1 pg_dump -U postgres -d gisdb -F t -f /tmp/afcd_backup.tar

**Step 3: Copy the file to your Windows folder Now**, use the Docker copy command (cp) to pull that safely generated `.tar` file out of the container and into your current working directory folder:
```sql
docker cp ri-rii-riii-aquafarm-db-1:/tmp/afcd_backup.tar .\afcd_backup.tar

You should now see the `afcd_backup.tar` file in your folder, ready for the Managers group to use.

## 2. Exercises for Managers

### Schemas, Roles, and Row-Level Security
Managers create distinct access levels and apply Row-Level Security (RLS) to restrict regional data access.

```sql

-- Create schemas for data separation
CREATE SCHEMA restricted;
CREATE SCHEMA public_data;

-- Create role groups
CREATE ROLE afcd_reader;
CREATE ROLE afcd_editor;
CREATE ROLE afcd_admin;

-- Grant basic schema usage
GRANT USAGE ON SCHEMA public_data TO afcd_reader;
GRANT ALL ON SCHEMA restricted TO afcd_editor;

-- Move the table from the public schema to the restricted schema
ALTER TABLE public.aquafarms SET SCHEMA restricted;

-- Apply Row-Level Security (RLS) on the table
ALTER TABLE restricted.aquafarms ENABLE ROW LEVEL SECURITY;

-- Create a policy ensuring users can only see/edit records matching their assigned region
CREATE POLICY region_policy 
ON restricted.aquafarms
FOR ALL
USING (Region = 'Region III (Central Luzon)');

## 3. Exercises for Users

### Baseline Aggregation Query
In QGIS DB Manager, Users need to find out which provinces have the highest concentration of aquafarms. (Note: Run this without the restricted. prefix if you are doing this step before the Managers move the table).
```sql
-- Aggregate total aquafarms by Province and Region
SELECT 
    Province,
    Region,
    SUM(Total_Aqua) AS aquafarm_count
FROM 
    restricted.aquafarms
GROUP BY 
    Province, Region
ORDER BY 
    aquafarm_count DESC
LIMIT 10;

### Stretch Task `ST_Transform` & `ST_Area`
This query computes the total area of the aquafarms dynamically using PostGIS spatial functions. It reprojects the geometry to a projected coordinate system (e.g., EPSG:32651) to calculate the area in square meters, then converts it to hectares.
```sql
-- Dynamically calculate area in hectares using PostGIS functions
SELECT 
    Province,
    SUM(ST_Area(ST_Transform(wkb_geometry, 32651)) / 10000) AS area_ha
FROM 
    restricted.aquafarms
GROUP BY 
    Province
ORDER BY 
    area_ha DESC;

### Error Diagnosis (Handling CRS Mismatches)
If a user runs an area calculation and gets microscopic numbers instead of realistic square meters, they should check the table's Coordinate Reference System (CRS).
```sql
-- Check the SRID (Spatial Reference Identifier) of your dataset. 
-- If it returns 4326 (WGS84), the data is in degrees, not meters!
SELECT ST_SRID(wkb_geometry) 
FROM restricted.aquafarms 
LIMIT 1;

-- This is the naive (broken) calculation that will return microscopic results:
SELECT 
    Province,
    SUM(ST_Area(wkb_geometry)) AS broken_area
FROM 
    restricted.aquafarms
GROUP BY 
    Province;

### Stretch Task Area Calculation
Perform the stretch task area calculation using QGIS DB Manager to dynamically calculate the physical area of the aquafarm geometries in hectares. Even though your dataset comes with a pre-existing Area_Aquaf column, the goal of this exercise is to learn how to compute spatial areas on the fly using PostGIS functions and avoid a common coordinate system error.

```sql
SELECT 
    Province,
    SUM(ST_Area(ST_Transform(wkb_geometry, 32651)) / 10000) AS area_ha
FROM 
    restricted.aquafarms
GROUP BY 
    Province
ORDER BY 
    area_ha DESC;

### Calculate the average farm size in hectares

To calculate the average farm size in hectares, you need to complete the second half of the Stretch Task, which asks you to add an **avg_farm_size_ha** column.

The mathematical logic is straightforward: **Average Farm Size = Total Area in Hectares / Total Number of Farms**.

While your dataset does have a pre-existing Area_Aquaf column, if you look closely at the raw tabular data, some records have an **asterisk (*)** instead of a number for their area (for example, Oyster Farms in Ilocos Norte or Fish Cages in Nueva Vizcaya). Because of these missing or suppressed values, relying on the pre-calculated column could result in errors.

Instead, you can combine the dynamic spatial area calculation we just built with the Total_Aqua (farm count) column to get a perfectly accurate, on-the-fly average.

Here is the complete SQL query to calculate the count, total area, and average farm size per province:

```sql
SELECT 
    Province,
    -- 1. Total number of aquafarms
    SUM(Total_Aqua) AS aquafarm_count,
    
    -- 2. Total area in hectares dynamically calculated
    SUM(ST_Area(ST_Transform(wkb_geometry, 32651)) / 10000) AS total_area_ha,
    
    -- 3. Average farm size in hectares (Total Area / Total Farms)
    SUM(ST_Area(ST_Transform(wkb_geometry, 32651)) / 10000) / SUM(Total_Aqua) AS avg_farm_size_ha
FROM 
    restricted.aquafarms
GROUP BY 
    Province
ORDER BY 
    avg_farm_size_ha DESC;

### Filter the results by specific aquafarm type

To filter your results by a specific aquafarm type, you need to add a `WHERE` clause to your SQL query.

Based on your dataset, the specific column that stores the farm type is named Aquafarm. The available categories you can filter by include **'Fishpond'**, **'Fish Cage'**, **'Fish Pen'**, **'Oyster Farm'**, **'Mussel Farm'**, **'Seaweed Farm'**, and **'Others'**.

Here is how you modify your baseline aggregation query to filter for only one specific type—for example, Fishponds:

```sql
SELECT 
    Province,
    Region,
    SUM(Total_Aqua) AS aquafarm_count
FROM 
    restricted.aquafarms
WHERE 
    Aquafarm = 'Fishpond'
GROUP BY 
    Province, Region
ORDER BY 
    aquafarm_count DESC
LIMIT 10;

### `ST_Intersects` & `ST_Buffer`

**Objective:** Connect QGIS to a live PostGIS database; query layers with **ST_Intersects** and **ST_Buffer**; interpret spatial query results; export a filtered layer to **GeoPackage**.

**Problem Framing:** A severe weather event (like a localized typhoon) is forecasted to hit a specific rectangular area in Central Luzon. The PSA-AFCD director wants to draw a coordinate "bounding box" to represent this storm path, find out which provinces intersect this box, and draw a 5-kilometer "peripheral hazard buffer" around those affected provinces to plan for supply chain disruptions.

#### Database Connection

**Action:** Open QGIS and connect to the training PostGIS instance using the pre-configured credentials.

**Steps:** Go to Layer → Add PostGIS Layer. Load the restricted.aquafarms layer into your QGIS map canvas.

#### Spatial Query (`ST_Intersects` with a Bounding Box & `ST_Buffer`)

**Action:** Open the DB Manager in QGIS. Write a query that generates a dummy bounding box for the storm, finds provinces that intersect it, and calculates a 5km buffer around them.

**Steps:** Paste and execute the following SQL code. How it works: We use ST_MakeEnvelope to draw a spatial box using Longitude/Latitude coordinates. We use ST_Intersects to filter only the provinces hit by that box. Finally, we use ST_Buffer (combined with ST_Transform to fix the CRS) to draw the 5,000-meter hazard zone.

```sql
-- Find provinces hit by a storm bounding box, sum their farms, and buffer them by 5km
SELECT 
    Province,
    Region,
    SUM(Total_Aqua) AS total_affected_farms,
    -- Create a 5km (5000m) buffer around the province geometry
    ST_Buffer(ST_Transform(wkb_geometry, 32651), 5000) AS hazard_buffer_geom
FROM 
    restricted.aquafarms
WHERE 
    -- Intersect the data with a bounding box (Longitude Min, Latitude Min, Longitude Max, Latitude Max, SRID)
    ST_Intersects(
        wkb_geometry, 
        ST_MakeEnvelope(119.5, 14.5, 121.0, 15.5, 4326)
    )
GROUP BY 
    Province, Region, wkb_geometry
ORDER BY 
    total_affected_farms DESC;

### Interpret Spatial Query Results

**Action:** Review the resulting data table in the DB Manager.

**Questions to answer:**
1. Based on the query results, which provinces fall inside the coordinates of our hypothetical storm path?
2. Which of these affected provinces has the highest total_affected_farms at risk?
3. Notice how we used ST_Transform before buffering. What would happen if we just ran ST_Buffer(wkb_geometry, 5000) without transforming it first? (Hint: Look back at the Error Analysis Variant regarding geographic versus metric coordinate systems!)

###  Export a Filtered Layer to GeoPackage

**Action:** Take the 5km buffer results and export them as a portable file format for the field response team to use offline.

**Steps:**
1. In the DB Manager, beneath your query results, check the box for Load as new layer.
2. Select hazard_buffer_geom as the Geometry column and set an appropriate Layer name (e.g., typhoon_5km_hazard_zones). Click Load.
3. The buffered hazard zones will now appear in your main QGIS Layers panel as large, rounded province shapes.
4. Right-click the newly loaded layer in the QGIS Layers panel, select Export → Save Features As...
5. Choose GeoPackage as the format, define a file path, and click OK to generate the export.
