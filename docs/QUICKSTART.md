# Quick Start Guide

## Local Development Setup (5 minutes with Docker)

### Prerequisites
- Docker & Docker Compose installed
- Git
- Code editor (VS Code recommended)

### Step 1: Clone & Setup

```bash
# Navigate to repo
cd bihar-land-records-mvp

# Copy environment template
cp .env.example .env

# Start services with Docker Compose
docker-compose up
```

This will automatically start:
- **PostgreSQL + PostGIS** (port 5432)
- **Express Backend** (port 3000)
- **React Frontend** (port 5173)

### Step 2: Access the Application

Open your browser:
- **Frontend:** http://localhost:5173
- **Backend API:** http://localhost:3000/health
- **PostgreSQL:** localhost:5432

### Step 3: Test the API

```bash
# In another terminal, test a basic request
curl http://localhost:3000/health

# Response:
# {"status":"ok","timestamp":"2026-05-31T10:00:00.000Z"}
```

---

## Manual Setup (Without Docker)

### Prerequisites
- Node.js 18+
- PostgreSQL 13+ with PostGIS
- npm or yarn

### Backend Setup

```bash
cd backend

# Install dependencies
npm install

# Create .env file
cat > .env << EOF
NODE_ENV=development
PORT=3000
DB_HOST=localhost
DB_PORT=5432
DB_NAME=bihar_land_records
DB_USER=postgres
DB_PASSWORD=postgres
EOF

# Run migrations
npm run migrate

# Start development server
npm run dev
```

Backend runs on: http://localhost:3000

### Frontend Setup

```bash
cd frontend

# Install dependencies
npm install

# Start development server
npm run dev
```

Frontend runs on: http://localhost:5173

---

## Project Structure

```
bihar-land-records-mvp/
├── backend/                    # Node.js + Express API
│   ├── src/
│   │   ├── app.ts             # Main Express app
│   │   ├── db.js              # Database connection
│   │   ├── routes/            # API routes
│   │   │   ├── search.js      # Coordinate/plot search
│   │   │   └── parcels.js     # Parcel data endpoints
│   │   └── migrations/        # Database schemas
│   ├── package.json
│   └── Dockerfile
│
├── frontend/                   # React + Leaflet
│   ├── src/
│   │   ├── App.jsx            # Main React component
│   │   ├── main.jsx           # Entry point
│   │   └── index.css          # Styles
│   ├── index.html
│   ├── package.json
│   └── Dockerfile
│
├── docker-compose.yml         # Local dev environment
├── .env.example               # Environment template
└── docs/                      # Documentation
```

---

## Key Features Implemented

✅ **Interactive Map**
- Leaflet.js powered map
- Click to search for parcels
- Real-time response

✅ **Backend API**
- Point-in-polygon search
- Plot number lookup
- Parcel detail retrieval

✅ **Database**
- PostgreSQL + PostGIS
- Spatial indexes for fast queries
- Schema for parcels, records, mutations

✅ **Docker Setup**
- One-command startup
- All services containerized
- Volume mounts for live code reload

---

## Adding Sample Data

### Option 1: Insert Test Data

```bash
# Inside PostgreSQL container
docker exec -it bihar_postgres psql -U postgres -d bihar_land_records

-- Insert sample district
INSERT INTO districts (name, code) VALUES ('SAHARSA', '10');

-- Insert sample parcel (as GeoJSON)
INSERT INTO parcels (
  district_id, plot_no, dag_no, khata_no, area, land_type,
  district, mauza, sheet_no, geometry
) VALUES (
  1, '3244', '456', '789', 0.5000, 'Agricultural',
  'SAHARSA', 'Simri Bakhtiarpur', '03',
  ST_GeomFromText('POLYGON((85.1200 25.5900, 85.1300 25.5900, 85.1300 25.6000, 85.1200 25.6000, 85.1200 25.5900))', 4326)
);
```

### Option 2: Import from Shapefile

```bash
# Download shapefile from BhuNaksha, then:
docker exec bihar_postgres ogr2ogr -f "PostgreSQL" \
  PG:"dbname=bihar_land_records user=postgres password=postgres" \
  /path/to/parcels.shp \
  -nln parcels \
  -overwrite
```

---

## Testing the Map

1. Open http://localhost:5173
2. Click anywhere on the map
3. Should see parcel details on the right panel

If you get "No parcel found" - add sample data first (see above).

---

## Stopping Services

```bash
# With Docker
docker-compose down

# Or Ctrl+C in terminal if running without Docker
```

---

## Troubleshooting

### Port Already in Use
```bash
# Find process using port 5173
lsof -i :5173

# Kill it
kill -9 <PID>
```

### Database Connection Error
```bash
# Check PostgreSQL is running
docker ps | grep postgres

# Or start it
docker-compose up postgres -d
```

### Module Not Found
```bash
# Reinstall dependencies
rm -rf node_modules package-lock.json
npm install
```

---

## Next Steps

1. ✅ Verify setup works with sample data
2. 📍 Import BhuNaksha shapefile data
3. 🔍 Test coordinate search functionality
4. 🎨 Customize map styling
5. 📱 Add mobile responsiveness
6. 🔐 Add authentication (optional)
7. 🚀 Deploy to production

---

## Need Help?

- Check `docs/` folder for detailed guides
- Review API documentation: `docs/API.md`
- See database setup: `docs/DATABASE.md`
