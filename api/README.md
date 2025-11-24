# OpenTelemetry E-commerce API

## Architecture

```
┌─────────────────┐
│   Express API   │
└────────┬────────┘
         │
    ┌────┴────┐
    │  OTEL   │ (Auto + Manual Instrumentation)
    │   SDK   │
    └────┬────┘
         │
    ┌────┴────┐
    │  OTLP   │ (HTTP Exporter)
    │ Exporter│
    └────┬────┘
         │
    ┌────┴────┐
    │ Sentry  │
    │ Platform│
    └─────────┘
```

This application supports two export modes:

- **Direct**: App → Sentry (default)
- **Collector**: App → OpenTelemetry Collector → Sentry

## Quick Start

### 1. Clone and Install

```bash
cd api
npm install
```

### 2. Setup Database

Create a free PostgreSQL database with Neon:

```bash
npx neonctl@latest projects create --name sentry-build-otlp-workshop -y
```

Copy the connection string provided (looks like `postgresql://user:pass@host.neon.tech/neondb?sslmode=require`)

### 3. Configure Environment

Copy the example environment file and configure:

```bash
cp .env.example .env
```

Edit `.env` and add:

1. Your **Neon database URL** from Step 2:

```bash
DATABASE_URL=postgresql://user:pass@host.neon.tech/neondb?sslmode=require
```

2. Your **Sentry OTLP endpoints** (get from Sentry: Project Settings > Client Keys):

```bash
OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=https://YOUR-ORG.ingest.sentry.io/api/YOUR-PROJECT-ID/integration/otlp/v1/traces
OTEL_EXPORTER_OTLP_TRACES_HEADERS="x-sentry-auth=sentry sentry_key=YOUR_PUBLIC_KEY"
```

### 4. Initialize Database

```bash
npm run db:setup
```

This creates tables and seeds sample data (users, products) in your Neon database.

### 5. Start the Application

```bash
npm start
```

For development with auto-reload:

```bash
npm run dev
```

The API will be available at `http://localhost:3000`

## API Endpoints

### Health Check

```bash
GET /health
```

### Products

```bash
# Get all products
GET /api/products

# Get product by ID
GET /api/products/:id

# Search products
GET /api/products/search?q=laptop
```

### Orders

```bash
# Create order
POST /api/orders
Body: {
  "userId": 1,
  "items": [
    { "productId": 1, "quantity": 2 }
  ],
  "paymentMethod": "credit_card"
}

# Get order by ID
GET /api/orders/:id

# Get user orders
GET /api/orders/user/:userId
```

## Testing

### Manual Testing

```bash
# Health check
curl http://localhost:3000/health

# Get products
curl http://localhost:3000/api/products

# Create order
curl -X POST http://localhost:3000/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "userId": 1,
    "items": [{"productId": 1, "quantity": 1}],
    "paymentMethod": "credit_card"
  }'
```

### Load Testing

Run the built-in load test to generate realistic traffic:

```bash
npm test
```

This will:

- Fetch products (testing cache)
- Create orders (testing transactions)
- Trigger various error scenarios
- Generate concurrent requests
- Display statistics

## OpenTelemetry Features Demonstrated

### Auto-Instrumentation

- ✅ HTTP requests (incoming/outgoing)
- ✅ Express routes
- ✅ PostgreSQL queries
- ✅ In-memory cache operations

### Manual Instrumentation

- ✅ Custom spans for business logic
- ✅ Custom attributes (user IDs, order IDs, SKUs)
- ✅ Events (cache hits, payment failures)
- ✅ Error recording
- ✅ Span status tracking

### Example Traces in Sentry

**Order Creation Flow:**

```
POST /api/orders
  ├─ order.create
  │   ├─ SELECT users (Postgres)
  │   ├─ SELECT products (Postgres)
  │   ├─ inventory.check
  │   │   └─ SELECT products (Postgres)
  │   ├─ BEGIN/INSERT/COMMIT (Transaction)
  │   ├─ inventory.reserve
  │   │   ├─ UPDATE products (Postgres)
  │   │   └─ cache.delete (in-memory)
  │   └─ payment.process
  │       └─ [External API simulation]
```

## Error Scenarios

The application includes realistic error scenarios:

1. **404 Not Found**: Invalid product/order IDs
2. **400 Bad Request**: Validation errors
3. **409 Conflict**: Insufficient inventory
4. **422 Unprocessable**: Payment failures (~10% random rate)
5. **500 Server Error**: Database connection issues

All errors are captured in spans and sent to Sentry with full context.

## Configuration

Key environment variables:

| Variable                             | Description                       | Default                        |
| ------------------------------------ | --------------------------------- | ------------------------------ |
| `PORT`                               | Server port                       | 3000                           |
| `NODE_ENV`                           | Environment                       | development                    |
| `DATABASE_URL`                       | Neon PostgreSQL connection string | (required)                     |
| `OTEL_SERVICE_NAME`                  | Service name in traces            | sentry-build-otlp-workshop-api |
| `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` | Sentry OTLP endpoint              | (required)                     |
| `OTEL_EXPORTER_OTLP_TRACES_HEADERS`  | Sentry auth header                | (required)                     |

## Development Tips

### Enable OTEL Debug Logs

Uncomment in `instrumentation.js`:

```javascript
diag.setLogger(new DiagConsoleLogger(), DiagLogLevel.DEBUG);
```

### Trigger Specific Errors

```bash
# 404 - Product not found
curl http://localhost:3000/api/products/99999

# 409 - Insufficient inventory
curl -X POST http://localhost:3000/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "userId": 1,
    "items": [{"productId": 1, "quantity": 99999}],
    "paymentMethod": "credit_card"
  }'

# 422 - Payment failure (10% random chance)
# Just create multiple orders - some will fail
```

### Monitor Slow Queries

Queries taking >1s are logged to console. Check your application logs.

## Troubleshooting

**Can't connect to database:**

- Verify your DATABASE_URL is correct in .env
- Make sure it includes `?sslmode=require` at the end
- Check your Neon project is active at https://console.neon.tech

**Not seeing traces in Sentry:**

1. Verify your OTLP endpoint URL is correct
2. Check that your Sentry public key is valid
3. Ensure the environment variables are properly set (no typos)
4. Enable OTEL debug logging to see export attempts

**Port already in use:**

```bash
# Change PORT in .env
PORT=3001
```

## Switching Export Modes

Switch between direct export to Sentry and using an OpenTelemetry Collector:

```bash
# Direct Mode (App → Sentry)
npm run mode:direct
npm start

# Collector Mode (App → Collector → Sentry)
npm run mode:collector
npm start

# Check current mode
npm run mode:status

# Health check for collector (if running)
npm run collector:health
```

## License

MIT
