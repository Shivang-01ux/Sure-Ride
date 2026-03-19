# ZoneShield 🛡️
### Parametric Micro-Zone Income Protection for Hyper-Local Gig Delivery Partners

> **Automatically pays delivery riders when their zone fails them — no claim, no receipts, no waiting.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Python 3.11](https://img.shields.io/badge/Python-3.11-blue)](https://python.org)
[![Node.js 20](https://img.shields.io/badge/Node.js-20_LTS-green)](https://nodejs.org)
[![React Native](https://img.shields.io/badge/React_Native-Expo-purple)](https://expo.dev)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15_+_PostGIS-blue)](https://postgresql.org)
[![Docker](https://img.shields.io/badge/Docker-Compose-blue)](https://docker.com)

---

## Table of Contents

- [What is ZoneShield?](#what-is-zoneshield)
- [The Problem We Solve](#the-problem-we-solve)
- [How It Works](#how-it-works)
- [System Architecture](#system-architecture)
- [Parametric Trigger Definitions](#parametric-trigger-definitions)
- [Fraud Detection System](#fraud-detection-system)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Environment Variables](#environment-variables)
  - [Running with Docker Compose](#running-with-docker-compose)
  - [Running Services Individually](#running-services-individually)
- [API Reference](#api-reference)
- [Mobile App](#mobile-app)
- [Admin Dashboard](#admin-dashboard)
- [Data Sources & Integrations](#data-sources--integrations)
- [Financial Model Summary](#financial-model-summary)
- [Regulatory Notes](#regulatory-notes)
- [Testing](#testing)
- [Contributing](#contributing)
- [Team](#team)
- [License](#license)

---

## What is ZoneShield?

ZoneShield is a **parametric micro-zone income insurance product** built for hyper-local gig delivery workers — riders employed by or contracted with platforms like **Zepto**, **Blinkit**, and **Swiggy Instamart**.

Unlike traditional insurance:
- ✅ No claim forms
- ✅ No document submission
- ✅ No adjuster visits
- ✅ Payout in **< 4 hours** via UPI
- ✅ Triggered automatically by **real-world environmental data**

When a rider's micro-zone becomes operationally impossible — due to flooding, hazardous air quality, dark store closure, or a civic disruption — ZoneShield detects it automatically and credits their UPI account within hours.

The weekly premium (₹49–₹99) is marketed as a **"Guaranteed Earnings Floor"** subscription, not insurance — a framing that resonates far better with gig workers who have historically distrusted traditional insurance products.

---

## The Problem We Solve

A Q-Commerce delivery partner in Dharavi, Mumbai operates within a **2–3 km radius** from their assigned dark store. On a normal day, they earn ₹600–800.

When 40mm of rain hits their ward in 90 minutes:
- Their lanes flood
- Their dark store's catchment zone becomes impassable
- They earn **₹0** for the day — sometimes 2–3 days
- The rider 3km away in a different micro-zone continues working normally

**This is the core problem:** disruption is hyper-local, income loss is total, and no safety net exists.

City-level weather averages miss these events entirely. Traditional insurance doesn't cover "bad weather day" income loss. Platform compensation doesn't exist for zone-level environmental disruptions.

ZoneShield's pincode/ward-level parametric triggers are **4× more precise** than city-level averages and can correctly identify and compensate affected riders while correctly *not* triggering for unaffected nearby zones.

---

## How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                        TRIGGER LIFECYCLE                        │
│                                                                 │
│  Environmental     Data Ingestion    Trigger Engine             │
│  Event Occurs  ──► (every 2-3 min) ──► Evaluates Thresholds    │
│                                              │                  │
│                                    ┌─────────▼──────────┐      │
│                                    │  Dual-Source Check  │      │
│                                    │  (IMD + IoT must    │      │
│                                    │   agree within 2mm) │      │
│                                    └─────────┬──────────┘      │
│                                              │                  │
│                                    ┌─────────▼──────────┐      │
│                                    │  Fraud Scoring      │      │
│                                    │  (Rules + ML)       │      │
│                                    │  Score < 30: AUTO ✅│      │
│                                    │  Score 30-70: LOG ⚠│      │
│                                    │  Score > 70: HOLD 🔍│      │
│                                    └─────────┬──────────┘      │
│                                              │                  │
│                                    ┌─────────▼──────────┐      │
│                                    │  Payout Calculation │      │
│                                    │  hours_inactive ×   │      │
│                                    │  floor_rate (capped)│      │
│                                    └─────────┬──────────┘      │
│                                              │                  │
│                                    ┌─────────▼──────────┐      │
│                                    │  UPI Disbursement   │      │
│                                    │  via Razorpay API   │      │
│                                    │  + WhatsApp Alert   │      │
│                                    └────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

**Step-by-step:**

1. **Environmental signal detected** — IMD reports 38mm rainfall in pincode 400017 in 90 minutes
2. **Data ingestion service** polls IMD API every 3 minutes and writes reading to PostgreSQL with PostGIS zone matching
3. **IoT cross-validation** — nearest rain gauge (within 500m of dark store) must confirm reading within 2mm tolerance
4. **Trigger Evaluation Engine** evaluates threshold (≥35mm in 2h) → threshold breached → `TriggerEvent` created with state `EVALUATING`
5. **Fraud detection** runs rule-based checks (< 500ms) + async ML scoring → rider passes → state moves to `CONFIRMED`
6. **Payout Orchestrator** calculates amount → bulk UPI payout via Razorpay → WhatsApp + push notification to rider
7. **Ledger** records double-entry transaction → Admin dashboard updates in real-time

---

## System Architecture

ZoneShield follows a **6-layer architecture**, each independently scalable:

```
┌──────────────────────────────────────────────────────────────────────┐
│  LAYER 6 — WORKER-FACING INTERFACE                                   │
│  React Native App (Android/iOS) · React.js Admin Dashboard           │
├──────────────────────────────────────────────────────────────────────┤
│  LAYER 5 — PAYOUT ORCHESTRATION                                      │
│  Node.js · Razorpay API · Twilio WhatsApp · Firebase FCM             │
├──────────────────────────────────────────────────────────────────────┤
│  LAYER 4 — FRAUD DETECTION                                           │
│  Python · scikit-learn (Isolation Forest) · Redis Feature Store      │
├──────────────────────────────────────────────────────────────────────┤
│  LAYER 3 — TRIGGER EVALUATION ENGINE                                 │
│  Python/FastAPI · APScheduler · PostgreSQL State Machine             │
├──────────────────────────────────────────────────────────────────────┤
│  LAYER 2 — ZONE REGISTRY                                             │
│  PostgreSQL 15 + PostGIS · Geospatial Zone Polygons                 │
├──────────────────────────────────────────────────────────────────────┤
│  LAYER 1 — DATA INGESTION                                            │
│  IMD API · CPCB API · NDMA Feed · Kafka · MQTT · InfluxDB           │
└──────────────────────────────────────────────────────────────────────┘
```

### Component Communication

```
IMD API ──────────────────────────────────────────────┐
CPCB API ─────────────────────────────────────────────┤
NDMA RSS ─────────────────────────────────────────────┤──► Ingestion Service
IoT Sensors ──► MQTT Broker ──► MQTT Subscriber ──────┘    (FastAPI / Python)
                                                                  │
                                                         PostgreSQL + PostGIS
                                                         InfluxDB (time-series)
                                                                  │
Rider GPS Events ──► Apache Kafka ──► GPS Processor ──► Redis (clustering)
                                                                  │
                                                      Trigger Evaluation Engine
                                                         (APScheduler, 2min loop)
                                                                  │
                                                       ┌──────────▼──────────┐
                                                       │  Fraud Detection     │
                                                       │  Rules + ML Model    │
                                                       └──────────┬──────────┘
                                                                  │
                                                       ┌──────────▼──────────┐
                                                       │  Payout Orchestrator │
                                                       │  Node.js + Razorpay  │
                                                       └──────────┬──────────┘
                                                                  │
                                                     ┌────────────┴──────────┐
                                                     │                       │
                                               Rider App              Admin Dashboard
                                            (React Native)              (React.js)
```

---

## Parametric Trigger Definitions

All triggers are evaluated automatically. No rider action is required.

### Trigger 1 — Hyperlocal Rainfall Event

| Parameter | Value |
|-----------|-------|
| **Data Sources** | IMD API (primary) + IoT rain gauge network (secondary) |
| **Geographic Unit** | Pincode / Revenue Ward level |
| **Threshold** | ≥ 35mm rainfall within any 2-hour rolling window |
| **Concordance Required** | IMD and IoT readings must agree within ±2mm |
| **Minimum Duration** | Rainfall must persist ≥ 90 minutes within the 2-hour window |
| **Active Period** | Trigger active for 6 hours post-event (accounts for road drainage lag) |
| **Poll Frequency** | Every 3 minutes |

**Why this threshold?** IMD field data for Mumbai, Delhi, and Bengaluru shows that 35mm/2h consistently results in street-level flooding in dense urban wards. Below 25mm, drainage typically manages. 25–34mm is a marginal zone — we err conservative to maintain loss ratio.

---

### Trigger 2 — Hazardous AQI Spike

| Parameter | Value |
|-----------|-------|
| **Data Source** | CPCB CAAQMS (Continuous Ambient Air Quality Monitoring System) |
| **Geographic Unit** | Nearest CPCB station to dark store centroid (< 3km radius) |
| **Threshold** | AQI ≥ 300 (Hazardous category) for ≥ 4 consecutive hours |
| **Pollutant Basis** | PM2.5 and PM10 combined index (dominant in Indian urban AQI) |
| **Verification** | Single source (CPCB government data — tamper-proof) |
| **Seasonal Relevance** | Oct–Feb (North India); Jun–Aug (monsoon crop burning) |
| **Poll Frequency** | Every 5 minutes |

---

### Trigger 3 — Dark Store Operational Closure

| Parameter | Value |
|-----------|-------|
| **Data Sources** | Platform API (primary) + GPS rider clustering (secondary) |
| **Detection Method** | Store status = `CLOSED` OR order fulfillment rate = 0% for ≥ 2 hours |
| **GPS Clustering** | ≥ 5 riders from same dark store showing clustered idle GPS required |
| **Concordance** | Both platform API signal AND GPS cluster must agree |
| **GPS Data Retention** | 30 minutes maximum (privacy compliance) |
| **Exclusion** | Temporary stockout (< 30 min) does not count |

> **Note:** For the hackathon prototype, the dark store API is simulated via a Node.js mock server. In production, this would be a real webhook integration with platform partners.

---

### Trigger 4 — Government-Declared Civic Disruption

| Parameter | Value |
|-----------|-------|
| **Data Sources** | NDMA alerts + Municipal Corporation RSS/XML feeds |
| **Event Types** | Bandh, Section 144 CrPC, NDMA-declared emergency, road closure order |
| **Geographic Unit** | Ward/district boundary overlapping rider's registered micro-zone |
| **Verification** | Official government notification reference number required |
| **Active Period** | Duration of official order only |
| **Exclusion** | Unofficial/social-media-only reports not eligible |

---

## Fraud Detection System

Because triggers are automated, fraud is attempted at the **trigger level** — not the claim level. ZoneShield's fraud architecture makes trigger manipulation economically and technically infeasible.

### Rule-Based Checks (Synchronous, < 500ms)

| Check | Method | Action on Failure |
|-------|--------|-------------------|
| **GPS Spoofing Detection** | Device attestation API checks for known spoofing apps (Fake GPS, Mock Location) | Auto-deny + flag account |
| **Accelerometer Cross-Check** | GPS shows idle but accelerometer shows movement → anomaly | Escalate to manual review |
| **Cell Tower Triangulation** | Cross-reference GPS with nearest tower location (±200m tolerance) | Flag if discrepancy > 200m |
| **Order History Check** | If rider completed orders during claimed trigger window → deny | Auto-deny |
| **Speed Plausibility** | GPS trajectory shows physically impossible movement (teleportation) | Auto-flag |
| **Concordance Enforcement** | Both data sources must agree (see triggers above) | Trigger does not fire |

### ML-Based Checks (Asynchronous)

| Model | Algorithm | Features | Action |
|-------|-----------|----------|--------|
| **Anomaly Detection** | Isolation Forest (scikit-learn) | Claim frequency, geographic clustering, temporal patterns, social graph score | Flag for retrospective review |
| **Cluster Ring Detection** | Density-based clustering | Time-correlation of idle GPS events within zone, social graph adjacency | Progressive payout throttle |

### Fraud Score Bands

```
Score 0–29   → Auto-Approve ✅
Score 30–69  → Enhanced Logging ⚠️  (approved, but flagged for review)
Score 70–89  → Manual Review Queue 🔍 (payout held pending human approval)
Score 90–100 → Auto-Deny ❌ (appeal pathway provided to rider)
```

### Model Explainability

Every fraud score generates **SHAP values** stored alongside the event record. This satisfies:
- Regulatory transparency requirements
- Rider appeal process (rider can see why they were flagged)
- Internal model improvement workflows

---

## Tech Stack

### Backend

| Technology | Version | Purpose |
|-----------|---------|---------|
| **Python** | 3.11 | Trigger Engine, Data Ingestion, Fraud ML |
| **FastAPI** | 0.110+ | REST APIs for all Python services |
| **Node.js** | 20 LTS | Payout Orchestration, Dark Store Mock API |
| **APScheduler** | 3.10+ | 2-minute trigger evaluation polling loop |
| **PostgreSQL** | 15 | Zone Registry, Ledger, Audit Log, TriggerEvents |
| **PostGIS** | 3.x | Geospatial zone polygon queries (ST_Within, ST_DWithin) |
| **InfluxDB** | 2.x | IoT sensor time-series (rainfall, AQI readings) |
| **Redis** | 7 | Fraud feature store, caching, rate limiting |
| **Apache Kafka** | 3.x | High-throughput GPS event stream from rider app |
| **Mosquitto** | 2.x | MQTT broker for IoT rain gauge simulation |

### Machine Learning & Data Science

| Technology | Purpose |
|-----------|---------|
| **scikit-learn** | Isolation Forest fraud anomaly detection model |
| **pandas + numpy** | Data transformation, windowed time-series aggregation |
| **shapely + geopandas** | Geospatial calculations — zone polygon membership checks |
| **MLflow** | Model experiment tracking and registration |
| **SHAP** | Fraud score explainability (feature importance per prediction) |

### Frontend & Mobile

| Technology | Purpose |
|-----------|---------|
| **React Native (Expo)** | Rider mobile app — single codebase for Android + iOS |
| **React.js 18** | Admin operations dashboard |
| **Tailwind CSS** | Admin dashboard styling |
| **MapLibre GL JS** | Live zone map with GeoJSON polygon rendering (no API cost) |
| **React Query (TanStack)** | API state management, caching, background refetch |

### External APIs & Integrations

| Service | Purpose |
|---------|---------|
| **IMD Open Data API** | Rainfall data — pincode level |
| **CPCB CAAQMS API** | AQI data — monitoring station level |
| **NDMA API / RSS** | Civic disruption alerts |
| **Razorpay Payout API** | UPI bulk disbursement with status webhooks |
| **Twilio WhatsApp API** | Rider payout notifications |
| **Firebase FCM** | In-app push notifications |
| **Account Aggregator (Sahamati)** | Income verification at onboarding (optional) |

### Infrastructure & DevOps

| Technology | Purpose |
|-----------|---------|
| **Docker + Docker Compose** | All services containerized, one-command startup |
| **GitHub Actions** | CI/CD — automated tests, linting, staging deploy |
| **Grafana + Prometheus** | System metrics dashboards |
| **PagerDuty** | On-call alerting for trigger engine / payout failures |
| **Sentry** | Real-time error tracking (backend + mobile) |
| **AWS Mumbai Region** | Hosting (all data stays in India — DPDPA compliance) |

### Security

| Technology | Purpose |
|-----------|---------|
| **AES-256** | PII data encryption at rest |
| **JWT + OAuth 2.0** | API authentication + admin SSO |
| **OWASP Dependency Check** | Supply chain vulnerability scanning in CI |

---

## Project Structure

```
zoneshield/
│
├── services/
│   ├── ingestion/                   # Layer 1 — Data Ingestion
│   │   ├── imd_poller.py            # IMD rainfall API poller (FastAPI + APScheduler)
│   │   ├── cpcb_poller.py           # CPCB AQI API poller
│   │   ├── ndma_parser.py           # NDMA RSS/XML feed parser
│   │   ├── mqtt_subscriber.py       # IoT rain gauge MQTT subscriber → InfluxDB
│   │   ├── kafka_gps_consumer.py    # Rider GPS event Kafka consumer
│   │   └── dark_store_mock/         # Node.js dark store status mock API
│   │       ├── server.js
│   │       └── data/stores.json
│   │
│   ├── zone_registry/               # Layer 2 — Zone Registry
│   │   ├── models.py                # SQLAlchemy models with PostGIS geometry fields
│   │   ├── seed_zones.py            # Seeds test zones (Delhi, Mumbai, Bengaluru)
│   │   └── geo_utils.py             # Haversine distance, zone polygon helpers
│   │
│   ├── trigger_engine/              # Layer 3 — Trigger Evaluation Engine
│   │   ├── main.py                  # FastAPI app + APScheduler loop
│   │   ├── evaluators/
│   │   │   ├── rainfall.py          # Rainfall trigger logic + dual-source concordance
│   │   │   ├── aqi.py               # AQI trigger logic
│   │   │   ├── dark_store.py        # Dark store closure trigger + GPS clustering
│   │   │   └── civic.py             # NDMA civic disruption trigger
│   │   ├── state_machine.py         # TriggerEvent state transitions
│   │   └── audit_logger.py          # Append-only audit log writer
│   │
│   ├── fraud_detection/             # Layer 4 — Fraud Detection
│   │   ├── rules_engine.py          # Synchronous rule-based checks (< 500ms)
│   │   ├── ml_model/
│   │   │   ├── train.py             # Isolation Forest training script
│   │   │   ├── predict.py           # Real-time fraud scoring
│   │   │   ├── shap_explainer.py    # SHAP value generation
│   │   │   └── models/              # MLflow-tracked model artifacts
│   │   ├── feature_store.py         # Redis feature extraction and caching
│   │   └── gps_analyzer.py          # GPS velocity plausibility, clustering logic
│   │
│   ├── payout_orchestrator/         # Layer 5 — Payout Orchestration
│   │   ├── server.js                # Node.js payout service
│   │   ├── razorpay_client.js       # Razorpay API integration (sandbox/production)
│   │   ├── calculator.js            # Payout amount calculation logic
│   │   ├── ledger.js                # Double-entry accounting ledger
│   │   ├── notifier.js              # Twilio WhatsApp + Firebase FCM
│   │   └── retry_handler.js         # Failed payout retry logic (3 attempts)
│   │
│   └── admin_api/                   # Admin REST API
│       ├── main.py
│       ├── routes/
│       │   ├── zones.py
│       │   ├── triggers.py
│       │   ├── fraud_queue.py
│       │   └── ledger.py
│       └── auth.py                  # JWT + OAuth 2.0
│
├── apps/
│   ├── rider_app/                   # Layer 6 — React Native Rider App
│   │   ├── app/
│   │   │   ├── (tabs)/
│   │   │   │   ├── index.tsx        # Zone Status Dashboard
│   │   │   │   ├── coverage.tsx     # My Coverage + Payout History
│   │   │   │   └── earnings.tsx     # Live Earnings Floor Tracker
│   │   │   ├── onboarding/
│   │   │   │   ├── plan_select.tsx  # Plan selection screen
│   │   │   │   └── upi_register.tsx # UPI registration
│   │   │   └── _layout.tsx
│   │   ├── i18n/                    # Multilingual strings
│   │   │   ├── en.json
│   │   │   ├── hi.json
│   │   │   ├── kn.json
│   │   │   ├── ta.json
│   │   │   └── mr.json
│   │   └── app.json
│   │
│   └── admin_dashboard/             # Layer 6 — React.js Admin Dashboard
│       ├── src/
│       │   ├── pages/
│       │   │   ├── ZoneMap.tsx      # Live MapLibre zone map
│       │   │   ├── TriggerFeed.tsx  # Real-time trigger event feed
│       │   │   ├── FraudQueue.tsx   # Manual review queue
│       │   │   └── Ledger.tsx       # Payout ledger view
│       │   └── components/
│       └── package.json
│
├── database/
│   ├── migrations/                  # Alembic migration files
│   ├── schema.sql                   # Base PostgreSQL + PostGIS schema
│   └── seed/
│       ├── zones.geojson            # Test zone polygon GeoJSON
│       └── riders.csv               # 100 synthetic test riders
│
├── ml/
│   ├── data/                        # Synthetic training data generator
│   │   └── generate_fraud_data.py
│   ├── notebooks/                   # EDA and model development notebooks
│   │   └── fraud_model_development.ipynb
│   └── experiments/                 # MLflow experiment tracking data
│
├── infra/
│   ├── docker-compose.yml           # Full stack — one-command startup
│   ├── docker-compose.dev.yml       # Development overrides (hot reload)
│   ├── prometheus.yml               # Prometheus scrape config
│   └── grafana/
│       └── dashboards/              # Pre-built Grafana dashboard JSON
│
├── tests/
│   ├── unit/
│   │   ├── test_rainfall_trigger.py
│   │   ├── test_aqi_trigger.py
│   │   ├── test_fraud_rules.py
│   │   └── test_payout_calculator.py
│   ├── integration/
│   │   ├── test_trigger_to_payout.py  # End-to-end trigger lifecycle
│   │   └── test_zone_matching.py
│   └── load/
│       └── simulate_1000_riders.py    # Load test: 1,000 concurrent rider GPS events
│
├── scripts/
│   ├── inject_flood_event.py        # Simulate flood event for demo
│   ├── inject_aqi_spike.py          # Simulate AQI spike for demo
│   ├── inject_store_closure.py      # Simulate dark store closure for demo
│   └── reset_demo_state.py          # Reset all demo data to clean state
│
├── docs/
│   ├── architecture.md              # Detailed architecture documentation
│   ├── api_reference.md             # Full REST API reference
│   ├── trigger_definitions.md       # Complete trigger specification
│   ├── fraud_detection.md           # Fraud system design document
│   └── financial_model.md           # Actuarial assumptions + projections
│
├── .env.example                     # Environment variable template
├── .github/
│   └── workflows/
│       ├── ci.yml                   # CI: test + lint on every push
│       └── deploy.yml               # CD: deploy to staging on main merge
├── docker-compose.yml
├── Makefile                         # Common dev commands
└── README.md
```

---

## Getting Started

### Prerequisites

Ensure the following are installed on your machine:

| Tool | Version | Install |
|------|---------|---------|
| Docker | 24.x+ | [docs.docker.com](https://docs.docker.com) |
| Docker Compose | 2.x+ | Included with Docker Desktop |
| Node.js | 20 LTS | [nodejs.org](https://nodejs.org) |
| Python | 3.11+ | [python.org](https://python.org) |
| Expo CLI | Latest | `npm install -g expo-cli` |
| Git | Any | [git-scm.com](https://git-scm.com) |

---

### Environment Variables

Copy the example file and fill in your values:

```bash
cp .env.example .env
```

```env
# ── DATABASE ──────────────────────────────────────────────────────────────
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=zoneshield
POSTGRES_USER=zoneshield_user
POSTGRES_PASSWORD=your_secure_password

# ── INFLUXDB (IoT Time-Series) ────────────────────────────────────────────
INFLUXDB_URL=http://localhost:8086
INFLUXDB_TOKEN=your_influxdb_token
INFLUXDB_ORG=zoneshield
INFLUXDB_BUCKET=sensor_data

# ── REDIS ─────────────────────────────────────────────────────────────────
REDIS_URL=redis://localhost:6379

# ── KAFKA ─────────────────────────────────────────────────────────────────
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
KAFKA_TOPIC_GPS=rider.gps.events

# ── EXTERNAL DATA APIs ────────────────────────────────────────────────────
IMD_API_KEY=your_imd_api_key
IMD_API_BASE_URL=https://api.imd.gov.in/v1
CPCB_API_KEY=your_cpcb_api_key
CPCB_API_BASE_URL=https://api.cpcb.gov.in/caaqms

# ── PAYOUT ────────────────────────────────────────────────────────────────
RAZORPAY_KEY_ID=rzp_test_xxxxxxxxxxxx        # Use rzp_test_ prefix for sandbox
RAZORPAY_KEY_SECRET=your_razorpay_secret
RAZORPAY_ACCOUNT_NUMBER=your_payout_account

# ── NOTIFICATIONS ─────────────────────────────────────────────────────────
TWILIO_ACCOUNT_SID=your_twilio_sid
TWILIO_AUTH_TOKEN=your_twilio_token
TWILIO_WHATSAPP_FROM=whatsapp:+14155238886   # Twilio sandbox number
FIREBASE_SERVICE_ACCOUNT_KEY=path/to/firebase-key.json

# ── AUTH ──────────────────────────────────────────────────────────────────
JWT_SECRET=your_jwt_secret_min_32_chars
GOOGLE_OAUTH_CLIENT_ID=your_google_oauth_client_id  # For admin dashboard SSO

# ── APP CONFIG ────────────────────────────────────────────────────────────
ENVIRONMENT=development                    # development | staging | production
TRIGGER_EVAL_INTERVAL_SECONDS=120          # How often trigger engine runs (default: 2 min)
FRAUD_AUTO_DENY_THRESHOLD=90               # Fraud score above which payouts auto-denied
FRAUD_MANUAL_REVIEW_THRESHOLD=70           # Fraud score above which payouts held for review
```

> **Hackathon Note:** For the demo, set `ENVIRONMENT=development` — this activates the dark store mock API and simulated IoT sensor data. Real external API calls are mocked in development mode.

---

### Running with Docker Compose

The fastest way to get everything running:

```bash
# Clone the repository
git clone https://github.com/your-team/zoneshield.git
cd zoneshield

# Copy environment variables
cp .env.example .env
# Edit .env with your API keys

# Start all services (first run downloads images — takes ~3 minutes)
docker compose up --build

# In a separate terminal, seed the database with test zones and riders
docker compose exec trigger_engine python database/seed/seed_zones.py
```

**Services started by Docker Compose:**

| Service | URL | Description |
|---------|-----|-------------|
| Admin Dashboard | http://localhost:3000 | React.js operations dashboard |
| Admin API | http://localhost:8001 | REST API for admin dashboard |
| Trigger Engine | http://localhost:8000 | Core parametric trigger service |
| Payout Orchestrator | http://localhost:8002 | Node.js payout service |
| Dark Store Mock | http://localhost:8003 | Simulated platform API |
| PostgreSQL | localhost:5432 | Primary database |
| InfluxDB | http://localhost:8086 | IoT time-series database |
| Redis | localhost:6379 | Feature store + cache |
| Kafka | localhost:9092 | GPS event streaming |
| Mosquitto (MQTT) | localhost:1883 | IoT sensor broker |
| Grafana | http://localhost:3001 | System metrics dashboards |
| Prometheus | http://localhost:9090 | Metrics collection |

---

### Running Services Individually

For development, you may prefer to run services individually with hot reload:

**1. Database setup (run once):**
```bash
# Start only infrastructure
docker compose up postgres influxdb redis kafka mosquitto -d

# Run migrations
cd database && alembic upgrade head

# Seed test data
python database/seed/seed_zones.py
```

**2. Trigger Engine (Python):**
```bash
cd services/trigger_engine
pip install -r requirements.txt
uvicorn main:app --reload --port 8000
```

**3. Payout Orchestrator (Node.js):**
```bash
cd services/payout_orchestrator
npm install
npm run dev
```

**4. Admin Dashboard (React):**
```bash
cd apps/admin_dashboard
npm install
npm run dev
# Opens at http://localhost:3000
```

**5. Rider App (React Native):**
```bash
cd apps/rider_app
npm install
npx expo start
# Scan QR code with Expo Go app on Android
```

---

### Running a Demo Scenario

Use the demo scripts to simulate events in the running system:

```bash
# Simulate a flood event in Delhi Zone 3
python scripts/inject_flood_event.py --zone delhi_zone_3 --rainfall_mm 42

# Simulate an AQI spike in Mumbai Zone 1
python scripts/inject_aqi_spike.py --zone mumbai_zone_1 --aqi 315

# Simulate a dark store closure
python scripts/inject_store_closure.py --store_id DST_MUM_007

# Reset all demo data to clean state
python scripts/reset_demo_state.py
```

Watch the admin dashboard at http://localhost:3000 as triggers fire, fraud checks run, and payouts are dispatched in real-time.

---

## API Reference

Full API documentation is available at `http://localhost:8000/docs` (FastAPI auto-generated Swagger UI) when running locally.

### Trigger Engine API

```
GET  /health                          Health check
GET  /zones                           List all registered micro-zones
GET  /zones/{zone_id}                 Get zone details + current trigger status
GET  /triggers                        List all trigger events (paginated)
GET  /triggers/{trigger_id}           Get specific trigger event + audit trail
POST /triggers/simulate               Inject a simulated trigger event (dev only)
GET  /triggers/active                 Get all currently active triggers
```

### Admin API

```
GET  /admin/dashboard                 Summary stats (active triggers, pending payouts, fraud queue)
GET  /admin/fraud/queue               Get fraud events pending manual review
POST /admin/fraud/{event_id}/approve  Manually approve a held payout
POST /admin/fraud/{event_id}/deny     Manually deny a held payout
GET  /admin/ledger                    Payout ledger (paginated, filterable)
GET  /admin/zones/map                 GeoJSON for live zone map rendering
```

### Payout Orchestrator API (Internal)

```
POST /payouts/initiate                Initiate payout for confirmed trigger event
GET  /payouts/{payout_id}/status      Check payout status
POST /payouts/{payout_id}/retry       Manually retry a failed payout
GET  /payouts/reconciliation          Daily Razorpay reconciliation report
```

---

## Mobile App

The rider app (React Native / Expo) covers the following journeys:

### Onboarding Flow
1. **Phone OTP verification** (Firebase Auth)
2. **Zone auto-assignment** based on GPS location → nearest dark store
3. **Plan selection** (Basic ₹49 / Plus ₹69 / Pro ₹99)
4. **UPI registration** — enter UPI ID, verify with ₹1 test credit

### Core Screens

**Zone Status Dashboard (Home)**
- Real-time trigger conditions in the rider's micro-zone
- Color coded: 🟢 Clear · 🟡 Watch · 🔴 Active Trigger
- Current rainfall reading, AQI level, dark store status
- Estimated payout if trigger activates

**My Coverage**
- Active plan details and weekly premium next deduction date
- Payout history (date, trigger type, amount, status)
- Downloadable payout receipts (PDF)

**Live Earnings Floor Tracker**
- Today's earned amount vs. guaranteed floor
- If trigger is active: projected payout amount shown in real-time
- Weekly cumulative view

### Language Support
English · हिन्दी (Hindi) · ಕನ್ನಡ (Kannada) · தமிழ் (Tamil) · मराठी (Marathi)

### Offline Mode
Last known zone status and coverage details cached locally. Premium status accessible without network. Critical for riders in low-connectivity delivery zones.

---

## Admin Dashboard

The React.js admin dashboard provides operations teams with:

- **Live Zone Map** — MapLibre GL JS rendering zone polygons, color-coded by trigger status. Click any zone for details.
- **Real-Time Trigger Feed** — Streaming list of TriggerEvents as they move through the state machine
- **Fraud Review Queue** — Events with fraud score 70–89 requiring human approval/denial
- **Payout Ledger** — Double-entry ledger view with Razorpay reconciliation status
- **ML Model Monitor** — Fraud model performance metrics, drift detection, SHAP feature importance charts
- **Grafana Integration** — Links to pre-built dashboards: trigger engine latency, payout success rate, fraud detection throughput

---

## Data Sources & Integrations

| Source | Type | Coverage | Reliability | Cost |
|--------|------|----------|-------------|------|
| **IMD Open Data** | Government API | All India, pincode level | High (official) | Free (non-commercial) |
| **CPCB CAAQMS** | Government API | 800+ stations nationwide | High (official) | Free |
| **NDMA Alerts** | RSS/XML Feed | National + State level | High (official) | Free |
| **Razorpay Payout** | Commercial API | Pan-India UPI | Very High | Per-transaction fee |
| **Twilio WhatsApp** | Commercial API | Global | Very High | Per-message fee |
| **Firebase FCM** | Commercial SDK | Global | High | Free tier sufficient |
| **Sahamati AA** | Regulated API | India banks | High | Per-consent fee |
| **IoT Rain Gauges** | Simulated (MQTT) | Hackathon only | — | — |
| **Platform API** | Simulated (Mock) | Hackathon only | — | — |

---

## Financial Model Summary

### Unit Economics (Plus Plan — ₹69/week)

| Line Item | Amount |
|-----------|--------|
| Weekly Gross Premium | ₹69.00 |
| GST (18%) | ₹10.53 |
| Net Premium to ZoneShield | ₹51.45 |
| Expected Loss per Rider per Week | ₹28.30 (55% loss ratio) |
| Operating Margin per Rider per Week | ₹18.00 |
| Break-Even Rider Count | ~1,850 riders |

### Actuarial Assumptions

| Trigger | Frequency | Basis |
|---------|-----------|-------|
| Rainfall (Mumbai/Delhi) | 2.1 events/zone/month | IMD historical data 2018–2023 |
| AQI (Delhi, Oct–Feb) | 3.2 events/zone/month | CPCB historical records |
| AQI (Mumbai/Bengaluru) | 0.4 events/zone/month | CPCB historical records |
| Dark Store Closure | 0.8 events/zone/month | Conservative estimate |
| Avg Payout per Event | ₹1,050 | 1.5 days × ₹700/day floor |

### 5-Year Projection (Conservative)

| Year | Riders | Revenue | EBITDA |
|------|--------|---------|--------|
| Y1 | 5,000 | ₹1.8 Cr | −₹0.8 Cr |
| Y2 | 25,000 | ₹8.9 Cr | +₹1.2 Cr |
| Y3 | 80,000 | ₹28.5 Cr | +₹7.1 Cr |
| Y4 | 180,000 | ₹64.2 Cr | +₹18.5 Cr |
| Y5 | 350,000 | ₹124.8 Cr | +₹37.4 Cr |

---

## Regulatory Notes

### IRDAI Pathway
- ZoneShield is structured as a **Micro-Insurance Benefit Product** (non-indemnity) under IRDAI Micro-Insurance Regulations 2005 (amended 2015)
- Eligible for **IRDAI Regulatory Sandbox** (2019 framework) — 12-month pilot with up to 10,000 policyholders
- Requires a licensed general insurer as underwriting partner (New India, HDFC Ergo, Bajaj Allianz have active insurtech programs)

### Data Privacy (DPDPA 2023)
- Explicit, granular rider consent collected at onboarding
- GPS data retained for **30 minutes maximum** — anonymized thereafter
- All data processed and stored within India (AWS Mumbai / equivalent)
- Rider right-to-erasure honored within 30 days of account closure

### RBI / Payments
- UPI payouts routed through licensed payment aggregator (Razorpay — RBI authorized)
- Premium deduction via rider-authorized auto-debit mandate
- Account Aggregator (AA) framework used for income verification — RBI regulated

---

## Testing

### Unit Tests
```bash
# Run all unit tests
pytest tests/unit/ -v

# Run specific module
pytest tests/unit/test_rainfall_trigger.py -v
pytest tests/unit/test_fraud_rules.py -v
```

### Integration Tests
```bash
# Requires Docker Compose stack running
pytest tests/integration/ -v

# Test full trigger-to-payout lifecycle
pytest tests/integration/test_trigger_to_payout.py -v
```

### Load Test
```bash
# Simulate 1,000 concurrent riders sending GPS events
python tests/load/simulate_1000_riders.py --riders 1000 --zones 10 --duration 60

# Expected: trigger engine latency < 30 seconds, payout API < 2 minutes
```

### CI Pipeline
Every push to any branch triggers:
1. **Black** (Python formatting check)
2. **ESLint** (JavaScript linting)
3. **pytest** (all unit tests)
4. **OWASP Dependency Check** (CVE scanning)

Every merge to `main` additionally triggers:
5. **Integration tests** (Docker Compose environment)
6. **Deploy to staging**

---

## Contributing

1. **Fork** the repository
2. Create a feature branch: `git checkout -b feature/your-feature-name`
3. Write tests for your changes
4. Ensure all tests pass: `pytest tests/unit/`
5. Run linting: `black . && eslint apps/`
6. Submit a **Pull Request** with a clear description

### Code Style
- **Python:** Black formatter, PEP 8, type hints on all public functions
- **JavaScript/TypeScript:** ESLint + Prettier, functional components in React
- **SQL:** All migrations via Alembic, no raw schema changes
- **Commits:** Conventional Commits format (`feat:`, `fix:`, `docs:`, `test:`)

---

## Team

| Role | Responsibility |
|------|---------------|
| **Backend Lead** | Trigger Engine, Data Ingestion, PostgreSQL/PostGIS |
| **ML Engineer** | Fraud Detection Model, Feature Store, SHAP Explainability |
| **Frontend Developer** | Admin Dashboard (React.js), MapLibre Integration |
| **Mobile Developer** | Rider App (React Native / Expo) |
| **Product / Finance** | Financial Model, Actuarial Assumptions, Regulatory Research |

---

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.

---

> Built with ❤️ for gig workers who keep India's cities running — one 10-minute delivery at a time.

---

*ZoneShield — Parametric Micro-Zone Income Protection | Hackathon Build | Confidential*
