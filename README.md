# FS_VaidehiMhatre
# 🚗 Student Commute Optimizer
## 📌 Problem
Students often commute individually, leading to higher travel costs, congestion, and inefficiency.
A system is needed to match students traveling along similar routes while keeping their identities private.

## 🎯 Proposed Solution
A carpooling & route-sharing platform for students with:
Map-based UI to enter origin/destination and see nearby students
Anonymous matching (unique usernames, no PII exposure)
Route overlap detection for suggesting carpool partners
In-app chat to coordinate rides
Friend groups & preferences (e.g., same class/year)
Push notifications (matches, chat, reminders)
Calendar & Maps integration for reminders and navigation

## 🏗️ High-Level Architecture

flowchart LR
    FE[Frontend App] -->|API Calls| BE[Backend Server]
    BE --> AUTH[Auth and Anonymity Service]
    BE --> COMM[Commute Service]
    BE --> MATCH[Matching Engine]
    BE --> CHAT[Chat Service]
    BE --> GROUPS[Groups and Preferences Service]
    BE --> NOTIFY[Notification Service]
    BE --> INTEGR[Calendar and Maps Integration]

    COMM --> DB[(Postgres + PostGIS)]
    MATCH --> DB
    GROUPS --> DB
    CHAT --> DB

    NOTIFY --> REDIS[(Redis Cache)]
    NOTIFY --> PUSH[(Push Service)]

    COMM --> ROUTER[Routing Engine]
    INTEGR --> EXTERNAL[External Maps and Calendar APIs]


## 📈 Sequence Flow

Student enters origin/destination.
Backend calls the routing engine → gets road-snapped route.
Backend saves this commute in DB.
Backend queries DB for overlaps with other students’ routes.
Backend sends back possible matches to student.
Student clicks on a match → starts chat (via WebSocket).
Push service sends notifications (e.g., “new match found” or “ride reminder”).

## 📈 Sequence Diagram

sequenceDiagram
    participant Student
    participant Backend
    participant DB
    participant Router
    participant Push
    participant Peer
    participant Groups
    participant CalendarAPI

    %% Login Flow
    Student->>Backend: Login / Signup (Anonymous handle)
    Backend->>DB: Create/Retrieve User
    DB-->>Backend: User record
    Backend-->>Student: Auth token + anon handle

    %% Commute Submission
    Student->>Backend: Submit commute (origin, destination, time, preferences)
    Backend->>Router: Get optimized route
    Router-->>Backend: Route geometry
    Backend->>DB: Save commute + preferences
    Backend->>DB: Query overlapping commutes
    DB-->>Backend: Candidate routes
    Backend-->>Student: Return potential matches

    %% Friend Groups & Preferences
    Student->>Backend: Create/Join "Commute Group"
    Backend->>Groups: Add to group
    Groups-->>Backend: Group confirmation
    Backend-->>Student: Updated group membership

    %% Chat & Notifications
    Student->>Backend: Start chat with Peer
    Backend-->>Peer: Deliver chat via WebSocket
    Backend->>Push: Send notification (new chat/match)
    Push-->>Student: Receive notification
    Push-->>Peer: Receive notification

    %% Calendar / Maps Integration
    Student->>Backend: Sync Calendar for commute reminders
    Backend->>CalendarAPI: Fetch events & reminders
    CalendarAPI-->>Backend: Event data
    Backend-->>Student: Reminder before commute



## ⚙️ Key Algorithms (Pseudocode)

Matching

function findMatches(my_commute, preferences):
    candidates = geoFilter(my_commute.route, my_commute.time)
    results = []
    for cand in candidates:
        if preferences["same_group"] and not inSameGroup(my_commute.user, cand.user):
            continue
        overlap = ST_Intersection(my_commute.route, cand.route)
        score = length(overlap) / length(my_commute.route)
        if score > 0.3:   # 30% overlap threshold
            results.append({cand, score})
    return sortByScore(results)

Notifications

function notify(user_id, title, body):
    token = db.getDeviceToken(user_id)
    if token:
        pushService.send(token, {title: title, body: body})

Calendar Integration

function exportToCalendar(commute):
    event = {
        summary: "Commute to College",
        location: commute.destination,
        start: commute.depart_time
    }
    calendarAPI.createEvent(commute.user_id, event)

## 📊 Core Data Model
Users → id, anon_handle, preferences, device_token
Commutes → id, user_id, origin, destination, route, time
Groups → id, name, members[]
Matches → commute_a, commute_b, score
Messages → channel_id, from_handle, body, timestamp

## 🔐 Privacy & Safety

Anonymous handles (Student_1234) for identity protection
Coarse map location until ride confirmed
Report/block features + SOS button

## ⚖️ Thought Process & Trade-Offs

Database (Postgres + PostGIS): best for spatial queries (route overlap, distance). Trade-off: heavier than MongoDB but more accurate for geo data.
Backend (FastAPI/Node): FastAPI is fast and Python-friendly for geospatial libs. Node is easier if team is JS-only. Trade-off: choose based on team skill.
Routing (Google Maps/OSRM): APIs are accurate but cost money; OSRM is free but harder to host. Trade-off: start with APIs for MVP, move to OSRM if scaling.
Matching: two-step (filter by area → compute overlap). Trade-off: more complex than brute force but faster and cheaper at scale.
Chat (WebSockets): real-time and user-friendly. Trade-off: more infra work than simple polling.
Push Notifications (FCM/APNs): good for reminders and new matches. Trade-off: extra setup and token management.
Groups & Preferences: builds trust and safety. Trade-off: reduces number of possible matches.
Calendar/Maps Integration: smooth reminders + navigation. Trade-off: dependency on external APIs.
Privacy: anonymous usernames protect identity. Trade-off: less personal trust, solved by optional group/verified handles later.


 ## 📊 Evaluation Metrics

| Metric                  | Definition                                   | Goal                 |
|--------------------------|-----------------------------------------------|----------------------|
| Match Accuracy           | % of carpools with ≥70% route overlap        | >80%                 |
| Average Match Time       | Time to return potential matches             | <5 sec               |
| Engagement Rate          | % of users who chat/join after a match       | >60%                 |
| Notification Effectiveness | % of notifications that trigger action     | >30% open rate       |
| Scalability              | Latency under load (10k users)               | <1 sec response time |
| User Retention           | % of users returning 3+ times a week         | >50%                 |
