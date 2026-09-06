# Sequence Diagrams - ES Backend

This document contains detailed sequence diagrams for major workflows in the system, using Mermaid syntax.

---

## 1. User Creation Flow

### Diagram
```mermaid
sequenceDiagram
    participant Client as Client App
    participant UPS as UserProfileService<br/>(Port 7000)
    participant MDB as MongoDBService<br/>(Port 14000)
    participant BS as BlobStorageService<br/>(Port 15000)

    Client->>UPS: POST /CreateNewUser<br/>(multipart: pic, banner, metadata)
    activate UPS

    UPS->>BS: POST /UserProfilePic/StoreImage<br/>(multipart: pic)
    activate BS
    BS->>BS: Save to MinIO<br/>bucket: user-profile-pic
    BS-->>UPS: Return pic_url
    deactivate BS

    UPS->>BS: POST /UserProfileBanner/StoreImage<br/>(multipart: banner)
    activate BS
    BS->>BS: Save to MinIO<br/>bucket: user-profile-banner
    BS-->>UPS: Return banner_url
    deactivate BS

    UPS->>MDB: POST /UserProfile/CreateNewUser<br/>(metadata + pic_url + banner_url)
    activate MDB
    MDB->>MDB: Generate UUID as USER_ID
    MDB->>MDB: Insert into USER_PROFILE collection
    MDB->>MDB: Create CREATED_AT timestamp
    MDB-->>UPS: Return created user object
    deactivate MDB

    UPS-->>Client: 201 Created<br/>(user_id, profile_pic, banner, etc.)
    deactivate UPS
```

### Theoretical Aspects

**Transaction Model**: Eventual Consistency
- Image uploads are atomic (succeed or fail completely)
- Database insert is atomic
- No cross-service transactions
- If DB write fails after image upload, images are orphaned (cleanup needed)

**Error Scenarios**:
1. **Image upload fails** → Return error immediately, no database changes
2. **Database write fails** → Images uploaded but user not created (inconsistency)
3. **Network timeout** → Client retries, duplicate detection needed

**Optimization Opportunities**:
- Parallel image uploads (not currently done)
- Caching user profile in Redis
- Batch user creation for bulk imports

---

## 2. Event Creation Flow

### Diagram
```mermaid
sequenceDiagram
    participant Client as Client App
    participant ES as EventsService<br/>(Port 13000)
    participant MDB as MongoDBService<br/>(Port 14000)
    participant BS as BlobStorageService<br/>(Port 15000)

    Client->>ES: POST /CreateNewEvent<br/>(multipart: banner, metadata)
    activate ES

    ES->>BS: POST /Event/StoreImage<br/>(multipart: banner)
    activate BS
    BS->>BS: Save to MinIO<br/>bucket: event-banner
    BS-->>ES: Return banner_url
    deactivate BS

    ES->>MDB: POST /Events/CreateNewEvent<br/>(metadata + banner_url)
    activate MDB
    MDB->>MDB: Generate UUID as EVENT_ID
    MDB->>MDB: Validate event data
    MDB->>MDB: Insert into EVENTS collection
    MDB->>MDB: Create CREATED_AT timestamp
    MDB-->>ES: Return created event object
    deactivate MDB

    ES-->>Client: 201 Created<br/>(event_id, banner_url, event_info)
    deactivate ES

    Note over Client,BS: Event now visible in system<br/>Other users can view/register
```

### Theoretical Aspects

**Resource Creation Pattern**:
- External resources (images) created first
- Database record created after external dependencies succeed
- Rollback not implemented (orphaned resources possible)

**Idempotency**:
- No idempotency key in current implementation
- Retried requests create duplicate events
- **Recommendation**: Add request ID tracking

**Event Lifecycle**:
```
CREATED → REGISTRATION_OPEN → REGISTRATION_CLOSED → IN_PROGRESS → COMPLETED
  ↑                                                                    ↓
  └────────────────────────────────────────────────────────────────┘
                         (Cancelled/Archived)
```

**Concurrent Registrations**:
- No reservation system
- Race conditions possible (participant limit)
- Needs optimistic locking or reservation queue

---

## 3. Team Creation Flow

### Diagram
```mermaid
sequenceDiagram
    participant Client as Client App
    participant TS as TeamsService<br/>(Port 17000)
    participant MDB as MongoDBService<br/>(Port 14000)
    participant BS as BlobStorageService<br/>(Port 15000)

    Client->>TS: POST /CreateNewTeam<br/>(multipart: logo, metadata)
    activate TS

    TS->>BS: POST /Team/StoreImage<br/>(multipart: logo)
    activate BS
    BS->>BS: Save to MinIO<br/>bucket: team-logo
    BS-->>TS: Return logo_url
    deactivate BS

    TS->>MDB: POST /Teams/CreateNewTeam<br/>(metadata + logo_url)
    activate MDB
    MDB->>MDB: Generate UUID as TEAM_ID
    MDB->>MDB: Insert into TEAMS collection
    MDB->>MDB: Initialize PARTICIPANTS array<br/>(creator as ADMIN)
    MDB->>MDB: Initialize MILESTONES array (empty)
    MDB->>MDB: Initialize EVENTS_ENROLLED array (empty)
    MDB-->>TS: Return created team object
    deactivate MDB

    TS-->>Client: 201 Created<br/>(team_id, logo_url, team_info)
    deactivate TS

    Note over MDB: Team Structure<br/>├─ TEAM_ID<br/>├─ PARTICIPANTS: [creator]<br/>├─ MILESTONES: []<br/>└─ EVENTS_ENROLLED: []
```

### Theoretical Aspects

**Roster Management**:
- Participants stored as objects with USER_ID + ACCESS_LEVEL
- Access levels: ADMIN (full control), MEMBER (read-only)
- No automatic role assignment based on creation

**Scalability Concerns**:
```
Team Size         Concerns
─────────────────────────────────
<50 players       None
50-500 players    Array grows large in document
500-5000 players  Performance degradation
5000+ players     Must use separate collection
```

**Recommendation**: For large teams, use separate TEAM_ROSTERS collection:
```
TEAMS → { TEAM_ID, NAME, LOGO, ... }
TEAM_ROSTERS → { ROSTER_ID, TEAM_ID, PARTICIPANTS: [...] }
```

---

## 4. Mentor Profile Creation Flow

### Diagram
```mermaid
sequenceDiagram
    participant Client as Client App
    participant MPS as MentorProfileService<br/>(Port 10000)
    participant MDB as MongoDBService<br/>(Port 14000)
    participant BS as BlobStorageService<br/>(Port 15000)

    Client->>MPS: POST /CreateNewMentor<br/>(multipart: pic, banner, metadata)
    activate MPS

    par Parallel Image Uploads
        MPS->>BS: POST /MentorProfilePic/StoreImage
        activate BS
        BS->>BS: Save to bucket: mentor-profile-pic
        BS-->>MPS: Return pic_url
        deactivate BS
    and
        MPS->>BS: POST /MentorProfileBanner/StoreImage
        activate BS
        BS->>BS: Save to bucket: mentor-profile-banner
        BS-->>MPS: Return banner_url
        deactivate BS
    end

    MPS->>MDB: POST /MentorProfile/CreateNewMentor<br/>(metadata + urls)
    activate MDB
    MDB->>MDB: Generate UUID as MENTOR_ID
    MDB->>MDB: Insert into MENTOR_PROFILE collection
    MDB->>MDB: Initialize RATING = 0
    MDB->>MDB: Initialize SESSIONS_COMPLETED = 0
    MDB->>MDB: Initialize SUCCESS_RATE = 100%
    MDB->>MDB: Set VERIFIED = false
    MDB-->>MPS: Return created mentor object
    deactivate MDB

    MPS-->>Client: 201 Created<br/>(mentor_id, mentor_info)
    deactivate MPS

    Note over Client,MDB: Mentor awaiting verification<br/>from admin before active
```

### Theoretical Aspects

**Verification Workflow**:
```
Mentor Signup → Admin Review → VERIFIED = true → Can Accept Sessions
                    ↓
                 Rejected → VERIFIED = false → Cannot Accept Sessions
```

**Rating System**:
```
Session Completed
    ↓
Student Rate Mentor (1-5 stars)
    ↓
new_rating = (old_rating × sessions_completed + new_rating) / (sessions_completed + 1)
    ↓
success_rate = (successful_sessions / total_sessions) × 100%
```

**Session Booking Flow** (Not yet implemented):
```
Student Request → Mentor Confirm → Session Scheduled → Session Completed → Rating
```

---

## 5. Event Registration Flow

### Diagram
```mermaid
sequenceDiagram
    participant Client as Client App
    participant ES as EventsService<br/>(Port 13000)
    participant MDB as MongoDBService<br/>(Port 14000)

    Client->>ES: POST /Events/Register<br/>(EVENT_ID, USER_ID/TEAM_ID)
    activate ES

    ES->>MDB: GET /Events/GetEventInfo<br/>(EVENT_ID)
    activate MDB
    MDB-->>ES: Return event object<br/>(status, deadline, participants)
    deactivate MDB

    ES->>ES: Validate Registration<br/>├─ Check deadline<br/>├─ Check participant limit<br/>├─ Check eligibility<br/>└─ Check duplicate

    alt Validation Fails
        ES-->>Client: 400 Bad Request<br/>(reason)
    else Validation Passes
        ES->>MDB: PUT /Events/Update<br/>(add participant)
        activate MDB
        MDB->>MDB: Update PARTICIPANTS array<br/>MDB->>MDB: Update participant count
        MDB-->>ES: Success
        deactivate MDB

        ES-->>Client: 200 OK<br/>(registration confirmed)
    end

    deactivate ES

    Note over MDB: Event State Updated<br/>├─ Participant added<br/>├─ Count incremented<br/>└─ Timestamp recorded
```

### Theoretical Aspects

**Race Condition Scenario**:
```
Scenario: Participant Limit = 100, Current = 99

Request A                        Request B
   │                               │
   ├─ Check count < 100 ✓         │
   │                               ├─ Check count < 100 ✓
   │                               │
   ├─ Insert participant           │
   │ (Count = 100) ✓               │
   │                               ├─ Insert participant
   │                               │ (Count = 101) ✗
   │                               │
   └─ Return 200                   └─ Return 500
                                      (Oversubscribed)
```

**Solution Options**:
1. **Optimistic Locking**: Version check before update
2. **Pessimistic Locking**: Lock document during update (slower)
3. **Reservation Queue**: Queue participants, assign slots sequentially
4. **Atomic Counter**: Use MongoDB atomic operations

**Recommendation**: MongoDB atomic update:
```javascript
db.EVENTS.findOneAndUpdate(
  { EVENT_ID: id, participants_count: { $lt: max } },
  { $inc: { participants_count: 1 }, $push: { participants: user_id } }
)
```

---

## 6. Team Enrollment in Event

### Diagram
```mermaid
sequenceDiagram
    participant Client as Client App
    participant TS as TeamsService<br/>(Port 17000)
    participant ES as EventsService<br/>(Port 13000)
    participant MDB as MongoDBService<br/>(Port 14000)

    Client->>TS: POST /Teams/EnrollInEvent<br/>(TEAM_ID, EVENT_ID)
    activate TS

    TS->>ES: GET /Events/GetEventInfo<br/>(EVENT_ID)
    activate ES
    ES->>MDB: GET /Events/GetEventInfo
    activate MDB
    MDB-->>ES: Return event
    deactivate MDB
    ES-->>TS: Return event details
    deactivate ES

    TS->>TS: Validate Enrollment<br/>├─ Check team size<br/>├─ Check event format<br/>├─ Check eligibility<br/>└─ Check game match

    alt Validation Fails
        TS-->>Client: 400 Bad Request
    else Validation Passes
        TS->>MDB: PUT /Teams/Update<br/>(add to EVENTS_ENROLLED)
        activate MDB
        MDB-->>TS: Success
        deactivate MDB

        TS->>MDB: PUT /Events/Update<br/>(add team to participants)
        activate MDB
        MDB-->>TS: Success
        deactivate MDB

        TS-->>Client: 200 OK<br/>(enrollment confirmed)
    end

    deactivate TS

    Note over MDB: Bi-directional Update<br/>├─ Team knows about event<br/>└─ Event knows about team
```

### Theoretical Aspects

**Data Consistency Issue**:
- Updates to both TEAMS and EVENTS collections
- No transaction spanning both
- If TEAMS update succeeds but EVENTS fails: inconsistency

**Better Approach** (Recommended):
```
Single source of truth: EVENTS collection stores all team registrations
Teams collection references events: { EVENTS_ENROLLED: [event_id1, event_id2] }
Queries fetch from single collection for consistency
```

---

## 7. Milestone Creation and Tracking

### Diagram
```mermaid
sequenceDiagram
    participant Client as Client App
    participant TS as TeamsService<br/>(Port 17000)
    participant MDB as MongoDBService<br/>(Port 14000)

    Client->>TS: POST /Milestone/CreateNewMilestone<br/>(TEAM_ID, milestone_data)
    activate TS

    TS->>MDB: POST /Milestone/CreateNewMilestone
    activate MDB
    MDB->>MDB: Generate UUID as MILESTONE_ID
    MDB->>MDB: Insert into MILESTONES collection
    MDB->>MDB: Initialize STATUS = "Not Started"
    MDB->>MDB: Set DEADLINE
    MDB-->>TS: Return milestone_id
    deactivate MDB

    TS->>MDB: PUT /Teams/Update<br/>(add milestone_id to MILESTONES array)
    activate MDB
    MDB-->>TS: Success
    deactivate MDB

    TS-->>Client: 201 Created<br/>(milestone_id, details)
    deactivate TS

    Note over MDB: Milestone Lifecycle<br/>Not Started → In Progress → Completed
```

### Milestone Update Flow
```mermaid
sequenceDiagram
    participant Client as Client App
    participant MDB as MongoDBService<br/>(Port 14000)

    Client->>MDB: PUT /Milestone/UpdateMilestoneStatus<br/>(MILESTONE_ID, STATUS)
    activate MDB

    alt Valid Status Transition
        MDB->>MDB: Update STATUS in MILESTONES
        MDB->>MDB: Update UPDATED_AT timestamp
        MDB-->>Client: 200 OK
    else Invalid Transition
        MDB-->>Client: 400 Bad Request<br/>(invalid status)
    end

    deactivate MDB
```

### Theoretical Aspects

**Milestone State Machine**:
```
Not Started
    ↓
    ├─→ In Progress
    │        ↓
    │        └─→ Completed ✓
    │
    └─→ Cancelled (Alternative end state)
```

**Deadline Management**:
```
Milestone Created with DEADLINE
    ↓
System Check (Daily/Hourly Job)
    ├─ If DEADLINE passed and STATUS ≠ "Completed"
    │  └─ Mark as "Overdue" or auto-fail
    │
    └─ Send notification to team
```

**Recommendation**: Add DEADLINE_EXCEEDED status or automatic job to mark overdue milestones.

---

## 8. Cross-Service Communication Pattern

### Diagram
```mermaid
sequenceDiagram
    participant Client as Client App
    participant Service1 as Service A<br/>Orchestrator
    participant Service2 as Service B<br/>Operator
    participant Service3 as Service C<br/>Operator

    Client->>Service1: POST /ComplexOperation
    activate Service1

    Service1->>Service2: POST /Operation1
    activate Service2
    Service2->>Service2: Process
    Service2-->>Service1: Response A
    deactivate Service2

    Service1->>Service3: POST /Operation2<br/>(depends on Response A)
    activate Service3
    Service3->>Service3: Process
    Service3-->>Service1: Response B
    deactivate Service3

    Service1->>Service2: PUT /Operation1Update<br/>(confirm with Response B)
    activate Service2
    Service2-->>Service1: Confirmation
    deactivate Service2

    Service1-->>Client: Final Response<br/>(all operations complete)
    deactivate Service1

    Note over Service1: Orchestration Pattern<br/>Sequential Dependency
```

### Theoretical Aspects

**Orchestration vs Choreography**:

**Current (Orchestration)**:
```
Client → ServiceA → ServiceB ─┐
                              └─→ ServiceC
ServiceA controls flow
```

**Alternative (Choreography)**:
```
ServiceA emits Event
    ↓
ServiceB listens → emits Event
    ↓
ServiceC listens → completes
```

**Trade-offs**:
| Aspect | Orchestration | Choreography |
|--------|---------------|--------------|
| Coupling | Higher | Lower |
| Complexity | Centralized | Distributed |
| Debuggability | Easier | Harder |
| Scalability | Limited | Better |

---

## 9. System-Wide Data Flow

### Diagram
```mermaid
graph TD
    A[Client] -->|POST /CreateUser| B[UserProfileService]
    A -->|POST /CreateEvent| C[EventsService]
    A -->|POST /CreateTeam| D[TeamsService]
    A -->|POST /CreateMentor| E[MentorProfileService]

    B -->|Upload Images| F[BlobStorageService]
    C -->|Upload Images| F
    D -->|Upload Images| F
    E -->|Upload Images| F

    F -->|Store in Buckets| G[MinIO S3]

    B -->|CRUD Operations| H[MongoDBService]
    C -->|CRUD Operations| H
    D -->|CRUD Operations| H
    E -->|CRUD Operations| H

    H -->|Read/Write| I[MongoDB Database]

    style A fill:#e1f5ff
    style B fill:#fff3e0
    style C fill:#fff3e0
    style D fill:#fff3e0
    style E fill:#fff3e0
    style F fill:#f3e5f5
    style H fill:#f3e5f5
    style G fill:#e8f5e9
    style I fill:#e8f5e9
```

---

## 10. Error Handling and Retry Flow

### Diagram
```mermaid
sequenceDiagram
    participant Client as Client
    participant Service as Service
    participant DB as Database

    Client->>Service: Request
    activate Service

    Service->>DB: Operation
    activate DB

    alt Success
        DB-->>Service: Data
        deactivate DB
        Service-->>Client: 200 OK
    else Connection Error
        DB--x Service: Timeout
        deactivate DB

        Service->>Service: Wait 100ms (exponential backoff)
        Service->>DB: Retry #1
        activate DB

        alt Success
            DB-->>Service: Data
            deactivate DB
            Service-->>Client: 200 OK
        else Still Failing
            DB--x Service: Timeout
            deactivate DB
            Service-->>Client: 503 Service Unavailable
        end
    else Database Error
        DB-->>Service: 500 Error
        deactivate DB
        Service-->>Client: 500 Internal Server Error
    else Validation Error
        Service->>Service: Validate Input
        Service-->>Client: 422 Unprocessable Entity
    end

    deactivate Service
```

---

## 11. Theoretical: Distributed Transaction Pattern

### The Problem
```
Transaction: User Registration + Team Creation + Event Enrollment

Step 1: Create User in MongoDB ✓
Step 2: Create Team in MongoDB ✓
Step 3: Enroll Team in Event in MongoDB... ✗ (Database down!)

Result: Inconsistent state (user + team exist, but not enrolled in event)
```

### Two-Phase Commit (Not Implemented)

```mermaid
sequenceDiagram
    participant Coordinator
    participant Service1
    participant Service2
    participant Service3

    Coordinator->>Service1: Prepare (Create User)
    Coordinator->>Service2: Prepare (Create Team)
    Coordinator->>Service3: Prepare (Enroll Event)

    Service1-->>Coordinator: Ready/Not Ready
    Service2-->>Coordinator: Ready/Not Ready
    Service3-->>Coordinator: Ready/Not Ready

    alt All Ready
        Coordinator->>Service1: Commit
        Coordinator->>Service2: Commit
        Coordinator->>Service3: Commit
        Service1-->>Coordinator: Committed
        Service2-->>Coordinator: Committed
        Service3-->>Coordinator: Committed
    else Any Failed
        Coordinator->>Service1: Rollback
        Coordinator->>Service2: Rollback
        Coordinator->>Service3: Rollback
    end
```

**Current System**: Does NOT use two-phase commit (eventual consistency instead)

---

## Summary

These sequence diagrams illustrate:
- **Current Implementation**: Synchronous REST-based orchestration
- **Theoretical Concerns**: Race conditions, eventual consistency, transaction boundaries
- **Future Improvements**: Async messaging, two-phase commit, saga pattern
- **Scalability**: Bottlenecks at database and orchestration service

All diagrams follow the microservices communication pattern: Client → Orchestrator → Infrastructure Services → Data Stores.
