# GIS Implementation Guide

## What is GIS?

**GIS = Geographic Information System**

A system that captures, stores, and analyzes spatial (location-based) data.

In this project, GIS is used to:
1. Store parcel boundaries (polygons)
2. Find which parcel contains a clicked coordinate (point-in-polygon)
3. Display parcel data on a map

---

## Key Concepts

### 1. Coordinates (Lat/Lon)

Every location on Earth can be described by:
- **Latitude:** Distance from equator (-90° to +90°)
- **Longitude:** Distance from prime meridian (-180° to +180°)

Example: Patna, Bihar = (25.5941, 85.1376)

### 2. Coordinate Systems (CRS / EPSG)

Different ways to represent Earth coordinates on a flat map.

**EPSG:4326 (WGS84)** - Most common
- Used by Google Maps, GPS devices
- Degrees latitude/longitude
- Format: (lat, lon) or (lon, lat) depending on context

**Example:**
```
Saharsa: EPSG:4326 = (25.5941, 85.1376)
```

### 3. Geometries

**Geometries** are shapes that represent locations:

**Point:** Single coordinate
```
POINT(85.1376 25.5941)
```

**LineString:** Series of connected points
```
LINESTRING(85.1 25.5, 85.2 25.6, 85.3 25.7)
```

**Polygon:** Closed shape with multiple points (used for parcels)
```
POLYGON((85.1 25.5, 85.2 25.5, 85.2 25.6, 85.1 25.6, 85.1 25.5))
```

### 4. Spatial Indexes

**Problem:** Finding which parcel contains a clicked point requires checking all parcels.
- 100 parcels = instant
- 10,000 parcels = 100ms
- 1,000,000 parcels = 10s (too slow!)

**Solution:** GIST Index (tree structure)
- Divides space into bounding boxes
- Eliminates 99% of parcels from search
- Makes 1,000,000 parcel search instant

---

## Spatial Data Formats

### 1. GeoJSON (Web Standard)

Human-readable JSON format for GIS data.

```json
{
  "type": "Feature",
  "properties": {
    "plot_no": "3244",
    "area": "0.5000"
  },
  "geometry": {
    "type": "Polygon",
    "coordinates": [
      [[85.1200, 25.5900], [85.1300, 25.5900], 
       [85.1300, 25.6000], [85.1200, 25.6000], 
       [85.1200, 25.5900]]
    ]
  }
}
```

**Used for:** Frontend map rendering, API responses

### 2. Shapefile (Industry Standard)

Binary format used by GIS professionals.

Components:
- `*.shp` - Geometry (polygons, points, lines)
- `*.dbf` - Attribute data (plot_no, khata_no, etc.)
- `*.shx` - Shape index (for fast access)
- `*.prj` - Projection info (CRS)

**Used for:** Data import/export, BhuNaksha downloads

### 3. PostGIS Native Format

PostgreSQL extension that stores geometries.

```sql
-- In database
geometry GEOMETRY(POLYGON, 4326)

-- Example value
POLYGON((85.1 25.5, 85.2 25.5, 85.2 25.6, 85.1 25.5))
```

**Used for:** Database storage, spatial queries

---

## PostGIS: Spatial Database Queries

### Installation

```sql
-- In PostgreSQL
CREATE EXTENSION postgis;

-- Verify
SELECT PostGIS_Version();
```

### Core Functions

#### 1. Create Point from Coordinates
```sql
SELECT ST_Point(85.1376, 25.5941)
-- Returns: POINT(85.1376 25.5941)
```

#### 2. Point-in-Polygon (Most Important)
```sql
SELECT * FROM parcels
WHERE ST_Contains(
  geometry,
  ST_Point(85.1376, 25.5941)
)
```

**What it does:**
- Finds all parcels that contain the given point
- Uses GIST index for speed
- Result: The parcel you clicked on

**In your MVP:**
```
User clicks map → coordinates (lat, lon) sent to backend
Backend runs: SELECT * FROM parcels WHERE ST_Contains(geometry, point)
Database returns: Parcel containing that point
Frontend displays: Parcel details
```

#### 3. Convert Geometry to GeoJSON
```sql
SELECT ST_AsGeoJSON(geometry) FROM parcels
-- Returns: {"type":"Polygon","coordinates":[...]}
```

Used to send geometries to frontend.

#### 4. Calculate Area
```sql
SELECT ST_Area(geometry::geography) / 10000 as area_hectares
```

#### 5. Find Nearby Parcels
```sql
SELECT * FROM parcels
WHERE ST_DWithin(
  geometry::geography,
  ST_Point(85.1376, 25.5941)::geography,
  1000  -- within 1000 meters
)
```

---

## Leaflet.js: Frontend Map Library

### Why Leaflet?

- Lightweight (39 KB)
- Works with OpenStreetMap, Google Maps
- Supports GeoJSON overlays
- Mobile-friendly
- Free and open-source

### Basic Usage

```jsx
import { MapContainer, TileLayer, GeoJSON } from 'react-leaflet';

export default function Map() {
  return (
    <MapContainer center={[25.5941, 85.1376]} zoom={10}>
      {/* Tile layer = base map (OpenStreetMap) */}
      <TileLayer
        url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
        attribution='&copy; OpenStreetMap contributors'
      />
      
      {/* GeoJSON overlay = parcel polygons */}
      <GeoJSON data={parcelGeoJSON} />
    </MapContainer>
  );
}
```

### Click to Get Coordinates

```jsx
const MapClickHandler = () => {
  useMapEvents({
    click(e) {
      console.log(e.latlng);  // { lat: 25.5941, lng: 85.1376 }
      
      // Send to backend
      fetch(`/api/search/coordinate?lat=${e.latlng.lat}&lon=${e.latlng.lng}`)
        .then(res => res.json())
        .then(data => console.log(data.parcel))
    }
  });
  return null;
};
```

### Display Parcel Boundary

```jsx
const parcelGeoJSON = {
  type: "Feature",
  geometry: {
    type: "Polygon",
    coordinates: [[[85.1, 25.5], [85.2, 25.5], [85.2, 25.6], [85.1, 25.5]]]
  }
};

<GeoJSON data={parcelGeoJSON} style={{ color: 'blue', weight: 2 }} />
```

---

## Data Flow in MVP

```
┌─────────────────────────────────────────────────────────┐
│ User clicks map at coordinate (25.5941, 85.1376)       │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│ Frontend (Leaflet) captures click event                │
│ Sends: GET /api/search/coordinate?lat=25.5941&lon=85.1376
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│ Backend (Express) receives request                     │
│ Executes PostGIS query:                               │
│ SELECT * FROM parcels                                 │
│ WHERE ST_Contains(geometry, ST_Point(85.1376, 25.5941))
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│ Database (PostGIS) searches parcel table              │
│ GIST spatial index narrows down candidates instantly │
│ ST_Contains function checks if point is inside polygon│
│ Returns: Row with parcel_id=123, plot_no=3244, etc.  │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│ Backend formats as JSON:                              │
│ {                                                     │
│   "success": true,                                   │
│   "parcel": {                                        │
│     "id": 123,                                       │
│     "plot_no": "3244",                              │
│     "geometry": {...},                              │
│     ...                                             │
│   }                                                 │
│ }                                                   │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│ Frontend renders:                                      │
│ 1. Highlight parcel polygon on map (blue outline)    │
│ 2. Show details panel with plot_no, area, owner, etc.│
└─────────────────────────────────────────────────────────┘
```

---

## WMS & WFS (Advanced)

### What They Are

**WMS (Web Map Service):**
- Returns static image tiles of the map
- Like Google Maps satellite view
- Used for visualization

**WFS (Web Feature Service):**
- Returns actual geometry data
- Can be queried and styled
- Used for data access

### BhuNaksha Usage

BhuNaksha likely uses:
- **WMS** to display the base cadastral map
- **Internal APIs** (like `getPlotAXY`) for data retrieval

For this MVP, we're building our own WMS-like service using PostGIS + Express.

---

## Common GIS Mistakes to Avoid

❌ **Storing coordinates as text** (e.g., "85.1376, 25.5941")
- Can't do spatial queries
- Slow to search

✅ **Use PostGIS GEOMETRY type**

❌ **Forgetting coordinate order**
- Different formats use (lat,lon) vs (lon,lat)
- Can cause 100km errors!

✅ **Always specify: Leaflet uses (lat,lon), PostGIS functions often use (lon,lat)**

❌ **Skipping spatial indexes**
- Works for 1000 parcels
- Dies at 100,000 parcels

✅ **Always create GIST index**

---

## Resources

- PostGIS Docs: https://postgis.net/documentation/
- Leaflet Docs: https://leafletjs.com/
- GeoJSON Spec: https://geojson.org/
- Shapefile Tutorial: https://gdal.org/drivers/vector/shapefile.html
