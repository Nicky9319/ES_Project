# ES Arena (by Gryork)

A full-stack platform for gamers, mentors, and communities to connect, compete, and grow together. Built with a **React frontend** and a **FastAPI microservices backend**.

---

## Project Structure

```
ES_Project/
├── ES_Frontend/          # React + Vite frontend
└── ES_Backend/          # FastAPI microservices backend
```

---

## Frontend — `ES_Frontend/`

**Stack:** React 18 · Vite · Tailwind CSS 4 · React Router 7 · Framer Motion · Recharts · Axios

### Key Dependencies
- **Routing:** react-router-dom (client-side routing)
- **Styling:** Tailwind CSS with custom design tokens and scrollbar styling
- **Animations:** framer-motion (page transitions) · aos (scroll animations)
- **HTTP:** axios (API communication)
- **Charts:** recharts (dashboard analytics)
- **UI:** lucide-react (icons) · react-icons · clsx · class-variance-authority

### Features
- **Landing Page** — Platform overview, feature highlights, FAQs
- **Authentication** — Login, registration with persona selection (Player / Mentor / Team)
- **Dashboard** — Personalized home feed and activity overview
- **Mentor Profiles** — Create/edit profiles, upload banner/avatar, tag specialities (3–5 tags), rating (default 3), verification status
- **Teams** — Create teams, add milestones, manage members
- **ES Events** — Browse and join tournaments (individual or team entries)
- **ES League & Tiers** — Competitive ranking system
- **Chat** — Real-time DMs and social messaging
- **Social** — Community feed and connections

### Getting Started

```sh
cd ES_Frontend
npm install
npm run dev        # Development server
npm run build      # Production build
npm run preview    # Preview production build
npm run lint       # ESLint check
```

**Note:** When developing locally, set the IP address of the Chat and Profile services to `127.0.0.1` to test against a local MongoDB instance.

---

## Backend — `ES_Backend/`

**Stack:** Python 3.12 · FastAPI · uvicorn · MongoDB · RabbitMQ · boto3 (S3)

### Architecture — Microservices

Each service is independently deployable and communicates via:
- **HTTP/REST** (synchronous request–response)
- **RabbitMQ** (async message queue for event-driven flows)

| Service | Port | Responsibility |
|---|---|---|
| `AUTH_SERVICE` | 5000 | Authentication & authorization |
| `USER_PROFILE_SERVICE` | 7000 | User account management |
| `MENTOR_PROFILE_SERVICE` | 10000 | Mentor profiles & specialities |
| `TEAMS_SERVICE` | 17000 | Team creation & management |
| `EVENTS_SERVICE` | 13000 | Tournament/event management |
| `NEWS_SERVICE` | 8000 | News & announcements feed |
| `FEED_SERVICE` | 9000 | Social feed aggregation |
| `CHAT_SERVICE` | 11000 | Real-time messaging |
| `LEAGUE_SYSTEM_SERVICE` | 12000 | Rankings & competitive tiers |
| `DASHBOARD_SERVICE` | 6000 | Dashboard data aggregation |
| `BLOB_STORAGE_SERVICE` | 15000 | File storage (S3-compatible) |
| `IMAGE_SERVER_SERVICE` | 16000 | Image serving & optimization |
| `MONGO_DB_SERVICE` | 14000 | Central MongoDB schema & queries |

### Data Model
- Schema defined in `MongoSchema.json` — single source of truth for all collection schemas
- Pass schema to an LLM to scaffold new service functions quickly

### Environment Setup

```sh
cd ES_Backend
# Create virtual environment
python -m venv venv
source venv/bin/activate      # or venv\Scripts\activate on Windows

# Install dependencies
pip install -r requirements.txt

# Configure environment variables in .env
```

### Running Services

**All services at once (local):**
```sh
cd ES_Backend/ServerScripts
bash TerminalStartServer.sh
```

**Individual service:**
```sh
cd ES_Backend/service_<ServiceName>
bash TerminalStartServer.sh
```

**Restart all services:**
```sh
cd ES_Backend/ServerScripts
bash TerminalRestartServer.sh
```

### Adding a New Service
1. Create `service_<ServiceName>/` directory
2. Add the service to `ServiceURLMapping.json`
3. Add required files to the service folder
4. Optionally add environment/dependency entries to `.env` and `requirements.txt`

### Testing
```sh
cd ES_Backend/Testing
python publisher.py   # Publish test messages to the queue
python subscriber.py  # Receive and verify messages
```

### Remote Deployment
1. Edit `CopyFilesToRemoteServer.sh` with the remote server IP and target directory
2. Run `bash CopyFilesToRemoteServer.sh` to transfer files
3. Follow `ServerScripts/setupServer.sh` for server-side component setup (RabbitMQ via Docker, MongoDB, etc.)

---

## Tech Stack Summary

| Layer | Technology |
|---|---|
| **Frontend** | React 18, Vite 6, Tailwind CSS 4, React Router 7 |
| **Backend** | FastAPI, Python 3.12, uvicorn |
| **Database** | MongoDB |
| **Message Queue** | RabbitMQ (Docker) |
| **Storage** | S3-compatible blob storage (boto3) |
| **Deployment** | Vercel (frontend), Linux remote server (backend) |
