# Getin-Scanner-be – Flow (endpoints, cron, data flow)

Base URL: `/api/v1/events`. All POST routes use **verifyLambdaToken** (query or body `token`).

---

## 1. Entry points overview

```mermaid
flowchart LR
    subgraph Triggers
        CRON["Cron (every 1 min)"]
        E1["POST /attendees"]
        E2["POST /uploadCSV"]
        E3["POST /attendees-cron"]
        E4["POST /create-attendee"]
        E5["POST /sync-event"]
    end

    subgraph Controller
        FEU["fetchEventsForUsers"]
        UPLOAD["uploadCSVAttendee"]
        CREATE["createAttendee"]
        SYNC["syncEvent"]
    end

    CRON --> FEU
    E1 --> FEU
    E3 --> FEU
    E2 --> UPLOAD
    E4 --> CREATE
    E5 --> SYNC
```

| Trigger | Handler | What it does (short) |
|--------|---------|----------------------|
| **Cron** (every 1 min, if DATABASE_URL set) | `fetchEventsForUsers({})` | Sync events + attendees for **all** users from DB |
| **POST /attendees** (body: `user_id`, token) | `fetchEventsForUsers({ user_id, res })` | Sync events + attendees for **one** user |
| **POST /attendees-cron** (body: token) | `fetchEventsForUsers({ isCron: true })` | Same as cron, but triggered by HTTP |
| **POST /uploadCSV** (file + eventId + token) | `uploadCSVAttendee` | Validate event → parse CSV → upsert attendees → delete file |
| **POST /create-attendee** (body: participants, creator_id, token) | `createAttendee` | Upsert attendees from payload; may trigger creator/event sync first |
| **POST /sync-event** (body: event_id, auth_user_id, …) | `syncEvent` | Find/create creator → processEventData → syncAddons |

---

## 2. fetchEventsForUsers flow (cron + /attendees + /attendees-cron)

```mermaid
flowchart TB
    START([fetchEventsForUsers]) --> GET_USERS["processUserData user_id"]
    GET_USERS --> USERS_SOURCE{"user_id given?"}
    USERS_SOURCE -->|Yes| ONE["Get or update 1 user from Get-in API - user-details"]
    USERS_SOURCE -->|No cron| ALL["prisma.creators.findMany - all users in DB"]
    ONE --> LOOP
    ALL --> LOOP

    LOOP["For each user"] --> MARK["markeUsersProcessing user, isCron"]
    MARK --> UPDATE_ALL["updateEventStatusAfterProcessing user_id, isCron"]
    UPDATE_ALL --> LOOP
    LOOP --> DONE([Done])

    subgraph markeUsersProcessing
        direction TB
        M1["POST Get-in: events/dashboard - events and seller_events"] --> M2["processEventData for each - prisma.events create or update"]
        M2 --> M3["syncAddons for each event"]
        M3 --> M4["prisma.events.findMany - live events end_date gt now"]
        M4 --> M5["For each live event: skip if completed or in-progress, else set InProgress"]
        M5 --> M6["fetchAndUpsertAttendeesBatchWise eventId"]
        M6 --> M7["finally: set sync_status and cron_status to Completed"]
    end
```

- **processUserData**: If `user_id` → call Get-in `/users/user-details`, then create/update **creators** in DB and return `[user]`. If no `user_id` (cron) → return all **creators** from DB.
- **markeUsersProcessing**: Fetches events from Get-in dashboard → writes/updates **events** and **add_ons** → for each **live** event fetches attendees from Get-in and upserts **attendee** (+ addons). Sets sync/cron status.

---

## 3. Where attendees come from (fetchAndUpsertAttendeesBatchWise)

```mermaid
flowchart LR
    subgraph Backend
        F["fetchAndUpsertAttendeesBatchWise"]
    end
    subgraph GetInAPI["Get-in API"]
        P["POST attendees/list - body event_id - query auth-user-id, scanner-api-key"]
    end
    subgraph DB["Scanner DB Supabase"]
        EV["events"]
        AT["attendee"]
        AO["attendee_add_ons"]
    end

    F --> P
    P -->|participants| F
    F --> EV
    F --> AT
    F --> AO
```

- **Single source of truth for list**: Get-in API `POST .../attendees/list` with `event_id`.
- Backend maps `response.data.participants` to your schema and upserts into **attendee** (and add-ons). Also updates **events** (total_attendees, fetched_attendees, sync_status, cron_status).

---

## 4. uploadCSV flow

```mermaid
flowchart TB
    A([POST /uploadCSV: file + eventId]) --> B["verifyLambdaToken"]
    B --> C["multer: save file to public/uploads"]
    C --> D["prisma.events.findUnique( eventId )"]
    D --> E{"Event exists?"}
    E -->|No| F["400/500 Event Not found"]
    E -->|Yes| G["EventService.uploadCSV( filePath, eventId )"]
    G --> H["Parse CSV → upsert attendee rows"]
    H --> I["fs.unlink( file )"]
    I --> J(["200 Uploaded CSV successfully"])
```

---

## 5. create-attendee flow

```mermaid
flowchart TB
    A([POST /create-attendee: participants, creator_id, token]) --> B["Send 200 'sync in progress'"]
    B --> C["eventId = participants[0].event_id"]
    C --> D{"Event in DB?"}
    D -->|No| E["processUserData( creator_id )\nmarkeUsersProcessing( creator )"]
    E --> F["Prepare serverAttendees from participants"]
    D -->|Yes| F
    F --> G["Batch upsert attendee (+ addons)"]
    G --> H["Update event total_attendees, fetched_attendees"]
    H --> I([Done])
```

---

## 6. sync-event flow

```mermaid
flowchart TB
    A([POST /sync-event: event_id, auth_user_id, type?, addons?, ...]) --> B["Validate event_id, auth_user_id"]
    B --> C["Find/create creator (auth_user_id)"]
    C --> D{"type === 3?"}
    D -->|Yes| E["sellerEventOperation( data, creator )"]
    D -->|No| F["prisma.events.findFirst( event_id )"]
    E --> G["processEventData( data, eventDetails, ... )"]
    F --> G
    G --> H["prisma.events.update/create"]
    H --> I{"addons?.length > 0?"}
    I -->|Yes| J["syncAddons( addons, event_id )"]
    I -->|No| K([Done])
    J --> K
```

---

## 7. External APIs used (Get-in)

| Purpose | Method + path | Query / body |
|--------|----------------|---------------|
| Events list | POST `{BACKEND_API_URL}/api/scanner-app/events/dashboard` | `auth-user-id`, `scanner-api-key`; body: `{ filter }` |
| Attendees list | POST `{BACKEND_API_URL}/api/scanner-app/attendees/list` | `auth-user-id`, `scanner-api-key`; body: `{ event_id }` |
| User details | POST `{BACKEND_API_URL}/api/scanner-app/users/user-details` | `auth-user-id`, `scanner-api-key` |

All use **SCANNER_API_KEY** and **BACKEND_API_URL** from env.

**Testing (e.g. curl):** Attendees list requires a JSON body with `event_id`. Empty body can return "This action is unauthorized".

```bash
curl -X POST 'https://api.getin-nextgen.com/api/scanner-app/attendees/list?auth-user-id=14462&scanner-api-key=YOUR_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"event_id": 12803}'
```

---

## 8. Quick reference

- **Cron**: `src/cron.ts` → `fetchEventsForUsers({})` every 1 min (if DATABASE_URL set).
- **Routes**: `src/routes/eventsRoutes.ts` → all under `/api/v1/events`.
- **Controller**: `src/controller/events.controller.ts` (fetchEventsForUsers, markeUsersProcessing, fetchAndUpsertAttendeesBatchWise, processEventData, syncAddons, uploadCSV, createAttendee, syncEvent).
- **Auth**: `verifyLambdaToken` → token from query or body must equal `LAMBDA_TOKEN`.
