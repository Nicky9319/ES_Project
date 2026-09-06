# System Overview - ES Backend

## Executive Summary

ES Backend is a **distributed microservices architecture** designed for a comprehensive esports platform. The system orchestrates multiple independent services to manage users, events, teams, mentors, and community features while maintaining scalability, modularity, and separation of concerns.

---

## System Context Diagram

```
                           ┌─────────────────────────────┐
                           │   Client Applications       │
                           │ (Web, Mobile, Desktop)      │
                           └──────────────┬──────────────┘
                                          │
                                          │ HTTP REST APIs
                                          │
                     ┌────────────────────┼────────────────────┐
                     │                    │                    │
          ┌──────────▼──────────┐  ┌──────▼──────────┐  ┌──────▼────────────┐
          │ User Management     │  │ Event/Teams     │  │ Mentorship        │
          │ Services            │  │ Management      │  │ Services          │
          │                     │  │                 │  │                   │
          │ - User Profiles     │  │ - Events        │  │ - Mentor Profiles │
          │ - Authentication    │  │ - Teams         │  │ - Sessions        │
          │ - Profile Images    │  │ - Milestones    │  │ - Ratings         │
          └──────┬──────────────┘  └────────┬────────┘  └────────┬──────────┘
                 │                          │                    │
                 └──────────────┬───────────┴────────────────────┘
                                │
                    ┌───────────▼──────────────┐
                    │  Core Service Hub       │
                    │  (MongoDB Service)      │
                    │  (Blob Storage Service) │
                    └───────────┬──────────────┘
                                │
                    ┌───────────┴──────────────┐
                    │                         │
          ┌─────────▼──────────┐  ┌──────────▼─────────┐
          │   MongoDB          │  │   MinIO/S3        │
          │   Database         │  │   Blob Storage    │
          └────────────────────┘  └───────────────────┘
```

---

## Architectural Principles

### 1. **Microservices Architecture**
- **Principle**: Decompose the system into small, independent, loosely-coupled services
- **Benefit**: Each service can be developed, deployed, and scaled independently
- **Implementation**: 13 services (6 implemented, 7 planned/in-progress)

### 2. **Service Orchestration Pattern**
- **Principle**: Higher-level services coordinate with lower-level services
- **Benefit**: Clear layering and responsibility separation
- **Example**: UserProfileService orchestrates MongoDB + BlobStorage operations

### 3. **Asynchronous-First Design**
- **Principle**: All operations use async/await patterns
- **Benefit**: Better resource utilization, high concurrency
- **Implementation**: FastAPI + Uvicorn + async httpx for inter-service calls

### 4. **Separation of Concerns**
- **Database Service**: Handles all CRUD operations
- **Blob Service**: Manages all file storage
- **API Services**: Business logic and orchestration
- **Benefit**: Easier to maintain, test, and evolve

### 5. **Loose Coupling via REST APIs**
- **Principle**: Services communicate via HTTP REST, not shared databases
- **Benefit**: Services are truly independent and can be deployed separately
- **Implementation**: Service discovery via ServiceURLMapping.json

---

## High-Level System Layers

```
┌─────────────────────────────────────────────────────────┐
│                  Presentation Layer                      │
│              (Client Applications)                       │
└────────────────────────┬────────────────────────────────┘
                         │ HTTP REST APIs
                         │
┌────────────────────────▼────────────────────────────────┐
│                 API Service Layer                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Domain Services (Orchestrators)                  │  │
│  │ - UserProfileService      - EventsService       │  │
│  │ - TeamsService            - MentorProfileSvc    │  │
│  │ - ChatService             - AuthService         │  │
│  │ - FeedService             - DashboardService    │  │
│  └──────────────────────────────────────────────────┘  │
│                         ▲                                │
│                         │ HTTP Calls                     │
└─────────────────────────┼────────────────────────────────┘
                          │
┌─────────────────────────▼────────────────────────────────┐
│              Infrastructure Service Layer                │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Core Services (Shared Infrastructure)            │  │
│  │ - MongoDBService         - BlobStorageService    │  │
│  │ - RabbitMQService (Future: Async Events)         │  │
│  └──────────────────────────────────────────────────┘  │
│                         ▲                                │
│                         │ Internal APIs                  │
└─────────────────────────┼────────────────────────────────┘
                          │
┌─────────────────────────▼────────────────────────────────┐
│              Data & Storage Layer                        │
│  ┌────────────────────┐      ┌──────────────────────┐  │
│  │   MongoDB          │      │   MinIO/S3           │  │
│  │   (Structured)     │      │   (Unstructured)     │  │
│  └────────────────────┘      └──────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## Service Inventory & Status

### Tier 1: Core Infrastructure (Always Running)
| Service | Port | Status | Role |
|---------|------|--------|------|
| **MongoDBService** | 14000 | ✅ Production | Central data repository |
| **BlobStorageService** | 15000 | ✅ Production | File/image management |

### Tier 2: User & Profile Management
| Service | Port | Status | Role |
|---------|------|--------|------|
| **UserProfileService** | 7000 | ✅ Production | User CRUD & profile mgmt |
| **MentorProfileService** | 10000 | ✅ Production | Mentor profiles & ratings |
| **AuthService** | 5000 | 🔄 WIP | OAuth 2.0 authentication |

### Tier 3: Entity Management
| Service | Port | Status | Role |
|---------|------|--------|------|
| **EventsService** | 13000 | ✅ Production | Event organization |
| **TeamsService** | 17000 | ✅ Production | Team management |
| **ChatService** | 11000 | 📋 Planned | Real-time messaging |

### Tier 4: Advanced Features
| Service | Port | Status | Role |
|---------|------|--------|------|
| **LeagueSystemService** | 12000 | 📋 Planned | Rankings & matchmaking |
| **FeedService** | 9000 | 📋 Planned | User feed generation |
| **NewsService** | 8000 | 📋 Planned | News aggregation |
| **DashboardService** | 6000 | 📋 Planned | Analytics dashboard |
| **ImageServerService** | 16000 | 📋 Planned | Image CDN/serving |

---

## Data Flow Hierarchy

### Level 1: Direct Client Requests
```
Client ──HTTP──> API Service ──HTTP──> Infrastructure Service ──→ Data Store
```

### Level 2: Orchestrated Flows
```
Client
  └──> UserProfileService
        ├──> MongoDBService (Create User Record)
        └──> BlobStorageService (Upload Images)
             └──> MongoDBService (Update URLs)
  └──> Response back to Client
```

### Level 3: Future Async Messaging (Available but Unused)
```
Event Service ──RabbitMQ──> Chat Service (Async notification)
                         ──> Notification Service (Async alert)
```

---

## Technology Stack Rationale

### Framework: FastAPI
- **Why**: Modern, fast, async-native Python framework
- **Benefit**: Built-in async support, auto-generated API docs (Swagger UI)
- **Alternative**: Could use aiohttp or Starlette, but FastAPI provides highest productivity

### Database: MongoDB
- **Why**: Flexible schema for evolving data models (early stage development)
- **Benefit**: Document-based (natural for complex nested objects like user profiles)
- **Alternative**: PostgreSQL would require strict schemas upfront

### File Storage: MinIO (S3-compatible)
- **Why**: Open-source, self-hosted alternative to AWS S3
- **Benefit**: Full control, no vendor lock-in, cost-effective
- **Alternative**: AWS S3 (cloud), File system (scalability issues)

### Message Queue: RabbitMQ
- **Why**: Mature, reliable message broker
- **Benefit**: Decouples services, enables async processing
- **Alternative**: Kafka (overkill for current needs), Redis (less robust)

### Infrastructure: Docker Compose
- **Why**: Simple orchestration for development and small deployments
- **Future**: Kubernetes for production scale

---

## Scalability Strategy

### Current State (Vertical Scaling)
- All services on single machine
- Resource limits via Docker memory/CPU allocation
- Load balanced via Nginx (or similar) if needed

### Future State (Horizontal Scaling)
```
                    ┌──────────────────┐
                    │   Load Balancer  │
                    └────────┬─────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
┌───────▼────────┐  ┌───────▼────────┐  ┌───────▼────────┐
│ UserProfile-1  │  │ UserProfile-2  │  │ UserProfile-3  │
└────────────────┘  └────────────────┘  └────────────────┘
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  Shared MongoDB  │
                    └──────────────────┘
```

**Strategy**:
1. Horizontal scaling of stateless API services
2. Database replication for read-heavy workloads
3. Caching layer (Redis) for frequently accessed data
4. Event streaming with Kafka for async operations

---

## Security Considerations

### Current Implementation
- CORS enabled for cross-origin requests
- Service discovery via configuration file
- No API authentication (OAuth2 in development)

### Production Recommendations
- **API Gateway**: Implement central authentication/authorization
- **TLS/HTTPS**: Encrypt all inter-service communication
- **Rate Limiting**: Protect endpoints from abuse
- **Input Validation**: Validate all user inputs
- **Audit Logging**: Track all operations for compliance

---

## Deployment Strategy

### Development
- Docker Compose with all services on single host
- RabbitMQ and MinIO containerized
- MongoDB containerized

### Staging
- Multi-host Docker Compose
- Separate database server
- Load balancing

### Production
- Kubernetes orchestration
- Managed database service (MongoDB Atlas or similar)
- Object storage (AWS S3 or MinIO cluster)
- Message queue cluster
- API gateway and load balancing
- CI/CD pipeline for automated deployments

---

## Performance Characteristics

### Latency Profile
- **P50**: 50-100ms (simple CRUD operations)
- **P95**: 200-500ms (multi-service orchestration)
- **P99**: 1-2 seconds (with file uploads)

### Throughput
- **Current**: ~100-500 concurrent users per service instance
- **Scalable to**: 10,000+ with horizontal scaling

### Resource Requirements
| Component | CPU | Memory | Disk |
|-----------|-----|--------|------|
| Per API Service | 0.5 core | 512MB | 100MB |
| MongoDB | 2 cores | 2GB | 10GB |
| MinIO | 1 core | 1GB | 100GB+ |

---

## Key System Constraints

1. **Single Database Instance**: No sharding/replication yet
2. **No API Gateway**: Services exposed directly to clients
3. **No Rate Limiting**: Services vulnerable to abuse
4. **No Caching Layer**: Every request hits the database
5. **Synchronous Only**: No async event streaming
6. **No Service Mesh**: No traffic management or observability

---

## Summary

The ES Backend system is designed with **modularity, scalability, and clear separation of concerns** as core principles. The microservices architecture allows independent development and deployment of features while maintaining loose coupling through REST APIs. The system is production-ready for small to medium scale, with a clear path to horizontal scaling and enhanced resilience through future architectural enhancements.
