# GigGuard AI — Technical Architecture Documentation

## Overview

GigGuard AI is a parametric insurance platform for India's gig economy delivery partners. This document explains the integration logic, automated trigger mechanisms, and key architectural decisions.

---

## 1. System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          GigGuard AI Platform                           │
│                                                                         │
│  ┌─────────────────┐    ┌──────────────────┐    ┌──────────────────┐   │
│  │  React Frontend  │    │  Express Backend  │    │   PostgreSQL DB   │   │
│  │  (Vite + Tw CSS) │◄──►│   (Node.js API)   │◄──►│  (Drizzle ORM)   │   │
│  └─────────────────┘    └──────────┬───────┘    └──────────────────┘   │
│                                    │                                     │
│                    ┌───────────────┼───────────────┐                     │
│                    │               │               │                     │
│             ┌──────▼──────┐ ┌─────▼──────┐ ┌─────▼──────┐              │
│             │  PricingService│ │FraudSentry │ │WeatherSvc  │              │
│             └─────────────┘ └────────────┘ └─────┬──────┘              │
│                                                    │                     │
│                                          ┌─────────▼──────────┐         │
│                                          │  OpenWeatherMap API │         │
│                                          │  (Mock Traffic API) │         │
│                                          └────────────────────┘         │
└─────────────────────────────────────────────────────────────────────────┘
```

## 2. Core Service Layers

### 2.1 PricingService (`artifacts/api-server/src/lib/pricing-service.ts`)

**Purpose:** Calculates weekly premium based on hyper-local disruption history.

**Algorithm:**
```
basePremium = ₹25
historicalDisruptions = COUNT(claims WHERE worker_id = X AND created_at > NOW() - 30 days)

IF disruptions >= 3:
  adjustedPremium = ₹30 (+₹5), riskLevel = "High"
ELSE IF disruptions >= 1:
  adjustedPremium = ₹25, riskLevel = "Medium"
ELSE:
  adjustedPremium = ₹20 (-₹5), riskLevel = "Low"
```

**Integration:** Called during `POST /api/policies` to auto-price each new weekly shield.

---

### 2.2 Parametric Trigger Engine (`artifacts/api-server/src/routes/triggers.ts`)

**Purpose:** Zero-touch claim creation when environmental thresholds are breached.

**Trigger Thresholds:**
- Weather: `rainfall > 5mm` (per hour)
- Traffic: `delay > 20 minutes`

**Flow:**
```
POST /api/triggers/check
  │
  ├─► fetchWeatherData(lat, lon)   → OpenWeatherMap API or mock
  ├─► getMockTrafficData(lat, lon) → Deterministic mock (GPS/traffic sensor sim)
  │
  ├─► IF rainfall > 5mm OR delay > 20min:
  │     triggered = TRUE
  │     ├─► runFraudSentry(devicePressure, stationPressure, mockLocation, ...)
  │     ├─► INSERT into claims table
  │     ├─► INSERT into disruption_events table
  │     └─► RETURN { triggered, claimId, weatherData, trafficData }
  │
  └─► ELSE: RETURN { triggered: false, message: "No conditions met" }
```

**OpenWeatherMap Integration:**
- Set `OPENWEATHER_API_KEY` environment variable for live data
- Falls back to deterministic mock data if API key is absent or call fails
- Mock data varies by lat/lon for realistic city-specific scenarios

---

### 2.3 FraudSentry Module (`artifacts/api-server/src/lib/fraud-sentry.ts`)

**Purpose:** Multi-signal fraud detection to prevent false claims.

**Detection Signals:**

| Signal | Method | Score |
|--------|--------|-------|
| Altitude Spoofing | Compare device barometric pressure vs met-station pressure (tolerance: ±5 hPa) | +35 |
| Pressure Mismatch | Minor pressure deviation | +15 |
| Mock Location | GPS mock location flag from device | +40 |
| Root/Jailbreak | System integrity flag check | +25 |
| Modified System | Non-standard system flag | +15 |

**Verdict:**
- Score ≥ 60 → `isFlagged = true` → Status: `conditional_payout`
- Score < 60 → `isFlagged = false` → Status: `approved`

**Verified Hold Workflow:**
When a claim is flagged as suspicious:
1. Claim status set to `conditional_payout`
2. A random 8-char handshake code generated and stored
3. Worker must complete "Digital Handshake" — submit the code via `POST /api/claims/:id/verify`
4. On match, claim moves to `approved` and payout proceeds

---

### 2.4 Stability Vault (`artifacts/api-server/src/routes/vault.ts`)

**Purpose:** Liquid micro-savings for gig workers.

**Operations:**
- `GET /api/vault` — Get current balance and totals
- `POST /api/vault/deposit` — Add funds to vault
- `POST /api/vault/withdraw` — Withdraw funds
- `GET /api/vault/transactions` — Transaction history

Each operation creates a `vault_transaction` record for full auditability.

---

## 3. Database Schema

```sql
workers              -- Gig worker profiles with risk scores
policies             -- Weekly Shield policies (7-day validity)
claims               -- Parametric claims with fraud scores
vaults               -- Worker savings vault accounts
vault_transactions   -- Immutable transaction ledger
disruption_events    -- Geographic disruption log for admin map
```

---

## 4. Authentication

- **Method:** JWT (JSON Web Tokens), HS256 algorithm
- **Expiry:** 7 days
- **Secret:** `JWT_SECRET` env variable (falls back to `SESSION_SECRET`)
- **Flow:** Register → receive token → include as `Authorization: Bearer <token>` header
- **Password hashing:** bcrypt with 10 salt rounds

---

## 5. API Routes Summary

| Method | Path | Purpose |
|--------|------|---------|
| POST | /api/auth/register | Worker onboarding |
| POST | /api/auth/login | Worker login |
| GET | /api/workers/me | Get profile |
| GET | /api/workers/me/dashboard | Dashboard summary |
| GET/POST | /api/policies | List / create policies |
| GET | /api/policies/active | Active policy |
| POST | /api/pricing/calculate | Dynamic premium calculation |
| GET/POST | /api/claims | List / create claims |
| POST | /api/claims/:id/verify | Digital handshake verification |
| POST | /api/claims/:id/payout | Process payout with choice |
| GET | /api/vault | Vault balance |
| POST | /api/vault/deposit | Deposit to vault |
| POST | /api/vault/withdraw | Withdraw from vault |
| GET | /api/vault/transactions | Transaction history |
| POST | /api/triggers/check | Run parametric trigger check |
| GET | /api/triggers/weather | Get weather data |
| GET | /api/admin/analytics | Admin KPI overview |
| GET | /api/admin/loss-ratio | Loss ratio time series |
| GET | /api/admin/disruptions | Disruption map data |
| GET | /api/admin/workers | All workers list |
| GET | /api/admin/claims | All claims list |

---

## 6. Environment Variables

| Variable | Required | Purpose |
|----------|----------|---------|
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `PORT` | Yes | API server port |
| `SESSION_SECRET` | Yes | JWT signing secret |
| `JWT_SECRET` | No | Override JWT secret |
| `OPENWEATHER_API_KEY` | No | Live weather data (mock used if absent) |
| `BASE_PATH` | Yes (frontend) | Vite base path |

---

## 7. Deployment Notes

1. **API Server:** `pnpm --filter @workspace/api-server run build` produces `dist/index.mjs`
2. **Frontend:** `pnpm --filter @workspace/gigguard run build` produces `dist/public/`
3. **DB Migrations:** `pnpm --filter @workspace/db run push` — push schema to PostgreSQL
4. **Admin Access:** Set `is_admin = true` in the `workers` table for admin dashboard
5. **Demo Credentials:** `admin@gigguard.ai` / `admin123`

---

## 8. Folder Structure

```
artifacts/
  api-server/
    src/
      lib/
        auth.ts           # JWT + bcrypt auth utilities
        fraud-sentry.ts   # FraudSentry multi-signal detection
        pricing-service.ts # AI dynamic pricing engine
        weather-service.ts # OpenWeatherMap + traffic mock
      routes/
        auth.ts           # Register/login endpoints
        workers.ts        # Worker profile + dashboard
        policies.ts       # Policy management + pricing
        claims.ts         # Claims + fraud verification
        vault.ts          # Stability Vault CRUD
        triggers.ts       # Parametric trigger engine
        admin.ts          # Admin analytics

  gigguard/
    src/
      pages/              # React route pages
      components/         # Reusable UI components
      hooks/              # Custom React hooks
      lib/                # Utils and API client setup

lib/
  api-spec/openapi.yaml   # Single source of truth for API contract
  api-client-react/       # Generated React Query hooks (from codegen)
  api-zod/                # Generated Zod schemas (from codegen)
  db/src/schema/          # Drizzle ORM table definitions

TECHNICAL.md              # This file
```

---

## 9. Running Locally

```bash
# Install dependencies
pnpm install

# Push DB schema
pnpm --filter @workspace/db run push

# Start API server (dev)
pnpm --filter @workspace/api-server run dev

# Start frontend (dev)
pnpm --filter @workspace/gigguard run dev
```

---

*Built by Team GigGuard — April 2026*
