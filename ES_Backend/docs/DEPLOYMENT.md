# Deployment & Infrastructure - ES Backend

## Deployment Architecture

### Current: Development/Single-Host Deployment

```
┌─────────────────────────────────────────────┐
│         Single Server/Laptop                 │
├─────────────────────────────────────────────┤
│                                              │
│  ┌─ Docker Container ────────────────────┐  │
│  │ UserProfileService (Port 7000)        │  │
│  └───────────────────────────────────────┘  │
│                                              │
│  ┌─ Docker Container ────────────────────┐  │
│  │ EventsService (Port 13000)            │  │
│  └───────────────────────────────────────┘  │
│                                              │
│  ┌─ Docker Container ────────────────────┐  │
│  │ TeamsService (Port 17000)             │  │
│  └───────────────────────────────────────┘  │
│                                              │
│  ┌─ Docker Container ────────────────────┐  │
│  │ MentorProfileService (Port 10000)     │  │
│  └───────────────────────────────────────┘  │
│                                              │
│  ┌─ Docker Container ────────────────────┐  │
│  │ MongoDBService (Port 14000)           │  │
│  └───────────────────────────────────────┘  │
│                                              │
│  ┌─ Docker Container ────────────────────┐  │
│  │ BlobStorageService (Port 15000)       │  │
│  └───────────────────────────────────────┘  │
│                                              │
│  ┌─ Docker Container ────────────────────┐  │
│  │ MongoDB Database                       │  │
│  │ Port: 27017                            │  │
│  └───────────────────────────────────────┘  │
│                                              │
│  ┌─ Docker Container ────────────────────┐  │
│  │ MinIO (S3 Storage)                     │  │
│  │ Port: 9000 (API), 9001 (UI)            │  │
│  └───────────────────────────────────────┘  │
│                                              │
│  ┌─ Docker Container ────────────────────┐  │
│  │ RabbitMQ                              │  │
│  │ Port: 5672 (AMQP), 15672 (UI)         │  │
│  └───────────────────────────────────────┘  │
│                                              │
└─────────────────────────────────────────────┘
```

---

## Docker Compose Configuration

### Location
`ServerScripts/docker-compose.yml`

### Services Defined

```yaml
version: '3.8'

services:
  # Message Queue
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"      # AMQP port
      - "15672:15672"    # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5

  # Blob Storage
  minio:
    image: minio/minio:latest
    ports:
      - "9000:9000"      # S3 API
      - "9001:9001"      # Web UI
    environment:
      MINIO_ROOT_USER: admin
      MINIO_ROOT_PASSWORD: password
    volumes:
      - "./blob-storage:/data"
    command: minio server /data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 10s
      retries: 5
```

### Environment Variables

Create `.env` file in project root:

```env
# Database
MONGODB_HOST=localhost
MONGODB_PORT=27017
MONGODB_DB=ES
MONGODB_USERNAME=root
MONGODB_PASSWORD=password

# Storage
MINIO_HOST=localhost
MINIO_PORT=9000
MINIO_ACCESS_KEY=admin
MINIO_SECRET_KEY=password
MINIO_BUCKET_PREFIX=es-

# Message Queue
RABBITMQ_HOST=localhost
RABBITMQ_PORT=5672
RABBITMQ_USER=guest
RABBITMQ_PASSWORD=guest

# Services
SERVER_IP=localhost
SERVICE_PORT=5000  # Default, override per service

# OAuth (Future)
GOOGLE_CLIENT_ID=your-client-id
GOOGLE_CLIENT_SECRET=your-client-secret
GOOGLE_REDIRECT_URI=http://localhost:5000/auth/google-callback

# Logging
LOG_LEVEL=INFO
LOG_FORMAT=json
```

---

## Port Allocation

### Reserved Ports

| Port | Service | Protocol | Purpose |
|------|---------|----------|---------|
| 5000 | AuthService | HTTP | OAuth 2.0 Authentication |
| 5672 | RabbitMQ | AMQP | Message Queue |
| 6000 | DashboardService | HTTP | Analytics Dashboard |
| 7000 | UserProfileService | HTTP | User Management |
| 8000 | NewsService | HTTP | News Aggregation |
| 9000 | MinIO | HTTP | S3-compatible Storage |
| 9001 | MinIO Console | HTTP | Storage Management UI |
| 9000-10000 | Reserved | - | Future services |
| 11000 | ChatService | WebSocket | Real-time Messaging |
| 12000 | LeagueSystemService | HTTP | Rankings/Matchmaking |
| 13000 | EventsService | HTTP | Event Management |
| 14000 | MongoDBService | HTTP | Database Hub |
| 15000 | BlobStorageService | HTTP | File Storage API |
| 15672 | RabbitMQ UI | HTTP | Queue Management |
| 16000 | ImageServerService | HTTP | CDN Serving |
| 17000 | TeamsService | HTTP | Team Management |
| 27017 | MongoDB | mongod | Database |

### Blocked Ports
- Standard ports (80, 443): Reserved for API Gateway
- System ports (<1000): Avoid if possible

---

## Startup & Shutdown

### Start All Services

**Using provided script**:
```bash
cd ServerScripts
./TerminalStartServer.sh
```

**Using Docker Compose**:
```bash
docker-compose -f ServerScripts/docker-compose.yml up -d
```

**Manual startup** (for debugging):
```bash
# Terminal 1: Infrastructure
docker-compose -f ServerScripts/docker-compose.yml up

# Terminal 2-7: Each service
cd service_userprofile && python UserProfileService.py
cd service_events && python EventsService.py
cd service_teams && python TeamsService.py
cd service_mentorprofile && python MentorProfileService.py
cd service_mongodb && python MongoDBService.py
cd service_blobstorage && python BlobStorageService.py
```

### Check Service Status

```bash
# Check all containers
docker-compose -f ServerScripts/docker-compose.yml ps

# Check logs
docker-compose -f ServerScripts/docker-compose.yml logs -f [service-name]

# Check service health
curl http://localhost:{PORT}/health  # Once health endpoints are added
```

### Restart Services

**All services**:
```bash
./TerminalRestartServer.sh
```

**Specific service**:
```bash
docker-compose -f ServerScripts/docker-compose.yml restart service-name
```

### Stop Services

```bash
python StopServer.py
# OR
docker-compose -f ServerScripts/docker-compose.yml down
```

---

## Infrastructure Requirements

### Development Environment
```
CPU:     2+ cores (quad-core recommended)
RAM:     4GB (8GB recommended)
Disk:    50GB (SSD recommended for database)
Network: Stable local network or localhost
```

### Staging Environment
```
CPU:     4 cores
RAM:     8GB
Disk:    100GB SSD
Network: 1Gbps+ network connection
```

### Production Environment
```
CPU:     8+ cores
RAM:     16GB+
Disk:    1TB+ SSD (RAID configuration)
Network: 10Gbps+ network, multiple regions
Backup:  Automated daily backups
Monitoring: 24/7 health monitoring
```

---

## Service Dependencies

### Startup Order (Critical)

```
1. RabbitMQ       (Message queue infrastructure)
2. MongoDB        (Data storage)
3. MinIO          (Blob storage)
4. MongoDBService (Database API hub)
5. BlobStorageService (File storage API)
6. API Services   (Can start in any order)
   ├─ UserProfileService
   ├─ EventsService
   ├─ TeamsService
   └─ MentorProfileService
```

### Health Checks

**MongoDB Service**:
```python
# Check connectivity
async def health_check():
    try:
        await db_client.admin.command('ping')
        return {"status": "healthy"}
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}
```

**BlobStorageService**:
```python
# Check S3 connectivity
async def health_check():
    try:
        response = await s3_client.list_buckets()
        return {"status": "healthy", "buckets": len(response['Buckets'])}
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}
```

---

## Data Persistence

### MongoDB Data

**Default location**: `/var/lib/mongodb`

**Docker volume**:
```yaml
services:
  mongodb:
    volumes:
      - mongodb_data:/data/db
      - mongodb_config:/data/configdb

volumes:
  mongodb_data:
  mongodb_config:
```

**Backup strategy**:
```bash
# Automated backup daily
0 2 * * * mongodump --archive=/backups/es-$(date +\%Y\%m\%d).archive

# Restore
mongorestore --archive=/backups/es-20240127.archive
```

### MinIO Data

**Default location**: `./blob-storage` (from docker-compose)

**Docker volume**:
```yaml
services:
  minio:
    volumes:
      - ./blob-storage:/data
      - ./minio-config:/etc/minio
```

**Backup strategy**:
```bash
# Sync buckets to S3
aws s3 sync s3://es-user-profile-pic s3://backup-bucket/user-profile-pic
```

---

## Logging & Monitoring

### Current Logging

Each service logs to stdout:
```bash
docker-compose logs -f service_userprofile
```

### Recommended Logging Stack

```
┌──────────────┐
│ Application  │
└──────┬───────┘
       │ (stdout)
       ↓
┌──────────────┐
│ Filebeat     │
└──────┬───────┘
       │ (forwarding)
       ↓
┌──────────────┐
│ Elasticsearch│
└──────┬───────┘
       │
       ↓
┌──────────────┐
│ Kibana       │ (Dashboard)
└──────────────┘
```

### Metrics Collection

**Recommended**: Prometheus + Grafana

```
Service Metrics
    ↓
Prometheus Scraper
    ↓
Time-series DB
    ↓
Grafana Dashboard
```

---

## Scaling Strategy

### Horizontal Scaling (Add More Instances)

#### Phase 1: Load Balancer
```
                    Load Balancer (Nginx)
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
    Service-1          Service-2          Service-3
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
                    Shared Database
```

**Implementation**:
```nginx
upstream userprofile {
    server localhost:7001;
    server localhost:7002;
    server localhost:7003;
}

server {
    listen 7000;
    location / {
        proxy_pass http://userprofile;
    }
}
```

#### Phase 2: Database Replication
```
Primary MongoDB          Secondary MongoDB
    (writes)                 (reads)
        │                       │
        └─────── Replication ───┘
```

#### Phase 3: Caching Layer
```
Redis Cache
    │
Service Instances
    │
Database
```

### Vertical Scaling (More Resources)

Increase per-service:
- CPU allocation (Docker: `cpus` limit)
- Memory allocation (Docker: `memory` limit)
- Worker threads (FastAPI: `workers` parameter)

---

## Continuous Integration/Deployment

### Git Workflow

```
Feature Branch
    ↓
Create Pull Request
    ↓
Run Tests (CI)
    ↓
Code Review
    ↓
Merge to Main
    ↓
Deploy to Staging (CD)
    ↓
Run Integration Tests
    ↓
Deploy to Production
```

### CI Pipeline (GitHub Actions Example)

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-python@v2
        with:
          python-version: 3.9
      - run: pip install -r requirements.txt
      - run: pytest tests/
      - run: black --check .
      - run: pylint service_*/

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - run: docker build -t es-backend:latest .
      - run: docker push registry/es-backend:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: docker-compose pull
      - run: docker-compose up -d
```

---

## Security Considerations

### Current State
- ⚠️ No authentication
- ⚠️ CORS allows all origins
- ⚠️ No HTTPS/TLS
- ⚠️ No rate limiting
- ⚠️ Services exposed directly

### Recommended for Production

#### 1. API Gateway
```
Client
    ↓ HTTPS
    ↓ Authentication
    ↓ Rate Limiting
    ↓ Request Validation
API Gateway
    ↓ HTTP (internal)
Services
```

#### 2. TLS/HTTPS
```
services:
  userprofile:
    volumes:
      - ./certs/:/etc/ssl/certs/
    environment:
      SSL_CERT: /etc/ssl/certs/server.crt
      SSL_KEY: /etc/ssl/certs/server.key
```

#### 3. Network Isolation
```
┌─────────────────────────────────────┐
│  Public Network (Firewall)          │
├─────────────────────────────────────┤
│  API Gateway (Port 443)             │
├─────────────────────────────────────┤
│  Private Network (VPC)              │
├─────────────────────────────────────┤
│  Internal Services                  │
│  Database (no external access)      │
│  Cache (no external access)         │
└─────────────────────────────────────┘
```

#### 4. Secrets Management
```
# Use environment variables (avoid hardcoding)
MONGODB_PASSWORD=${MONGODB_PASSWORD}
MINIO_SECRET_KEY=${MINIO_SECRET_KEY}

# Or use secrets manager
docker secret create mongodb_password -
```

#### 5. Input Validation
```python
from pydantic import BaseModel, EmailStr, validator

class UserCreate(BaseModel):
    user_name: str = Field(..., min_length=3, max_length=50)
    email: EmailStr

    @validator('user_name')
    def user_name_alphanumeric(cls, v):
        assert v.isalnum(), 'must be alphanumeric'
        return v
```

---

## Database Backup & Recovery

### Automated Backup Script

```bash
#!/bin/bash
# backup.sh

BACKUP_DIR="/backups/es"
DATE=$(date +%Y%m%d_%H%M%S)

# MongoDB backup
mongodump --archive="$BACKUP_DIR/mongodb_$DATE.archive"

# MinIO backup
s3cmd sync s3://es-prod-bucket "$BACKUP_DIR/minio_$DATE/"

# Compress backups
tar -czf "$BACKUP_DIR/backup_$DATE.tar.gz" "$BACKUP_DIR"

# Upload to S3 for redundancy
aws s3 cp "$BACKUP_DIR/backup_$DATE.tar.gz" s3://backup-bucket/

# Cleanup old backups (keep 30 days)
find "$BACKUP_DIR" -type f -mtime +30 -delete

echo "Backup completed: $DATE"
```

### Recovery Procedure

```bash
# Restore MongoDB from archive
mongorestore --archive=/backups/mongodb_20240127_020000.archive

# Restore MinIO from backup
s3cmd sync /backups/minio_20240127_020000/ s3://es-prod-bucket
```

---

## Disaster Recovery Plan

### RTO/RPO Targets
```
RTO (Recovery Time Objective): 4 hours
RPO (Recovery Point Objective): 1 hour
```

### Failover Strategy

**Current**: Single point of failure

**Recommended**:
```
Primary Datacenter      Secondary Datacenter
    (Active)                (Standby)
        │                        │
        └──────  Replication ────┘

On failure:
1. DNS failover to secondary
2. Promote secondary to primary
3. Alert ops team
```

---

## Monitoring & Alerting

### Health Check Endpoints (To Implement)

```python
@app.get("/health")
async def health():
    return {"status": "healthy", "service": "UserProfileService"}

@app.get("/health/live")
async def liveness():
    # Check if service is responding
    return {"status": "alive"}

@app.get("/health/ready")
async def readiness():
    # Check if service is ready to accept traffic
    db_connected = await check_db()
    storage_connected = await check_storage()
    return {
        "ready": db_connected and storage_connected
    }
```

### Alerting Rules

```
Priority 1 (Critical):
├─ Service down (no response for 2 minutes)
├─ Database unreachable
└─ Storage service unavailable

Priority 2 (High):
├─ Error rate > 5%
├─ Response time > 2s (P95)
└─ Memory usage > 80%

Priority 3 (Medium):
├─ Error rate > 1%
├─ Response time > 500ms (P95)
└─ Disk usage > 80%
```

---

## Summary

**Current Deployment**:
- Single-host Docker Compose
- Suitable for development and small staging
- All services on localhost
- File-based storage for data

**Recommended Next Steps**:
1. Implement health check endpoints
2. Set up centralized logging (ELK stack)
3. Add metrics collection (Prometheus)
4. Implement backup automation
5. Plan for multi-region deployment
6. Add API gateway and authentication

**Production Readiness Checklist**:
- [ ] Authentication (OAuth 2.0)
- [ ] HTTPS/TLS encryption
- [ ] Rate limiting & DDoS protection
- [ ] Database replication & backups
- [ ] Monitoring & alerting
- [ ] Log aggregation
- [ ] Disaster recovery plan
- [ ] Load balancing
- [ ] Secrets management
- [ ] Security audit

