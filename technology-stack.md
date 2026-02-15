# FloatChat Technology Stack

## Overview

FloatChat leverages a modern, scalable technology stack designed for high-performance data processing, intelligent querying, and interactive visualization of oceanographic data. The stack is optimized for both local development and cloud deployment on AWS.

---

## 1. Backend Technologies

### 1.1 Core Backend Framework

**FastAPI** (Primary Backend Framework)
- **Version**: 0.104+
- **Purpose**: RESTful API server for all client-server communication
- **Key Features**:
  - Async/await support for high concurrency
  - Automatic OpenAPI documentation
  - Pydantic data validation
  - High performance (comparable to Node.js and Go)
  - Type hints and IDE support
- **Use Cases**:
  - API endpoints for float data queries
  - Real-time data streaming
  - Forecast API integration
  - Heatwave detection endpoints
  - User authentication and authorization

**Example FastAPI Implementation**:
```python
from fastapi import FastAPI, Query, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from typing import Optional, List
import uvicorn

app = FastAPI(
    title="FloatChat API",
    description="AI-powered ocean data access API",
    version="1.0.0"
)

# CORS middleware for web UI
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/api/floats/{float_id}/temperature")
async def get_temperature_data(
    float_id: str,
    start_date: Optional[str] = None,
    end_date: Optional[str] = None,
    depth_min: Optional[float] = None,
    depth_max: Optional[float] = None
):
    """Retrieve temperature data for a specific float"""
    # Implementation here
    pass

@app.get("/api/floats/nearby")
async def get_nearby_floats(
    lat: float = Query(..., ge=-90, le=90),
    lon: float = Query(..., ge=-180, le=180),
    radius_km: int = Query(100, ge=1, le=1000)
):
    """Find floats within radius of coordinates"""
    # Implementation here
    pass

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### 1.2 ASGI Server

**Uvicorn**
- **Purpose**: Lightning-fast ASGI server for FastAPI
- **Features**:
  - HTTP/1.1 and WebSocket support
  - Async request handling
  - Auto-reload for development
  - Production-ready performance

**Alternative**: Gunicorn with Uvicorn workers for production
```bash
gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app
```

### 1.3 Data Processing

**Python 3.8+**
- **Core Libraries**:
  - `pandas`: Data manipulation and analysis
  - `numpy`: Numerical computing
  - `xarray`: NetCDF file handling
  - `netCDF4`: NetCDF file I/O
  - `scipy`: Scientific computing and statistics

**Data Processing Pipeline**:
```python
import xarray as xr
import pandas as pd
import numpy as np

def process_netcdf_to_dataframe(netcdf_path: str) -> pd.DataFrame:
    """Convert NetCDF to pandas DataFrame"""
    ds = xr.open_dataset(netcdf_path)
    
    # Extract dimensions
    n_prof = ds.dims['N_PROF']
    n_levels = ds.dims['N_LEVELS']
    
    # Flatten and create DataFrame
    data = []
    for prof in range(n_prof):
        for level in range(n_levels):
            record = {
                'float_id': ds['PLATFORM_NUMBER'][prof].values,
                'cycle': ds['CYCLE_NUMBER'][prof].values,
                'lat': ds['LATITUDE'][prof].values,
                'lon': ds['LONGITUDE'][prof].values,
                'date': ds['JULD'][prof].values,
                'pressure': ds['PRES'][prof, level].values,
                'temperature': ds['TEMP'][prof, level].values,
                'salinity': ds['PSAL'][prof, level].values,
            }
            data.append(record)
    
    return pd.DataFrame(data)
```

---

## 2. Database Technologies

### 2.1 Relational Database

**PostgreSQL 14+**
- **Purpose**: Primary structured data storage
- **Extensions**:
  - **PostGIS**: Geospatial queries and indexing
  - **pg_trgm**: Fuzzy text search
  - **btree_gin**: Multi-column indexing
- **Features**:
  - ACID compliance
  - Complex queries with JOINs
  - Geospatial indexing (radius searches)
  - Time-series optimization
  - Connection pooling

**Database Driver**: `psycopg2` or `asyncpg` (async)
```python
import asyncpg
from typing import List, Dict

class DatabaseManager:
    def __init__(self, connection_string: str):
        self.connection_string = connection_string
        self.pool = None
    
    async def connect(self):
        self.pool = await asyncpg.create_pool(
            self.connection_string,
            min_size=5,
            max_size=20
        )
    
    async def get_temperature_data(
        self, 
        float_id: str, 
        start_date: str, 
        end_date: str
    ) -> List[Dict]:
        async with self.pool.acquire() as conn:
            query = """
                SELECT juld, pres, temp_adjusted, temp_qc
                FROM argo_data
                WHERE float_id = $1
                AND juld BETWEEN $2 AND $3
                AND temp_qc IN ('1', '2', '5')
                ORDER BY juld, pres
            """
            rows = await conn.fetch(query, float_id, start_date, end_date)
            return [dict(row) for row in rows]
```

### 2.2 Vector Database

**Pinecone**
- **Purpose**: Semantic search and similarity queries
- **Configuration**:
  - Dimension: 384 (sentence-transformers)
  - Metric: Cosine similarity
  - Namespaces: `bgc-floats`, `non-bgc-floats`
- **Features**:
  - Fast vector similarity search
  - Metadata filtering
  - Hybrid search (vector + metadata)
  - Scalable to billions of vectors

**Python Client**: `pinecone-client`
```python
import pinecone
from sentence_transformers import SentenceTransformer

class VectorSearchManager:
    def __init__(self, api_key: str, environment: str, index_name: str):
        pinecone.init(api_key=api_key, environment=environment)
        self.index = pinecone.Index(index_name)
        self.encoder = SentenceTransformer('all-MiniLM-L6-v2')
    
    def semantic_search(self, query: str, top_k: int = 10, filters: dict = None):
        # Generate query embedding
        query_vector = self.encoder.encode(query).tolist()
        
        # Search Pinecone
        results = self.index.query(
            vector=query_vector,
            top_k=top_k,
            filter=filters,
            include_metadata=True
        )
        
        return results['matches']
```

---

## 3. AI & Machine Learning

### 3.1 Natural Language Processing

**LangGraph**
- **Purpose**: Conversational AI workflow orchestration
- **Features**:
  - State machine for conversation flow
  - Tool calling and chaining
  - Context management
  - Error handling and retries

**LangChain**
- **Purpose**: LLM integration and prompt management
- **Components**:
  - Prompt templates
  - Output parsers
  - Memory management
  - Chain composition

### 3.2 Embeddings

**Sentence Transformers**
- **Model**: `all-MiniLM-L6-v2`
- **Dimension**: 384
- **Purpose**: Generate embeddings for semantic search
- **Performance**: Fast inference, good quality

### 3.3 Forecasting Models

**TensorFlow / Keras**
- **Purpose**: LSTM models for time-series forecasting
- **Models**:
  - Temperature forecasting (7-day, 30-day)
  - Salinity forecasting
- **Deployment**: AWS SageMaker endpoints

**Scikit-learn**
- **Purpose**: Statistical analysis and preprocessing
- **Use Cases**:
  - Data normalization
  - Feature engineering
  - Model evaluation metrics

---

## 4. Frontend Technologies

### 4.1 Core Framework

**React 18**
- **Purpose**: UI component library
- **Features**:
  - Concurrent rendering
  - Automatic batching
  - Server components (future)
  - Hooks for state management

**TypeScript**
- **Purpose**: Type-safe JavaScript
- **Benefits**:
  - Compile-time error detection
  - Better IDE support
  - Self-documenting code
  - Refactoring safety

**Vite**
- **Purpose**: Build tool and dev server
- **Features**:
  - Lightning-fast HMR (Hot Module Replacement)
  - Optimized production builds
  - Plugin ecosystem
  - Native ESM support

### 4.2 Visualization Libraries

#### **Plotly.js / React-Plotly.js**
- **Purpose**: Interactive scientific charts and graphs
- **Use Cases**:
  - Time-series line charts (temperature, salinity over time)
  - Depth profile plots (vertical ocean profiles)
  - 3D scatter plots (multi-parameter visualization)
  - Heatmaps (correlation matrices)
  - Contour plots (spatial temperature distributions)
  - Box plots (statistical distributions)
- **Key Features**:
  - Highly interactive (zoom, pan, hover)
  - Export to PNG/SVG
  - Responsive design
  - WebGL acceleration for large datasets
  - Built-in statistical charts

**Example Implementation**:
```typescript
import Plot from 'react-plotly.js';

interface TimeSeriesChartProps {
  data: {
    timestamps: string[];
    temperatures: number[];
    forecast?: number[];
  };
}

export const TimeSeriesChart: React.FC<TimeSeriesChartProps> = ({ data }) => {
  return (
    <Plot
      data={[
        {
          x: data.timestamps,
          y: data.temperatures,
          type: 'scatter',
          mode: 'lines+markers',
          name: 'Observed',
          line: { color: '#2563eb' },
        },
        {
          x: data.timestamps,
          y: data.forecast,
          type: 'scatter',
          mode: 'lines',
          name: 'Forecast',
          line: { color: '#dc2626', dash: 'dash' },
        },
      ]}
      layout={{
        title: 'Temperature Time Series',
        xaxis: { title: 'Date' },
        yaxis: { title: 'Temperature (°C)' },
        hovermode: 'x unified',
        showlegend: true,
      }}
      config={{
        responsive: true,
        displayModeBar: true,
        toImageButtonOptions: {
          format: 'png',
          filename: 'temperature_timeseries',
        },
      }}
    />
  );
};
```

**Depth Profile Example**:
```typescript
export const DepthProfileChart: React.FC<{ data: ProfileData }> = ({ data }) => {
  return (
    <Plot
      data={[
        {
          x: data.temperature,
          y: data.depth,
          type: 'scatter',
          mode: 'lines+markers',
          name: 'Temperature',
          line: { color: '#ef4444' },
        },
        {
          x: data.salinity,
          y: data.depth,
          type: 'scatter',
          mode: 'lines+markers',
          name: 'Salinity',
          yaxis: 'y',
          xaxis: 'x2',
          line: { color: '#3b82f6' },
        },
      ]}
      layout={{
        title: 'Ocean Depth Profile',
        xaxis: { title: 'Temperature (°C)' },
        xaxis2: { title: 'Salinity (PSU)', overlaying: 'x', side: 'top' },
        yaxis: { title: 'Depth (m)', autorange: 'reversed' },
        showlegend: true,
      }}
    />
  );
};
```

#### **Leaflet / React-Leaflet**
- **Purpose**: Interactive mapping and geospatial visualization
- **Use Cases**:
  - Float location markers on world map
  - Geospatial clustering of dense float regions
  - Marine heatwave overlay (colored zones)
  - Temperature gradient layers
  - Region boundaries (Arabian Sea, Bay of Bengal, etc.)
  - User interaction (click, hover, draw)
  - Route visualization (float trajectories)
- **Key Features**:
  - Lightweight (38KB gzipped)
  - Mobile-friendly
  - Plugin ecosystem
  - Custom markers and popups
  - GeoJSON support
  - Tile layer customization

**Example Implementation**:
```typescript
import { MapContainer, TileLayer, Marker, Popup, CircleMarker } from 'react-leaflet';
import MarkerClusterGroup from 'react-leaflet-cluster';
import L from 'leaflet';

interface FloatMapProps {
  floats: Array<{
    id: string;
    lat: number;
    lon: number;
    temperature: number;
    hasBGC: boolean;
  }>;
  heatwaves?: Array<{
    lat: number;
    lon: number;
    intensity: number;
  }>;
}

export const FloatMap: React.FC<FloatMapProps> = ({ floats, heatwaves }) => {
  // Custom marker icon
  const floatIcon = new L.Icon({
    iconUrl: '/float-marker.png',
    iconSize: [25, 25],
  });

  return (
    <MapContainer
      center={[15, 70]}
      zoom={5}
      style={{ height: '600px', width: '100%' }}
    >
      <TileLayer
        url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
        attribution='&copy; OpenStreetMap contributors'
      />

      {/* Float markers with clustering */}
      <MarkerClusterGroup>
        {floats.map((float) => (
          <Marker
            key={float.id}
            position={[float.lat, float.lon]}
            icon={floatIcon}
          >
            <Popup>
              <div>
                <h3>Float {float.id}</h3>
                <p>Temperature: {float.temperature}°C</p>
                <p>Type: {float.hasBGC ? 'BGC' : 'Standard'}</p>
              </div>
            </Popup>
          </Marker>
        ))}
      </MarkerClusterGroup>

      {/* Marine heatwave overlay */}
      {heatwaves?.map((hw, idx) => (
        <CircleMarker
          key={idx}
          center={[hw.lat, hw.lon]}
          radius={hw.intensity * 10}
          pathOptions={{
            color: 'red',
            fillColor: '#ff0000',
            fillOpacity: 0.3,
          }}
        >
          <Popup>
            <div>
              <h3>Marine Heatwave</h3>
              <p>Intensity: +{hw.intensity.toFixed(1)}°C</p>
            </div>
          </Popup>
        </CircleMarker>
      ))}
    </MapContainer>
  );
};
```

**Leaflet Plugins**:
- `leaflet.heat`: Heatmap layer for temperature gradients
- `leaflet-draw`: Drawing tools for region selection
- `leaflet-fullscreen`: Fullscreen map control
- `react-leaflet-cluster`: Marker clustering

#### **D3.js** (Supplementary)
- **Purpose**: Custom, complex visualizations
- **Use Cases**:
  - Custom network graphs
  - Sankey diagrams (data flow)
  - Force-directed graphs
  - Advanced animations

### 4.3 State Management

**Redux Toolkit**
- **Purpose**: Global state management
- **Features**:
  - Simplified Redux setup
  - Built-in immutability
  - DevTools integration
  - Async thunk support

**React Query (TanStack Query)**
- **Purpose**: Server state management
- **Features**:
  - Automatic caching
  - Background refetching
  - Optimistic updates
  - Pagination support

### 4.4 UI Component Library

**Tailwind CSS**
- **Purpose**: Utility-first CSS framework
- **Benefits**:
  - Rapid UI development
  - Consistent design system
  - Small bundle size
  - Responsive design utilities

**shadcn/ui** (Optional)
- **Purpose**: Pre-built accessible components
- **Components**:
  - Buttons, inputs, modals
  - Dropdowns, tooltips
  - Data tables
  - Form components

### 4.5 API Client

**Axios**
- **Purpose**: HTTP client for API requests
- **Features**:
  - Request/response interceptors
  - Automatic JSON transformation
  - Error handling
  - Request cancellation

```typescript
import axios from 'axios';

const apiClient = axios.create({
  baseURL: 'http://localhost:8000/api',
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('auth_token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Response interceptor
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Handle unauthorized
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

---

## 5. Cloud Infrastructure (AWS)

### 5.1 Compute

**AWS Lambda**
- **Purpose**: Serverless function execution
- **Runtime**: Python 3.11
- **Use Cases**:
  - Data ingestion
  - Data processing
  - API endpoints
  - Scheduled tasks

**Amazon ECS / Fargate** (Alternative)
- **Purpose**: Container orchestration for FastAPI
- **Benefits**:
  - Long-running processes
  - WebSocket support
  - More control over environment

### 5.2 Storage

**Amazon S3**
- **Purpose**: Object storage for data lake
- **Buckets**:
  - Raw NetCDF files
  - Processed CSV files
  - Static website hosting
  - Backups

**Amazon RDS (PostgreSQL)**
- **Purpose**: Managed relational database
- **Configuration**:
  - Multi-AZ deployment
  - Automated backups
  - Read replicas

### 5.3 API & Networking

**Amazon API Gateway**
- **Purpose**: RESTful API management
- **Features**:
  - Rate limiting
  - API keys
  - CORS configuration
  - Request validation

**Amazon CloudFront**
- **Purpose**: CDN for global content delivery
- **Benefits**:
  - Low latency
  - Edge caching
  - DDoS protection

### 5.4 Machine Learning

**Amazon SageMaker**
- **Purpose**: ML model hosting
- **Features**:
  - Real-time inference endpoints
  - Batch transform jobs
  - Model versioning
  - Auto-scaling

### 5.5 Monitoring

**Amazon CloudWatch**
- **Purpose**: Logging and monitoring
- **Metrics**:
  - Lambda invocations
  - API latency
  - Database performance
  - Custom application metrics

**AWS X-Ray**
- **Purpose**: Distributed tracing
- **Benefits**:
  - Request flow visualization
  - Performance bottleneck identification

---

## 6. Development Tools

### 6.1 Version Control

**Git + GitHub**
- **Purpose**: Source code management
- **Workflows**:
  - Feature branches
  - Pull requests
  - Code reviews
  - CI/CD integration

### 6.2 Package Management

**Python**: `pip` + `virtualenv` or `conda`
**JavaScript**: `npm` or `yarn`

### 6.3 Code Quality

**Python**:
- `black`: Code formatting
- `flake8`: Linting
- `mypy`: Type checking
- `pytest`: Testing

**TypeScript**:
- `ESLint`: Linting
- `Prettier`: Code formatting
- `Jest`: Unit testing
- `React Testing Library`: Component testing

### 6.4 API Documentation

**Swagger UI / ReDoc**
- **Purpose**: Interactive API documentation
- **Features**:
  - Auto-generated from FastAPI
  - Try-it-out functionality
  - Schema visualization

---

## 7. Complete Technology Stack Summary

### Backend Stack
```
FastAPI (Web Framework)
├── Uvicorn (ASGI Server)
├── PostgreSQL (Relational DB)
│   └── PostGIS (Geospatial Extension)
├── Pinecone (Vector DB)
├── Python 3.8+
│   ├── pandas (Data Processing)
│   ├── numpy (Numerical Computing)
│   ├── xarray (NetCDF Handling)
│   ├── psycopg2 / asyncpg (DB Driver)
│   ├── sentence-transformers (Embeddings)
│   └── TensorFlow/Keras (ML Models)
└── LangGraph + LangChain (AI Orchestration)
```

### Frontend Stack
```
React 18 + TypeScript
├── Vite (Build Tool)
├── Plotly.js (Scientific Charts)
│   ├── Time-series plots
│   ├── Depth profiles
│   ├── 3D visualizations
│   └── Statistical charts
├── Leaflet (Interactive Maps)
│   ├── Float markers
│   ├── Heatwave overlays
│   ├── Clustering
│   └── GeoJSON layers
├── Redux Toolkit (State Management)
├── React Query (Server State)
├── Axios (HTTP Client)
└── Tailwind CSS (Styling)
```

### Cloud Stack (AWS)
```
AWS Cloud Platform
├── Lambda (Serverless Compute)
├── API Gateway (API Management)
├── RDS PostgreSQL (Managed Database)
├── S3 (Object Storage)
├── SageMaker (ML Hosting)
├── CloudFront (CDN)
├── CloudWatch (Monitoring)
└── X-Ray (Tracing)
```

---

## 8. Installation & Setup

### Backend Dependencies
```bash
# Create virtual environment
python -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install fastapi uvicorn[standard]
pip install psycopg2-binary asyncpg
pip install pinecone-client sentence-transformers
pip install pandas numpy xarray netCDF4
pip install python-dotenv pydantic
pip install langchain langgraph
pip install tensorflow scikit-learn
```

### Frontend Dependencies
```bash
# Initialize React + TypeScript + Vite project
npm create vite@latest floatchat-ui -- --template react-ts

cd floatchat-ui

# Install core dependencies
npm install react react-dom
npm install react-router-dom

# Install visualization libraries
npm install plotly.js react-plotly.js
npm install leaflet react-leaflet
npm install @types/leaflet
npm install react-leaflet-cluster

# Install state management
npm install @reduxjs/toolkit react-redux
npm install @tanstack/react-query

# Install utilities
npm install axios
npm install tailwindcss postcss autoprefixer
npm install date-fns  # Date manipulation

# Development dependencies
npm install -D @types/react @types/react-dom
npm install -D eslint prettier
npm install -D @testing-library/react @testing-library/jest-dom
```

---

## 9. Why This Stack?

### Performance
- **FastAPI**: One of the fastest Python frameworks
- **Async/await**: Non-blocking I/O for high concurrency
- **Plotly WebGL**: Hardware-accelerated rendering for large datasets
- **Leaflet**: Lightweight mapping library (38KB)

### Developer Experience
- **TypeScript**: Type safety and better tooling
- **Vite**: Lightning-fast development server
- **FastAPI auto-docs**: Automatic API documentation
- **Hot reload**: Instant feedback during development

### Scalability
- **Serverless**: Auto-scaling with AWS Lambda
- **CDN**: Global content delivery with CloudFront
- **Database**: Horizontal scaling with read replicas
- **Caching**: ElastiCache for performance

### Ecosystem
- **Rich libraries**: Extensive Python and JavaScript ecosystems
- **Community support**: Large communities for troubleshooting
- **Documentation**: Excellent documentation for all technologies
- **Integrations**: Easy integration between components

---

## 10. Future Considerations

### Potential Additions
- **GraphQL**: Alternative to REST API (Apollo Server)
- **WebSockets**: Real-time data streaming
- **Redis**: Caching layer for frequent queries
- **Elasticsearch**: Full-text search capabilities
- **Kubernetes**: Container orchestration (alternative to Lambda)
- **Next.js**: Server-side rendering for SEO
- **Three.js**: 3D ocean visualizations

### Mobile Support
- **React Native**: Cross-platform mobile app
- **Progressive Web App (PWA)**: Offline capabilities

---

**This technology stack provides a solid foundation for building a scalable, performant, and user-friendly ocean data platform.**
