# Detailed Architecture - ES Backend

## Architecture Overview Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                      CLIENT LAYER                               │
│         (Web App, Mobile App, Desktop App, API Clients)         │
└──────────────────────────────────────┬──────────────────────────┘
                                       │
                    ┌──────────────────┴──────────────────┐
                    │         EDGE SERVICES              │
                    │  (Future API Gateway, Auth)        │
                    └──────────────────┬──────────────────┘
                                       │
                 ┌─────────────────────┼─────────────────────┐
                 │                     │                     │
        ┌────────▼─────────┐  ┌────────▼────────┐  ┌────────▼────────┐
        │  USER DOMAIN     │  │  ENTITY DOMAIN  │  │ COMMUNITY DOMAIN│
        │                  │  │                 │  │                 │
        │ • UserProfile    │  │ • Events        │  │ • Chat          │
        │ • Auth           │  │ • Teams         │  │ • Feed          │
        │ • Mentor         │  │ • Milestones    │  │ • News          │
        │ • Dashboard      │  │ • League        │  │ • Dashboard     │
        └────────┬─────────┘  └────────┬────────┘  └────────┬────────┘
                 │                     │                     │
                 └─────────────────────┼─────────────────────┘
                                       │
                ┌──────────────────────▼──────────────────────┐
                │     INFRASTRUCTURE SERVICES LAYER           │
                │  (Persistence, Storage, Messaging)          │
                │                                              │
                │  ┌─────────────────────────────────────┐   │
                │  │   MongoDB Service (Port 14000)      │   │
                │  │  • CRUD operations                  │   │
                │  │  • Data validation                  │   │
                │  │  • Schema management                │   │
                │  └─────────────────────────────────────┘   │
                │                                              │
                │  ┌─────────────────────────────────────┐   │
                │  │ Blob Storage Service (Port 15000)  │   │
                │  │  • File uploads                     │   │
                │  │  • Image management                 │   │
                │  │  • URL generation                   │   │
                │  └─────────────────────────────────────┘   │
                │                                              │
                │  ┌─────────────────────────────────────┐   │
                │  │  Message Queue (Available, unused)  │   │
                │  │  • RabbitMQ integration             │   │
                │  │  • Async event handling (future)    │   │
                │  └─────────────────────────────────────┘   │
                └──────────────────────┬──────────────────────┘
                                       │
                ┌──────────────────────▼──────────────────────┐
                │        DATA & STORAGE LAYER                 │
                │                                              │
                │  ┌──────────────────┐  ┌────────────────┐  │
                │  │   MongoDB        │  │  MinIO (S3)    │  │
                │  │   Database       │  │  Object Store  │  │
                │  └──────────────────┘  └────────────────┘  │
                └──────────────────────────────────────────────┘
```

---

## Component Architecture

### Layer 1: Service Foundation (ServiceTemplates/)

#### HTTPServer Class
```python
class HTTPServer:
    ├── FastAPI Application
    ├── CORS Configuration
    ├── Route Registration
    ├── Error Handling
    └── Uvicorn Server Management
```

**Responsibilities**:
- Wrap FastAPI application initialization
- Enable CORS for cross-origin requests
- Start/stop Uvicorn server
- Route all HTTP requests

**Usage Pattern**:
```python
# In each service main file
server = HTTPServer("service-name")
server.register_routes(router)
server.start()
```

#### MessageQueue Class
```python
class MessageQueue:
    ├── RabbitMQ Connection
    ├── Queue Management
    ├── Publisher Methods
    ├── Consumer Methods
    └── Callback Handler Mapping
```

**Responsibilities**:
- Manage RabbitMQ connections
- Publish messages to queues
- Consume messages with async callbacks
- Handle message acknowledgment

**Current Status**: Implemented but listener disabled in all services

---

### Layer 2: Infrastructure Services

#### MongoDBService (Port 14000)
```
                    MongoDBService
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
    ┌───▼────┐        ┌───▼────┐       ┌───▼────┐
    │ Events │        │ Profiles│       │ Teams  │
    │ CRUD   │        │ CRUD    │       │ CRUD   │
    └───┬────┘        └───┬────┘       └───┬────┘
        │                 │                 │
        └─────────────────┼─────────────────┘
                          │
                  ┌───────▼────────┐
                  │  MongoDB Conn  │
                  └────────────────┘
```

**Endpoints**:
- Events: Create, Read, Update, Delete, List
- User Profiles: Create, Read, Update, List
- Mentor Profiles: Create, Read, Update, List
- Teams: Create, Read, Update, Delete, List
- Milestones: CRUD operations
- Chat: Messages and conversation management

**Key Implementation Details**:
- UUID-based resource IDs
- Timestamp tracking (CREATED_AT, DEADLINE)
- Nested documents (SOCIAL_LINKS, GAME_RELATED_INFO)
- Array support (GAMES_PLAYED, HISTORY, PARTICIPANTS)

#### BlobStorageService (Port 15000)
```
            BlobStorageService
                     │
        ┌────────────┼────────────┐
        │            │            │
    ┌───▼────┐   ┌───▼────┐   ┌──▼────┐
    │ Upload │   │Retrieve│   │Delete │
    │Handler │   │Handler │   │Handler│
    └───┬────┘   └───┬────┘   └──┬────┘
        │            │            │
        └────────────┼────────────┘
                     │
            ┌────────▼────────┐
            │  MinIO/S3 Conn  │
            │  (boto3 client) │
            └─────────────────┘
```

**Bucket Organization**:
- `user-profile-pic` - User profile pictures
- `user-profile-banner` - User profile banners
- `mentor-profile-pic` - Mentor profile pictures
- `mentor-profile-banner` - Mentor profile banners
- `team-logo` - Team logos
- `event-banner` - Event banners
- `chat-files` - Chat attachments (future)

**Key Implementation Details**:
- Multipart file upload handling
- URL construction for retrieval
- File naming: `{resource_id}.{extension}`
- MIME type validation

---

### Layer 3: Domain Services

#### UserProfileService (Port 7000)
```
         Client Request
              │
    ┌─────────▼──────────┐
    │ UserProfileService │
    │                    │
    │ POST /CreateUser   │
    │ GET /GetProfile    │
    │ GET /ListProfiles  │
    └─────────┬──────────┘
              │
        ┌─────┴──────────────────┐
        │                        │
    ┌───▼───────────┐    ┌───────▼──────┐
    │MongoDBService │    │BlobStorage   │
    │ • Create      │    │ • Upload Pic │
    │ • Update      │    │ • Upload    │
    │ • URLs        │    │   Banner    │
    └───┬───────────┘    └───────┬──────┘
        │                        │
        └────────────┬───────────┘
                     │
              Response to Client
```

**Responsibilities**:
- User CRUD operations
- Profile picture/banner upload orchestration
- Social links management
- Games played tracking

**Communication Pattern**:
1. Receive multipart request (file + metadata)
2. Upload files to BlobStorageService → get URLs
3. Create user in MongoDBService with URLs
4. Return created user with all data

#### EventsService (Port 13000)
```
         Client Request
              │
    ┌─────────▼──────────┐
    │  EventsService     │
    │                    │
    │ POST /CreateEvent  │
    │ GET /GetEvent      │
    │ GET /ListEvents    │
    │ DELETE /Event      │
    └─────────┬──────────┘
              │
        ┌─────┴──────────────────┐
        │                        │
    ┌───▼───────────┐    ┌───────▼──────┐
    │MongoDBService │    │BlobStorage   │
    │ • Create      │    │ • Upload     │
    │ • Update      │    │   Banner     │
    │ • Delete      │    └──────────────┘
    └───────────────┘
```

**Responsibilities**:
- Event CRUD operations
- Event banner upload
- Event registration management
- Milestone tracking
- Discussion/Q&A management

#### TeamsService (Port 17000)
```
         Client Request
              │
    ┌─────────▼──────────┐
    │   TeamsService     │
    │                    │
    │ POST /CreateTeam   │
    │ GET /GetTeam       │
    │ GET /UserTeams     │
    │ DELETE /Team       │
    └─────────┬──────────┘
              │
        ┌─────┴──────────────────┐
        │                        │
    ┌───▼───────────┐    ┌───────▼──────┐
    │MongoDBService │    │BlobStorage   │
    │ • Create      │    │ • Upload     │
    │ • Update      │    │   Logo       │
    │ • Delete      │    └──────────────┘
    └───────────────┘
```

**Responsibilities**:
- Team CRUD operations
- Team logo management
- Participant management (roster)
- Access level control (ADMIN/MEMBER)
- Milestone association

#### MentorProfileService (Port 10000)
```
         Client Request
              │
    ┌──────────▼──────────┐
    │MentorProfileService │
    │                     │
    │ POST /CreateMentor  │
    │ GET /GetMentor      │
    │ GET /ListMentors    │
    └──────────┬──────────┘
               │
        ┌──────┴──────────────────┐
        │                         │
    ┌───▼────────────┐    ┌───────▼──────┐
    │MongoDBService  │    │BlobStorage   │
    │ • Create       │    │ • Upload Pic │
    │ • Update       │    │ • Upload     │
    │ • Rating       │    │   Banner     │
    └────────────────┘    └──────────────┘
```

**Responsibilities**:
- Mentor profile CRUD
- Profile image management
- Rating and session tracking
- Speciality and language management
- Verification status

---

## Communication Patterns

### Pattern 1: Synchronous REST (Primary)
```
Service A                Service B
   │                       │
   │  HTTP POST/GET        │
   ├──────────────────────>│
   │                       │ Process
   │  JSON Response        │
   │<──────────────────────┤
   │                       │
```

**Implementation**:
```python
import httpx

async def call_other_service(endpoint: str, data: dict):
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{SERVICE_URL}/{endpoint}",
            json=data
        )
        return response.json()
```

**Advantages**:
- Simple, request-response semantics
- Easy to debug
- Built-in error handling via HTTP status

**Disadvantages**:
- Tight coupling
- Cascading failures
- Synchronous blocking

### Pattern 2: Service Orchestration
```
Client
   │
   └──> UserProfileService (Orchestrator)
         ├──────────────────> MongoDBService
         │                    (Create User)
         │
         ├──────────────────> BlobStorageService
         │                    (Upload Images)
         │
         └──────────────────> MongoDBService
                              (Update URLs)
```

**Example Flow**:
```python
# In UserProfileService
@app.post("/CreateNewUser")
async def create_user(request: UserCreationRequest):
    # Step 1: Upload images
    pic_url = await upload_to_blob_storage(request.pic)
    banner_url = await upload_to_blob_storage(request.banner)

    # Step 2: Create user in DB
    user_data = {
        **request.dict(),
        "PROFILE_PIC": pic_url,
        "PROFILE_BANNER": banner_url
    }
    user = await call_mongodb_service("CreateNewUser", user_data)

    return user
```

### Pattern 3: Service Discovery
```json
// ServiceURLMapping.json
{
  "MongoDBService": "http://localhost:14000",
  "BlobStorageService": "http://localhost:15000",
  "UserProfileService": "http://localhost:7000",
  ...
}
```

**Advantages**:
- Centralized service registry
- Easy to update endpoints
- Environment-independent

---

## Error Handling Strategy

### Level 1: Input Validation
```python
from pydantic import BaseModel

class EventCreationRequest(BaseModel):
    EVENT_NAME: str  # Required
    EVENT_DATE: datetime  # Type validation
    PRIZE_POOL: int >= 0  # Range validation

@app.post("/CreateEvent")
async def create_event(request: EventCreationRequest):
    # Automatic validation, returns 422 if invalid
    ...
```

### Level 2: Service-Level Errors
```python
try:
    response = await mongodb_service.create_event(data)
except httpx.HTTPError as e:
    raise HTTPException(
        status_code=500,
        detail=f"Database service error: {str(e)}"
    )
```

### Level 3: Database Errors
```python
from pymongo.errors import DuplicateKeyError

try:
    result = await users_collection.insert_one(doc)
except DuplicateKeyError:
    raise HTTPException(
        status_code=409,
        detail="User already exists"
    )
```

---

## Data Consistency Model

### Eventual Consistency (Current)
```
State A                    State B (After Operation)
  │                              │
  ├─ User created in DB ────────>├─ User record exists
  │                              │
  ├─ Images uploaded ──────────> ├─ Images in storage
  │                              │
  ├─ URLs updated in DB ───────> ├─ All data consistent
  │                              │
  └─ Response to client ────────>└─ Operation complete
```

**Characteristics**:
- No transactions across services
- Each service commits atomically
- Temporary inconsistency possible if failures occur

### Atomicity Within Service
```python
# MongoDB Service - Single operation atomic
session = await db_client.start_session()
async with session.start_transaction():
    await users_collection.insert_one(user_doc)
    # Either fully succeeds or fully fails
```

### Failure Scenarios
1. **Upload succeeds, DB write fails**: Images orphaned in storage
2. **DB write succeeds, response fails**: Client doesn't know, can retry
3. **Service timeout**: Automatic 500ms retry with backoff

---

## Scalability Architecture

### Horizontal Scaling Strategy

#### API Services (Stateless)
```
                    Load Balancer
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
   ┌────▼──┐         ┌────▼──┐        ┌────▼──┐
   │ User  │         │ User  │        │ User  │
   │ Svc-1 │         │ Svc-2 │        │ Svc-3 │
   └────┬──┘         └────┬──┘        └────┬──┘
        │                 │                 │
        └─────────────────┼─────────────────┘
                          │
                  ┌───────▼────────┐
                  │  MongoDB       │
                  │  (Replica Set) │
                  └────────────────┘
```

**Process**:
1. Add more instances of stateless services
2. Place behind load balancer
3. Share single MongoDB database

#### Database Replication
```
Primary MongoDB                Secondary MongoDB
    │                              │
    ├─ Writes ────────────────────>│
    ├─ Replication ───────────────>│
    │                              │
    ├─ Reads <──────────────────────┤
```

#### Cache Layer (Future)
```
                    Client
                      │
                ┌─────▼─────┐
                │  Redis    │
                │  Cache    │
                └─────┬─────┘
                      │
              ┌───────┴───────┐
              │               │
         ┌────▼──┐        ┌───▼────┐
         │  HIT  │        │ MISS   │
         └───────┘        └───┬────┘
                              │
                         ┌────▼─────┐
                         │ MongoDB  │
                         └──────────┘
```

---

## Summary

The ES Backend architecture emphasizes:
- **Modularity**: Independent services with clear responsibilities
- **Scalability**: Stateless API services + data persistence layer
- **Maintainability**: Separation of concerns, clear communication patterns
- **Extensibility**: Template-based service creation, message queue infrastructure

Each layer has specific responsibilities and can be evolved independently while maintaining backward compatibility at API boundaries.
