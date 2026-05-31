# Bihar Land Records MVP

A Google Maps-style interface for government land records in Bihar. Click on any land parcel to instantly view publicly available land record information.

## Vision

Transform access to Bihar's digitized cadastral data by providing a modern, intuitive web interface that allows citizens, researchers, and government officials to query land records by location.

## Current Status: MVP Phase

- **Scope:** Saharsa District (pilot)
- **Features:** 
  - Interactive map with parcel overlays
  - Click-to-query parcel information
  - Plot/Dag/Khata number lookup
  - Area and land type display
  - Search by location, plot number, or khata number

## Tech Stack

### Frontend
- **React 18** - UI framework
- **Leaflet.js** - Interactive mapping
- **TypeScript** - Type safety
- **Tailwind CSS** - Styling
- **Vite** - Fast build tool

### Backend
- **Node.js + Express** - REST API server
- **PostgreSQL + PostGIS** - Spatial database
- **Knex.js** - Query builder
- **Axios** - HTTP client

### GIS/Data
- **PostGIS** - Spatial queries (point-in-polygon)
- **GDAL/OGR** - Shapefile processing
- **GeoJSON** - Vector format for parcels
- **Leaflet-GeoJSON** - Frontend GIS rendering

### DevOps
- **Docker** - Containerization
- **Docker Compose** - Local development
- **GitHub Actions** - CI/CD (future)

## Quick Start

### Prerequisites
- Node.js 18+
- PostgreSQL 13+ with PostGIS extension
- Docker & Docker Compose (recommended)

### Installation

```bash
# Clone repository
git clone https://github.com/poddarmanjit5-dot/bihar-land-records-mvp.git
cd bihar-land-records-mvp

# Using Docker (recommended)
docker-compose up

# Manual setup
cd backend
npm install
npm run migrate
npm start

cd ../frontend
npm install
npm run dev
```

### Access
- Frontend: http://localhost:5173
- Backend API: http://localhost:3000
- PostGIS DB: localhost:5432

## Architecture Overview

```
User Interface (React + Leaflet)
           ↓
    REST API (Express)
           ↓
Point-in-Polygon Query (PostGIS)
           ↓
Parcel Geometry + Metadata (PostgreSQL)
           ↓
Land Records Database
```

## API Endpoints

### Map Tiles & Parcel Data
- `GET /api/tiles/:z/:x/:y` - Map tiles
- `GET /api/parcels` - All parcels (GeoJSON)
- `GET /api/parcels/:id` - Single parcel details

### Queries
- `GET /api/search/coordinate?lat=X&lon=Y` - Find parcel at coordinate
- `GET /api/search/plot?plot_no=XXX&khata=XXX` - Search by plot/khata
- `GET /api/records/:plot_id` - Get full land record

### Admin
- `POST /api/admin/upload-shapefile` - Bulk parcel import
- `POST /api/admin/sync-records` - Sync with BhuNaksha

## Data Flow

```
1. User clicks map coordinate (lat, lon)
2. Frontend sends: GET /api/search/coordinate?lat=X&lon=Y
3. Backend queries: SELECT * FROM parcels WHERE ST_Contains(geometry, point)
4. Database returns: parcel_id, dag_no, khata_no, area, land_type
5. Backend fetches: land_records JOIN mutations ON parcel_id
6. Response includes: ownership, mutation history, ROR details
7. Frontend renders: parcel highlight + details panel
```

## Directory Structure

```
bihar-land-records-mvp/
├── frontend/                 # React web app
│   ├── src/
│   │   ├── components/      # Reusable React components
│   │   ├── pages/           # Page components
│   │   ├── services/        # API calls
│   │   ├── hooks/           # Custom React hooks
│   │   ├── types/           # TypeScript interfaces
│   │   ├── utils/           # Utilities
│   │   └── App.tsx
│   ├── public/
│   └── package.json
│
├── backend/                 # Node.js + Express API
│   ├── src/
│   │   ├── routes/          # API endpoints
│   │   ├── controllers/      # Business logic
│   │   ├── services/        # Database/GIS services
│   │   ├── models/          # Data models
│   │   ├── middleware/      # Auth, validation, etc
│   │   ├── utils/           # Helpers
│   │   └── app.ts
│   ├── migrations/          # Database migrations
│   ├── seeds/               # Test data
│   └── package.json
│
├── gis/                     # GIS data & processing
│   ├── shapefiles/          # Parcel geometries (*.shp, *.dbf, etc)
│   ├── geojson/             # Converted GeoJSON
│   └── scripts/             # GDAL conversion scripts
│
├── docker-compose.yml       # Local dev environment
├── .env.example             # Environment template
├── docs/                    # Documentation
│   ├── API.md
│   ├── DATABASE.md
│   ├── GIS.md
│   ├── DEPLOYMENT.md
│   └── ROADMAP.md
└── README.md
```

## Development Roadmap

### Week 1: Core MVP
- [ ] Database schema & migrations
- [ ] Parcel data import (Saharsa District)
- [ ] Point-in-polygon API endpoint
- [ ] Basic map interface
- [ ] Click-to-query feature

### Week 2: Polish & Features
- [ ] Search by plot/khata number
- [ ] Land record display
- [ ] Mutation history
- [ ] Export as PDF
- [ ] Mobile responsiveness

### Week 3: Scaling
- [ ] Add Muzaffarpur District
- [ ] Performance optimization
- [ ] Caching layer
- [ ] User analytics

### Month 2: Production
- [ ] Authentication (government officials)
- [ ] Audit logs
- [ ] API rate limiting
- [ ] Deployment to staging

### Month 3: Expansion
- [ ] All Bihar districts
- [ ] Assam integration
- [ ] Official API documentation
- [ ] Government partnerships

## Legal & Compliance

- ✅ Uses publicly available government data
- ✅ No AI/ML ownership prediction
- ✅ Read-only access to land records
- ✅ GDPR-compliant (no personal data storage)
- ⚠️ Terms of Service compliance with BhuNaksha (pending)

## Contributing

This is an open-source project. Contributions welcome.

```bash
git checkout -b feature/your-feature
# Make changes
git push origin feature/your-feature
# Create Pull Request
```

## License

MIT License - See LICENSE file

## Contact

- **Founder:** Manjit Poddar (@poddarmanjit5-dot)
- **Email:** [your-email]
- **Twitter:** [your-handle]

## Acknowledgments

- Bihar BhuNaksha for cadastral data
- PostGIS community for spatial queries
- Leaflet.js for mapping library
- All contributors to open-source GIS

---

**Last Updated:** May 31, 2026
**Status:** MVP in development
**Next Milestone:** Week 1 backend complete