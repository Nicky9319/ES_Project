# API Documentation - ES Backend

## API Overview

The ES Backend uses **REST APIs** for service-to-service and client-to-service communication. All APIs are built on **FastAPI** and follow standard HTTP conventions.

---

## General API Conventions

### Base URL
```
http://localhost:{SERVICE_PORT}
```

### Content Types
- Request: `application/json` or `multipart/form-data`
- Response: `application/json`

### Authentication
**Current**: None (OAuth 2.0 in development)
**Future**: JWT Bearer tokens required

### Error Responses

All errors follow this format:
```json
{
  "detail": "Error message describing the issue"
}
```

| Status Code | Meaning |
|------------|---------|
| 200 | Success |
| 201 | Created |
| 204 | No Content |
| 400 | Bad Request (validation error) |
| 404 | Not Found |
| 409 | Conflict (duplicate, capacity full) |
| 422 | Unprocessable Entity (invalid schema) |
| 500 | Internal Server Error |
| 502 | Bad Gateway (upstream service error) |
| 503 | Service Unavailable |

---

## User Profile Service (Port 7000)

### Create New User

```
POST /UserProfile/CreateNewUser

Content-Type: multipart/form-data

Form Data:
├─ pic: File (image/jpeg, image/png)
├─ banner: File (image/jpeg, image/png)
└─ metadata: JSON
   {
     "USER_NAME": "string (required)",
     "BIO": "string (optional)",
     "TAGLINE": "string (optional)",
     "LOCATION": "string (optional)",
     "PLATFORM_STATUS": "ACTIVE",
     "TEAM_STATUS": "LOOKING_FOR_TEAM",
     "GAMES_PLAYED": [
       {
         "GAME_NAME": "Valorant",
         "HOURS_PLAYED": 500,
         "RANK": "Radiant",
         "SPECIALITY": "IGL"
       }
     ],
     "SOCIAL_LINKS": {
       "INSTAGRAM": "https://instagram.com/...",
       "DISCORD": "username#1234",
       ...
     }
   }

Response (201 Created):
{
  "USER_ID": "550e8400-e29b-41d4-a716-446655440000",
  "USER_NAME": "ProPlayer",
  "BIO": "Competitive esports player",
  "PROFILE_PIC": "http://localhost:15000/Image/RetrieveImage?bucket=user-profile-pic&key=550e8400-e29b-41d4-a716-446655440000.jpg",
  "PROFILE_BANNER": "http://localhost:15000/Image/RetrieveImage?bucket=user-profile-banner&key=550e8400-e29b-41d4-a716-446655440000.jpg",
  "CREATED_AT": "2024-01-27T10:30:00Z"
}

Possible Errors:
├─ 400: Image format not supported
├─ 413: File too large (exceeds 50MB)
├─ 409: Username already exists
└─ 502: BlobStorageService unavailable
```

### Get User Profile

```
GET /UserProfile/GetUserProfile?USER_ID={USER_ID}

Query Parameters:
└─ USER_ID: string (uuid)

Response (200 OK):
{
  "USER_ID": "550e8400-e29b-41d4-a716-446655440000",
  "USER_NAME": "ProPlayer",
  "BIO": "Competitive esports player",
  "PROFILE_PIC": "http://...",
  "GAMES_PLAYED": [...],
  "HISTORY": [
    {
      "TEAM_ID": "...",
      "TEAM_NAME": "Team Name",
      "JOINED_DATE": "2024-01-01T00:00:00Z",
      "ROLE": "IGL"
    }
  ],
  ...
}

Possible Errors:
└─ 404: User not found
```

### Get All User Profiles

```
GET /UserProfile/GetAllUserProfiles

Response (200 OK):
[
  { USER_ID, USER_NAME, BIO, ... },
  { USER_ID, USER_NAME, BIO, ... },
  ...
]

Performance Note: Returns all users (no pagination)
Recommendation: Add pagination for large datasets
```

---

## Events Service (Port 13000)

### Create New Event

```
POST /Events/CreateNewEvent

Content-Type: multipart/form-data

Form Data:
├─ banner: File (image/jpeg, image/png)
└─ metadata: JSON
   {
     "EVENT_NAME": "string (required)",
     "EVENT_DATE": "2024-02-15T18:00:00Z (ISO 8601)",
     "REGISTRATION_DEADLINE": "2024-02-10T23:59:59Z",
     "VENUE": "Online or location string",
     "GAME": "Valorant",
     "CONSOLE": "PC",
     "PRIZE_POOL": 10000,
     "FORMAT": "5v5",
     "ORGANIZER": "USER_ID",
     "CONTACT_INFO": {
       "EMAIL": "organizer@example.com",
       "MOBILE_NUMBER": "+1234567890"
     },
     "ELIGIBILITY": [
       { "TYPE": "RANK", "VALUE": "GOLD or above" },
       { "TYPE": "REGION", "VALUE": ["NA", "EU"] }
     ],
     "QUESTIONNAIRE": [
       {
         "QUESTION": "Previous tournament experience?",
         "ANSWER_TYPE": "TEXT",
         "REQUIRED": true
       }
     ],
     "FAQ": [
       {
         "QUESTION": "What's the entry fee?",
         "ANSWER": "Free"
       }
     ]
   }

Response (201 Created):
{
  "EVENT_ID": "660e8400-e29b-41d4-a716-446655440000",
  "EVENT_NAME": "Grand Tournament 2024",
  "EVENT_DATE": "2024-02-15T18:00:00Z",
  "IMAGE": "http://localhost:15000/Image/RetrieveImage?bucket=event-banner&key=660e8400-e29b-41d4-a716-446655440000.jpg",
  "STATUS": "DRAFT",
  "CREATED_AT": "2024-01-27T10:30:00Z"
}

Possible Errors:
├─ 400: Invalid event data
├─ 409: Event already exists (same name/date)
└─ 502: Storage service error
```

### Get Event Info

```
GET /Events/GetEventInfo?EVENT_ID={EVENT_ID}

Query Parameters:
└─ EVENT_ID: string (uuid)

Response (200 OK):
{
  "EVENT_ID": "660e8400-e29b-41d4-a716-446655440000",
  "EVENT_NAME": "Grand Tournament 2024",
  "EVENT_DATE": "2024-02-15T18:00:00Z",
  "PARTICIPANTS": [
    {
      "USER_ID": "550e8400-e29b-41d4-a716-446655440000",
      "NAME": "ProPlayer",
      "REGISTRATION_DATE": "2024-01-27T11:00:00Z"
    }
  ],
  "STATUS": "OPEN",
  ...
}

Possible Errors:
└─ 404: Event not found
```

### Get All Events

```
GET /Events/AllEvents

Response (200 OK):
[
  { EVENT_ID, EVENT_NAME, EVENT_DATE, STATUS, ... },
  { EVENT_ID, EVENT_NAME, EVENT_DATE, STATUS, ... },
  ...
]

Performance Note: Returns all events (unfiltered)
Recommendation: Add pagination and filters (status, game, date range)
```

### Get User's Organized Events

```
GET /Events/Organized/User/AllEvents?USER_ID={USER_ID}

Query Parameters:
└─ USER_ID: string (uuid)

Response (200 OK):
[
  { EVENT_ID, EVENT_NAME, STATUS, ... },
  ...
]

Filters events where ORGANIZER == USER_ID
```

### Delete Event

```
DELETE /Events/DeleteEvent?EVENT_ID={EVENT_ID}

Query Parameters:
└─ EVENT_ID: string (uuid)

Response (204 No Content)

Possible Errors:
├─ 404: Event not found
├─ 409: Event already in progress (cannot delete)
└─ 500: Database error
```

### Add Discussion Question

```
POST /Events/Discussion/AddNewQuestion

JSON Body:
{
  "EVENT_ID": "660e8400-e29b-41d4-a716-446655440000",
  "QUESTION": "What time should the event start?",
  "QUESTIONER": "550e8400-e29b-41d4-a716-446655440000"
}

Response (201 Created):
{
  "QUESTION_ID": "uuid",
  "EVENT_ID": "...",
  "QUESTION": "...",
  "CREATED_AT": "2024-01-27T10:30:00Z"
}
```

---

## Teams Service (Port 17000)

### Create New Team

```
POST /Teams/CreateNewTeam

Content-Type: multipart/form-data

Form Data:
├─ logo: File (image/jpeg, image/png)
└─ metadata: JSON
   {
     "NAME": "string (required)",
     "SHORT_NAME": "string (e.g., TGK)",
     "TAGLINE": "string (optional)",
     "GAME": "Valorant",
     "TEAM_DESCRIPTION": "string (optional)",
     "TEAM_SIZE": 5
   }

Response (201 Created):
{
  "TEAM_ID": "770e8400-e29b-41d4-a716-446655440000",
  "NAME": "Team Name",
  "TEAM_LOGO": "http://localhost:15000/Image/RetrieveImage?bucket=team-logo&key=770e8400-e29b-41d4-a716-446655440000.jpg",
  "PARTICIPANTS": [
    {
      "USER_ID": "550e8400-e29b-41d4-a716-446655440000",
      "ACCESS_LEVEL": "ADMIN",
      "JOINED_DATE": "2024-01-27T10:30:00Z"
    }
  ],
  "CREATED_AT": "2024-01-27T10:30:00Z"
}

Possible Errors:
├─ 400: Invalid team data
├─ 409: Team name already exists
└─ 502: Storage service error
```

### Get Team Info

```
GET /Teams/GetTeamInfo?TEAM_ID={TEAM_ID}

Query Parameters:
└─ TEAM_ID: string (uuid)

Response (200 OK):
{
  "TEAM_ID": "770e8400-e29b-41d4-a716-446655440000",
  "NAME": "Team Name",
  "PARTICIPANTS": [
    {
      "USER_ID": "...",
      "USER_NAME": "...",
      "ROLE": "IGL",
      "ACCESS_LEVEL": "ADMIN",
      "JOINED_DATE": "2024-01-27T10:30:00Z"
    }
  ],
  "MILESTONES": [...],
  "EVENTS_ENROLLED": [...]
}

Possible Errors:
└─ 404: Team not found
```

### Get User's Teams

```
GET /Teams/User/GetAllTeams?USER_ID={USER_ID}

Query Parameters:
└─ USER_ID: string (uuid)

Response (200 OK):
[
  { TEAM_ID, NAME, SHORT_NAME, ... },
  { TEAM_ID, NAME, SHORT_NAME, ... },
  ...
]

Returns teams where USER_ID is in PARTICIPANTS array
```

### Update Team Logo

```
PUT /Teams/Update/TeamLogo

Content-Type: multipart/form-data

Form Data:
├─ TEAM_ID: string (uuid)
└─ logo: File (image/jpeg, image/png)

Response (200 OK):
{
  "TEAM_ID": "...",
  "TEAM_LOGO": "http://..."
}
```

### Disband Team

```
DELETE /Teams/DisbandTeam?TEAM_ID={TEAM_ID}

Query Parameters:
└─ TEAM_ID: string (uuid)

Response (204 No Content)

Possible Errors:
├─ 404: Team not found
├─ 403: Unauthorized (not admin)
└─ 409: Team has active event enrollment
```

---

## Mentor Profile Service (Port 10000)

### Create New Mentor

```
POST /MentorProfile/CreateNewMentor

Content-Type: multipart/form-data

Form Data:
├─ pic: File (image/jpeg, image/png)
├─ banner: File (image/jpeg, image/png)
└─ metadata: JSON
   {
     "USER_NAME": "string (required)",
     "BIO": "string (optional)",
     "TAGLINE": "string (optional)",
     "LOCATION": "string (optional)",
     "EXPERIENCE_YEARS": 5,
     "PRICE_PER_SESSION": 50,
     "GAMES": ["Valorant", "CS:GO"],
     "SPECIALITIES": [
       {
         "GAME": "Valorant",
         "AREA": "Agent Selection",
         "PROFICIENCY": "EXPERT"
       }
     ],
     "LANGUAGES": ["English", "Hindi"],
     "SKILL_TAGS": ["coaching", "strategy"]
   }

Response (201 Created):
{
  "MENTOR_ID": "880e8400-e29b-41d4-a716-446655440000",
  "USER_NAME": "ProCoach",
  "RATING": 0,
  "VERIFIED": false,
  "SESSIONS_COMPLETED": 0,
  "PROFILE_PIC": "http://...",
  "CREATED_AT": "2024-01-27T10:30:00Z"
}
```

### Get Mentor Profile

```
GET /MentorProfile/GetMentorProfile?MENTOR_ID={MENTOR_ID}

Query Parameters:
└─ MENTOR_ID: string (uuid)

Response (200 OK):
{
  "MENTOR_ID": "880e8400-e29b-41d4-a716-446655440000",
  "USER_NAME": "ProCoach",
  "BIO": "Professional esports coach",
  "EXPERIENCE_YEARS": 5,
  "PRICE_PER_SESSION": 50,
  "RATING": 4.5,
  "VERIFIED": true,
  "SESSIONS_COMPLETED": 100,
  "SUCCESS_RATE": 95,
  "GAMES": ["Valorant", "CS:GO"],
  "SPECIALITIES": [...]
}
```

### Get All Mentors

```
GET /MentorProfile/GetAllMentorProfiles

Response (200 OK):
[
  { MENTOR_ID, USER_NAME, RATING, VERIFIED, ... },
  { MENTOR_ID, USER_NAME, RATING, VERIFIED, ... },
  ...
]

Recommendation: Add filtering by game, rating, availability
Recommendation: Add pagination for large result sets
```

### Get Mentor Dashboard

```
GET /MentorProfile/Dashboard/GetMentorProfile?MENTOR_ID={MENTOR_ID}

Response (200 OK):
{
  "MENTOR_ID": "...",
  "USER_NAME": "...",
  "TOTAL_EARNINGS": 5000,
  "PENDING_EARNINGS": 250,
  "SESSIONS_THIS_MONTH": 8,
  "RATING": 4.5,
  "REVIEWS_COUNT": 100,
  "UPCOMING_SESSIONS": [...]
}
```

---

## Blob Storage Service (Port 15000)

### Upload User Profile Picture

```
POST /UserProfilePic/StoreImage

Content-Type: multipart/form-data

Form Data:
├─ pic: File (image/jpeg, image/png)
└─ USER_ID: string (uuid, optional, auto-generated if not provided)

Response (200 OK):
{
  "pic_url": "http://localhost:15000/Image/RetrieveImage?bucket=user-profile-pic&key=550e8400-e29b-41d4-a716-446655440000.jpg"
}
```

### Upload User Profile Banner

```
POST /UserProfileBanner/StoreImage

(Same as UserProfilePic, bucket: user-profile-banner)
```

### Upload Mentor Profile Picture

```
POST /MentorProfilePic/StoreImage

(Same format, bucket: mentor-profile-pic)
```

### Upload Mentor Profile Banner

```
POST /MentorProfileBanner/StoreImage

(Same format, bucket: mentor-profile-banner)
```

### Upload Team Logo

```
POST /Team/StoreImage

(Same format, bucket: team-logo)
```

### Upload Event Banner

```
POST /Event/StoreImage

(Same format, bucket: event-banner)
```

### Retrieve Image

```
GET /Image/RetrieveImage?bucket={bucket}&key={key}

Query Parameters:
├─ bucket: string (user-profile-pic, user-profile-banner, etc.)
└─ key: string (resource_id.jpg)

Response (200 OK):
[Binary image data]

Possible Errors:
├─ 404: File not found
└─ 400: Invalid bucket or key
```

---

## MongoDB Service (Port 14000)

### Event CRUD Endpoints

```
GET    /Events/AllEvents
GET    /Events/GetEventInfo?EVENT_ID=
GET    /Events/Organized/User/AllEvents?USER_ID=
POST   /Events/CreateNewEvent
PUT    /Events/Update
PUT    /Events/Update/EventBanner
DELETE /Events/DeleteEvent?EVENT_ID=
POST   /Events/Discussion/AddNewQuestion
GET    /Events/Organizer/GetEventInfo?EVENT_ID=
```

### User Profile CRUD Endpoints

```
GET    /UserProfile/GetUserProfile?USER_ID=
GET    /UserProfile/GetAllUserProfiles
POST   /UserProfile/CreateNewUser
PUT    /UserProfile/Update/ProfilePic
PUT    /UserProfile/Update/ProfileBanner
```

### Mentor Profile CRUD Endpoints

```
GET    /MentorProfile/GetMentorProfile?MENTOR_ID=
GET    /MentorProfile/GetAllMentorProfiles
GET    /MentorProfile/Dashboard/GetMentorProfile?MENTOR_ID=
POST   /MentorProfile/CreateNewMentor
PUT    /MentorProfile/Update/ProfilePic
PUT    /MentorProfile/Update/ProfileBanner
```

### Teams CRUD Endpoints

```
GET    /Teams/User/GetAllTeams?USER_ID=
GET    /Teams/GetTeamInfo?TEAM_ID=
POST   /Teams/CreateNewTeam
PUT    /Teams/Update/TeamLogo
DELETE /Teams/DisbandTeam?TEAM_ID=
```

### Milestones CRUD Endpoints

```
POST   /Milestone/CreateNewMilestone
GET    /Milestone/GetMilestoneInfo
PUT    /Milestone/UpdateMilestone
DELETE /Milestone/DeleteMilestone
PUT    /Milestone/UpdateMilestoneStatus
```

---

## Common Request/Response Patterns

### Multipart File Upload Pattern

**Request**:
```
POST /Service/Endpoint

Content-Type: multipart/form-data
boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW

------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="file"; filename="image.jpg"
Content-Type: image/jpeg

[Binary file content]
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="metadata"
Content-Type: application/json

{"key": "value"}
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```

**Response**:
```json
{
  "file_url": "http://...",
  "file_size": 51234,
  "mime_type": "image/jpeg"
}
```

### Pagination Pattern (Future)

```
GET /Service/ListEndpoint?page=1&limit=20&sort=CREATED_AT&order=DESC

Response:
{
  "items": [...],
  "total": 150,
  "page": 1,
  "limit": 20,
  "has_more": true
}
```

### Filter Pattern (Future)

```
GET /Service/ListEndpoint?game=Valorant&status=VERIFIED&sort=RATING

Response:
[filtered items]
```

---

## Rate Limiting (Future)

```
Currently: No rate limiting
Planned:
├─ Per-IP limits: 100 requests/minute
├─ Per-user limits: 1000 requests/hour
└─ Per-endpoint limits: Specific thresholds
```

---

## API Versioning

**Current**: v0 (no versioning)
**Future**: Use URL versioning (e.g., `/v1/Events/CreateNewEvent`)

---

## CORS Configuration

**Enabled Services**:
- UserProfileService
- EventsService
- TeamsService
- MentorProfileService

**Configuration**:
```
Allow-Origin: *
Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Allow-Headers: *
```

**Security Note**: Current configuration allows all origins. For production, restrict to specific domains.

---

## Authentication (Future)

```
Authorization: Bearer {JWT_TOKEN}

Token Format:
{
  "sub": "USER_ID",
  "exp": 1704067200,
  "iat": 1704067200,
  "scopes": ["read", "write"]
}
```

---

## API Clients

### Using cURL

```bash
# Create user
curl -X POST http://localhost:7000/UserProfile/CreateNewUser \
  -F "pic=@photo.jpg" \
  -F "banner=@banner.jpg" \
  -F 'metadata={"USER_NAME":"ProPlayer"}'

# Get user
curl http://localhost:7000/UserProfile/GetUserProfile?USER_ID=550e8400-e29b-41d4-a716-446655440000
```

### Using Python Requests

```python
import requests

# Create event
files = {
    'banner': open('banner.jpg', 'rb'),
    'metadata': ('', json.dumps({
        'EVENT_NAME': 'Tournament',
        'EVENT_DATE': '2024-02-15T18:00:00Z'
    }))
}
response = requests.post(
    'http://localhost:13000/Events/CreateNewEvent',
    files=files
)
```

### Using Axios (JavaScript)

```javascript
const formData = new FormData();
formData.append('logo', logoFile);
formData.append('metadata', JSON.stringify({
  NAME: 'Team Name',
  GAME: 'Valorant'
}));

const response = await axios.post(
  'http://localhost:17000/Teams/CreateNewTeam',
  formData
);
```

---

## Summary

The ES Backend provides a comprehensive REST API for all major operations. All services follow consistent patterns for request/response handling and error reporting. The API is well-suited for client applications and service-to-service communication.

**Next Steps**:
1. Add authentication layer (AuthService)
2. Implement pagination and filtering
3. Add rate limiting
4. Set up API versioning
5. Generate OpenAPI/Swagger documentation automatically

