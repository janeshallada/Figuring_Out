# Architecture Overview

## System Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT SIDE                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │           React Frontend (Port 5173)                     │  │
│  │  • matrix-port/                                          │  │
│  │  • React + TypeScript + Tailwind + shadcn UI           │  │
│  │  • Pages: Landing, Login, Dashboard, Matches, Rooms    │  │
│  │  • Real-time chat with WebSocket                        │  │
│  └────────┬─────────────────────┬──────────────────────────┘  │
│           │                     │                               │
│        HTTP API              WebSocket                          │
│        (Axios)               (Socket.IO)                        │
│           │                     │                               │
└───────────┼─────────────────────┼───────────────────────────────┘
            │                     │
            ├─────────────────────┤
            │                     │
┌───────────┼─────────────────────┼───────────────────────────────┐
│           │                     │      SERVER SIDE              │
│  ┌────────▼─────────────────────▼──────────────────────────┐  │
│  │      Express.js Backend (Port 3000)                     │  │
│  │  • api/                                                  │  │
│  │  • Node.js + TypeScript + Express                      │  │
│  │                                                         │  │
│  │  Routes:                      Middleware:              │  │
│  │  ├── /auth                    ├── auth.ts (JWT)       │  │
│  │  ├── /users                   ├── CORS                │  │
│  │  ├── /rooms                   └── Error handling      │  │
│  │  └── /progress                                        │  │
│  │                                                         │  │
│  │  WebSocket (Socket.IO):                               │  │
│  │  ├── message:send                                      │  │
│  │  ├── message:subscribe                                 │  │
│  │  └── typing indicators                                 │  │
│  └────────────────┬────────────────────────────────────────┘  │
│                   │                                             │
│                PostgreSQL                                      │
│                   │                                             │
│  ┌────────────────▼────────────────────────────────────────┐  │
│  │       Database (Collaboration DB)                       │  │
│  │  • Prisma ORM (Type-safe)                              │  │
│  │                                                        │  │
│  │  Tables:                                              │  │
│  │  ├── users (accounts, profiles)                       │  │
│  │  ├── rooms (collaboration spaces)                     │  │
│  │  ├── room_members (membership)                        │  │
│  │  ├── messages (chat)                                  │  │
│  │  └── progress_updates (feed)                          │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## Authentication Flow

```
┌─────────────┐
│   User      │
└──────┬──────┘
       │
       │ Enter credentials
       ▼
┌─────────────────────────────────┐
│  POST /auth/register or login   │
│  { email, password }            │
└────────────┬────────────────────┘
             │
             ▼
    ┌────────────────────┐
    │ Verify password    │
    │ (bcrypt compare)   │
    └────────┬───────────┘
             │
      ✓ Match │
             │
             ▼
    ┌────────────────────┐
    │ Generate JWT Token │
    │ (7 days expiry)    │
    └────────┬───────────┘
             │
             ▼
  Return: { token, user }
             │
             │ Store in localStorage
             │
             ▼
   ┌──────────────────────┐
   │ Include in header:   │
   │ Authorization:       │
   │ Bearer <JWT_TOKEN>   │
   └──────────────────────┘
             │
   Used for all authenticated requests
```

---

## Real-Time Messaging Flow

```
┌─────────────────┐          ┌─────────────────┐
│  User A         │          │  User B         │
│  (Browser)      │          │  (Browser)      │
└────────┬────────┘          └────────┬────────┘
         │                           │
         │ WebSocket              WebSocket
         │ Connected             Connected
         │                           │
         └──────────┬────────────────┘
                    │
         ┌──────────▼──────────┐
         │  Socket.IO Server   │
         │  (Express Backend)  │
         └──────────┬──────────┘
                    │
     ┌──────────────┼──────────────┐
     │              │              │
 Subscribe      Subscribe      Subscribe
 room:1         room:1         room:1
     │              │              │
     └──────────────┼──────────────┘
                    │
            Room 1 Channel
                    │
         ┌──────────┴──────────┐
         │                     │
    User A types         User B types
    message:send         message:send
         │                     │
         └──────────┬──────────┘
                    │
              Save to DB
         (prisma.message.create)
                    │
                    ▼
         Broadcast to room:
         message:received
                    │
         ┌──────────┴──────────┐
         │                     │
      User A             User B
      receives           receives
      message            message
         │                     │
         ▼                     ▼
      Display            Display
```

---

## API Endpoint Organization

```
Express App
    │
    ├── /auth                    (No auth required)
    │   ├── POST /register       → Create account
    │   ├── POST /login          → Get JWT token
    │   └── GET /me              → Requires auth
    │
    ├── /users                   (All require auth)
    │   ├── GET /:id             → Get user profile
    │   ├── GET /me/profile      → Get my profile
    │   ├── PUT /me/profile      → Update profile
    │   └── GET /matches         → Get matches
    │
    ├── /rooms                   (All require auth)
    │   ├── GET /                → List all rooms
    │   ├── POST /               → Create room
    │   ├── GET /:id             → Room details
    │   ├── POST /:id/join       → Join room
    │   ├── POST /:id/leave      → Leave room
    │   └── GET /:id/messages    → Chat history
    │
    ├── /progress               (All require auth)
    │   ├── GET /               → Get feed
    │   └── POST /              → Post update
    │
    └── WebSocket Events        (Real-time)
        ├── message:subscribe   → Join room channel
        ├── message:send        → Send message
        ├── message:received    → Receive message
        ├── typing              → Typing indicator
        └── typing:stop         → Stop typing
```

---

## Database Schema Relationships

```
┌──────────────┐
│    USERS     │
├──────────────┤
│ id (PK)      │◄─────┐
│ email        │      │
│ name         │      │
│ password     │      │
│ bio          │      │
│ skills[]     │      │
│ goals[]      │      │
│ interests[]  │      │
│ location     │      │
└──────────────┘      │
        ▲              │
        │ ownerId      │ userId
        │              │
    ┌───┴──────────┐   ├──────────────────┐
    │              │   │                  │
┌───┴──────┐  ┌───┴────────┐  ┌──────────┴──┐
│  ROOMS   │  │ROOM_MEMBERS│  │ MESSAGES    │
├──────────┤  ├────────────┤  ├─────────────┤
│id (PK)   │  │id (PK)     │  │id (PK)     │
│name      │◄─┤roomId (FK) │  │roomId (FK) │◄──┐
│desc      │  │userId (FK)─┼─→│userId (FK) │   │
│topic     │  └────────────┘  └─────────────┘   │
│ownerId   │                                     │
│createdAt │                  ┌──────────────┐  │
└──────────┘                   │   PROGRESS   │  │
                               ├──────────────┤  │
                               │id (PK)      │  │
                               │userId (FK)──┼──┘
                               │text         │
                               │goal         │
                               │createdAt    │
                               └──────────────┘

PK  = Primary Key
FK  = Foreign Key
[]  = Array type
```

---

## Request/Response Lifecycle

```
1. CLIENT REQUEST
   ┌─────────────────────────────────────┐
   │ POST /auth/login                    │
   │ { "email": "...", "password": "..." }
   └────────────────┬────────────────────┘
                    │
2. MIDDLEWARE PROCESSING
                    ▼
   ┌─────────────────────────────────────┐
   │ Express receives request            │
   │ Parse JSON body                     │
   │ Check CORS                          │
   │ Route to handler                    │
   └────────────────┬────────────────────┘
                    │
3. AUTHENTICATION (if needed)
                    ▼
   ┌─────────────────────────────────────┐
   │ Check Authorization header          │
   │ Verify JWT token                    │
   │ Extract user ID                     │
   │ Attach to req.user                  │
   └────────────────┬────────────────────┘
                    │
4. VALIDATION
                    ▼
   ┌─────────────────────────────────────┐
   │ Validate request body (Zod)         │
   │ Check required fields               │
   │ Type check                          │
   └────────────────┬────────────────────┘
                    │
5. BUSINESS LOGIC
                    ▼
   ┌─────────────────────────────────────┐
   │ Execute handler function            │
   │ Query/modify database               │
   │ Apply matching algorithm            │
   │ etc.                                │
   └────────────────┬────────────────────┘
                    │
6. DATABASE OPERATION (via Prisma)
                    ▼
   ┌─────────────────────────────────────┐
   │ Prisma generates SQL                │
   │ Execute on PostgreSQL               │
   │ Return results                      │
   └────────────────┬────────────────────┘
                    │
7. RESPONSE FORMATTING
                    ▼
   ┌─────────────────────────────────────┐
   │ Format response object              │
   │ Set HTTP status code                │
   │ Add headers                         │
   └────────────────┬────────────────────┘
                    │
8. SERVER RESPONSE
                    ▼
   ┌─────────────────────────────────────┐
   │ 200 OK / 201 Created / 400 Error    │
   │ { "token": "...", "user": {...} }   │
   └────────────────┬────────────────────┘
                    │
9. CLIENT RECEIVES
                    ▼
   ┌─────────────────────────────────────┐
   │ Frontend receives JSON              │
   │ Store token in localStorage         │
   │ Update UI / React state             │
   │ Redirect/navigate                   │
   └─────────────────────────────────────┘
```

---

## Directory Structure After Setup

```
pora/
│
├── matrix-port/                    ← Frontend (already built)
│   ├── src/
│   │   ├── pages/
│   │   ├── components/
│   │   ├── services/
│   │   │   ├── api.ts              ← Update to use real endpoints
│   │   │   └── websocket.ts        ← New: WebSocket client
│   │   ├── context/
│   │   │   └── AuthContext.tsx     ← Update to use real API
│   │   └── ...
│   ├── .env                        ← Add VITE_API_URL
│   ├── package.json
│   └── ...
│
├── api/                            ← Backend (create this)
│   ├── src/
│   │   ├── index.ts                ← Server entry point
│   │   ├── routes/
│   │   │   ├── auth.ts
│   │   │   ├── users.ts
│   │   │   ├── rooms.ts
│   │   │   └── progress.ts
│   │   ├── middleware/
│   │   │   └── auth.ts
│   │   └── websocket/
│   │       └── handlers.ts
│   ├── prisma/
│   │   └── schema.prisma           ← Database schema
│   ├── .env                        ← Configuration
│   ├── package.json
│   ├── tsconfig.json
│   └── ...
│
├── README.md                       ← This guide
├── BACKEND_SETUP.md               ← Setup instructions
├── FRONTEND_INTEGRATION.md        ← Frontend migration
├── API_REFERENCE.md               ← Endpoint docs
├── SETUP_CHECKLIST.md             ← Progress tracker
├── IMPLEMENTATION_SUMMARY.md      ← What was built
└── Files for api/ folder:
    ├── tsconfig.json
    ├── .env.example
    ├── src-index.ts
    ├── src-middleware-auth.ts
    ├── src-routes-auth.ts
    ├── src-routes-users.ts
    ├── src-routes-rooms.ts
    ├── src-routes-progress.ts
    ├── src-websocket-handlers.ts
    └── prisma-schema.prisma
```

---

## Matching Algorithm

```
For User A finding matches:

1. Get User A profile:
   - goals: ["Learn Node.js", "Build startup"]
   - interests: ["Backend", "Startups"]
   - location: "San Francisco"

2. Get all other users

3. For each user B:
   ├── Count shared goals
   │   └── Each shared goal = +3 points
   │
   ├── Count shared interests
   │   └── Each shared interest = +2 points
   │
   └── Check location match
       └── Same location = +1 point

4. Example scores:
   User B: goals[2 shared], interests[1 shared], location[yes]
   Score = (2 × 3) + (1 × 2) + 1 = 9 points
   
   User C: goals[1 shared], interests[2 shared], location[no]
   Score = (1 × 3) + (2 × 2) + 0 = 7 points

5. Sort by score (highest first)

6. Return matches with scores
```

---

This architecture provides:
✅ Separation of concerns (frontend vs backend)
✅ Real-time communication (WebSocket)
✅ Secure authentication (JWT)
✅ Type safety (TypeScript)
✅ Database integrity (Prisma + PostgreSQL)
✅ Scalability (stateless API design)
