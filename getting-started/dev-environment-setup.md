# Infrar Development Environment Setup

**Complete guide to setting up your local development environment**

---

## Prerequisites

Before starting, ensure you have these installed:

### Required
- **Docker Desktop** (or Docker Engine + Docker Compose)
- **Go 1.21+** - [Download](https://go.dev/dl/)
- **Node.js 18+** - [Download](https://nodejs.org/)
- **Python 3.11+** - [Download](https://www.python.org/downloads/)
- **PostgreSQL Client** (`psql`) - For running migrations

### Optional (but recommended)
- **Git** - For version control
- **VS Code** - With Go, Python, and TypeScript extensions
- **Postman** or **curl** - For API testing

---

## Quick Start

### 1. Start Infrastructure Services

```bash
cd /home/alexb/projects/infrar
./dev-start.sh
```

This will:
- Start PostgreSQL on port 5432
- Start Redis on port 6379
- Create databases: `infrar_platform_dev` and `infrar_cost_dev`
- Wait for services to be healthy

**Output**:
```
🚀 Starting Infrar Development Environment...

✓ Docker is running
✓ Infrastructure services started

Services running:
  • PostgreSQL: localhost:5432
    - Platform DB: infrar_platform_dev
    - Cost DB: infrar_cost_dev
    - User: infrar / Password: devpassword

  • Redis: localhost:6379
```

### 2. Setup infrar-platform (Backend API)

```bash
cd infrar-platform

# Copy environment configuration
cp .env.dev .env

# Install Go dependencies
go mod download

# Run database migrations
./scripts/run-migrations.sh

# Start the API server
go run cmd/api/main.go
```

**Expected output**:
```
Starting Infrar Platform API...
Environment: development
Database: Connected to infrar_platform_dev
Redis: Connected
Server: Listening on :8080
```

**Test it**:
```bash
curl http://localhost:8080/health
# Expected: {"status":"healthy"}
```

### 3. Setup infrar-cost-intelligence (Cost API)

```bash
cd infrar-cost-intelligence

# Copy environment configuration
cp .env.dev .env

# Create Python virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Start the API server
uvicorn app.main:app --reload --port 8000
```

**Expected output**:
```
INFO:     Uvicorn running on http://127.0.0.1:8000
INFO:     Application startup complete
```

**Test it**:
```bash
curl http://localhost:8000/health
# Expected: {"status":"healthy","service":"infrar-cost-intelligence"}
```

### 4. Setup infrar-web (Frontend UI)

```bash
cd infrar-web

# Install dependencies
npm install

# Start development server
npm run dev
```

**Expected output**:
```
▲ Next.js 14.0.4
- Local:        http://localhost:3000
- Environments: .env.local

✓ Ready in 2.5s
```

**Test it**:
Open http://localhost:3000 in your browser

---

## Service Ports

| Service | Port | URL |
|---------|------|-----|
| PostgreSQL | 5432 | localhost:5432 |
| Redis | 6379 | localhost:6379 |
| infrar-platform (API) | 8080 | http://localhost:8080 |
| infrar-cost-intelligence | 8000 | http://localhost:8000 |
| infrar-web (Frontend) | 3000 | http://localhost:3000 |
| pgAdmin (optional) | 5050 | http://localhost:5050 |
| Redis Commander (optional) | 8081 | http://localhost:8081 |

---

## Database Setup

### Auto-created Databases

When you run `./dev-start.sh`, these databases are created automatically:

1. **infrar_platform_dev** - Used by infrar-platform
   - Tables: users, projects, deployment_configs, cloud_credentials, etc.

2. **infrar_cost_dev** - Used by infrar-cost-intelligence
   - Tables: pricing, predictions (to be created)

### Running Migrations

**infrar-platform**:
```bash
cd infrar-platform
./scripts/run-migrations.sh
```

This applies all SQL migrations in `migrations/` directory:
- 001_create_users_table.sql
- 002_create_projects_table.sql
- 003_create_deployment_configs_table.sql
- 004_create_cloud_credentials_table.sql
- 005_create_terraform_states_table.sql
- 006_create_deployments_table.sql
- 007_create_pricing_data_table.sql

### Viewing Database Content

**Option 1: Using psql**
```bash
# Connect to platform database
psql -h localhost -p 5432 -U infrar -d infrar_platform_dev

# List tables
\dt

# Query users
SELECT * FROM users;
```

**Option 2: Using pgAdmin**
```bash
# Start pgAdmin
docker-compose -f docker-compose.dev.yml --profile tools up -d pgadmin

# Open http://localhost:5050
# Login: admin@infrar.io / admin

# Add server:
# - Host: postgres (or host.docker.internal on Mac)
# - Port: 5432
# - Database: infrar_platform_dev
# - Username: infrar
# - Password: devpassword
```

---

## Environment Configuration

### infrar-platform (.env.dev)

```bash
# Database
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=infrar_platform_dev
DATABASE_USER=infrar
DATABASE_PASSWORD=devpassword

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# JWT
JWT_SECRET=dev-secret-key-change-in-production-1234567890

# Cost Intelligence API
COST_API_URL=http://localhost:8000
COST_API_KEY=infrar-mvp-dev-key-123

# CORS
CORS_ALLOWED_ORIGINS=http://localhost:3000
```

### infrar-cost-intelligence (.env.dev)

```bash
# Database
DATABASE_URL=postgresql://infrar:devpassword@localhost:5432/infrar_cost_dev

# Redis (using DB 1, separate from platform)
REDIS_URL=redis://localhost:6379/1

# API
API_HOST=0.0.0.0
API_PORT=8000
DEV_API_KEY=infrar-mvp-dev-key-123

# CORS
CORS_ORIGINS=http://localhost:3000,http://localhost:8080
```

### infrar-web (.env.local)

```bash
# API URL
NEXT_PUBLIC_API_URL=http://localhost:8080
```

---

## Testing the Complete Stack

### 1. Test Health Endpoints

```bash
# Platform API
curl http://localhost:8080/health
# Expected: {"status":"healthy"}

# Cost Intelligence API
curl http://localhost:8000/health
# Expected: {"status":"healthy","service":"infrar-cost-intelligence"}
```

### 2. Test Database Connection

```bash
# Platform database
psql -h localhost -p 5432 -U infrar -d infrar_platform_dev -c "SELECT NOW();"

# Cost intelligence database
psql -h localhost -p 5432 -U infrar -d infrar_cost_dev -c "SELECT NOW();"
```

### 3. Test Redis Connection

```bash
# Test Redis
docker exec infrar-redis redis-cli ping
# Expected: PONG

# Set and get a value
docker exec infrar-redis redis-cli SET test "Hello Infrar"
docker exec infrar-redis redis-cli GET test
# Expected: "Hello Infrar"
```

### 4. Test API Integration

```bash
# From platform, test cost API call
curl -X POST http://localhost:8000/api/v1/predict \
  -H "Content-Type: application/json" \
  -H "X-API-Key: infrar-mvp-dev-key-123" \
  -d '{
    "capabilities": ["storage"],
    "requirements": {"storage_gb": 100},
    "providers": ["aws", "gcp"]
  }'
# Should return cost predictions (currently stub)
```

---

## Troubleshooting

### Issue: PostgreSQL won't start

**Symptoms**: `docker-compose up` fails or postgres container exits

**Solution**:
```bash
# Stop all containers
docker-compose -f docker-compose.dev.yml down

# Remove volumes (WARNING: deletes all data)
docker-compose -f docker-compose.dev.yml down -v

# Start fresh
./dev-start.sh
```

### Issue: Port already in use

**Symptoms**: `Error: bind: address already in use`

**Solution**:
```bash
# Find what's using the port (example: 5432)
lsof -i :5432

# Kill the process or change the port in docker-compose.dev.yml
```

### Issue: Database connection refused

**Symptoms**: `could not connect to server: Connection refused`

**Solution**:
```bash
# Check if PostgreSQL is running
docker ps | grep postgres

# Check PostgreSQL logs
docker logs infrar-postgres

# Restart PostgreSQL
docker restart infrar-postgres
```

### Issue: Migration fails

**Symptoms**: `ERROR: relation "users" already exists`

**Solution**:
```bash
# Check which migrations were applied
psql -h localhost -U infrar -d infrar_platform_dev \
  -c "SELECT * FROM schema_migrations ORDER BY applied_at;"

# If you need to reset (WARNING: deletes all data)
psql -h localhost -U infrar -d infrar_platform_dev \
  -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"

# Run migrations again
cd infrar-platform && ./scripts/run-migrations.sh
```

### Issue: Go dependencies fail to download

**Symptoms**: `go: error loading module requirements`

**Solution**:
```bash
cd infrar-platform
go mod tidy
go mod download
```

### Issue: Python dependencies fail to install

**Symptoms**: `ERROR: Could not find a version that satisfies...`

**Solution**:
```bash
# Upgrade pip
pip install --upgrade pip

# Install dependencies again
pip install -r requirements.txt

# If still fails, create fresh venv
deactivate
rm -rf venv
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### Issue: npm install fails

**Symptoms**: `ERESOLVE unable to resolve dependency tree`

**Solution**:
```bash
# Clear npm cache
npm cache clean --force

# Remove node_modules and package-lock.json
rm -rf node_modules package-lock.json

# Install again
npm install
```

---

## Stopping the Environment

### Stop All Services
```bash
cd /home/alexb/projects/infrar
./dev-stop.sh
```

### Stop Individual Services

```bash
# Stop infrastructure only
docker-compose -f docker-compose.dev.yml down

# Stop API servers (Ctrl+C in their terminals)
```

### Keep Data vs Clean Slate

**Keep data** (default):
```bash
docker-compose -f docker-compose.dev.yml down
# Volumes are preserved, data persists
```

**Clean slate** (delete all data):
```bash
docker-compose -f docker-compose.dev.yml down -v
# Volumes are removed, all data deleted
```

---

## Development Workflow

### Daily Workflow

1. **Start infrastructure**:
   ```bash
   ./dev-start.sh
   ```

2. **Start services in separate terminals**:
   ```bash
   # Terminal 1: Platform API
   cd infrar-platform && go run cmd/api/main.go

   # Terminal 2: Cost Intelligence API
   cd infrar-cost-intelligence && source venv/bin/activate && uvicorn app.main:app --reload

   # Terminal 3: Web UI
   cd infrar-web && npm run dev
   ```

3. **Make changes and test**

4. **Stop services**: Ctrl+C in each terminal

5. **Stop infrastructure**:
   ```bash
   ./dev-stop.sh
   ```

### Testing Changes

**Backend API**:
```bash
# Run tests
cd infrar-platform
go test ./... -v

# Test specific endpoint
curl http://localhost:8080/api/v1/auth/register \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}'
```

**Cost Intelligence**:
```bash
# Test endpoint
curl http://localhost:8000/api/v1/predict \
  -X POST \
  -H "Content-Type: application/json" \
  -H "X-API-Key: infrar-mvp-dev-key-123" \
  -d '{"capabilities":["storage"],"providers":["aws"]}'
```

**Frontend**:
- Open http://localhost:3000 in browser
- Changes auto-reload

---

## Optional Development Tools

### Start pgAdmin (Database UI)

```bash
docker-compose -f docker-compose.dev.yml --profile tools up -d pgadmin
```

Open http://localhost:5050
- Email: admin@infrar.io
- Password: admin

### Start Redis Commander (Redis UI)

```bash
docker-compose -f docker-compose.dev.yml --profile tools up -d redis-commander
```

Open http://localhost:8081

---

## Next Steps

Once your environment is running:

1. ✅ **Implement Authentication** - Start with `infrar-platform/internal/api/auth/`
2. ✅ **Implement Cost Stub** - Update `infrar-cost-intelligence/app/main.py`
3. ✅ **Create Basic UI** - Start with `infrar-web/src/app/` pages
4. ✅ **Test End-to-End** - Register user, create project, see costs

See: [MVP_IMPLEMENTATION_PLAN.md](MVP_IMPLEMENTATION_PLAN.md) for detailed next steps.

---

## Quick Reference

### Start Everything
```bash
./dev-start.sh
cd infrar-platform && go run cmd/api/main.go &
cd infrar-cost-intelligence && uvicorn app.main:app --reload &
cd infrar-web && npm run dev &
```

### Stop Everything
```bash
./dev-stop.sh
pkill -f "go run"
pkill -f "uvicorn"
pkill -f "next dev"
```

### Reset Database
```bash
docker-compose -f docker-compose.dev.yml down -v
./dev-start.sh
cd infrar-platform && ./scripts/run-migrations.sh
```

---

**Status**: ✅ Development environment ready!

**Next**: See [MVP_IMPLEMENTATION_PLAN.md](MVP_IMPLEMENTATION_PLAN.md) for implementation steps.
