# Software Architecture Design Document
## Autonomous Cement Plant GenAI Platform - PoC

**Version:** 1.0  
**Date:** October 22, 2025  
**Status:** Design Phase

---

## 1. Executive Summary

This document defines the **software architecture** for the Autonomous Cement Plant GenAI Platform PoC. It details the code structure, module responsibilities, API contracts, deployment strategy, and development workflow for a serverless, event-driven system built entirely on Google Cloud Platform.

### Architecture Principles
- **Serverless-First**: Zero idle costs, pay-per-use pricing model
- **Event-Driven**: Asynchronous communication via Pub/Sub
- **Microservices**: Loosely coupled, independently deployable services
- **API-First**: Well-defined contracts between all components
- **Infrastructure as Code**: Reproducible deployments via Terraform/YAML
- **Security by Design**: Authentication, authorization, and encryption at every layer

---

## 2. System Architecture Overview

### 2.1 High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │   React SPA (Firebase Hosting)                           │  │
│  │   - Real-time Dashboard    - NL Query Interface          │  │
│  │   - Alert Feed             - Optimization UI             │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │ HTTPS + JWT
┌────────────────────────────▼────────────────────────────────────┐
│                   API GATEWAY LAYER                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │   Cloud API Gateway + Firebase Auth                      │  │
│  │   - Route Management    - Token Validation               │  │
│  │   - Rate Limiting       - CORS Configuration             │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │ Internal Routes
┌────────────────────────────▼────────────────────────────────────┐
│               APPLICATION SERVICES LAYER                        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐ │
│  │ Optimization    │  │ Analytics       │  │ Quality        │ │
│  │ Service         │  │ Service         │  │ Service        │ │
│  │ (Cloud Run)     │  │ (Cloud Run)     │  │ (Cloud Run)    │ │
│  └────────┬────────┘  └────────┬────────┘  └────────┬───────┘ │
└───────────┼────────────────────┼────────────────────┼──────────┘
            │                    │                    │
            ├────────────────────┴────────────────────┘
            │
┌───────────▼─────────────────────────────────────────────────────┐
│                     AI/ML SERVICES LAYER                        │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │   Vertex AI Gemini API (Pay-per-use)                     │  │
│  │   - Process Optimization  - NL to SQL  - Text Generation │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                  DATA PROCESSING LAYER                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐ │
│  │ Stream          │  │ Alert           │  │ Metrics        │ │
│  │ Processor       │  │ Handler         │  │ Aggregator     │ │
│  │ (Cloud Func)    │  │ (Cloud Func)    │  │ (Cloud Func)   │ │
│  └────────┬────────┘  └────────┬────────┘  └────────┬───────┘ │
└───────────┼────────────────────┼────────────────────┼──────────┘
            │                    │                    │
            │         ┌──────────▼────────────┐       │
            │         │   Pub/Sub Topics      │       │
            │         │ - sensor-data-raw     │       │
            └─────────│ - alerts-critical     │───────┘
                      │ - optimization-results│
                      └───────────────────────┘
                                 ▲
┌────────────────────────────────┴────────────────────────────────┐
│                   DATA INGESTION LAYER                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │   Cloud Scheduler → Data Simulator (Cloud Function)      │  │
│  │   - Generates sensor data    - Publishes to Pub/Sub      │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                     DATA STORAGE LAYER                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │  BigQuery    │  │  Firestore   │  │  Cloud Storage       │ │
│  │  (Analytics) │  │  (Real-time) │  │  (Blobs/Archives)    │ │
│  └──────────────┘  └──────────────┘  └──────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Component Architecture

### 3.1 Data Ingestion Layer

#### 3.1.1 Data Simulator (Cloud Function)
**Purpose**: Generate realistic sensor data and publish to Pub/Sub

**Technology Stack**:
- Runtime: Python 3.11
- Trigger: Cloud Scheduler (HTTP)
- Libraries: `pandas`, `google-cloud-pubsub`, `numpy`

**Code Structure**:
```
/data-simulator/
├── main.py                    # Entry point, HTTP handler
├── generators/
│   ├── sensor_generator.py    # Sensor reading generation logic
│   ├── material_generator.py  # Raw material batch generation
│   └── patterns.py            # Realistic data patterns (sinusoidal, noise)
├── config.py                  # Configuration (pub/sub topics, ranges)
├── requirements.txt
└── README.md
```

**Key Functions**:
```python
def generate_sensor_reading(sensor_id: str, timestamp: datetime) -> dict:
    """Generate a single sensor reading with realistic patterns"""
    
def publish_to_pubsub(topic: str, data: dict) -> str:
    """Publish message to Pub/Sub topic"""
    
def simulate_batch(request) -> tuple:
    """Main HTTP handler triggered by Cloud Scheduler"""
```

**Configuration**:
```python
SENSORS = {
    'kiln_temp': {'min': 1400, 'max': 1500, 'pattern': 'sinusoidal'},
    'feed_rate': {'min': 180, 'max': 250, 'pattern': 'random_walk'},
    'energy_kwh': {'min': 95, 'max': 110, 'pattern': 'normal'},
    'quality_score': {'min': 0.85, 'max': 0.98, 'pattern': 'trending'}
}

MATERIAL_COMPOSITION = {
    'cao': {'mean': 44.0, 'std': 1.5},
    'sio2': {'mean': 22.0, 'std': 1.0},
    'al2o3': {'mean': 5.5, 'std': 0.5},
    'fe2o3': {'mean': 3.2, 'std': 0.3}
}
```

**Deployment**:
```bash
gcloud functions deploy data-simulator \
  --runtime python311 \
  --trigger-http \
  --entry-point simulate_batch \
  --set-env-vars PROJECT_ID=your-project-id
```

---

### 3.2 Data Processing Layer

#### 3.2.1 Stream Processor (Cloud Function)
**Purpose**: Process incoming sensor data, write to storage, detect anomalies

**Technology Stack**:
- Runtime: Node.js 20
- Trigger: Pub/Sub (`sensor-data-raw`)
- Libraries: `@google-cloud/bigquery`, `@google-cloud/firestore`, `@google-cloud/pubsub`

**Code Structure**:
```
/stream-processor/
├── index.js                   # Entry point, Pub/Sub handler
├── processors/
│   ├── validator.js           # Data validation and cleaning
│   ├── bigquery-writer.js     # BigQuery insert logic
│   ├── firestore-updater.js   # Firestore live state update
│   └── anomaly-detector.js    # Threshold-based anomaly detection
├── config.js                  # Thresholds, table names
├── package.json
└── README.md
```

**Key Functions**:
```javascript
async function processSensorData(message, context) {
  // Main Pub/Sub handler
  const data = parseMessage(message);
  const validated = validateData(data);
  
  await Promise.all([
    writeToBigQuery(validated),
    updateFirestore(validated),
    checkAnomaly(validated)
  ]);
}

function checkAnomaly(data) {
  // Threshold-based detection
  if (data.sensor_id === 'kiln_temp' && data.value > 1500) {
    publishAlert('CRITICAL', 'Kiln temperature exceeds safe limit');
  }
}
```

**Anomaly Thresholds**:
```javascript
const THRESHOLDS = {
  kiln_temp: { max: 1500, severity: 'CRITICAL' },
  quality_score: { min: 0.85, severity: 'CRITICAL' },
  feed_rate: { min: 1, max: 300, severity: 'WARNING' },
  energy_kwh: { max: 120, severity: 'WARNING' }
};
```

#### 3.2.2 Alert Handler (Cloud Function)
**Purpose**: Process alert events and write to Firestore

**Technology Stack**:
- Runtime: Node.js 20
- Trigger: Pub/Sub (`alerts-critical`)
- Libraries: `@google-cloud/firestore`

**Code Structure**:
```
/alert-handler/
├── index.js                   # Entry point
├── handlers/
│   ├── firestore-writer.js    # Write alert to /alerts collection
│   └── notification.js        # Future: send push notifications
├── config.js
└── package.json
```

#### 3.2.3 Metrics Aggregator (Cloud Function)
**Purpose**: Generate production_metrics table every 5 minutes

**Technology Stack**:
- Runtime: Python 3.11
- Trigger: Cloud Scheduler (HTTP)
- Libraries: `google-cloud-bigquery`

**Logic**:
```python
def aggregate_metrics(request):
    """
    1. Query sensor_readings for last 5 minutes
    2. Calculate production_rate_tph, energy_per_ton_kwh, quality_score
    3. Insert into production_metrics table
    """
```

---

### 3.3 Application Services Layer

All services are deployed on **Cloud Run** with the following common configuration:
- Auto-scaling: 0-10 instances
- CPU: 1 vCPU
- Memory: 512 MB
- Concurrency: 80 requests/instance
- Authentication: JWT validation via API Gateway

#### 3.3.1 Optimization Service
**Purpose**: Generate AI-powered optimization recommendations

**Technology Stack**:
- Runtime: Python 3.11 (Flask)
- Container: Cloud Run
- Libraries: `google-cloud-aiplatform`, `google-cloud-firestore`, `google-cloud-bigquery`

**Code Structure**:
```
/optimization-service/
├── main.py                    # Flask app entry point
├── services/
│   ├── gemini_client.py       # Vertex AI Gemini API client
│   ├── data_fetcher.py        # Fetch state from Firestore/BigQuery
│   └── recommendation.py      # Recommendation logic
├── prompts/
│   └── optimization_prompt.txt # Gemini prompt template
├── models/
│   └── schemas.py             # Pydantic models for I/O validation
├── Dockerfile
├── requirements.txt
└── README.md
```

**API Endpoints**:
```
POST /optimize
Request:
{
  "plant_id": "poc_plant_01",
  "constraints": ["DO_NOT_EXCEED_TEMP_1500"]
}

Response:
{
  "recommendation_id": "rec_abc123",
  "suggested_parameters": {
    "feed_rate_setpoint": 218.0,
    "fuel_mix_ratio": 0.45
  },
  "predicted_outcomes": {
    "energy_reduction_pct": 3.5,
    "quality_score_impact": "+0.01"
  },
  "explanation": "Reducing feed rate...",
  "timestamp": "2025-10-22T12:30:00Z"
}
```

**Gemini Integration**:
```python
def call_gemini(current_state: dict, recent_trends: list) -> dict:
    """
    Construct prompt with:
    - Current sensor readings
    - Material composition
    - Recent quality trend
    - Active constraints
    
    Call Gemini API with JSON response mode
    Parse and validate response
    """
    
    prompt = f"""
    You are an AI optimization expert for cement manufacturing.
    
    Current State: {json.dumps(current_state)}
    Recent Quality Trend: {json.dumps(recent_trends)}
    Constraints: {constraints}
    
    Generate a JSON response with these keys:
    - suggested_parameters: dict
    - predicted_outcomes: dict
    - explanation: string
    
    Respond ONLY with valid JSON, no markdown.
    """
    
    response = gemini_client.generate_content(prompt)
    return json.loads(response.text)
```

#### 3.3.2 Analytics Service
**Purpose**: Natural language to SQL query execution

**Technology Stack**:
- Runtime: Node.js 20 (Express)
- Container: Cloud Run
- Libraries: `@google-cloud/aiplatform`, `@google-cloud/bigquery`

**Code Structure**:
```
/analytics-service/
├── index.js                   # Express app
├── services/
│   ├── gemini-client.js       # Gemini API client
│   ├── sql-generator.js       # NL → SQL conversion
│   ├── sql-validator.js       # SQL safety checks
│   └── bigquery-executor.js   # Execute SQL on BigQuery
├── prompts/
│   └── text-to-sql.txt        # Prompt with table schemas
├── config.js
├── Dockerfile
└── package.json
```

**API Endpoints**:
```
POST /query
Request:
{
  "question": "What was the average kiln temperature yesterday?",
  "plant_id": "poc_plant_01"
}

Response:
{
  "question": "What was the average kiln temperature yesterday?",
  "sql": "SELECT AVG(value) as avg_temp FROM sensor_readings WHERE...",
  "results": [
    {"avg_temp": 1452.3}
  ],
  "summary": "The average kiln temperature yesterday was 1452.3°C",
  "execution_time_ms": 234
}
```

**Text-to-SQL Flow**:
```javascript
async function processQuery(question, plantId) {
  // 1. Send question + table schemas to Gemini
  const sql = await generateSQLFromNL(question);
  
  // 2. Validate SQL (check for dangerous operations)
  validateSQL(sql);
  
  // 3. Execute on BigQuery
  const results = await executeBigQuery(sql);
  
  // 4. Send results back to Gemini for summarization
  const summary = await summarizeResults(question, results);
  
  return { sql, results, summary };
}
```

**Table Schemas Prompt**:
```javascript
const SCHEMA_PROMPT = `
Available Tables:

1. sensor_readings
   Columns: timestamp, plant_id, sensor_id, value
   Description: Time-series sensor data
   
2. production_metrics
   Columns: timestamp, plant_id, production_rate_tph, 
            energy_per_ton_kwh, clinker_quality_score
   Description: Aggregated production KPIs
   
3. raw_material_batches
   Columns: analysis_timestamp, plant_id, batch_id, composition
   Description: Raw material chemical analysis
   
4. ai_recommendations
   Columns: recommendation_id, timestamp, parameters, 
            predicted_outcomes, implementation_status
   Description: AI optimization recommendations

Generate a valid BigQuery SQL query for: "{question}"
Return ONLY the SQL, no explanation.
`;
```

#### 3.3.3 Quality Service
**Purpose**: Simulated quality predictions (PoC placeholder)

**Technology Stack**:
- Runtime: Python 3.11 (FastAPI)
- Container: Cloud Run

**Code Structure**:
```
/quality-service/
├── main.py                    # FastAPI app
├── simulators/
│   └── defect_classifier.py   # Returns mock predictions
├── Dockerfile
└── requirements.txt
```

**API Endpoints**:
```
POST /quality/inspect
Request:
{
  "image_url": "gs://bucket/inspection_123.jpg"
}

Response:
{
  "defect_type": "Clinker Porosity",
  "severity": "High",
  "confidence": 0.91,
  "recommendation": "Adjust kiln temperature"
}
```

---

### 3.4 API Gateway Layer

**Technology**: Cloud API Gateway + OpenAPI Spec

**OpenAPI Configuration**:
```yaml
openapi: 3.0.0
info:
  title: Cement Plant API
  version: 1.0.0

security:
  - firebase: []

paths:
  /api/v1/optimize:
    post:
      x-google-backend:
        address: https://optimization-service-xyz.run.app
        jwt_audience: optimization-service
      responses:
        200:
          description: Optimization recommendation
          
  /api/v1/query:
    post:
      x-google-backend:
        address: https://analytics-service-xyz.run.app
        jwt_audience: analytics-service
      responses:
        200:
          description: Query results

securityDefinitions:
  firebase:
    type: oauth2
    authorizationUrl: https://securetoken.google.com/your-project-id
    flow: implicit
```

**Authentication Flow**:
```
1. User logs in via Firebase Auth (React app)
2. Firebase returns JWT token
3. React includes token in Authorization header
4. API Gateway validates JWT signature
5. If valid, forwards request to Cloud Run service
6. Cloud Run service can optionally validate claims
```

---

### 3.5 Presentation Layer

**Technology Stack**:
- Framework: React 18 + Vite
- Styling: Tailwind CSS
- State Management: React Context + Hooks
- Real-time: Firestore SDK (`onSnapshot`)
- Charts: Recharts
- Deployment: Firebase Hosting

**Code Structure**:
```
/frontend/
├── public/
├── src/
│   ├── main.jsx               # Entry point
│   ├── App.jsx                # Root component
│   ├── components/
│   │   ├── Dashboard.jsx      # Real-time dashboard
│   │   ├── AlertFeed.jsx      # Live alert list
│   │   ├── OptimizationPanel.jsx  # Optimization UI
│   │   ├── QueryInterface.jsx # NL query input
│   │   └── Charts/
│   │       ├── LineChart.jsx
│   │       └── GaugeChart.jsx
│   ├── services/
│   │   ├── api.js             # API client (axios)
│   │   ├── firestore.js       # Firestore listeners
│   │   └── auth.js            # Firebase Auth
│   ├── contexts/
│   │   ├── AuthContext.jsx    # User authentication state
│   │   └── PlantContext.jsx   # Plant data state
│   ├── hooks/
│   │   ├── useRealTimeData.js # Firestore onSnapshot hook
│   │   └── useAlerts.js       # Alert subscription hook
│   └── utils/
│       └── formatting.js      # Data formatting helpers
├── package.json
├── vite.config.js
└── tailwind.config.js
```

**Key React Components**:

```jsx
// useRealTimeData.js - Custom hook for Firestore real-time updates
import { useState, useEffect } from 'react';
import { doc, onSnapshot } from 'firebase/firestore';
import { db } from '../services/firestore';

export function useRealTimeData() {
  const [liveData, setLiveData] = useState(null);
  
  useEffect(() => {
    const unsubscribe = onSnapshot(
      doc(db, 'plant_status', 'live_data'),
      (doc) => {
        setLiveData(doc.data());
      }
    );
    
    return () => unsubscribe();
  }, []);
  
  return liveData;
}
```

```jsx
// Dashboard.jsx - Main dashboard component
export function Dashboard() {
  const liveData = useRealTimeData();
  const alerts = useAlerts();
  
  return (
    <div className="grid grid-cols-3 gap-4">
      <MetricCard 
        title="Kiln Temperature" 
        value={liveData?.kiln_temp} 
        unit="°C" 
      />
      <MetricCard 
        title="Feed Rate" 
        value={liveData?.feed_rate} 
        unit="TPH" 
      />
      <MetricCard 
        title="Quality Score" 
        value={liveData?.quality_score} 
        unit="" 
      />
      <AlertFeed alerts={alerts} />
      <TemperatureChart />
    </div>
  );
}
```

---

## 4. Data Storage Architecture

### 4.1 BigQuery Schema

**Dataset**: `cement_plant_poc`

**Tables**:

```sql
-- 1. sensor_readings (High-frequency time-series)
CREATE TABLE sensor_readings (
  timestamp TIMESTAMP NOT NULL,
  plant_id STRING NOT NULL,
  sensor_id STRING NOT NULL,
  value FLOAT64
)
PARTITION BY DATE(timestamp)
CLUSTER BY plant_id, sensor_id
OPTIONS(
  partition_expiration_days=90,
  description="Raw sensor telemetry data"
);

-- 2. raw_material_batches (Hourly material analysis)
CREATE TABLE raw_material_batches (
  analysis_timestamp TIMESTAMP NOT NULL,
  plant_id STRING NOT NULL,
  batch_id STRING NOT NULL,
  composition STRUCT<
    cao FLOAT64,
    sio2 FLOAT64,
    al2o3 FLOAT64,
    fe2o3 FLOAT64
  >
)
PARTITION BY DATE(analysis_timestamp)
OPTIONS(description="Raw material chemical composition");

-- 3. production_metrics (Aggregated KPIs)
CREATE TABLE production_metrics (
  timestamp TIMESTAMP NOT NULL,
  plant_id STRING NOT NULL,
  production_rate_tph FLOAT64,
  energy_per_ton_kwh FLOAT64,
  clinker_quality_score FLOAT64
)
PARTITION BY DATE(timestamp)
OPTIONS(description="Aggregated production metrics");

-- 4. ai_recommendations (AI recommendation audit trail)
CREATE TABLE ai_recommendations (
  recommendation_id STRING NOT NULL,
  timestamp TIMESTAMP NOT NULL,
  plant_id STRING NOT NULL,
  parameters JSON,
  predicted_outcomes JSON,
  implementation_status STRING,
  actual_outcomes JSON
)
PARTITION BY DATE(timestamp)
OPTIONS(description="AI-generated recommendations");
```

### 4.2 Firestore Schema

**Database**: `(default)`

**Collections**:

```
/plant_status/
  live_data (single document)
  {
    kiln_temp: 1452.1,
    feed_rate: 220.8,
    energy_kwh: 101.3,
    quality_score: 0.91,
    material_cao: 44.5,
    last_update: "2025-10-22T12:30:05Z"
  }

/plants/
  poc_plant_01
  {
    name: "Mumbai PoC Plant",
    location: "Mumbai, India",
    type: "Integrated (Dry Process)",
    production_capacity_tpd: 5000,
    commissioned_date: "2024-01-15"
  }

/alerts/
  {alert_id} (auto-generated)
  {
    timestamp: "2025-10-22T12:32:10Z",
    severity: "CRITICAL",
    message: "Kiln temperature over 1500°C!",
    sensor_id: "kiln_temp",
    value: 1502.5,
    status: "new"  // or "acknowledged"
  }

/users/
  {user_id}
  {
    email: "user@example.com",
    display_name: "John Doe",
    role: "operator",
    created_at: "2025-10-20T10:00:00Z"
  }
```

### 4.3 Cloud Storage Buckets

```
gs://{project-id}-simulator-data/
  └── kaggle_cement_data.csv (if needed in future)

gs://{project-id}-quality-images/
  └── inspections/
      ├── 2025-10-22/
      │   ├── inspection_001.jpg
      │   └── inspection_002.jpg

gs://{project-id}-reports/
  └── optimization-reports/
      └── 2025-10-22/
          └── report_abc123.pdf
```

---

## 5. API Contracts

### 5.1 REST API Standards

**Base URL**: `https://api.cement-plant-poc.example.com/api/v1`

**Authentication**: Bearer token (JWT from Firebase)
```
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Standard Response Format**:
```json
{
  "success": true,
  "data": { ... },
  "error": null,
  "timestamp": "2025-10-22T12:30:00Z"
}
```

**Error Response**:
```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "INVALID_REQUEST",
    "message": "Missing required field: plant_id"
  },
  "timestamp": "2025-10-22T12:30:00Z"
}
```

### 5.2 Endpoint Specifications

#### POST /optimize
Generate optimization recommendation

**Request**:
```json
{
  "plant_id": "poc_plant_01",
  "constraints": ["DO_NOT_EXCEED_TEMP_1500", "MAINTAIN_QUALITY_ABOVE_0.90"]
}
```

**Response**:
```json
{
  "success": true,
  "data": {
    "recommendation_id": "rec_abc123",
    "timestamp": "2025-10-22T12:30:00Z",
    "suggested_parameters": {
      "feed_rate_setpoint": 218.0,
      "fuel_mix_ratio": 0.45
    },
    "predicted_outcomes": {
      "energy_reduction_pct": 3.5,
      "quality_score_impact": "+0.01",
      "confidence": 0.88
    },
    "explanation": "Reducing feed rate slightly..."
  }
}
```

#### POST /query
Execute natural language query

**Request**:
```json
{
  "question": "What was the average kiln temperature yesterday?",
  "plant_id": "poc_plant_01"
}
```

**Response**:
```json
{
  "success": true,
  "data": {
    "question": "What was the average kiln temperature yesterday?",
    "sql": "SELECT AVG(value) as avg_temp FROM sensor_readings WHERE sensor_id='kiln_temp' AND DATE(timestamp) = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)",
    "results": [
      {"avg_temp": 1452.3}
    ],
    "summary": "The average kiln temperature yesterday was 1452.3°C",
    "execution_time_ms": 234,
    "row_count": 1
  }
}
```

#### GET /alerts
Retrieve recent alerts

**Query Parameters**:
- `plant_id` (required)
- `status` (optional): "new" | "acknowledged" | "all"
- `limit` (optional): default 50

**Response**:
```json
{
  "success": true,
  "data": {
    "alerts": [
      {
        "alert_id": "alert_xyz",
        "timestamp": "2025-10-22T12:32:10Z",
        "severity": "CRITICAL",
        "message": "Kiln temperature over 1500°C!",
        "sensor_id": "kiln_temp",
        "value": 1502.5,
        "status": "new"
      }
    ],
    "total": 1
  }
}
```

---

## 6. Deployment Architecture

### 6.1 Infrastructure as Code (Terraform)

**Directory Structure**:
```
/infrastructure/
├── main.tf                    # Provider and project config
├── variables.tf               # Input variables
├── outputs.tf                 # Output values
├── modules/
│   ├── bigquery/
│   │   └── main.tf           # BigQuery dataset and tables
│   ├── cloud-functions/
│   │   └── main.tf           # Cloud Functions deployment
│   ├── cloud-run/
│   │   └── main.tf           # Cloud Run services
│   ├── pubsub/
│   │   └── main.tf           # Pub/Sub topics and subscriptions
│   ├── firestore/
│   │   └── main.tf           # Firestore database
│   ├── api-gateway/
│   │   └── main.tf           # API Gateway config
│   └── iam/
│       └── main.tf           # Service accounts and permissions
└── environments/
    ├── dev.tfvars
    └── prod.tfvars
```

**Sample Terraform (Cloud Run Service)**:
```hcl
resource "google_cloud_run_service" "optimization_service" {
  name     = "optimization-service"
  location = var.region

  template {
    spec {
      containers {
        image = "gcr.io/${var.project_id}/optimization-service:latest"
        
        resources {
          limits = {
            cpu    = "1000m"
            memory = "512Mi"
          }
        }
        
        env {
          name  = "PROJECT_ID"
          value = var.project_id
        }
        
        env {
          name  = "GEMINI_MODEL"
          value = "gemini-pro"
        }
      }
      
      service_account_name = google_service_account.optimization_sa.email
    }
    
    metadata {
      annotations = {
        "autoscaling.knative.dev/minScale" = "0"
        "autoscaling.knative.dev/maxScale" = "10"
      }
    }
  }

  traffic {
    percent         = 100
    latest_revision = true
  }
}

resource "google_cloud_run_service_iam_member" "optimization_invoker" {
  service  = google_cloud_run_service.optimization_service.name
  location = google_cloud_run_service.optimization_service.location
  role     = "roles/run.invoker"
  member   = "serviceAccount:${google_service_account.api_gateway_sa.email}"
}
```

### 6.2 CI/CD Pipeline

**Technology**: Cloud Build + GitHub Actions

**GitHub Actions Workflow**:
```yaml
# .github/workflows/deploy.yml
name: Deploy to GCP

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  REGION: us-central1

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
      
      - name: Install dependencies
        run: npm ci
        working-directory: ./stream-processor
      
      - name: Run tests
        run: npm test
        working-directory: ./stream-processor
      
      - name: Run linter
        run: npm run lint
        working-directory: ./stream-processor

  build-and-deploy-functions:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
      
      - name: Deploy Data Simulator
        run: |
          gcloud functions deploy data-simulator \
            --runtime python311 \
            --trigger-http \
            --entry-point simulate_batch \
            --region $REGION \
            --set-env-vars PROJECT_ID=$PROJECT_ID
        working-directory: ./data-simulator
      
      - name: Deploy Stream Processor
        run: |
          gcloud functions deploy stream-processor \
            --runtime nodejs20 \
            --trigger-topic sensor-data-raw \
            --region $REGION \
            --set-env-vars PROJECT_ID=$PROJECT_ID
        working-directory: ./stream-processor

  build-and-deploy-services:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        service: [optimization-service, analytics-service, quality-service]
    
    steps:
      - uses: actions/checkout@v3
      
      - uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
      
      - name: Build Docker image
        run: |
          docker build -t gcr.io/$PROJECT_ID/${{ matrix.service }}:$GITHUB_SHA .
          docker tag gcr.io/$PROJECT_ID/${{ matrix.service }}:$GITHUB_SHA \
                     gcr.io/$PROJECT_ID/${{ matrix.service }}:latest
        working-directory: ./${{ matrix.service }}
      
      - name: Push to Container Registry
        run: |
          gcloud auth configure-docker
          docker push gcr.io/$PROJECT_ID/${{ matrix.service }}:$GITHUB_SHA
          docker push gcr.io/$PROJECT_ID/${{ matrix.service }}:latest
      
      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy ${{ matrix.service }} \
            --image gcr.io/$PROJECT_ID/${{ matrix.service }}:$GITHUB_SHA \
            --platform managed \
            --region $REGION \
            --allow-unauthenticated=false

  deploy-frontend:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
      
      - name: Install dependencies
        run: npm ci
        working-directory: ./frontend
      
      - name: Build
        run: npm run build
        working-directory: ./frontend
        env:
          VITE_API_URL: https://api.cement-plant-poc.example.com
          VITE_FIREBASE_API_KEY: ${{ secrets.FIREBASE_API_KEY }}
      
      - name: Deploy to Firebase Hosting
        uses: FirebaseExtended/action-hosting-deploy@v0
        with:
          repoToken: '${{ secrets.GITHUB_TOKEN }}'
          firebaseServiceAccount: '${{ secrets.FIREBASE_SERVICE_ACCOUNT }}'
          channelId: live
          projectId: ${{ env.PROJECT_ID }}
```

### 6.3 Environment Configuration

**Development Environment**:
```bash
# .env.development
PROJECT_ID=cement-plant-poc-dev
REGION=us-central1
PUBSUB_TOPIC_SENSOR_DATA=sensor-data-raw-dev
PUBSUB_TOPIC_ALERTS=alerts-critical-dev
BIGQUERY_DATASET=cement_plant_poc_dev
FIRESTORE_DATABASE=(default)
GEMINI_MODEL=gemini-pro
API_GATEWAY_URL=https://api-dev.cement-plant-poc.example.com
```

**Production Environment**:
```bash
# .env.production
PROJECT_ID=cement-plant-poc-prod
REGION=us-central1
PUBSUB_TOPIC_SENSOR_DATA=sensor-data-raw
PUBSUB_TOPIC_ALERTS=alerts-critical
BIGQUERY_DATASET=cement_plant_poc
FIRESTORE_DATABASE=(default)
GEMINI_MODEL=gemini-pro
API_GATEWAY_URL=https://api.cement-plant-poc.example.com
```

---

## 7. Security Architecture

### 7.1 Authentication & Authorization

**Authentication Flow**:
```
1. User opens React app
2. Redirected to Firebase Auth login
3. User authenticates (email/password or Google OAuth)
4. Firebase returns JWT token with claims:
   {
     "user_id": "abc123",
     "email": "user@example.com",
     "role": "operator",
     "plant_id": "poc_plant_01"
   }
5. React stores token in memory (not localStorage)
6. All API requests include: Authorization: Bearer {token}
7. API Gateway validates JWT signature and expiration
8. Cloud Run services can read claims for authorization
```

**Role-Based Access Control (RBAC)**:
```javascript
// Custom claims in Firebase Auth tokens
const ROLES = {
  OPERATOR: {
    permissions: ['view_dashboard', 'acknowledge_alerts']
  },
  ENGINEER: {
    permissions: ['view_dashboard', 'acknowledge_alerts', 'request_optimization', 'run_queries']
  },
  ADMIN: {
    permissions: ['*']
  }
};

// Middleware in Cloud Run services
function checkPermission(req, res, next) {
  const token = req.headers.authorization?.split('Bearer ')[1];
  const decoded = verifyToken(token);
  
  if (!decoded.role || !hasPermission(decoded.role, req.endpoint)) {
    return res.status(403).json({ error: 'Forbidden' });
  }
  
  next();
}
```

### 7.2 Service Account Architecture

**Service Accounts**:
```
1. data-simulator-sa@project.iam.gserviceaccount.com
   - Roles: pubsub.publisher, storage.objectViewer

2. stream-processor-sa@project.iam.gserviceaccount.com
   - Roles: bigquery.dataEditor, datastore.user, pubsub.publisher

3. optimization-service-sa@project.iam.gserviceaccount.com
   - Roles: aiplatform.user, bigquery.dataViewer, datastore.user

4. analytics-service-sa@project.iam.gserviceaccount.com
   - Roles: aiplatform.user, bigquery.user

5. api-gateway-sa@project.iam.gserviceaccount.com
   - Roles: run.invoker (for all Cloud Run services)
```

**Principle of Least Privilege**: Each service account has ONLY the permissions required for its specific function.

### 7.3 Network Security

**Cloud Run Configuration**:
```yaml
# All Cloud Run services configured with:
- Ingress: Internal and Cloud Load Balancing only
- Authentication: Require authentication
- VPC Connector: (optional for future VPC integration)
```

**API Gateway Security**:
```yaml
# Rate Limiting
quota:
  requests_per_minute: 100
  requests_per_day: 10000

# CORS Configuration
cors:
  allow_origins:
    - https://cement-plant-poc.web.app
  allow_methods:
    - GET
    - POST
  allow_headers:
    - Authorization
    - Content-Type
```

### 7.4 Data Security

**Encryption**:
- **At Rest**: All data in BigQuery, Firestore, Cloud Storage encrypted by default with Google-managed keys
- **In Transit**: All communication over HTTPS/TLS 1.3

**Sensitive Data Handling**:
```python
# Never log sensitive data
def log_request(data):
    safe_data = {k: v for k, v in data.items() if k not in ['token', 'api_key']}
    logger.info(f"Request: {safe_data}")

# Firestore security rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /plant_status/{document} {
      allow read: if request.auth != null;
      allow write: if false; // Only backend can write
    }
    
    match /alerts/{alertId} {
      allow read: if request.auth != null;
      allow update: if request.auth != null && 
                       request.resource.data.diff(resource.data).affectedKeys()
                       .hasOnly(['status']);
    }
    
    match /users/{userId} {
      allow read: if request.auth.uid == userId;
      allow write: if request.auth.uid == userId;
    }
  }
}
```

---

## 8. Observability & Monitoring

### 8.1 Logging Strategy

**Structured Logging**:
```javascript
// Standard log format for all services
const log = {
  timestamp: new Date().toISOString(),
  severity: 'INFO', // DEBUG, INFO, WARNING, ERROR, CRITICAL
  service: 'optimization-service',
  version: '1.0.0',
  trace_id: req.headers['x-cloud-trace-context'],
  user_id: req.user?.id,
  message: 'Optimization request received',
  data: {
    plant_id: 'poc_plant_01',
    execution_time_ms: 234
  }
};

console.log(JSON.stringify(log));
```

**Log Levels by Component**:
- **Data Simulator**: INFO (log every batch generated)
- **Stream Processor**: WARNING (only log anomalies and errors)
- **Cloud Run Services**: INFO (log all requests/responses)
- **Frontend**: ERROR only (sent to Cloud Logging via API)

### 8.2 Metrics & Dashboards

**Key Metrics to Monitor**:

**System Health**:
- Cloud Function execution count, duration, errors
- Cloud Run request count, latency (p50, p95, p99), errors
- Pub/Sub message publish/delivery rate, backlog
- BigQuery query count, bytes processed, slot utilization
- Firestore read/write operations

**Business Metrics**:
- Sensor data points ingested per minute
- Alerts generated per hour
- Optimization recommendations generated per day
- Natural language queries executed per day
- Average optimization response time

**Cloud Monitoring Dashboard Configuration**:
```yaml
# monitoring/dashboard.yaml
displayName: "Cement Plant PoC Dashboard"
gridLayout:
  widgets:
    - title: "Sensor Data Ingestion Rate"
      xyChart:
        dataSets:
          - timeSeriesQuery:
              timeSeriesFilter:
                filter: 'resource.type="cloud_function" resource.labels.function_name="data-simulator"'
                aggregation:
                  alignmentPeriod: 60s
                  perSeriesAligner: ALIGN_RATE
    
    - title: "Stream Processor Latency"
      xyChart:
        dataSets:
          - timeSeriesQuery:
              timeSeriesFilter:
                filter: 'resource.type="cloud_function" resource.labels.function_name="stream-processor"'
                aggregation:
                  alignmentPeriod: 60s
                  perSeriesAligner: ALIGN_MEAN
    
    - title: "Optimization Service Requests"
      xyChart:
        dataSets:
          - timeSeriesQuery:
              timeSeriesFilter:
                filter: 'resource.type="cloud_run_revision" resource.labels.service_name="optimization-service"'
```

### 8.3 Alerting Rules

**Critical Alerts** (PagerDuty/Email):
```yaml
# monitoring/alerts.yaml
displayName: "Critical System Alerts"
conditions:
  - displayName: "High Cloud Function Error Rate"
    conditionThreshold:
      filter: 'resource.type="cloud_function" metric.type="cloudfunctions.googleapis.com/function/execution_count" metric.labels.status!="ok"'
      aggregations:
        alignmentPeriod: 300s
        perSeriesAligner: ALIGN_RATE
      comparison: COMPARISON_GT
      thresholdValue: 0.1  # >10% error rate
      duration: 300s
  
  - displayName: "Pub/Sub Message Backlog"
    conditionThreshold:
      filter: 'resource.type="pubsub_subscription" metric.type="pubsub.googleapis.com/subscription/num_undelivered_messages"'
      comparison: COMPARISON_GT
      thresholdValue: 1000
      duration: 600s
  
  - displayName: "Optimization Service High Latency"
    conditionThreshold:
      filter: 'resource.type="cloud_run_revision" metric.type="run.googleapis.com/request_latencies"'
      aggregations:
        alignmentPeriod: 60s
        perSeriesAligner: ALIGN_DELTA
        crossSeriesReducer: REDUCE_PERCENTILE_95
      comparison: COMPARISON_GT
      thresholdValue: 5000  # 5 seconds
      duration: 300s

notificationChannels:
  - email: ops-team@example.com
  - pagerduty: service_key_abc123
```

### 8.4 Distributed Tracing

**Cloud Trace Integration**:
```python
# Python Cloud Function with tracing
from google.cloud import trace_v1
import google.cloud.logging

tracer = trace_v1.TraceServiceClient()

def stream_processor(event, context):
    trace_id = context.trace
    
    with tracer.span(name="process_sensor_data"):
        data = parse_message(event)
        
        with tracer.span(name="write_to_bigquery"):
            write_to_bigquery(data)
        
        with tracer.span(name="update_firestore"):
            update_firestore(data)
        
        with tracer.span(name="check_anomaly"):
            check_anomaly(data)
```

---

## 9. Testing Strategy

### 9.1 Unit Testing

**Data Simulator Tests**:
```python
# data-simulator/tests/test_generator.py
import unittest
from generators.sensor_generator import generate_sensor_reading

class TestSensorGenerator(unittest.TestCase):
    def test_kiln_temp_within_range(self):
        reading = generate_sensor_reading('kiln_temp', datetime.now())
        self.assertGreaterEqual(reading['value'], 1400)
        self.assertLessEqual(reading['value'], 1500)
    
    def test_reading_schema(self):
        reading = generate_sensor_reading('feed_rate', datetime.now())
        self.assertIn('timestamp', reading)
        self.assertIn('sensor_id', reading)
        self.assertIn('value', reading)
        self.assertIsInstance(reading['value'], float)
```

**Stream Processor Tests**:
```javascript
// stream-processor/tests/validator.test.js
const { validateData } = require('../processors/validator');

describe('Data Validator', () => {
  test('accepts valid sensor reading', () => {
    const data = {
      timestamp: '2025-10-22T12:30:00Z',
      plant_id: 'poc_plant_01',
      sensor_id: 'kiln_temp',
      value: 1450.5
    };
    
    expect(validateData(data)).toBeTruthy();
  });
  
  test('rejects out-of-range value', () => {
    const data = {
      timestamp: '2025-10-22T12:30:00Z',
      plant_id: 'poc_plant_01',
      sensor_id: 'kiln_temp',
      value: -100  // Invalid negative temperature
    };
    
    expect(() => validateData(data)).toThrow();
  });
});
```

### 9.2 Integration Testing

**Optimization Service Integration Test**:
```python
# optimization-service/tests/test_integration.py
import pytest
from unittest.mock import Mock, patch
from main import app

@pytest.fixture
def client():
    app.config['TESTING'] = True
    with app.test_client() as client:
        yield client

@patch('services.gemini_client.call_gemini')
@patch('services.data_fetcher.get_current_state')
def test_optimization_endpoint(mock_get_state, mock_gemini, client):
    # Mock Firestore response
    mock_get_state.return_value = {
        'kiln_temp': 1450,
        'feed_rate': 220
    }
    
    # Mock Gemini response
    mock_gemini.return_value = {
        'suggested_parameters': {'feed_rate_setpoint': 218},
        'predicted_outcomes': {'energy_reduction_pct': 3.5}
    }
    
    response = client.post('/optimize', json={
        'plant_id': 'poc_plant_01',
        'constraints': []
    })
    
    assert response.status_code == 200
    data = response.get_json()
    assert 'recommendation_id' in data
    assert data['suggested_parameters']['feed_rate_setpoint'] == 218
```

### 9.3 End-to-End Testing

**E2E Test Scenario**:
```javascript
// e2e-tests/optimization-flow.spec.js
const { test, expect } = require('@playwright/test');

test('complete optimization flow', async ({ page }) => {
  // 1. Login
  await page.goto('https://cement-plant-poc.web.app');
  await page.fill('[data-testid="email"]', 'test@example.com');
  await page.fill('[data-testid="password"]', 'password123');
  await page.click('[data-testid="login-button"]');
  
  // 2. Wait for dashboard to load
  await page.waitForSelector('[data-testid="dashboard"]');
  
  // 3. Verify real-time data is updating
  const tempBefore = await page.textContent('[data-testid="kiln-temp"]');
  await page.waitForTimeout(5000);
  const tempAfter = await page.textContent('[data-testid="kiln-temp"]');
  expect(tempBefore).not.toBe(tempAfter);
  
  // 4. Request optimization
  await page.click('[data-testid="optimize-button"]');
  await page.waitForSelector('[data-testid="recommendation"]');
  
  // 5. Verify recommendation appears
  const recommendation = await page.textContent('[data-testid="recommendation"]');
  expect(recommendation).toContain('feed_rate');
});
```

### 9.4 Load Testing

**Locust Load Test**:
```python
# load-tests/locustfile.py
from locust import HttpUser, task, between

class CementPlantUser(HttpUser):
    wait_time = between(1, 3)
    
    def on_start(self):
        # Login and get token
        response = self.client.post("/auth/login", json={
            "email": "test@example.com",
            "password": "password123"
        })
        self.token = response.json()['token']
    
    @task(3)
    def view_dashboard(self):
        self.client.get("/api/v1/current-state", headers={
            "Authorization": f"Bearer {self.token}"
        })
    
    @task(1)
    def request_optimization(self):
        self.client.post("/api/v1/optimize", 
            headers={"Authorization": f"Bearer {self.token}"},
            json={"plant_id": "poc_plant_01", "constraints": []}
        )
    
    @task(2)
    def natural_language_query(self):
        self.client.post("/api/v1/query",
            headers={"Authorization": f"Bearer {self.token}"},
            json={
                "question": "What was the average temperature today?",
                "plant_id": "poc_plant_01"
            }
        )

# Run: locust -f locustfile.py --host=https://api.cement-plant-poc.example.com
```

---

## 10. Development Workflow

### 10.1 Repository Structure

```
cement-plant-poc/
├── .github/
│   └── workflows/
│       ├── deploy.yml
│       └── test.yml
├── infrastructure/
│   ├── terraform/
│   └── scripts/
├── services/
│   ├── data-simulator/
│   ├── stream-processor/
│   ├── alert-handler/
│   ├── metrics-aggregator/
│   ├── optimization-service/
│   ├── analytics-service/
│   └── quality-service/
├── frontend/
│   ├── src/
│   ├── public/
│   └── tests/
├── shared/
│   ├── schemas/            # Shared data schemas
│   └── utils/              # Shared utility functions
├── docs/
│   ├── architecture.md
│   ├── api-reference.md
│   └── deployment-guide.md
├── monitoring/
│   ├── dashboards/
│   └── alerts/
├── tests/
│   ├── e2e/
│   └── load/
├── scripts/
│   ├── setup-project.sh
│   ├── deploy-all.sh
│   └── seed-data.sh
├── .gitignore
├── README.md
└── ARCHITECTURE.md
```

### 10.2 Git Branching Strategy

**Branch Model**:
```
main            # Production-ready code
├── develop     # Integration branch
│   ├── feature/simulator-improvements
│   ├── feature/nl-query-enhancement
│   └── bugfix/alert-duplicate-issue
└── hotfix/critical-security-patch
```

**Commit Message Convention**:
```
<type>(<scope>): <subject>

Types:
- feat: New feature
- fix: Bug fix
- docs: Documentation changes
- style: Code formatting
- refactor: Code restructuring
- test: Test additions/changes
- chore: Build/tooling changes

Examples:
feat(simulator): add sinusoidal pattern for temperature
fix(stream-processor): prevent duplicate BigQuery inserts
docs(api): update authentication flow diagram
```

### 10.3 Code Review Process

**Pull Request Template**:
```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Testing
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Manual testing completed

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Comments added for complex logic
- [ ] Documentation updated
- [ ] No new warnings generated
- [ ] Deployment tested in dev environment

## Screenshots (if applicable)

## Related Issues
Closes #123
```

### 10.4 Local Development Setup

**Prerequisites**:
```bash
# Install required tools
- Node.js 20+
- Python 3.11+
- Docker Desktop
- gcloud CLI
- Terraform 1.5+
```

**Setup Script**:
```bash
#!/bin/bash
# scripts/setup-local-dev.sh

echo "Setting up local development environment..."

# 1. Authenticate with GCP
gcloud auth login
gcloud auth application-default login
gcloud config set project cement-plant-poc-dev

# 2. Install Cloud Functions emulator
npm install -g @google-cloud/functions-framework

# 3. Install Python dependencies
cd data-simulator
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cd ..

# 4. Install Node.js dependencies
cd stream-processor
npm install
cd ..

cd frontend
npm install
cd ..

# 5. Set up Firestore emulator
firebase emulators:start --only firestore &

# 6. Create .env files
cat > services/optimization-service/.env.local <<EOF
PROJECT_ID=cement-plant-poc-dev
FIRESTORE_EMULATOR_HOST=localhost:8080
BIGQUERY_DATASET=cement_plant_poc_dev
GEMINI_MODEL=gemini-pro
EOF

echo "✅ Local development environment ready!"
echo "Run 'npm run dev' in each service directory to start"
```

---

## 11. Performance Optimization

### 11.1 BigQuery Optimization

**Query Best Practices**:
```sql
-- ❌ BAD: Full table scan
SELECT AVG(value) 
FROM sensor_readings 
WHERE sensor_id = 'kiln_temp';

-- ✅ GOOD: Partition pruning + clustering
SELECT AVG(value) 
FROM sensor_readings 
WHERE DATE(timestamp) = CURRENT_DATE()
  AND sensor_id = 'kiln_temp'
LIMIT 1000;

-- ✅ BEST: Materialized view for common queries
CREATE MATERIALIZED VIEW daily_avg_temps AS
SELECT 
  DATE(timestamp) as date,
  sensor_id,
  AVG(value) as avg_value
FROM sensor_readings
WHERE sensor_id IN ('kiln_temp', 'feed_rate')
GROUP BY date, sensor_id;
```

**Cost Control**:
```sql
-- Set maximum bytes billed for queries
SET @@query_budget.max_bytes_billed = 1073741824; -- 1 GB

-- Use table preview instead of SELECT *
SELECT * FROM sensor_readings LIMIT 10;
```

### 11.2 Firestore Optimization

**Indexing Strategy**:
```
# firestore.indexes.json
{
  "indexes": [
    {
      "collectionGroup": "alerts",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "plant_id", "order": "ASCENDING" },
        { "fieldPath": "timestamp", "order": "DESCENDING" },
        { "fieldPath": "status", "order": "ASCENDING" }
      ]
    }
  ]
}
```

**Read Optimization**:
```javascript
// ❌ BAD: Reading entire document when only need one field
const doc = await db.collection('plant_status').doc('live_data').get();
const temp = doc.data().kiln_temp;

// ✅ GOOD: Use field mask (not supported in Firestore, use denormalization instead)
// Solution: Store frequently accessed fields in separate documents
const tempDoc = await db.collection('plant_status').doc('kiln_temp_only').get();
```

### 11.3 Cloud Run Optimization

**Cold Start Reduction**:
```dockerfile
# Use distroless base images
FROM gcr.io/distroless/python3-debian11

# Multi-stage build to reduce image size
FROM python:3.11-slim as builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

FROM python:3.11-slim
COPY --from=builder /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "main.py"]
```

**Concurrency Tuning**:
```yaml
# Cloud Run service configuration
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: optimization-service
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "1"  # Keep 1 warm instance
        autoscaling.knative.dev/maxScale: "10"
        autoscaling.knative.dev/target: "70"    # Target 70% CPU utilization
    spec:
      containerConcurrency: 80  # Max concurrent requests per instance
      timeoutSeconds: 300       # 5-minute timeout for long-running requests
```

### 11.4 Frontend Optimization

**React Performance**:
```jsx
// Use React.memo for expensive components
export const MetricCard = React.memo(({ title, value, unit }) => {
  return (
    <div className="metric-card">
      <h3>{title}</h3>
      <p>{value} {unit}</p>
    </div>
  );
});

// Debounce Firestore listeners
import { useMemo, useCallback } from 'react';
import { debounce } from 'lodash';

export function useRealTimeData() {
  const [data, setData] = useState(null);
  
  const updateData = useCallback(
    debounce((newData) => setData(newData), 500),
    []
  );
  
  useEffect(() => {
    const unsubscribe = onSnapshot(
      doc(db, 'plant_status', 'live_data'),
      (doc) => updateData(doc.data())
    );
    return () => unsubscribe();
  }, [updateData]);
  
  return data;
}
```

**Code Splitting**:
```javascript
// vite.config.js
export default {
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor': ['react', 'react-dom'],
          'firebase': ['firebase/app', 'firebase/auth', 'firebase/firestore'],
          'charts': ['recharts']
        }
      }
    }
  }
};

// Lazy load routes
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./components/Dashboard'));
const Analytics = lazy(() => import('./components/Analytics'));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/analytics" element={<Analytics />} />
      </Routes>
    </Suspense>
  );
}
```

---

## 12. Cost Management Strategy

### 12.1 Cost Allocation & Tracking

**Resource Labeling Strategy**:
```hcl
# Terraform resource labels
locals {
  common_labels = {
    project     = "cement-plant-poc"
    environment = var.environment
    managed_by  = "terraform"
    cost_center = "engineering"
  }
}

resource "google_cloud_run_service" "optimization_service" {
  name     = "optimization-service"
  location = var.region
  
  metadata {
    labels = merge(local.common_labels, {
      component = "application-services"
      service   = "optimization"
    })
  }
}
```

**Budget Alerts**:
```yaml
# Budget alert configuration
budget:
  displayName: "PoC Monthly Budget"
  budgetFilter:
    projects:
      - "projects/cement-plant-poc"
    labels:
      project: "cement-plant-poc"
  amount:
    specifiedAmount:
      currencyCode: "USD"
      units: "500"
  thresholdRules:
    - thresholdPercent: 0.5   # Alert at 50%
      spendBasis: CURRENT_SPEND
    - thresholdPercent: 0.8   # Alert at 80%
      spendBasis: CURRENT_SPEND
    - thresholdPercent: 1.0   # Alert at 100%
      spendBasis: CURRENT_SPEND
  notificationsRule:
    pubsubTopic: "projects/cement-plant-poc/topics/budget-alerts"
    monitoringNotificationChannels:
      - "projects/cement-plant-poc/notificationChannels/email-ops"
```

### 12.2 Cost Optimization Checklist

**Daily Cost Review**:
```bash
#!/bin/bash
# scripts/daily-cost-check.sh

# Get current month costs
gcloud billing projects describe cement-plant-poc \
  --format="value(billingAccountName)" | \
  xargs -I {} gcloud billing accounts describe {} \
  --format="table(displayName, open)"

# Check BigQuery usage (top cost driver)
bq query --use_legacy_sql=false '
SELECT
  DATE(creation_time) as date,
  SUM(total_bytes_processed) / POW(10, 12) as tb_processed,
  COUNT(*) as query_count
FROM `cement_plant_poc.INFORMATION_SCHEMA.JOBS_BY_PROJECT`
WHERE creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
GROUP BY date
ORDER BY date DESC
'

# Check Cloud Run invocations
gcloud logging read \
  "resource.type=cloud_run_revision AND timestamp>=\"$(date -u -d '1 day ago' '+%Y-%m-%dT%H:%M:%S')\"" \
  --format="value(resource.labels.service_name)" | \
  sort | uniq -c | sort -nr
```

### 12.3 Projected Monthly Costs (PoC)

| Service | Usage Pattern | Estimated Cost |
|---------|--------------|----------------|
| **Cloud Scheduler** | 3 jobs × 1440 runs/day | $0.30 |
| **Cloud Functions** | Simulator: 43,200 invocations/month<br>Stream Processor: 86,400 invocations/month<br>Alert Handler: ~1,000 invocations/month | $5-10 |
| **Pub/Sub** | ~3M messages/month | $2 |
| **BigQuery Storage** | ~10 GB (partitioned) | $0.20 |
| **BigQuery Queries** | ~500 queries/month, 100 GB processed | $2.50 |
| **Firestore** | 500K reads, 100K writes/month | $2 |
| **Cloud Storage** | 5 GB storage, minimal egress | $0.10 |
| **Cloud Run** | 3 services × 100 requests/day<br>Scale-to-zero when idle | $8-12 |
| **Vertex AI (Gemini)** | ~200 API calls/month<br>10K input + 2K output tokens/call | $20-30 |
| **Firebase Hosting** | CDN, SSL included | $0 (free tier) |
| **Cloud Monitoring** | Logs, metrics, traces | $5 |
| **API Gateway** | ~3,000 requests/month | $1 |
| **TOTAL** | | **~$50-70/month** |

**Buffer for Development**: $430-450 remains for testing, experimentation, and unexpected usage.

---

## 13. Error Handling & Resilience

### 13.1 Error Handling Patterns

**Cloud Function Error Handling**:
```javascript
// stream-processor/index.js
const { PubSub } = require('@google-cloud/pubsub');
const pubsub = new PubSub();

exports.processSensorData = async (message, context) => {
  const messageId = context.eventId;
  const data = Buffer.from(message.data, 'base64').toString();
  
  try {
    // Parse and validate
    const parsed = JSON.parse(data);
    validateSchema(parsed);
    
    // Process with Promise.allSettled for partial failure handling
    const results = await Promise.allSettled([
      writeToBigQuery(parsed),
      updateFirestore(parsed),
      checkAnomaly(parsed)
    ]);
    
    // Log any failures but don't throw
    results.forEach((result, index) => {
      if (result.status === 'rejected') {
        console.error(`Operation ${index} failed:`, result.reason);
        // Optionally publish to dead-letter queue
        publishToDeadLetter(messageId, parsed, result.reason);
      }
    });
    
    return { success: true, messageId };
    
  } catch (error) {
    console.error('Critical error processing message:', error);
    
    // For Pub/Sub, throwing an error triggers retry
    // Only throw if we want automatic retry
    if (isRetryableError(error)) {
      throw error;  // Pub/Sub will retry
    } else {
      // Log and acknowledge message to prevent infinite retries
      await publishToDeadLetter(messageId, data, error);
      return { success: false, error: error.message };
    }
  }
};

function isRetryableError(error) {
  const retryableCodes = [
    'UNAVAILABLE',
    'DEADLINE_EXCEEDED',
    'RESOURCE_EXHAUSTED'
  ];
  return retryableCodes.includes(error.code);
}
```

**Cloud Run Retry Logic**:
```python
# optimization-service/services/gemini_client.py
from google.api_core import retry
from google.api_core import exceptions
import time

@retry.Retry(
    predicate=retry.if_exception_type(
        exceptions.ResourceExhausted,
        exceptions.ServiceUnavailable,
        exceptions.DeadlineExceeded
    ),
    initial=1.0,
    maximum=60.0,
    multiplier=2.0,
    deadline=300.0
)
def call_gemini_with_retry(prompt: str) -> dict:
    """Call Gemini API with exponential backoff retry"""
    try:
        response = gemini_client.generate_content(prompt)
        return parse_response(response)
    
    except exceptions.InvalidArgument as e:
        # Don't retry client errors
        raise ValueError(f"Invalid prompt: {e}")
    
    except Exception as e:
        logging.error(f"Gemini API call failed: {e}")
        raise

def call_gemini_with_fallback(prompt: str) -> dict:
    """Call Gemini with fallback to cached response"""
    try:
        return call_gemini_with_retry(prompt)
    
    except Exception as e:
        logging.warning(f"Gemini unavailable, using fallback: {e}")
        return get_cached_recommendation()
```

### 13.2 Circuit Breaker Pattern

**Implementation for External Services**:
```python
# shared/circuit_breaker.py
from datetime import datetime, timedelta
from enum import Enum

class CircuitState(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Failing, reject requests
    HALF_OPEN = "half_open"  # Testing if service recovered

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout  # seconds
        self.failure_count = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED
    
    def call(self, func, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if self._should_attempt_reset():
                self.state = CircuitState.HALF_OPEN
            else:
                raise Exception("Circuit breaker is OPEN")
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        
        except Exception as e:
            self._on_failure()
            raise e
    
    def _on_success(self):
        self.failure_count = 0
        self.state = CircuitState.CLOSED
    
    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = datetime.now()
        
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN
    
    def _should_attempt_reset(self):
        return (datetime.now() - self.last_failure_time).total_seconds() >= self.timeout

# Usage in service
gemini_breaker = CircuitBreaker(failure_threshold=3, timeout=30)

def optimize(request):
    try:
        result = gemini_breaker.call(call_gemini_api, request.data)
        return result
    except Exception:
        # Return cached or default recommendation
        return get_default_recommendation()
```

### 13.3 Dead Letter Queue

**Pub/Sub DLQ Configuration**:
```hcl
# Terraform configuration
resource "google_pubsub_topic" "sensor_data_raw" {
  name = "sensor-data-raw"
}

resource "google_pubsub_topic" "sensor_data_dlq" {
  name = "sensor-data-raw-dlq"
}

resource "google_pubsub_subscription" "sensor_data_subscription" {
  name  = "sensor-data-subscription"
  topic = google_pubsub_topic.sensor_data_raw.name
  
  ack_deadline_seconds = 60
  
  retry_policy {
    minimum_backoff = "10s"
    maximum_backoff = "600s"
  }
  
  dead_letter_policy {
    dead_letter_topic     = google_pubsub_topic.sensor_data_dlq.id
    max_delivery_attempts = 5
  }
}

# DLQ handler function
resource "google_cloudfunctions_function" "dlq_handler" {
  name        = "dlq-handler"
  runtime     = "nodejs20"
  entry_point = "handleDeadLetter"
  
  event_trigger {
    event_type = "google.pubsub.topic.publish"
    resource   = google_pubsub_topic.sensor_data_dlq.name
  }
}
```

**DLQ Handler**:
```javascript
// dlq-handler/index.js
const { Firestore } = require('@google-cloud/firestore');
const db = new Firestore();

exports.handleDeadLetter = async (message, context) => {
  const data = Buffer.from(message.data, 'base64').toString();
  
  // Store in Firestore for manual review
  await db.collection('failed_messages').add({
    original_message: data,
    error_count: message.attributes.deliveryAttempt || 'unknown',
    timestamp: new Date(),
    message_id: context.eventId
  });
  
  // Alert ops team
  await sendSlackAlert(`Dead letter received: ${context.eventId}`);
  
  console.log(`Dead letter logged: ${context.eventId}`);
};
```

---

## 14. Data Migration & Seeding

### 14.1 Initial Data Seeding

**Seed Script for Development**:
```python
#!/usr/bin/env python3
# scripts/seed-data.py

from google.cloud import bigquery, firestore
from datetime import datetime, timedelta
import random

def seed_bigquery():
    """Seed BigQuery with historical data"""
    client = bigquery.Client()
    table_id = "cement_plant_poc.sensor_readings"
    
    # Generate 7 days of historical data
    rows = []
    start_date = datetime.now() - timedelta(days=7)
    
    for day in range(7):
        for hour in range(24):
            for minute in range(0, 60, 5):  # Every 5 minutes
                timestamp = start_date + timedelta(days=day, hours=hour, minutes=minute)
                
                for sensor_id in ['kiln_temp', 'feed_rate', 'energy_kwh', 'quality_score']:
                    value = generate_realistic_value(sensor_id, hour)
                    
                    rows.append({
                        'timestamp': timestamp.isoformat(),
                        'plant_id': 'poc_plant_01',
                        'sensor_id': sensor_id,
                        'value': value
                    })
    
    # Insert in batches
    batch_size = 1000
    for i in range(0, len(rows), batch_size):
        batch = rows[i:i+batch_size]
        errors = client.insert_rows_json(table_id, batch)
        if errors:
            print(f"Errors inserting batch {i}: {errors}")
    
    print(f"✅ Seeded {len(rows)} sensor readings")

def generate_realistic_value(sensor_id, hour):
    """Generate realistic sensor values with daily patterns"""
    base_values = {
        'kiln_temp': 1450,
        'feed_rate': 220,
        'energy_kwh': 100,
        'quality_score': 0.92
    }
    
    # Add daily variation (production typically lower at night)
    if 0 <= hour < 6:
        multiplier = 0.85
    elif 6 <= hour < 18:
        multiplier = 1.0
    else:
        multiplier = 0.95
    
    base = base_values[sensor_id]
    noise = random.gauss(0, base * 0.02)  # 2% standard deviation
    
    return base * multiplier + noise

def seed_firestore():
    """Seed Firestore with initial plant configuration"""
    db = firestore.Client()
    
    # Plant metadata
    db.collection('plants').document('poc_plant_01').set({
        'name': 'Mumbai PoC Plant',
        'location': 'Mumbai, India',
        'type': 'Integrated (Dry Process)',
        'production_capacity_tpd': 5000,
        'commissioned_date': '2024-01-15',
        'created_at': firestore.SERVER_TIMESTAMP
    })
    
    # Initial live state
    db.collection('plant_status').document('live_data').set({
        'kiln_temp': 1450.0,
        'feed_rate': 220.0,
        'energy_kwh': 100.0,
        'quality_score': 0.92,
        'material_cao': 44.0,
        'last_update': firestore.SERVER_TIMESTAMP
    })
    
    print("✅ Seeded Firestore collections")

if __name__ == '__main__':
    print("Starting data seeding...")
    seed_bigquery()
    seed_firestore()
    print("✅ Data seeding complete!")
```

### 14.2 Data Backup Strategy

**Automated BigQuery Snapshots**:
```sql
-- Create scheduled query for daily backups
CREATE OR REPLACE TABLE FUNCTION backup_sensor_readings()
AS (
  SELECT * FROM `cement_plant_poc.sensor_readings`
  WHERE DATE(timestamp) = CURRENT_DATE()
);

-- Schedule via Cloud Scheduler
gcloud scheduler jobs create http backup-bigquery-daily \
  --schedule="0 2 * * *" \
  --uri="https://bigquery.googleapis.com/bigquery/v2/projects/cement-plant-poc/queries" \
  --message-body='{"query": "EXPORT DATA OPTIONS(uri=\"gs://cement-plant-poc-backups/bq/$(date +%Y%m%d)/*.parquet\", format=\"PARQUET\") AS SELECT * FROM sensor_readings WHERE DATE(timestamp) = CURRENT_DATE()"}'
```

**Firestore Backup**:
```bash
#!/bin/bash
# scripts/backup-firestore.sh

gcloud firestore export \
  "gs://cement-plant-poc-backups/firestore/$(date +%Y%m%d)" \
  --collection-ids=plant_status,alerts,plants

# Schedule with Cloud Scheduler
gcloud scheduler jobs create http backup-firestore-daily \
  --schedule="0 3 * * *" \
  --uri="https://firestore.googleapis.com/v1/projects/cement-plant-poc/databases/(default):exportDocuments" \
  --message-body='{"outputUriPrefix": "gs://cement-plant-poc-backups/firestore/'$(date +%Y%m%d)'"}'
```

---

## 15. Documentation Standards

### 15.1 Code Documentation

**Function Documentation Standard**:
```python
def generate_optimization_recommendation(
    current_state: dict,
    recent_trends: list,
    constraints: list
) -> dict:
    """
    Generate AI-powered optimization recommendation using Gemini.
    
    Args:
        current_state: Dictionary containing current sensor readings
            Example: {'kiln_temp': 1450, 'feed_rate': 220}
        recent_trends: List of recent quality scores with timestamps
            Example: [{'timestamp': '...', 'score': 0.92}, ...]
        constraints: List of operational constraints to respect
            Example: ['DO_NOT_EXCEED_TEMP_1500']
    
    Returns:
        Dictionary containing recommendation with structure:
        {
            'recommendation_id': str,
            'suggested_parameters': dict,
            'predicted_outcomes': dict,
            'explanation': str
        }
    
    Raises:
        ValueError: If current_state is missing required keys
        GeminiAPIError: If Gemini API call fails after retries
    
    Example:
        >>> recommendation = generate_optimization_recommendation(
        ...     current_state={'kiln_temp': 1450, 'feed_rate': 220},
        ...     recent_trends=[{'timestamp': '2025-10-22T12:00:00Z', 'score': 0.92}],
        ...     constraints=['DO_NOT_EXCEED_TEMP_1500']
        ... )
        >>> print(recommendation['suggested_parameters'])
        {'feed_rate_setpoint': 218.0}
    """
    pass
```

### 15.2 API Documentation

**OpenAPI Specification**:
```yaml
# docs/openapi.yaml
openapi: 3.0.0
info:
  title: Cement Plant GenAI API
  version: 1.0.0
  description: API for autonomous cement plant optimization and analytics

servers:
  - url: https://api.cement-plant-poc.example.com/api/v1
    description: Production server

security:
  - BearerAuth: []

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    OptimizationRequest:
      type: object
      required:
        - plant_id
      properties:
        plant_id:
          type: string
          example: "poc_plant_01"
        constraints:
          type: array
          items:
            type: string
          example: ["DO_NOT_EXCEED_TEMP_1500"]
    
    OptimizationResponse:
      type: object
      properties:
        recommendation_id:
          type: string
        timestamp:
          type: string
          format: date-time
        suggested_parameters:
          type: object
          properties:
            feed_rate_setpoint:
              type: number
            fuel_mix_ratio:
              type: number
        predicted_outcomes:
          type: object
          properties:
            energy_reduction_pct:
              type: number
            quality_score_impact:
              type: string

paths:
  /optimize:
    post:
      summary: Generate optimization recommendation
      tags:
        - Optimization
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/OptimizationRequest'
      responses:
        '200':
          description: Successful optimization
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OptimizationResponse'
        '401':
          description: Unauthorized
        '500':
          description: Internal server error
```

### 15.3 Architecture Decision Records (ADR)

**ADR Template**:
```markdown
# ADR-001: Use Firestore Instead of Cloud Bigtable for Real-Time State

## Status
Accepted

## Context
We need a database to store the current state of the cement plant (latest sensor readings) that can be accessed with low latency (<100ms) by the frontend for real-time dashboard updates.

Original architecture proposed Cloud Bigtable, but:
- Minimum cost: ~$470/month
- Requires 24/7 running nodes
- Overkill for our PoC scale (single plant, ~10 data points)

## Decision
Use Firestore for storing real-time plant state:
- Single document `/plant_status/live_data` updated by stream processor
- Real-time listeners via `onSnapshot()` in React frontend
- Free tier: 50K reads/day, 20K writes/day (sufficient for PoC)

## Consequences

**Positive:**
- Cost: ~$0-5/month vs $470/month
- Simplified architecture (no separate time-series DB)
- Native real-time support (no polling needed)
- Easy integration with Firebase Auth

**Negative:**
- Not suitable for high-throughput time-series queries (solved by BigQuery)
- Document size limit: 1MB (not an issue for our use case)
- If we scale beyond PoC, may need to revisit

## Alternatives Considered
1. **Cloud Bigtable**: Too expensive for PoC
2. **Redis (Memorystore)**: $48/month minimum, adds complexity
3. **Datastore**: Similar to Firestore but less real-time support
```

---

## 16. Production Readiness Checklist

### 16.1 Pre-Launch Checklist

```markdown
## Security
- [ ] All services require authentication
- [ ] Service accounts follow least-privilege principle
- [ ] Secrets managed via Secret Manager (not environment variables)
- [ ] API rate limiting configured
- [ ] CORS properly configured
- [ ] Firestore security rules tested
- [ ] No hardcoded credentials in code

## Performance
- [ ] Load testing completed (1000 concurrent users)
- [ ] All Cloud Run services scale to zero when idle
- [ ] BigQuery queries use partition pruning
- [ ] Firestore indexes created for all queries
- [ ] Frontend bundle size < 500KB
- [ ] Lighthouse performance score > 90

## Reliability
- [ ] Circuit breakers implemented for external APIs
- [ ] Dead letter queues configured
- [ ] Retry logic with exponential backoff
- [ ] Health check endpoints for all services
- [ ] Error rates < 0.1% in staging
- [ ] All critical paths have unit tests (>80% coverage)

## Observability
- [ ] Structured logging implemented
- [ ] Distributed tracing configured
- [ ] Key metrics dashboards created
- [ ] Alerting rules defined and tested
- [ ] Log retention policy configured
- [ ] Uptime checks configured

## Operations
- [ ] CI/CD pipeline functional
- [ ] Infrastructure as Code (Terraform) complete
- [ ] Backup and restore procedures tested
- [ ] Runbook documentation complete
- [ ] Incident response plan defined
- [ ] On-call rotation schedule

## Compliance
- [ ] Data residency requirements met
- [ ] Privacy policy updated
- [ ] GDPR compliance reviewed (if applicable)
- [ ] Audit logging enabled
- [ ] Data retention policy enforced
```

---

## 17. Future Enhancements (Post-PoC)

### 17.1 Phase 2 Roadmap

**Multi-Plant Support** (Month 2-3):
- Extend data model to support multiple plants
- Add plant selection UI in frontend
- Implement cross-plant analytics and benchmarking

**Advanced ML Models** (Month 3-4):
- Train custom quality defect classifier on real images
- Deploy predictive maintenance model on Vertex AI
- Implement reinforcement learning for control optimization

**Real Hardware Integration** (Month 4-6):
- Replace simulator with OPC UA connector
- Deploy edge gateway at physical plant
- Implement bidirectional control (read + write to PLCs)

### 17.2 Scalability Considerations

**Beyond 10 Plants**:
```
Current PoC Architecture → Enterprise Architecture

Data Ingestion:
  Cloud Functions → Cloud Run with autoscaling

Stream Processing:
  Cloud Functions → Dataflow streaming (for complex aggregations)

Real-Time Storage:
  Firestore → Cloud Bigtable (for time-series at scale)

API Layer:
  Cloud Run → GKE (for complex service mesh needs)

Frontend:
  Firebase Hosting → Cloud CDN + Cloud Run (for SSR)
```

---

## 18. Appendix

### 18.1 Glossary

| Term | Definition |
|------|------------|
| **OLAP** | Online Analytical Processing - optimized for complex queries on large datasets |
| **OLTP** | Online Transaction Processing - optimized for frequent, small transactions |
| **TPH** | Tons Per Hour - production rate metric |
| **Clinker** | Intermediate cement product before grinding |
| **Kiln** | Rotating furnace that heats raw materials to 1450°C |
| **DLQ** | Dead Letter Queue - holds failed messages for manual review |
| **JWT** | JSON Web Token - authentication token standard |
| **RBAC** | Role-Based Access Control - permission model |

### 18.2 Key Contacts

| Role | Responsibility | Contact |
|------|---------------|---------|
| **Tech Lead** | Overall architecture and technical decisions | TBD |
| **DevOps Engineer** | Infrastructure, CI/CD, monitoring | TBD |
| **ML Engineer** | AI/ML model development and Gemini integration | TBD |
| **Frontend Developer** | React dashboard and user experience | TBD |
| **QA Engineer** | Testing strategy and execution | TBD |

### 18.3 References

- [Google Cloud Architecture Framework](https://cloud.google.com/architecture/framework)
- [Vertex AI Gemini API Documentation](https://cloud.google.com/vertex-ai/docs/generative-ai/model-reference/gemini)
- [Cloud Run Best Practices](https://cloud.google.com/run/docs/best-practices)
- [BigQuery Best Practices](https://cloud.google.com/bigquery/docs/best-practices)
- [Firestore Data Model Best Practices](https://firebase.google.com/docs/firestore/best-practices)

---

## Document Change Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-10-22 | Architecture Team | Initial software architecture design |

---

**END OF DOCUMENT**