# OTEL E-Commerce Demo

A full-stack e-commerce application demonstrating distributed tracing between Sentry SDK (frontend) and OpenTelemetry (backend), with data flowing to Sentry.

## What This Demonstrates

1. **Backend**: OpenTelemetry instrumentation sending traces to Sentry via OTLP

   - Automatic instrumentation (HTTP, Express, PostgreSQL)
   - Manual instrumentation (custom spans, events, attributes, in-memory cache)
   - Two modes: Direct to Sentry or via OTEL Collector

2. **Frontend**: Sentry SDK capturing browser performance

   - React Router tracing
   - API call tracking with `traceparent` header propagation

3. **Distributed Tracing**: Connected traces across frontend and backend
   - Browser → API → Database → Cache
   - Single unified trace view in Sentry
   - Error propagation across services

## Quick Start

### Prerequisites

- Node.js 18+
- Free Neon account (https://neon.tech)
- Sentry account with OTLP enabled

### 1. Start Backend

```bash
cd api
npm install

# Create Neon database (prompts for login/signup)
npx neondb -y

# Configure environment
cp .env.example .env
# Edit .env: Add your Neon DATABASE_URL and Sentry endpoints

# Initialize database and start
npm run db:setup
npm start
```

**See [api/QUICKSTART.md](api/QUICKSTART.md) for detailed instructions.**

Backend runs on: `http://localhost:3000`

### 2. Start Frontend

```bash
cd frontend
npm install
cp .env.example .env
# Add Sentry SDK configuration (see DISTRIBUTED_TRACING_PLAN.md)
npm run dev
```

Frontend runs on: `http://localhost:5173` or `http://localhost:5174`

### 3. Test Distributed Tracing

1. Open frontend in browser
2. Browse products and create an order
3. Check Sentry → Explore → Traces
4. See unified trace: Browser → API → Database → Redis

## Demo Flows

### Backend Only (OTEL)

- Shows OpenTelemetry traces sent to Sentry via OTLP
- Direct mode or Collector mode
- Run: `cd api && npm test`

### Full Distributed Tracing

- Shows Sentry SDK (frontend) + OTEL (backend) connected
- Single trace ID spans both systems
- See: `DISTRIBUTED_TRACING_PLAN.md`

## Architecture

```
┌─────────────────┐
│  React Frontend │ (Sentry SDK)
│  Port: 5173     │
└────────┬────────┘
         │ HTTP + traceparent header
         ▼
┌─────────────────────────┐
│    Express API          │ (OpenTelemetry)
│    Port: 3000           │
│  ┌──────────────────┐   │
│  │ In-Memory Cache  │   │
│  └──────────────────┘   │
└────────┬────────────────┘
         │
    ┌────┴────┬─────────┐
    ▼         ▼         ▼
┌────────┐ ┌───────┐ ┌───────┐
│  Neon  │ │Payment│ │Sentry │
│Postgres│ │  API  │ │(OTLP) │
└────────┘ └───────┘ └───────┘
```

## License

MIT
