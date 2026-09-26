# SIH2
# SMART PATH NER - Complete Backend & Frontend Architecture
**AI-Based Smart Logistics & Accessibility Intelligence Platform for North Eastern Region**

---

## Table of Contents
1. [System Overview](#system-overview)
2. [Backend Architecture](#backend-architecture)
3. [Frontend Architecture](#frontend-architecture)
4. [Data Flow & Integration](#data-flow--integration)
5. [Database Schema](#database-schema)
6. [API Specifications](#api-specifications)
7. [Deployment & Infrastructure](#deployment--infrastructure)
8. [Implementation Phases](#implementation-phases)

---

## System Overview

### Project Context (from SMART PATH NER Presentation)
- **Problem**: Transportation of essential goods (medicines, food, materials) across NER faces delays due to terrain, weather disruptions, and limited connectivity
- **Solution Approach**: 
  - Terrain-aware route optimization using weighted graph model
  - Realistic time & risk estimates for logistics planning
  - Segment-based transport guidance (truck vs van vs motorbike per route section)
  - Interactive dashboard for real-time accessibility monitoring

### Key Innovation: Weighted Graph Model
Instead of flat-map routing, SMART PATH uses:
- **Elevation changes** (terrain difficulty scoring)
- **Surface quality** (paved, unpaved, risky)
- **Road width** (vehicle suitability)
- **Historical risk data** (flood zones, landslide corridors, weather patterns)

This creates realistic ETAs and identifies truly safe routes, not just shortest ones.

---

## Backend Architecture

### Tech Stack
```
Node.js / Express.js          (HTTP API Server)
Python Flask/FastAPI          (Routing Engine Microservice)
PostgreSQL + PostGIS          (Spatial Database)
Redis                         (Caching, Real-time tracking)
NetworkX (Python)             (Graph algorithms)
```

### Core Modules

#### 1. **REST API Layer (Express.js)**

**Responsibilities:**
- Authentication & authorization (JWT tokens for field officers, admins, logistics operators)
- Handle incoming road status updates, vehicle tracking, incident reports
- Serve real-time accessibility data to dashboards
- Rate limiting and request validation

**Key Endpoints:**
```
POST   /auth/login                    Login field officers / admins
GET    /districts/:id/connectivity    Get district-wide accessibility status
GET    /routes/optimize               Request optimized route (calls Python engine)
POST   /incidents/report              File geo-tagged incident report
GET    /vehicles/track                Live vehicle GPS location
POST   /vehicles/track                Update vehicle location
GET    /alerts                        Fetch active alerts for a region
POST   /alerts/subscribe              Subscribe to push notifications
GET    /dashboard/bottlenecks         Identify logistics chokepoints
GET    /dashboard/emergency-routes    Pre-computed disaster-time corridors
```

**Request/Response Examples:**

```json
// POST /routes/optimize
// Request
{
  "origin": { "lat": 24.9142, "lng": 93.9497 },     // Imphal
  "destination": { "lat": 24.8170, "lng": 92.9605 }, // Senapati
  "vehicleType": "truck",                             // truck, van, motorbike
  "constraints": {
    "avoidFloodzones": true,
    "preferPavedRoads": true,
    "weatherAlerts": ["heavy_rain"]
  }
}

// Response
{
  "route": {
    "pathId": "rte_12345",
    "distance": 68.5,                    // km
    "estimatedTime": 145,                // minutes
    "riskScore": 0.32,                   // 0-1 scale
    "segments": [
      {
        "id": "seg_001",
        "from": "Imphal",
        "to": "Ukhrul",
        "distance": 42.3,
        "time": 85,
        "riskScore": 0.25,
        "recommendedVehicle": "truck",
        "roadQuality": "paved",
        "elevationGain": 580,
        "hazards": ["steep_gradient", "tight_curves"]
      },
      {
        "id": "seg_002",
        "from": "Ukhrul",
        "to": "Senapati",
        "distance": 26.2,
        "time": 60,
        "riskScore": 0.42,
        "recommendedVehicle": "van",
        "roadQuality": "unpaved",
        "elevationGain": 220,
        "hazards": ["monsoon_zone", "loose_gravel"]
      }
    ],
    "alternateRoutes": [
      { "pathId": "rte_12346", "distance": 82.1, "time": 165, "riskScore": 0.28 }
    ]
  }
}
```

#### 2. **Routing Engine Microservice (Python)**

**Technology**: Flask/FastAPI + NetworkX

**Responsibilities:**
- Build and maintain weighted graph model of NER road network
- Execute Dijkstra + A* algorithms for optimal routing
- Score routes based on terrain, weather, historical incidents
- Cache route computations for frequent queries

**Architecture:**

```python
# Core Components (pseudocode)

class TerrainWeightedGraph:
    """
    Models NER as weighted directed graph where:
    - Nodes = villages, towns, junction points
    - Edges = road segments with weights
    """
    def __init__(self, osm_data, terrain_data):
        self.graph = nx.MultiDiGraph()
        self.load_osm_roads(osm_data)
        self.assign_terrain_weights(terrain_data)
    
    def assign_terrain_weights(self, terrain_data):
        """
        Calculate edge weights from:
        - Distance (km)
        - Elevation change (meters)
        - Surface quality (paved=0.1, unpaved=0.3, risky=0.5)
        - Historical disruption frequency
        - Weather vulnerability (flood zones, landslide corridors)
        
        Weight = base_distance * surface_factor * elevation_factor * weather_risk_factor
        """
        for edge in self.graph.edges(data=True):
            u, v, key, attrs = edge
            distance = attrs['distance']
            elevation = attrs['elevation_change']
            surface = attrs['surface_type']
            
            # Surface factor (multiplier)
            surface_factors = {'paved': 1.0, 'unpaved': 1.4, 'risky': 2.0}
            
            # Elevation factor (steeper = more time)
            elevation_factor = 1.0 + (elevation / 100)  # 100m rise = 1x multiplier
            
            # Weather factor (flood/landslide zones)
            weather_risk = attrs.get('flood_risk', 0) + attrs.get('landslide_risk', 0)
            weather_factor = 1.0 + (weather_risk * 0.5)
            
            # Final weight
            attrs['weight'] = distance * surface_factors[surface] * elevation_factor * weather_factor
            attrs['travel_time'] = distance * 1.5 * elevation_factor * weather_factor  # minutes

class RoutingEngine:
    """Execute pathfinding algorithms"""
    
    def find_optimal_route(self, origin, destination, vehicle_type, constraints):
        """
        1. Prune edges not suitable for vehicle_type
           (e.g., trucks can't use footpaths, monsoon routes flagged)
        2. Run Dijkstra + A* for shortest weighted path
        3. Return route with segment-level details & alternate routes
        """
        pass
    
    def segment_guidance(self, route, vehicle_type):
        """
        Recommend transport mode per segment:
        - Paved, flat (elevation gain < 150m): truck (max cargo)
        - Unpaved, moderate slopes: van (balanced)
        - Risky, steep terrain: motorbike (agility), or no-go
        """
        pass

class DisruptionPredictor:
    """
    Predict route accessibility degradation:
    - Monsoon season + elevation < 500m + historical floods → avoid
    - Steep terrain + heavy rain forecast → landslide risk alert
    - Time-based: roads always blocked 6pm-8am during winter fog
    """
    def predict_accessibility(self, route_segment, weather_forecast, timestamp):
        risk_score = 0.0
        # rules based on historical data
        return risk_score
```

**Data Sources:**
- OpenStreetMap (road network, basic geometry)
- Simulated terrain datasets (elevation, slope)
- Historical incident logs (disruption frequency per segment)
- Weather APIs (current + forecast)

**API to Express:**
```python
# POST http://localhost:5000/routing/optimize
# Called by Express, returns optimized routes
```

#### 3. **Real-Time Tracking Module (Redis + Express)**

**For Vehicle Tracking:**
```javascript
// Express with Socket.io for live updates
io.on('connection', (socket) => {
  // Field officer updates vehicle location
  socket.on('vehicle:location-update', async (data) => {
    const { vehicleId, lat, lng, timestamp } = data;
    
    // Store in Redis for real-time access
    await redis.geoadd('vehicles:locations', lng, lat, vehicleId);
    
    // Store in PostgreSQL for historical tracking
    await Vehicle.update({ currentLat: lat, currentLng: lng }, 
                         { where: { id: vehicleId } });
    
    // Broadcast to all subscribed dashboards
    io.emit('vehicle:moved', { vehicleId, lat, lng, timestamp });
    
    // Check if vehicle entering high-risk zone
    const riskZones = await checkRiskZones(lat, lng);
    if (riskZones.length > 0) {
      io.emit('alert:risk-zone-entry', { vehicleId, zones: riskZones });
    }
  });
});
```

#### 4. **Incident Reporting & Alert System**

**Field Officer Reports:**
```javascript
// POST /incidents/report
{
  "officerId": "officer_123",
  "location": { "lat": 24.82, "lng": 93.96 },
  "type": "landslide",         // landslide, flood, blocked_road, pothole, bridge_damage
  "severity": "high",          // low, medium, high
  "description": "Debris blocking NH-39 near Manipur-Nagaland border",
  "photos": ["base64_image_1", "base64_image_2"],  // Geo-tagged
  "affectedRoads": ["NH-39", "State_Road_42"]
}

// Triggers:
// 1. Store in PostgreSQL with PostGIS geometry
// 2. Identify all vehicles on affected routes
// 3. Send push notifications + SMS to drivers
// 4. Auto-generate alternate routes
// 5. Notify district admin dashboard
// 6. Update accessibility status
```

#### 5. **Authentication & Role-Based Access Control**

```javascript
// Roles:
// - admin: Full access, system configuration
// - district_officer: View/manage incidents in assigned district
// - logistics_operator: View routes, track own vehicles
// - field_officer: Report incidents, update road status
// - emergency_responder: Access to emergency routes, priority alerts

const rbac = {
  admin: ['*'],
  district_officer: [
    'view:district-accessibility',
    'create:incident-reports',
    'view:vehicles-in-district'
  ],
  logistics_operator: [
    'view:routes',
    'create:route-requests',
    'view:own-vehicles'
  ],
  field_officer: [
    'create:incident-reports',
    'update:road-status'
  ],
  emergency_responder: [
    'view:all-routes',
    'view:emergency-corridors',
    'override:alerts'
  ]
};
```

#### 6. **Database Connection Pool & ORM**

```javascript
// Sequelize with connection pooling
const sequelize = new Sequelize(process.env.DB_URL, {
  pool: { max: 20, min: 5, idle: 10000 },
  logging: false, // Disable in production
});

// Models for key entities
- District
- Village
- RoadSegment (PostGIS geometry)
- Vehicle
- IncidentReport (PostGIS point geometry)
- Route (cached computed routes)
- Alert
- User (with roles)
```

---

## Frontend Architecture

### 1. **Dashboard Application (React + Leaflet.js)**

**Tech Stack:**
```
React 18+
React Router (navigation)
Leaflet.js + React-Leaflet (map rendering)
Redux / Zustand (state management)
Socket.io-client (real-time updates)
Tailwind CSS (styling)
```

**Key Views:**

#### a) **District Connectivity Dashboard** (Admin/District Officer)
```
┌─────────────────────────────────────────────────────────────┐
│  SMART PATH NER - District Accessibility Monitor            │
├─────────────────────────────────────────────────────────────┤
│  [District Selector ▼]  [Date Range Picker]  [Refresh]      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────────────────┐  ┌──────────────────────┐ │
│  │                              │  │  STATUS SUMMARY      │ │
│  │                              │  │  ─────────────────   │ │
│  │   MAP VIEW:                  │  │  ✓ Accessible: 45%   │ │
│  │   - Road segments color-coded│  │  ⚠ At Risk: 35%      │ │
│  │   - Green: Accessible        │  │  ✗ Blocked: 20%      │ │
│  │   - Yellow: Risky            │  │                      │ │
│  │   - Red: Blocked             │  │  BOTTLENECKS:        │ │
│  │   - Villages as dots         │  │  • NH-39 (landslide) │ │
│  │   - Vehicles as moving icons │  │  • State Rd-42       │ │
│  │   - Heatmap: Disruption freq │  │  • Bridge at Ukhrul  │ │
│  │                              │  │                      │ │
│  └──────────────────────────────┘  └──────────────────────┘ │
│                                                               │
│  ACTIVE INCIDENTS                                            │
│  ┌─ [Landslide] NH-39 | High | 2 hours ago | 6 vehicles     │
│  ├─ [Flood] Senapati Rd | Medium | 45 mins ago | 3 vehicles │
│  └─ [Pothole] State Rd 42 | Low | 1 hour ago | Update >>    │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**Component Hierarchy:**
```
DashboardContainer
├── MapComponent
│   ├── RoadLayerRenderer (PostGIS road data)
│   ├── VehicleMarkerLayer (live GPS, real-time updates via Socket.io)
│   ├── IncidentMarkerLayer (recent incidents with severity color)
│   ├── HeatmapLayer (disruption frequency overlay)
│   └── InteractivePopups (click to see details)
├── StatusPanel
│   ├── DistrictMetrics (accessible%, at-risk%, blocked%)
│   └── BottlenecksList (sorted by impact)
├── IncidentsPanel
│   └── IncidentCards (sortable, filterable)
└── RealTimeUpdater (Socket.io listener for live changes)
```

#### b) **Route Planning Interface** (Logistics Operator)
```
┌─────────────────────────────────────────────────────────────┐
│  SMART PATH NER - Route Planner                              │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  INPUT PANEL:                                                │
│  ┌──────────────────────────┐                               │
│  │ Origin: [Imphal ▼]       │                               │
│  │ Destination: [Senapati▼] │                               │
│  │ Vehicle: [Truck ▼]       │                               │
│  │                          │                               │
│  │ [✓] Avoid flood zones    │                               │
│  │ [✓] Prefer paved roads   │                               │
│  │ [✓] Current weather      │                               │
│  │                          │                               │
│  │ [Find Optimal Route]     │                               │
│  └──────────────────────────┘                               │
│                                                               │
│  RESULTS:                                                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ PRIMARY ROUTE (Recommended)                            │ │
│  │ 68.5 km | 2h 25m | Risk: LOW (0.32)                   │ │
│  │                                                        │ │
│  │ Segment-by-Segment Breakdown:                         │ │
│  │ ┌─ [1] Imphal → Ukhrul                                │ │
│  │ │   42.3 km | 1h 25m | Truck ✓ | Paved | 580m elev   │ │
│  │ │   Hazards: Steep gradient (12%), tight curves       │ │
│  │ │                                                    │ │
│  │ └─ [2] Ukhrul → Senapati                              │ │
│  │     26.2 km | 1h 0m | Van ⚠ | Unpaved | 220m elev    │ │
│  │     Hazards: Monsoon zone, loose gravel              │ │
│  │     → Switch to van or motorbike for this segment    │ │
│  │                                                        │ │
│  │ [Map View] [Turn-by-Turn] [Download Route]            │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                               │
│  ALTERNATIVES:                                               │
│  [ ] Route 2: 82.1 km | 2h 45m | Risk: LOW (0.28)           │
│  [ ] Route 3: 71.5 km | 2h 30m | Risk: MEDIUM (0.45)        │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**Component Hierarchy:**
```
RoutePlannerContainer
├── InputForm
│   ├── LocationSearchField (autocomplete villages/towns)
│   ├── VehicleTypeSelector
│   ├── ConstraintCheckboxes
│   └── [Submit Button]
├── RouteResultsPanel
│   ├── PrimaryRouteCard
│   │   ├── RouteMetrics (distance, time, risk)
│   │   ├── SegmentAccordion (expandable segments)
│   │   │   └── SegmentDetail (hazards, vehicle recommendations)
│   │   ├── MapPreview
│   │   └── ActionButtons (download, share, start navigation)
│   └── AlternateRoutesList
└── WeatherAlerts (dynamic, shown if relevant)
```

#### c) **Real-Time Vehicle Tracking** (Logistics Operator)
```
┌─────────────────────────────────────────────────────────────┐
│  SMART PATH NER - Live Vehicle Tracking                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  FLEET OVERVIEW:                                             │
│  Total Vehicles: 12 | On Route: 10 | Delayed: 2 | Idle: 0   │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                        │ │
│  │   [Interactive Map with Live Vehicle Icons]           │ │
│  │   - Green: On schedule                                │ │
│  │   - Yellow: Minor delay                               │ │
│  │   - Red: Significant delay / Alert                    │ │
│  │   - Click vehicle icon for details                    │ │
│  │                                                        │ │
│  │   [Realtime updates via Socket.io]                    │ │
│  │                                                        │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                               │
│  VEHICLE DETAILS PANEL:                                      │
│  ┌─ TRUCK-001 (Selected)                                    │
│  │  Status: On Route | Last update: 2 mins ago             │
│  │  Route: Imphal → Senapati                               │
│  │  Progress: 42.3/68.5 km (62%)                            │
│  │  ETA: 1h 28m (5 mins delayed due to weather alert)      │
│  │  Cargo: Medical supplies (perishable)                    │
│  │  Driver: Rajesh Kumar                                    │
│  │  [Reroute] [Contact Driver] [Alert Manager]              │
│  │                                                           │
│  │  TRIP HISTORY:                                           │
│  │  • 14:30 - Left Imphal                                   │
│  │  • 14:45 - Entered monsoon zone (caution alert)          │
│  │  • 15:10 - Delayed 5 mins due to road closure           │
│  │  • 15:15 - Rerouted via alternate segment               │
│  │                                                           │
│  └─ TRUCK-002                                              │
│     Status: Delayed | Reason: Landslide alert (NH-39)      │
│     [View Details]                                          │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

#### d) **Incident Reporting Interface** (Field Officer / Mobile-First)
```
┌─────────────────────────────────────────────────────────────┐
│  SMART PATH NER - Field Incident Report (Mobile App)         │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  [Auto-locate: 24.82°N, 93.96°E] [Manual adjust on map]     │
│                                                               │
│  Incident Type: [Landslide ▼]                               │
│                                                               │
│  Severity:                                                   │
│  ( ) Low   ( ) Medium   (●) High                             │
│                                                               │
│  Description:                                                │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Debris blocking NH-39 near Manipur-Nagaland border  │   │
│  │                                                      │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  [+ Add Photo] [+ Add Photo]                                 │
│  [Landslide_1.jpg ✓] [Debris_view.jpg ✓]                    │
│                                                               │
│  Affected Roads:                                             │
│  [x] NH-39     [ ] State Road 42     [ ] Local Road          │
│                                                               │
│  Estimated clearance: [12 hours ▼]                          │
│                                                               │
│  [SUBMIT REPORT]                                             │
│                                                               │
│  ─────────────────────────────────────────────────────────   │
│  ✓ Report submitted successfully (Offline queue: 2)         │
│  ✓ 12 vehicles notified of alternate routes                 │
│  ✓ District admin alerted                                   │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### 2. **Mobile App (React Native / Flutter)**

**Key Features:**
- **Offline-first architecture**: Cache maps, pre-computed routes, cached incident data
- **Low-bandwidth mode**: Reduced map detail, text alerts instead of images
- **Voice/SMS support**: For illiterate/low-literacy users
- **Background tracking**: GPS updates even when app not in foreground (for delivery vehicles)

**Tech Stack:**
```
React Native (iOS + Android)
or Flutter (if better offline support needed)

Libraries:
- react-native-maps (mapping)
- react-native-geolocation (GPS)
- AsyncStorage (local cache)
- react-native-push-notifications (push alerts)
- expo-sqlite (local database for offline sync)
```

**Key Screens:**
1. **Home**: Quick access to route planning, vehicle tracking, incident reporting
2. **Map**: Interactive map with offline map tiles cached locally
3. **Incident Report**: Geo-tagged photo + location capture (works offline, syncs on reconnect)
4. **Vehicle Tracking**: See your assigned vehicle's location in real-time
5. **Alerts**: Recent disruptions & notifications (actionable, dismissible)
6. **Offline Mode**: Shows cached data, queues new reports for sync

---

## Data Flow & Integration

### Real-Time Data Flow Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    FIELD OFFICERS (Mobile)                   │
│    - Report incidents (geo-tagged photos)                    │
│    - Update road conditions                                  │
│    - Track vehicle GPS                                       │
└────────────────┬──────────────────────────────────────────┘
                 │ (REST API + Socket.io)
                 │
    ┌────────────▼──────────────┐
    │   EXPRESS.JS API SERVER   │
    │  - Authentication (JWT)   │
    │  - Incident ingestion     │
    │  - GPS location updates   │
    │  - Rate limiting          │
    └────────┬───────────────┬──┘
             │               │
    ┌────────▼──────┐   ┌────▼─────────────────┐
    │   REDIS       │   │  PYTHON ROUTING      │
    │  - Vehicle    │   │  - Dijkstra routing  │
    │    geo-cache  │   │  - Disruption score  │
    │  - Session    │   │  - Segment guidance  │
    │    cache      │   └────┬─────────────────┘
    │  - Pub/Sub    │        │
    │    alerts     │   ┌────▼───────────────────────┐
    └────┬──────────┘   │  PYTHON MICROSERVICE       │
         │              │  - NetworkX graph model    │
         │              │  - ML disruption predict   │
         │              │  - Weather integration     │
         │              └────┬───────────────────────┘
         │                   │
    ┌────▼────────────────────▼──────────────────┐
    │     POSTGRESQL + PostGIS Database          │
    │  - Road segments (geometry)                │
    │  - Villages (points)                       │
    │  - Incident reports (geo-tagged)           │
    │  - Vehicles (current position + history)   │
    │  - Routes (cached computations)            │
    │  - Users (roles, permissions)              │
    └────────────────────────────────────────────┘
         │
    ┌────┴──────────────────────────────────┐
    │   EXTERNAL DATA SOURCES               │
    │  - OpenStreetMap (road network)       │
    │  - Weather API (current + forecast)   │
    │  - Government transport databases     │
    │  - Historical incident logs           │
    └───────────────────────────────────────┘
         │
    ┌────▼─────────────────────────────────┐
    │   REACT DASHBOARDS                   │
    │  - Admin connectivity view           │
    │  - Logistics route planning          │
    │  - Real-time vehicle tracking        │
    │  - Emergency response corridors      │
    │                                      │
    │   (Socket.io for live updates)       │
    └──────────────────────────────────────┘
```

### Incident-to-Alert Flow

```
Field Officer Reports Incident
            ↓
    [POST /incidents/report]
            ↓
  Express validates & stores in PostgreSQL
            ↓
  PostGIS queries: Which vehicles on affected routes?
            ↓
  Retrieve vehicle records + GPS locations
            ↓
  For each vehicle:
  - Generate alternate route via Python engine
  - Send push notification to driver
  - Send SMS backup alert
  - Broadcast to district admin dashboard
            ↓
  Update road accessibility status
  Update real-time heatmap
  Alert emergency responders if critical
            ↓
  Logistics operators see:
  - Incident marker on map
  - Affected vehicles list
  - Suggested reroutes for pending deliveries
```

---

## Database Schema

### Key Tables with PostGIS Extensions

```sql
-- SPATIAL TABLES (PostGIS)

-- Road network segments (geometric)
CREATE TABLE road_segments (
  id BIGSERIAL PRIMARY KEY,
  name VARCHAR(255),
  road_type VARCHAR(50),  -- highway, state_road, district_road, local
  surface VARCHAR(50),    -- paved, unpaved, risky
  geometry GEOMETRY(LineString, 4326),  -- GIS coordinates
  distance_km DECIMAL(10, 2),
  elevation_gain_m INT,
  width_m DECIMAL(5, 2),
  historical_disruption_count INT,
  flood_risk_score DECIMAL(3, 2),      -- 0-1 scale
  landslide_risk_score DECIMAL(3, 2),  -- 0-1 scale
  last_updated TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Villages & towns (point geometry)
CREATE TABLE villages (
  id BIGSERIAL PRIMARY KEY,
  name VARCHAR(255),
  district_id BIGINT REFERENCES districts(id),
  population INT,
  has_hospital BOOLEAN,
  has_school BOOLEAN,
  has_market BOOLEAN,
  geometry GEOMETRY(Point, 4326),  -- GIS coordinates
  created_at TIMESTAMP DEFAULT NOW()
);

-- Incidents (point + details)
CREATE TABLE incident_reports (
  id BIGSERIAL PRIMARY KEY,
  reporter_id BIGINT REFERENCES users(id),
  incident_type VARCHAR(50),       -- landslide, flood, blocked_road, etc
  severity VARCHAR(20),            -- low, medium, high
  description TEXT,
  location_geometry GEOMETRY(Point, 4326),
  affected_road_segment_ids BIGINT[],  -- array of road segment IDs
  photos_urls TEXT[],              -- base64 or S3 URLs
  estimated_clearance_hours INT,
  created_at TIMESTAMP DEFAULT NOW(),
  resolved_at TIMESTAMP NULL
);

-- Vehicles (point, updating frequently)
CREATE TABLE vehicles (
  id BIGSERIAL PRIMARY KEY,
  vehicle_number VARCHAR(20),
  vehicle_type VARCHAR(50),       -- truck, van, motorbike
  operator_id BIGINT REFERENCES users(id),
  current_location GEOMETRY(Point, 4326),
  current_lat DECIMAL(9, 6),
  current_lng DECIMAL(9, 6),
  status VARCHAR(30),             -- on_route, idle, delayed, maintenance
  last_update TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Road accessibility (cache of current status per segment)
CREATE TABLE road_accessibility_status (
  id BIGSERIAL PRIMARY KEY,
  road_segment_id BIGINT REFERENCES road_segments(id),
  status VARCHAR(30),             -- accessible, risky, blocked
  reason VARCHAR(255),            -- incident type causing status
  incident_report_id BIGINT REFERENCES incident_reports(id),
  last_updated TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

-- REGULAR TABLES

-- Districts
CREATE TABLE districts (
  id BIGSERIAL PRIMARY KEY,
  name VARCHAR(255),
  state VARCHAR(100),
  created_at TIMESTAMP DEFAULT NOW()
);

-- Users (roles)
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE,
  password_hash VARCHAR(255),
  name VARCHAR(255),
  role VARCHAR(50),               -- admin, district_officer, logistics_operator, field_officer
  assigned_district_id BIGINT REFERENCES districts(id),
  phone_number VARCHAR(20),
  created_at TIMESTAMP DEFAULT NOW()
);

-- Routes (cache of computed optimal routes)
CREATE TABLE computed_routes (
  id BIGSERIAL PRIMARY KEY,
  origin_village_id BIGINT REFERENCES villages(id),
  destination_village_id BIGINT REFERENCES villages(id),
  vehicle_type VARCHAR(50),
  distance_km DECIMAL(10, 2),
  estimated_time_minutes INT,
  risk_score DECIMAL(3, 2),
  route_geojson TEXT,             -- GeoJSON LineString
  segment_details JSONB,          -- array of segment objects
  alternate_routes JSONB,         -- array of alternate route objects
  computed_at TIMESTAMP,
  ttl_minutes INT DEFAULT 120,    -- Cache validity
  created_at TIMESTAMP DEFAULT NOW()
);

-- Alerts
CREATE TABLE alerts (
  id BIGSERIAL PRIMARY KEY,
  alert_type VARCHAR(50),         -- route_blocked, weather_warning, vehicle_delayed
  severity VARCHAR(20),           -- info, warning, critical
  incident_report_id BIGINT REFERENCES incident_reports(id),
  affected_vehicles BIGINT[],     -- array of vehicle IDs
  message TEXT,
  is_sent BOOLEAN DEFAULT FALSE,
  sent_at TIMESTAMP NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Create spatial indexes
CREATE INDEX idx_road_segments_geom ON road_segments USING GIST(geometry);
CREATE INDEX idx_villages_geom ON villages USING GIST(geometry);
CREATE INDEX idx_incidents_geom ON incident_reports USING GIST(location_geometry);
CREATE INDEX idx_vehicles_geom ON vehicles USING GIST(current_location);
```

---

## API Specifications

### Authentication

```javascript
// POST /auth/login
{
  "email": "officer@nerdisaster.gov.in",
  "password": "secure_password"
}

Response:
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": 1,
    "name": "Rajesh Kumar",
    "role": "field_officer",
    "assignedDistrict": "Imphal East"
  },
  "expiresIn": "24h"
}
```

### Route Optimization Endpoint

```javascript
// GET /routes/optimize?origin=24.9142,93.9497&dest=24.8170,92.9605&vehicle=truck

Response: { route, alternateRoutes }  [See earlier example]
```

### Incident Reporting

```javascript
// POST /incidents/report
{
  "officerId": "officer_123",
  "location": { "lat": 24.82, "lng": 93.96 },
  "type": "landslide",
  "severity": "high",
  "description": "...",
  "photos": ["base64_1", "base64_2"],
  "affectedRoads": ["NH-39", "State_Road_42"]
}

Response:
{
  "incidentId": "inc_56789",
  "status": "created",
  "affectedVehicles": 6,
  "alertsSent": true,
  "alternateRoutesGenerated": true
}
```

### Vehicle Tracking

```javascript
// POST /vehicles/track
{
  "vehicleId": "TRUCK-001",
  "lat": 24.8501,
  "lng": 93.9489,
  "speed": 45,  // km/h
  "timestamp": "2026-01-15T14:32:00Z"
}

// GET /vehicles/track?vehicleId=TRUCK-001
Response:
{
  "vehicleId": "TRUCK-001",
  "currentLocation": { "lat": 24.8501, "lng": 93.9489 },
  "status": "on_route",
  "routeId": "rte_12345",
  "progress": 62,  // percentage
  "eta": "2026-01-15T16:00:00Z",
  "speed": 45,
  "lastUpdate": "2026-01-15T14:32:00Z",
  "alerts": []
}
```

### Dashboard Connectivity

```javascript
// GET /districts/1/connectivity
Response:
{
  "districtId": 1,
  "districtName": "Imphal East",
  "status": {
    "accessible": 45,     // percentage of road network
    "atRisk": 35,
    "blocked": 20
  },
  "bottlenecks": [
    {
      "roadSegmentId": 123,
      "name": "NH-39 Ukhrul Section",
      "severity": "high",
      "reason": "Landslide",
      "affectedVehicles": 6,
      "estimatedClearance": "12 hours"
    }
  ],
  "recentIncidents": [...]
}
```

---

## Deployment & Infrastructure

### Containerized Architecture (Docker + Kubernetes)

```yaml
# docker-compose.yml for local development

version: '3.9'
services:
  postgres:
    image: postgis/postgis:latest
    environment:
      POSTGRES_DB: smartpath_ner
      POSTGRES_USER: smartpath_user
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  express-api:
    build: ./backend
    environment:
      NODE_ENV: development
      DB_URL: postgres://smartpath_user:${DB_PASSWORD}@postgres:5432/smartpath_ner
      REDIS_URL: redis://redis:6379
      PYTHON_ROUTING_SERVICE: http://routing-engine:5000
    ports:
      - "3000:3000"
    depends_on:
      - postgres
      - redis
      - routing-engine

  routing-engine:
    build: ./routing-service
    environment:
      FLASK_ENV: development
    ports:
      - "5000:5000"
    depends_on:
      - postgres

  react-dashboard:
    build: ./frontend/dashboard
    ports:
      - "3001:3000"
    environment:
      REACT_APP_API_URL: http://localhost:3000
```

### Production Deployment (AWS / GCP / Azure)

```
Cloud Provider: AWS/GCP/Azure
Container Registry: ECR/GCR/ACR
Orchestration: EKS/GKE/AKS (Kubernetes)

Components:
1. RDS PostgreSQL + PostGIS (managed database)
2. ElastiCache Redis (managed cache)
3. ECS/EKS for Express API, Routing microservice
4. CloudFront + S3 for dashboard static assets
5. CloudWatch for logs & monitoring
6. SNS/SQS for async notifications
7. VPC with private subnets for database security
```

---

## Implementation Phases

### Phase 1: Foundation (Weeks 1-3)

**Backend:**
- [x] Set up Express.js + PostgreSQL + PostGIS
- [x] Sequelize models (districts, villages, roads, vehicles, incidents, users)
- [x] JWT authentication & RBAC middleware
- [x] Basic REST endpoints (CRUD for incidents, vehicles, routes)
- [x] Database migrations & seed data (OSM road network for pilot district)

**Frontend:**
- [x] React setup with Leaflet.js
- [x] Map component rendering OSM data
- [x] District selector & real-time status panel (mock data initially)
- [x] Login / role-based navigation

**Python Engine:**
- [x] NetworkX graph builder from PostgreSQL road data
- [x] Dijkstra algorithm implementation
- [x] Basic weight calculation (distance + elevation)
- [x] Exposed as Flask API

**Deliverable:** MVP dashboard showing district accessibility + manual route planning

---

### Phase 2: Real-Time & Intelligence (Weeks 4-6)

**Backend:**
- [ ] Socket.io integration for real-time vehicle tracking
- [ ] Incident reporting API with geo-tagged photo upload
- [ ] Alert generation & push notification system
- [ ] Redis caching for frequently accessed routes
- [ ] Weather API integration (OpenWeather)

**Frontend:**
- [ ] Vehicle tracking dashboard with live markers
- [ ] Incident reporting form (mobile-friendly)
- [ ] Real-time alert notifications
- [ ] Enhanced map with incident layer & heatmap

**Python Engine:**
- [ ] Disruption prediction (rule-based: flood zones, landslide corridors, weather)
- [ ] Segment-level transport guidance (truck vs van vs motorbike)
- [ ] Alternate route generation
- [ ] Performance optimization (caching, parallel queries)

**Deliverable:** Incident reporting workflow + live vehicle tracking + intelligent routing

---

### Phase 3: Mobile & Offline Support (Weeks 7-8)

**Mobile App (React Native):**
- [ ] Offline-first architecture with AsyncStorage
- [ ] Offline map tiles cached locally
- [ ] Incident report queuing (sync on reconnect)
- [ ] Background GPS tracking
- [ ] Push notifications

**Frontend Enhancements:**
- [ ] Emergency response corridor pre-computation
- [ ] Supply chain bottleneck detection
- [ ] Multilingual UI (Hindi, regional languages)
- [ ] Voice/SMS alerting for low-literacy users (text-to-speech API)

**Backend:**
- [ ] Offline data synchronization endpoint
- [ ] Bulk export of map tiles & pre-computed routes
- [ ] SMS gateway integration (Twilio)

**Deliverable:** Fully functional mobile app + offline support + accessibility features

---

### Phase 4: Polish & Scaling (Weeks 9+)

- [ ] Performance testing & optimization
- [ ] Security audit & hardening
- [ ] Disaster recovery & backup strategy
- [ ] User training & documentation
- [ ] Expand to multiple NER districts
- [ ] Historical data analysis & ML model training

---

## Summary

**SMART PATH NER** is a geospatially-aware logistics platform tailored to NER's unique challenges:

1. **Terrain-Intelligent Routing**: Weighted graph model accounts for elevation, road quality, and historical disruptions—not just distance.
2. **Real-Time Monitoring**: Live vehicle tracking, incident reporting, and automated alerts.
3. **Scalable Backend**: Node.js + PostgreSQL + PostGIS for spatial queries, Python microservice for routing intelligence.
4. **Accessible Frontend**: React dashboards for admins + logistics operators, mobile app for field officers, offline-first support.
5. **Emergency-Ready**: Pre-computed disaster corridors, rapid rerouting, SMS alerts for low-connectivity areas.

The phased approach allows MVP deployment in 3-4 weeks, with intelligent features added incrementally.

---

**Next Steps:**
1. Finalize tech stack choices (Node/Python versions, deployment platform)
2. Acquire or simulate terrain data for pilot district
3. Set up development environment (Docker Compose)
4. Begin Phase 1 implementation
