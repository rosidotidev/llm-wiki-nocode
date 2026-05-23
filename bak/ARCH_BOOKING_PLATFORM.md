# Corporate Booking Platform — Architecture Document

> **Version:** 1.0  
> **Last Updated:** 2026-05-04  
> **Status:** Draft

---

## 1. Overview & Context

The **Corporate Booking Platform** (codename: *SpaceHub*) is an enterprise-grade application designed to streamline the reservation of meeting rooms and hot-desks across multiple corporate office locations. The platform enables employees to discover available spaces, make bookings, manage their reservations, and receive real-time notifications about upcoming events or changes.

### 1.1 Stakeholders

| Role | Interest |
|------|----------|
| Employees | Book rooms and desks quickly with minimal friction |
| Office Managers | Configure spaces, set policies, monitor utilization |
| Facilities Team | Understand occupancy patterns, plan maintenance windows |
| IT Operations | Ensure platform reliability, security, and compliance |
| Executive Leadership | Access utilization reports and cost optimization insights |

### 1.2 High-Level Goals

- Reduce room booking conflicts by 90% compared to the legacy calendar-based system.
- Provide a responsive, mobile-first user interface accessible from any corporate device.
- Integrate with corporate identity providers (Azure AD / Okta) for seamless SSO.
- Support multi-tenancy across geographically distributed offices.
- Deliver actionable analytics to facilities and leadership teams.

---

## 2. Architecture Overview

The system follows a classic **layered architecture** with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────┐
│                    Angular 21 SPA (Frontend)                 │
└─────────────────────────┬───────────────────────────────────┘
                          │ HTTPS / REST
┌─────────────────────────▼───────────────────────────────────┐
│          BFF — Experience Layer (Spring Boot / Java 21)      │
└───┬──────────────┬──────────────┬──────────────┬────────────┘
    │              │              │              │
    ▼              ▼              ▼              ▼
┌────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────┐
│Booking │  │  Space   │  │  User    │  │ Notification   │
│Service │  │ Service  │  │ Service  │  │   Service      │
│(Spring │  │(Spring   │  │(Spring   │  │(Spring Boot/   │
│ Boot)  │  │  Boot)   │  │  Boot)   │  │   Java 21)     │
└───┬────┘  └────┬─────┘  └────┬─────┘  └───────┬────────┘
    │             │             │                 │
    ▼             ▼             ▼                 ▼
┌────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────┐
│PostgreSQL│ │PostgreSQL│  │PostgreSQL│  │  Redis / SMTP  │
└──────────┘ └──────────┘  └──────────┘  └────────────────┘
```

### 2.1 Technology Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Frontend | Angular | 21.x |
| BFF (Experience Layer) | Spring Boot / Java | 21 |
| Microservices | Spring Boot / Java | 21 |
| Primary Database | PostgreSQL | 16 |
| Cache / Pub-Sub | Redis | 7.x |
| Message Broker | Apache Kafka | 3.7 |
| Identity Provider | Azure AD (OIDC) | — |
| Container Runtime | Kubernetes (AKS) | 1.29 |
| CI/CD | Azure DevOps Pipelines | — |
| Observability | OpenTelemetry + Grafana Stack | — |

---

## 3. Feature Catalog

| ID | Feature | Description | Priority |
|----|---------|-------------|----------|
| F-01 | Room Booking | Search, filter, and reserve meeting rooms by capacity, equipment, and floor | P0 |
| F-02 | Desk Booking | Reserve hot-desks on an interactive floor map with real-time availability | P0 |
| F-03 | Recurring Reservations | Create weekly/daily recurring bookings with conflict detection | P1 |
| F-04 | Calendar Sync | Bi-directional sync with Outlook 365 calendars | P1 |
| F-05 | Interactive Floor Map | SVG-based floor plans with clickable desks and rooms showing live status | P0 |
| F-06 | Check-In / No-Show | QR code check-in; auto-release bookings after 15-minute no-show window | P1 |
| F-07 | Favorites & Quick Book | One-tap booking for favorite spaces | P2 |
| F-08 | Admin Console | Office managers can CRUD spaces, define policies, and set blackout dates | P0 |
| F-09 | Notifications | Push, email, and in-app alerts for booking confirmations, changes, and reminders | P1 |
| F-10 | Reporting & Analytics | Utilization heatmaps, peak-hour analysis, per-floor occupancy trends | P1 |
| F-11 | Guest Registration | Pre-register external visitors linked to a room booking | P2 |
| F-12 | Multi-Location Support | Switch between offices/buildings; location-aware defaults | P0 |

---

## 4. UI Descriptions

The Angular 21 frontend is built as a Single Page Application using standalone components, signals for state management, and the Angular CDK for accessibility compliance (WCAG 2.1 AA).

### 4.1 Dashboard (`/dashboard`)

The landing page after login. Displays a personalized summary for the current day:

- **Today's Schedule** — Chronological list of the user's confirmed bookings (room and desk) with location, time, and quick-action buttons (cancel, extend, check-in).
- **Quick Book Widget** — A compact form allowing one-click booking of a favorite room or desk for the next available slot.
- **Office Announcements** — Banner area showing facility notifications (e.g., floor closures, maintenance).
- **Utilization Sparkline** — Mini chart showing this week's personal usage vs. team average.

### 4.2 Booking Wizard (`/book`)

A multi-step guided flow for creating a new reservation:

1. **Step 1 — Type Selection:** Choose between "Room" or "Desk."
2. **Step 2 — Filters:** Date/time picker, location dropdown, capacity slider (rooms), amenities checkboxes (whiteboard, video conferencing, standing desk, dual monitor).
3. **Step 3 — Results:** Grid/list view of matching spaces with availability timeline bars. Each card shows a thumbnail photo, capacity, floor, and equipment icons.
4. **Step 4 — Confirmation:** Summary of selected space and time, optional attendees field (for rooms), recurring toggle, notes textarea. Submit triggers the booking creation.

### 4.3 Interactive Floor Map (`/map/:floorId`)

An SVG-rendered floor plan with the following interactive behaviors:

- Desks and rooms are color-coded: **green** (available), **amber** (booked by colleague — visible), **red** (booked / occupied), **gray** (out of service).
- Clicking a green space opens a popover with details and a "Book Now" button.
- A time scrubber at the top allows previewing availability at different times of the day.
- Zoom and pan controls; pinch-to-zoom on mobile.
- Legend panel on the right side explaining the color codes and icons.

### 4.4 My Reservations (`/reservations`)

A tabbed view of the user's bookings:

- **Upcoming** — Sorted by date ascending. Cards show space name, time, floor, and action buttons (modify, cancel, check-in).
- **Past** — Read-only history. Option to "rebook" the same space.
- **Recurring** — Manage recurring series (edit pattern, skip occurrences, terminate series).

Each entry links to the floor map with the booked space highlighted.

### 4.5 Admin Console (`/admin`)

Accessible only to users with the `office-manager` or `admin` role. Sections include:

- **Space Management** — CRUD operations for rooms and desks. Bulk import via CSV. Define equipment lists, upload photos, set max capacity.
- **Policy Engine** — Configure rules such as max booking duration, advance booking window (e.g., up to 14 days ahead), auto-release timers, and blackout dates.
- **User Management** — View employees, assign roles, set location defaults.
- **Reports** — Embedded analytics dashboards (utilization heatmaps, peak-hour bar charts, per-floor occupancy over time).

---

## 5. BFF — Experience Services (REST)

The Backend-for-Frontend layer is a Spring Boot 21 application that serves as the single entry point for the Angular SPA. It aggregates, transforms, and orchestrates calls to downstream microservices, shielding the frontend from internal service topology.

**Base URL:** `https://api.spacehub.corp.net/experience/v1`

### 5.1 Authentication & Session

| Method | Path | Description |
|--------|------|-------------|
| GET | `/auth/me` | Returns the current authenticated user profile (aggregated from User Service + Azure AD claims) |
| POST | `/auth/logout` | Invalidate the session and revoke tokens |

### 5.2 Dashboard Experience

| Method | Path | Description |
|--------|------|-------------|
| GET | `/dashboard` | Aggregated dashboard payload: today's bookings, quick-book suggestions, announcements, utilization sparkline |

**Response Example:**
```json
{
  "todayBookings": [ ... ],
  "quickBookSuggestions": [ ... ],
  "announcements": [ ... ],
  "utilizationSummary": {
    "thisWeek": 4,
    "teamAverage": 3.2
  }
}
```

### 5.3 Availability Experience

| Method | Path | Description |
|--------|------|-------------|
| GET | `/availability/rooms` | Search available rooms by location, date range, capacity, amenities |
| GET | `/availability/desks` | Search available desks by location, date range, floor, zone |
| GET | `/availability/floor-map/{floorId}` | Full floor map state with availability status per space for a given time window |

**Query Parameters (rooms):**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `locationId` | UUID | Yes | Office/building identifier |
| `from` | ISO 8601 | Yes | Start datetime |
| `to` | ISO 8601 | Yes | End datetime |
| `minCapacity` | int | No | Minimum seating capacity |
| `amenities` | string[] | No | Filter by amenities (e.g., `whiteboard,videoconf`) |

### 5.4 Booking Experience

| Method | Path | Description |
|--------|------|-------------|
| POST | `/bookings` | Create a new booking (room or desk). Orchestrates validation, conflict check, calendar sync, and notification dispatch |
| GET | `/bookings/mine` | List current user's bookings with pagination and status filter |
| GET | `/bookings/{id}` | Get booking details including space info, attendees, and check-in status |
| PUT | `/bookings/{id}` | Modify an existing booking (time change, add attendees) |
| DELETE | `/bookings/{id}` | Cancel a booking. Triggers notification and calendar removal |
| POST | `/bookings/{id}/check-in` | Mark the user as checked in for a booking |

**Create Booking Request:**
```json
{
  "spaceId": "uuid",
  "type": "ROOM | DESK",
  "from": "2026-05-05T09:00:00Z",
  "to": "2026-05-05T10:00:00Z",
  "attendees": ["user-uuid-1", "user-uuid-2"],
  "recurring": {
    "pattern": "WEEKLY",
    "until": "2026-06-30"
  },
  "notes": "Project kickoff meeting"
}
```

**Create Booking Response (201):**
```json
{
  "bookingId": "uuid",
  "status": "CONFIRMED",
  "space": { "id": "...", "name": "Room Everest", "floor": 3 },
  "from": "2026-05-05T09:00:00Z",
  "to": "2026-05-05T10:00:00Z",
  "checkInDeadline": "2026-05-05T09:15:00Z"
}
```

### 5.5 User Experience

| Method | Path | Description |
|--------|------|-------------|
| GET | `/users/me/preferences` | Retrieve user preferences (default location, favorites, notification settings) |
| PUT | `/users/me/preferences` | Update user preferences |
| GET | `/users/me/favorites` | List favorite spaces |
| POST | `/users/me/favorites/{spaceId}` | Add a space to favorites |
| DELETE | `/users/me/favorites/{spaceId}` | Remove a space from favorites |

### 5.6 Admin Experience

| Method | Path | Description |
|--------|------|-------------|
| GET | `/admin/spaces` | List all spaces with filters (location, type, status) |
| POST | `/admin/spaces` | Create a new space (room or desk) |
| PUT | `/admin/spaces/{id}` | Update space details |
| DELETE | `/admin/spaces/{id}` | Decommission a space (soft delete) |
| GET | `/admin/policies` | Retrieve booking policies for a location |
| PUT | `/admin/policies` | Update booking policies |
| GET | `/admin/reports/utilization` | Aggregated utilization report (date range, location) |

---

## 6. Microservices

All microservices are Spring Boot 3.x applications running on Java 21 with Virtual Threads enabled. Each service owns its data and communicates asynchronously via Kafka events where eventual consistency is acceptable.

### 6.1 Booking Service

**Domain:** Manages the lifecycle of all reservations (create, modify, cancel, check-in, auto-release).

**Base URL:** `http://booking-service.internal:8080/api/v1`

| Method | Path | Description |
|--------|------|-------------|
| POST | `/bookings` | Create a booking after conflict validation |
| GET | `/bookings/{id}` | Retrieve booking by ID |
| GET | `/bookings` | Query bookings (by user, space, date range, status) |
| PUT | `/bookings/{id}` | Update booking details |
| PATCH | `/bookings/{id}/status` | Transition booking status (CONFIRMED → CHECKED_IN → COMPLETED / CANCELLED / NO_SHOW) |
| DELETE | `/bookings/{id}` | Cancel a booking |
| GET | `/bookings/conflicts` | Check for conflicts given a space + time range |

**Events Published (Kafka):**
- `booking.created`
- `booking.modified`
- `booking.cancelled`
- `booking.checked-in`
- `booking.no-show`

**Database Schema (PostgreSQL — `booking_db`):**

| Table | Key Columns |
|-------|-------------|
| `bookings` | id (UUID PK), space_id, user_id, type, status, start_time, end_time, recurring_group_id, created_at |
| `booking_attendees` | booking_id (FK), user_id |
| `recurring_groups` | id (UUID PK), pattern, until_date, parent_booking_id |

### 6.2 Space Service

**Domain:** Manages the inventory of physical spaces (rooms, desks), their attributes, locations, and floor plans.

**Base URL:** `http://space-service.internal:8080/api/v1`

| Method | Path | Description |
|--------|------|-------------|
| GET | `/locations` | List all office locations |
| GET | `/locations/{id}/floors` | List floors for a location |
| GET | `/floors/{id}/spaces` | List spaces on a floor |
| GET | `/spaces/{id}` | Get space details (capacity, amenities, coordinates, photo URL) |
| POST | `/spaces` | Create a new space (admin) |
| PUT | `/spaces/{id}` | Update space attributes |
| DELETE | `/spaces/{id}` | Soft-delete / decommission a space |
| GET | `/floors/{id}/map` | Retrieve floor map SVG metadata with space coordinates |
| GET | `/amenities` | List all available amenity types |

**Events Published (Kafka):**
- `space.created`
- `space.updated`
- `space.decommissioned`

**Database Schema (PostgreSQL — `space_db`):**

| Table | Key Columns |
|-------|-------------|
| `locations` | id (UUID PK), name, address, timezone |
| `floors` | id (UUID PK), location_id (FK), floor_number, map_svg_url |
| `spaces` | id (UUID PK), floor_id (FK), type (ROOM/DESK), name, capacity, status, coordinates_json, photo_url |
| `space_amenities` | space_id (FK), amenity_id (FK) |
| `amenities` | id (UUID PK), code, label, icon_url |

### 6.3 User Service

**Domain:** Manages user profiles, preferences, favorites, and role assignments. Acts as the local cache for identity provider data.

**Base URL:** `http://user-service.internal:8080/api/v1`

| Method | Path | Description |
|--------|------|-------------|
| GET | `/users/{id}` | Get user profile (synced from Azure AD) |
| GET | `/users/{id}/preferences` | Get user preferences |
| PUT | `/users/{id}/preferences` | Update preferences (default location, notification channels) |
| GET | `/users/{id}/favorites` | List favorite spaces |
| POST | `/users/{id}/favorites` | Add favorite |
| DELETE | `/users/{id}/favorites/{spaceId}` | Remove favorite |
| GET | `/users/{id}/roles` | Get assigned roles |
| PUT | `/users/{id}/roles` | Assign/remove roles (admin only) |

**Events Consumed (Kafka):**
- `azure-ad.user-sync` — Periodic sync of employee directory

**Database Schema (PostgreSQL — `user_db`):**

| Table | Key Columns |
|-------|-------------|
| `users` | id (UUID PK), email, display_name, department, location_id, avatar_url, synced_at |
| `user_preferences` | user_id (FK PK), default_location_id, notification_email, notification_push, notification_slack |
| `user_favorites` | user_id (FK), space_id |
| `user_roles` | user_id (FK), role (ENUM: EMPLOYEE, OFFICE_MANAGER, ADMIN) |

### 6.4 Notification Service

**Domain:** Responsible for dispatching notifications across multiple channels (email, push, in-app, Slack). Consumes domain events and applies user notification preferences before sending.

**Base URL:** `http://notification-service.internal:8080/api/v1`

| Method | Path | Description |
|--------|------|-------------|
| GET | `/notifications/user/{userId}` | List in-app notifications (paginated) |
| PATCH | `/notifications/{id}/read` | Mark a notification as read |
| POST | `/notifications/send` | Manually trigger a notification (admin/system use) |

**Events Consumed (Kafka):**
- `booking.created` → Send confirmation email + push
- `booking.cancelled` → Notify attendees
- `booking.no-show` → Notify user of auto-release
- `space.decommissioned` → Notify affected users with active bookings

**Channels:**
- **Email** — Via corporate SMTP relay (SendGrid adapter available)
- **Push** — Firebase Cloud Messaging
- **In-App** — Stored in Redis sorted sets, served via SSE (Server-Sent Events) to the frontend
- **Slack** — Webhook integration to workspace channels

**Database/Store:**

| Store | Purpose |
|-------|---------|
| Redis (sorted sets) | In-app notification feed per user |
| PostgreSQL (`notification_db`) | Audit log of all dispatched notifications |

### 6.5 Reporting Service

**Domain:** Aggregates booking and space data to produce analytics dashboards, utilization metrics, and exportable reports.

**Base URL:** `http://reporting-service.internal:8080/api/v1`

| Method | Path | Description |
|--------|------|-------------|
| GET | `/reports/utilization` | Utilization percentage by location/floor/space over a date range |
| GET | `/reports/peak-hours` | Peak booking hours histogram |
| GET | `/reports/no-show-rate` | No-show rate trends |
| GET | `/reports/occupancy-heatmap` | Per-floor heatmap data (hourly granularity) |
| GET | `/reports/export` | Generate CSV/PDF export of a report |

**Events Consumed (Kafka):**
- `booking.*` — All booking lifecycle events are materialized into the reporting data store for aggregation.

**Database:**
- **PostgreSQL (`reporting_db`)** — Materialized views and pre-aggregated time-series tables optimized for read-heavy queries.
- **Redis** — Caching layer for frequently accessed dashboard queries (TTL: 5 minutes).

---

## 7. Data Layer

### 7.1 Database Strategy

Each microservice follows the **Database-per-Service** pattern to ensure loose coupling and independent deployability.

| Service | Database | Engine | Notes |
|---------|----------|--------|-------|
| Booking Service | `booking_db` | PostgreSQL 16 | Row-level locking for conflict prevention |
| Space Service | `space_db` | PostgreSQL 16 | Stores spatial coordinates as JSONB |
| User Service | `user_db` | PostgreSQL 16 | Synced from Azure AD |
| Notification Service | `notification_db` + Redis | PostgreSQL 16 + Redis 7 | Redis for real-time feeds |
| Reporting Service | `reporting_db` | PostgreSQL 16 | TimescaleDB extension for time-series aggregation |

### 7.2 Key Data Entities (Logical)

```
Location 1──* Floor 1──* Space
                              │
User 1──* Booking *──1 Space
│                   │
│                   *──* Attendee (User)
│
User 1──* Favorite (Space)
User 1──1 Preferences
```

### 7.3 Consistency Model

- **Strong consistency** within a single service boundary (ACID transactions via PostgreSQL).
- **Eventual consistency** across services via Kafka events with at-least-once delivery semantics and idempotent consumers.
- **Conflict detection** in the Booking Service uses `SELECT FOR UPDATE` with a short lock timeout to prevent double-booking race conditions.

---

## 8. Cross-Cutting Concerns

### 8.1 Authentication & Authorization

- **Protocol:** OpenID Connect (OIDC) with Azure AD as the identity provider.
- **Token Flow:** Authorization Code Flow with PKCE (SPA) → BFF validates and exchanges tokens → JWT propagated to microservices via internal `Authorization` header.
- **Role-Based Access Control (RBAC):** Roles (`EMPLOYEE`, `OFFICE_MANAGER`, `ADMIN`) are stored in the User Service and enriched into the JWT claims at the BFF layer.
- **Service-to-Service:** mTLS within the Kubernetes cluster; no external exposure of internal service APIs.

### 8.2 Observability

| Pillar | Tool | Notes |
|--------|------|-------|
| Traces | OpenTelemetry → Tempo | Distributed tracing across BFF and all services |
| Metrics | Micrometer → Prometheus → Grafana | JVM metrics, request latency, booking throughput |
| Logs | Structured JSON → Loki | Correlated via trace-id |
| Alerts | Grafana Alerting | SLO-based alerts (p99 latency, error rate) |

### 8.3 Rate Limiting & Throttling

- Implemented at the BFF layer using Spring Cloud Gateway filters.
- Default: 100 requests/minute per user for booking endpoints.
- Admin endpoints: 30 requests/minute.
- Floor map endpoints: 200 requests/minute (higher due to frequent polling).

### 8.4 Error Handling

All services follow a unified error response format:

```json
{
  "timestamp": "2026-05-04T10:30:00Z",
  "status": 409,
  "error": "Conflict",
  "code": "BOOKING_CONFLICT",
  "message": "The selected time slot overlaps with an existing booking.",
  "traceId": "abc123def456"
}
```

Standard HTTP status codes are used consistently:
- `400` — Validation errors
- `401` — Unauthenticated
- `403` — Insufficient permissions
- `404` — Resource not found
- `409` — Conflict (e.g., double-booking)
- `429` — Rate limit exceeded
- `500` — Internal server error

---

## 9. Deployment Topology

### 9.1 Kubernetes Namespaces

| Namespace | Contents |
|-----------|----------|
| `spacehub-frontend` | Angular SPA served via NGINX ingress |
| `spacehub-bff` | BFF Experience Layer (2–4 replicas, HPA) |
| `spacehub-services` | All microservices (individual Deployments, 2–3 replicas each) |
| `spacehub-data` | PostgreSQL (managed via Azure Database for PostgreSQL Flexible Server), Redis (Azure Cache) |
| `spacehub-infra` | Kafka (Strimzi operator), observability stack |

### 9.2 CI/CD Pipeline

```
Code Push → Azure DevOps Pipeline
  ├── Build (Gradle / Java 21)
  ├── Unit Tests + Integration Tests (Testcontainers)
  ├── SAST Scan (SonarQube)
  ├── Container Image Build (Jib)
  ├── Push to Azure Container Registry
  ├── Deploy to Staging (Helm chart)
  ├── E2E Tests (Playwright for Angular, REST Assured for APIs)
  └── Promote to Production (manual gate)
```

### 9.3 Scaling Strategy

- **BFF:** Horizontal Pod Autoscaler based on CPU (target 60%) and request latency (p95 < 200ms).
- **Booking Service:** Scaled on custom metric (active booking transactions).
- **Notification Service:** Scaled on Kafka consumer lag.
- **Reporting Service:** Scheduled scaling during business hours; scaled down during off-hours.

---

## Appendix A — API Versioning Strategy

All APIs (BFF and microservices) follow URI-based versioning (`/v1/`, `/v2/`). Breaking changes result in a new version; non-breaking additions are made in-place. Deprecated versions are supported for a minimum of 6 months with sunset headers.

## Appendix B — Glossary

| Term | Definition |
|------|-----------|
| BFF | Backend-for-Frontend — an aggregation layer tailored to a specific UI |
| Hot-Desk | A shared desk not permanently assigned to any employee |
| Space | Generic term for any bookable resource (room or desk) |
| Check-In | Physical confirmation of presence at a booked space |
| No-Show | Failure to check in within the grace period, triggering auto-release |
| Experience API | A coarse-grained API designed for a specific user journey, aggregating multiple microservice calls |
