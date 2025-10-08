# Mission Control API Reference Guide

## Overview

The Mission Control API **(v1)** provides programmatic control and monitoring of autonomous surface vessels **(ASVs)** and their associated fleet operations, including mission planning, telemetry, configuration, logging, and event webhooks.

This API is designed to balance operational flexibility with the reliability and traceability requirements common to autonomous systems.Typical users include systems integrators, mission operators, and automation engineers who need to script or automate fleet behaviors. Common use cases are mission coordination (create/launch/abort), live telemetry reporting, configuration management, and retrieval of mission logs for analysis.

### Example: 	
`GET /vessels/{vessel_id}/logs?since=ISO8601` — retrieves vessel logs with event timestamps, severity levels, sources, messages, and contextual details.

## Appendix/Reference and Quick Start Guide

### Mission Priority Enum

| Value | Description |
|--------|------|
| `low`  | Non-urgent mission |
| `routine`  | Standard mission priority |
| `high`  | Urgent mission requiring immediate action |


### Fail-Safe Modes

| Mode | Description |
|--------|------|
| `return_to_base`  | Vessel returns to home base in case of failure |
| `hold_position`  | Vessel stops and holds its current position |
| `continue_autonomous`  | Vessel continues mission autonomously |


### Vessel Types

| Value | Description |
|--------|------|
| `ASV`  | Autonomous Surface Vehicle |
| `ROV`  | Remotely Operated Vehicle |
| `USV`  | Unmanned Surface Vehicle |
| `UUV`  | Unmanned Underwater Vehicle |

## Quick Start
**1. Client creates a mission** — API returns queued status.

**2. API sends commands to the vessel** — vessel starts executing mission.

**3. Vessel sends telemetry updates** — API relays them via webhooks.

**4. Mission completes or aborts** —  API notifies client via webhook

### Quick Start Examples

#### 1. Authenticate

```http
POST /v1/auth/token
Content-Type: application/json

{
  "client_id": "testing",
  "client_secret": "testing",
  "grant_type": "client_credentials"
}
```
**Response:**

```json
{
"access_token": "gfamedorn12998",
"expires_in": 120
}
```
**Use the token for subsequent requests:**

```http
Authorization: Bearer gfamedorn12998
```

#### 2. Create a Mission

```http
POST /v1/missions
Content-Type: application/json
Authorization: Bearer gfamedorn12998

{
  "mission_name": "Harbor Sweep Alpha",
  "priority": "routine",
  "vessel_id": "ASV-00012",
  "start_time": "2025-10-06T08:00:00Z",
  "end_time": "2025-10-06T12:00:00Z"
}
```

**Response:**

```json
{
 "mission_id": "MC-13023",
  "status": "queued"
}
```

#### 3. Subscribe to Webhooks

```http
POST /v1/webhooks
Content-Type: application/json
Authorization: Bearer gfamedorn12998

{
  "url": "https://myserver.com/webhook",
  "events": ["mission.status_changed", "vessel.alert"]
}
```

#### 4. Retrieve Telemetry

```http
GET /v1/vessels/ASV-00012/telemetry
Authorization: Bearer gfamedorn12998
```

**Response:**

```json
{
  "vessel_id": "ASV-00012",
  "position": {"lat": 27.9510, "lon": -82.4585},
  "speed_knots": 5.1,
  "battery_pct": 82
}
```


## Authentication & Authorization

Tokens are obtained via `POST /auth/token` using a valid `client_id` and `client_secret`. Each token allows up to **120 requests per minute** (burst requests allowed) and expires after **120 seconds**. After expiration, requests return a `403 Forbidden` response.

### Example: Obtain a Token

**Request:**
```json
POST /auth/token
{
  "client_id": "testing",
  "client_secret": "testing",
  "grant_type": "client_credentials"
}
```
**Response:**
```json
{
  "access_token": "gfamedorn12998",
  "expires_in": 120
}
```

All subsequent requests must include this token in the `Authorization` header:
```json
Authorization: Bearer gfamedorn12998
```

**Error Responses:**

`401 Unauthorized` — returned when the token is invalid or missing.

`403 Forbidden` — returned when the token has expired.


## Rate Limits & Versioning — Purpose
### Rate Limits:
Each authorization token allows up to **120 requests per minute**. Short bursts of up to **20 extra requests** are allowed.
### Versioning:
The Mission Control API is currently at ***v1***. All endpoints will include the version in the URL path, i.e., `/v1/missions`. Future versions may introduce new features or changes. Clients specifying an older version will continue to receive compatible responses until support is officially deprecated. 
*Note: there is no version before v1.*

### Example: Requesting a Mission with Versioning
```http
GET /v1/missions/0987
Authorization: Bearer gfamedorn12998
```
### Example: Rate Limit Exceeded
```
HTTP/1 429 Too Many Requests
Retry-After: 30
Content-Type: application/json

{
	"error": "Rate limit exceeded. Retry after 30 seconds."
}
```

## Core Endpoints — Purpose
This section documents the main API operations, grouped logically. Each endpoint includes the method, path, description, parameters, request and response examples, and common error codes.

#### Reference Enums:
##### - **Mission Priority:** See Appendix → Mission Priority Enum (`low`, `routine`, `high`).
##### - **Fail-Safe Modes:** See Appendix → Fail-Safe Modes (`return_to_base`, `hold_position`, `continue_autonomous`).
##### - **Vessel Types:** See Appendix → Vessel Types (`ASV`, `ROV`, `USV`, `UUV`).

### Missions:

The `missions` endpoint will allow the client to **create, launch, abort, list, and get details about specific missions**.

| Method | Path | Description |
|--------|------|------------|
| POST   | /v1/missions | Create a new mission |
| GET    | /v1/missions | List missions (filters: status, created_after, completed) |
| GET    | /v1/missions/{mission_id} | Get mission details |
| GET    | /v1/missions/{mission_id}/status | Retrieve current progress and health |
| PUT    | /v1/missions/{mission_id}/abort | Abort a mission |
| PATCH  | /v1/missions/{mission_id} | Update mission parameters before start |


**Mission Create (Request):**
```json
{
  "mission_name": "Harbor Sweep Alpha",
  "priority": "routine",             // see Mission Priority Enum
  "vessel_id": "ASV-00012",             // see Vessel Types
  "start_time": "2025-10-06T08:00:00Z",
  "end_time": "2025-10-06T12:00:00Z",
  "waypoints": [
    {"lat": 27.9506, "lon": -82.4572, "alt": 0},
    {"lat": 27.9512, "lon": -82.4601, "alt": 0}
  ],
  "behavior": { "mode": "autonomous", "fail_safe": "return_to_base" },     // see Fail-Safe Modes
  "metadata": {"customer_job_id": "CJ-555"}
}
```
**Response:**
```json
{
  "mission_id": "MC-13023",
  "status": "queued",
  "created_at": "2025-10-05T19:12:00Z",
  "links": {"self": "/missions/MC-13023"}
}
```
## Vessels & Telemetry

The `vessels` endpoint will allow the client to **receive lists, register, and receive details for a vessel**. The `telemetry` endpoint provides **live data for vessels**, including position, heading, speed, battery status, and sensor summaries.

| Method | Path | Description |
|--------|------|------------|
| GET   | /v1/vessels | List vessels and static properties |
| GET    | /v1/vessels/{vessel_id} | Get vessel details (hardware, firmware) |
| GET    | /v1/vessels/{vessel_id}/telemetry | Live telemetry snapshot |
| GET    | /v1/vessels/{vessel_id}/sensors/{sensor_id}/data | Sensor-specific feed (e.g., sonar, camera) |

### Example: Telemetry

```json
{
"vessel_id": "ASV-00012",      // see Vessel Types
  "timestamp": "2025-10-06T08:43:00Z",
  "position": {"lat": 27.9510, "lon": -82.4585},
  "heading_deg": 94.3,
  "speed_knots": 5.1,
  "battery_pct": 82,
  "health": { "comm_status": "ok", "cpu_load_pct": 24 }
}
```

## System Configuration

The `system` endpoint allows the client to **receive and update parameters**.
*Note: Updating parameters requires **admin role***

| Method | Path | Description |
|--------|------|------------|
| GET   | /v1/system/config | Retrieve system configuration |
| PUT    | /v1/system/config | Update system configuration (admin only) |

### Example: Configuration Schema

```json
{
  "max_speed_knots": 8.0,
  "comms_timeout_s": 30,
  "mission_concurrency_limit": 5,
  "default_fail_safe": "return_to_base"     // see Fail-Safe Modes
}
```

## Diagnostics & Logs

The `logs` endpoint allows clients to **retrieve system logs**, while the `events` endpoint provides **mission event timelines** (e.g.,: waypoint reached, errors)

| Method | Path | Description |
|--------|------|------------|
| GET   | /v1/vessels/{vessel_id}/logs?since=2025-10-05T14:00:00Z | Retrieve logs since a timestamp |
| GET    | /missions/{mission_id}/events | Retrieve mission event timeline |


### Example: Log Entry

```json
{
  "timestamp": "2025-10-05T14:00:00Z",
  "level": "WARN",
  "source": "nav_module",
  "message": "minor GPS drift detected",
  "context": {"hdop": 1.8}
}
```

## Webhooks

Also known as event push, the `webhooks` endpoint allows clients to register, list, and delete event subscriptions.
Events include: `mission.created`, `mission.status_changed`, `vessel.telemetry`, `vessel.log`
*Note: HMAC signature required in header `X-Signature` for authenticity*
`POST /webhooks` — register webhook URL and event types
Webhook payload includes: `event_type`, `timestamp`, `resource`, and `data`
| Method | Path | Description |
|--------|------|------------|
| POST  | /v1/webhooks | Register webhook URL and event types |


### Example: Webhook Payload

```json
{
  "event_type": "mission.status_changed",
  "timestamp": "2025-10-06T08:50:00Z",
  "resource": "/missions/MC-90812",
  "data": {"mission_id": "MC-90812", "status": "completed"}
}
```

## Standardized Error Response

Clients may encounter the following error codes:

| Code | Description |
|------|------------|
| 400  | Bad Request — invalid input |
| 401  | Unauthorized — invalid token |
| 403  | Forbidden — expired token |
| 404  | Not Found — resource missing |
| 409  | Conflict — conflict detected |
| 429  | Too Many Requests — rate limit exceeded |
| 500  | Internal Server Error — server-side error |

## Webhooks & Event Handling

### Webhook Event Types

Webhooks enable clients to receive event-driven notifications and real-time updates. Events are grouped into three categories: `mission`, `vessel`, and `system`/`fleet`.

#### Mission Events

| Event Type | Description |
|------------|------------|
| `mission.created` | A new mission has been created |
| `mission.updated` | Mission parameters have been updated |
| `mission.status_changed` | Mission status changed (queued → in-progress → completed → aborted) |
| `mission.aborted` | Mission was manually aborted |
| `mission.completed` | Mission finished successfully |

#### Vessel Events

| Event Type | Description |
|------------|------------|
| `vessel.registered` | A new vessel is added to the fleet |
| `vessel.updated` | Vessel configuration or metadata has changed |
| `vessel.telemetry` | Regular telemetry update (position, heading, speed, battery, health) |
| `vessel.sensor_data` | Sensor-specific data available (sonar, camera, LIDAR) |
| `vessel.alert` | Vessel health alert (battery low, comms error, hardware fault) |

#### System / Fleet Events

| Event Type | Description |
|------------|------------|
| `system.config_changed` | System configuration parameters were updated |
| `fleet.alert` | High-level fleet alert (multiple vessels reporting errors) |
| `log.entry` | New log entry written (includes severity, source, message) |

### Security Considerations

- **Authentication & Authorization:** Only clients with valid access tokens can register webhooks. Each webhook should have a **secret key** for HMAC verification to ensure security.
- **HMAC Signature Validation:** Every webhook payload includes an `X-Signature` header containing an HMAC hash of the payload using the **secret key**. This ensures the payload comes from the trusted API server and has not been tampered with. 
- **TLS/HTTPS:** Webhooks must be delivered over HTTPS (Hypertext Transfer Protocol Secure) to encrypt data in transit and ensure payload integrity.
- **Replay Protection:** Timestamps and unique `event_id` are included in payloads to prevent processing of old or duplicate events. Events older than a short threshold are rejected
- **Rate Limiting:** Clients are limited to a maximum number of webhook events per minute (e.g., 120 requests/min) to prevent abuse and ensure reliability. 

### Reliability Considerations

- **Retry Mechanisms:**The webhook system ensures events reach the client server reliably, even if something temporarily fails. If the client’s server returns a 5xx error code or times out, the API retries delivery (e.g.,: exponential backoff). 
- **Idempotency:** Each webhook has an unique `event_id`, so that clients can safely process retries without duplication. Even if an event is retired, the server can detect duplicates and process it only once.
- **Event Ordering:** Clients will register a webhook URL with the API, which pushes events to this URL as they occur (e.g.,, mission updates, vessel telemetry). Mission-critical events may arrive out of order, so clients should implement ordering logic if needed.
- **Monitoring & Logging:** API logs webhook delivery attempts and responses for troubleshooting and auditing purposes.

### Critical vs Optional Events

**Critical Events** — require immediate attention or trigger downstream actions:
- `mission.status_changed` — mission progress and safety-critical actions.
- `vessel.alert` — hardware faults, low battery, or comms errors.
- `mission.completed` — mission finished, triggers logging, reporting, or next steps.

**Optional Events** — useful and/or informational, but not time-sensitive.
- `mission.created` — a new mission was registered
- `mission.updated` — changes to mission parameters
- `vessel.telemetry` — routine telemetry updates (position, speed, battery)
- `vessel.sensor_data` — sensor-specific data for analysis purposes
- `log.entry` — general logging, warnings, or info messages.