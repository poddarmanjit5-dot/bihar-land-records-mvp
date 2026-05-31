# Database Schema & PostGIS Guide

## Overview

The MVP uses PostgreSQL with PostGIS extension for:
- Storing parcel geometries (polygon boundaries)
- Point-in-polygon queries (core feature)
- Spatial indexing for performance

## Tables

### `parcels` (Main Table)

Stores all land parcel information and geometries.

```sql
CREATE TABLE parcels (
  id SERIAL PRIMARY KEY,
  district_id INTEGER REFERENCES districts(id),
  plot_no VARCHAR(50),          -- Plot/Dag number (e.g., "3244")
  dag_no VARCHAR(50),           -- Dag number
  khata_no VARCHAR(50),         -- Khata/Account number
  area DECIMAL(15,4),           -- Area in square units
  land_type VARCHAR(100),       -- e.g., "Agricultural", "Residential"
  district VARCHAR(100),        -- District name
  mauza VARCHAR(100),           -- Revenue subdivision
  sheet_no VARCHAR(50),         -- Survey sheet number
  geometry GEOMETRY(POLYGON, 4326),  -- Parcel boundary (WGS84)
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

**Key Field:** `geometry` - A polygon representing the parcel boundary in latitude/longitude.

**Coordinate System:** EPSG:4326 (WGS84) - Same as Google Maps

### `land_records` (Ownership Info)

Stores owner and land record details.

```sql
CREATE TABLE land_records (
  id SERIAL PRIMARY KEY,
  parcel_id INTEGER REFERENCES parcels(id),
  owner_name VARCHAR(255),
  father_name VARCHAR(255),
  owner_address TEXT,
  ownership_type VARCHAR(50),   -- e.g., "Individual", "Joint", "Trust"
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

### `mutations` (History of Ownership Changes)

Tracks ownership changes over time.

```sql
CREATE TABLE mutations (
  id SERIAL PRIMARY KEY,
  parcel_id INTEGER REFERENCES parcels(id),
  land_record_id INTEGER REFERENCES land_records(id),
  mutation_type VARCHAR(50),    -- e.g., "Inheritance", "Sale", "Gift"
  mutation_date DATE,
  previous_owner VARCHAR(255),
  current_owner VARCHAR(255),
  document_no VARCHAR(100),     -- Government document reference
  created_at TIMESTAMP DEFAULT NOW()
);
```

### `districts` (Reference Table)

List of districts for filtering.

```sql
CREATE TABLE districts (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) UNIQUE NOT NULL,
  code VARCHAR(10) UNIQUE,
  created_at TIMESTAMP DEFAULT NOW()
);
```

---

## Spatial Indexes

**GIST Index (for fast point-in-polygon):**
```sql
CREATE INDEX idx_parcels_geometry ON parcels USING GIST(geometry);
```

This makes point-in-polygon queries fast even with millions of parcels.

**Regular Indexes (for text search):**
```sql
CREATE INDEX idx_parcels_plot_no ON parcels(plot_no);
CREATE INDEX idx_parcels_khata_no ON parcels(khata_no);
CREATE INDEX idx_parcels_district ON parcels(district);
```

---

## Key PostGIS Functions

### 1. Point-in-Polygon (Most Important)

```sql
SELECT * FROM parcels
WHERE ST_Contains(
  geometry,
  ST_SetSRID(ST_Point(85.1376, 25.5941), 4326)
)
```

**What it does:**
- `ST_Point(lon, lat)` - Create a point from coordinates
- `ST_SetSRID(..., 4326)` - Set coordinate system to WGS84
- `ST_Contains(polygon, point)` - Check if polygon contains point
- **Result:** Returns the parcel containing that coordinate

### 2. Convert Geometry to GeoJSON

```sql
SELECT ST_AsGeoJSON(geometry) FROM parcels
```

**Output:**
```json
{
  "type": "Polygon",
  "coordinates": [[[85.1234, 25.5678], [85.1235, 25.5678], ...]]
}
```

### 3. Calculate Area

```sql
SELECT ST_Area(geometry::geography) / 10000 as area_hectares
FROM parcels
```

### 4. Distance Between Points

```sql
SELECT ST_Distance(
  ST_SetSRID(ST_Point(85.1, 25.5), 4326)::geography,
  geometry::geography
) as distance_meters
FROM parcels
```

---

## Data Import (Shapefile to PostGIS)

### Step 1: Download Shapefile from BhuNaksha

BhuNaksha provides shapefiles in the format:
- `parcels.shp` - Polygon geometries
- `parcels.dbf` - Attribute data (plot_no, khata_no, etc.)
- `parcels.shx` - Shape index
- `parcels.prj` - Projection info

### Step 2: Convert Shapefile to PostgreSQL

Using GDAL (open-source GIS tool):

```bash
ogr2ogr -f "PostgreSQL" \
  PG:"dbname=bihar_land_records user=postgres password=postgres" \
  parcels.shp \
  -nln parcels \
  -overwrite
```

### Step 3: Add PostGIS Index

```sql
CREATE INDEX idx_parcels_geometry ON parcels USING GIST(geometry);
```

---

## Setup Instructions

### 1. Install PostgreSQL + PostGIS

**macOS:**
```bash
brew install postgresql postgis
brew services start postgresql
```

**Linux (Ubuntu):**
```bash
sudo apt-get install postgresql postgresql-contrib postgis
sudo service postgresql start
```

**Docker (Recommended):**
```bash
docker run -d \
  --name bihar_postgres \
  -e POSTGRES_DB=bihar_land_records \
  -e POSTGRES_PASSWORD=postgres \
  -p 5432:5432 \
  postgis/postgis:15-3.3
```

### 2. Create Database & Enable PostGIS

```bash
psql -U postgres

CREATE DATABASE bihar_land_records;
\c bihar_land_records

CREATE EXTENSION postgis;
CREATE EXTENSION postgis_topology;

\d  # List tables
```

### 3. Run Schema Migration

```bash
psql -U postgres -d bihar_land_records -f backend/migrations/001_initial_schema.sql
```

### 4. Verify Setup

```sql
SELECT PostGIS_Version();
-- Should return something like: POSTGIS="3.3.0" ...

SELECT * FROM parcels LIMIT 1;
-- Should return empty (no data yet)
```

---

## Query Performance Tips

1. **Always use GIST index for spatial queries** - PostGIS automatically uses indexes
2. **Filter by district first** - Reduces polygon comparisons
3. **Use LIMIT** - Don't return all matches at once
4. **Analyze queries** - Use `EXPLAIN ANALYZE` to check index usage

Example optimized query:
```sql
EXPLAIN ANALYZE
SELECT * FROM parcels
WHERE district = 'SAHARSA'
  AND ST_Contains(geometry, ST_SetSRID(ST_Point(85.1376, 25.5941), 4326))
LIMIT 1;
```

---

## Backups

```bash
# Backup database
pg_dump bihar_land_records > backup.sql

# Restore database
psql bihar_land_records < backup.sql
```
