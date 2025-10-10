# Mission Control API Reference Guide

## Overview

Programmatic control and monitoring of autonomous surface vessels (ASVs) and the fleet operations that go along with them, such as mission planning, telemetry, configuration, logging, and event webhooks, are made possible by the Mission Control API (v1).

The reliability and traceability requirements typical of autonomous systems are balanced with operational flexibility in the design of this API. Systems integrators, mission operators, and automation engineers who need to program or automate fleet behaviors are examples of typical users. Mission coordination (`create`, `launch`, and `abort`), live telemetry reporting, configuration management, and the retrieval of mission logs for analysis are examples of common use cases.

### Example: 	
`GET /vessels/{vessel_id}/logs?since=ISO8601` — retrieves logs of vessels which contain event timestamps, severity levels, sources, messages, and other relevant information.

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
**Use the token for future requests:**

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

Using a valid `client_id` and `client_secret`, tokens are acquired via `POST /auth/token`. Each token expires after 120 seconds and can receive up to 120 requests per minute (burst requests are permitted). Requests receive a `403 Forbidden` response after expiration.

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
A maximum of 120 requests per minute are permitted for each authorization token. Up to 20 additional requests in brief bursts are permitted.

### Versioning: 
V1 is the current version of the Mission Control API. Every endpoint will have the version in the URL path, such as `/v1/missions`. New features or modifications might be added in later iterations. Until support is formally deprecated, clients who specify an older version will continue to receive compatible responses. 
*Note: v1 is the only version available.*

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
## Core Endpoints — Objective
The primary API operations are logically grouped and documented in this section. The method, path, description, parameters, sample requests and responses, and common error codes are all included in each endpoint.

#### Reference Enums: 
##### - **Mission Priority:** Refer to Appendix → Mission Priority Enum (`low`, `routine`, `high`).
##### - **Fail-Safe Modes:** Refer to Appendix (`return_to_base`, `hold_position`, `continue_autonomous`).
##### - **Vessel Types:** Refer to Appendix → Vessel Types (`ASV`, `ROV`, `USV`, `UUV`).

### Missions:

The client can **create, launch, abort, list, and get details about specific missions** using the `missions` endpoint.

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
  "priority": "routine"
  "vessel_id": "ASV-00012",
  "start_time": "2025-10-06T08:00:00Z",
  "end_time": "2025-10-06T12:00:00Z",
  "waypoints": [
    {"lat": 27.9506, "lon": -82.4572, "alt": 0},
    {"lat": 27.9512, "lon": -82.4601, "alt": 0}
  ],
  "behavior": { "mode": "autonomous", "fail_safe": "return_to_base" },
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

The client will be able to **receive lists, register, and receive details for a vessel** through the `vessels` endpoint. Position, heading, speed, battery life, and sensor summaries are among the **live data for vessels** that are provided by the `telemetry` endpoint.

| Method | Path | Description |
|--------|------|------------|
| GET   | /v1/vessels | List vessels and static properties |
| GET    | /v1/vessels/{vessel_id} | Get vessel details (hardware, firmware) |
| GET    | /v1/vessels/{vessel_id}/telemetry | Live telemetry snapshot |
| GET    | /v1/vessels/{vessel_id}/sensors/{sensor_id}/data | Sensor-specific feed (e.g., sonar, camera) |

### Example: Telemetry

```json
{
"vessel_id": "ASV-00012",
  "timestamp": "2025-10-06T08:43:00Z",
  "position": {"lat": 27.9510, "lon": -82.4585},
  "heading_deg": 94.3,
  "speed_knots": 5.1,
  "battery_pct": 82,
  "health": { "comm_status": "ok", "cpu_load_pct": 24 }
}
```

## System Configuration

The client can **receive and update parameters** via the `system` endpoint. *Note: **admin role** is required to update parameters.*

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
  "default_fail_safe": "return_to_base"
}
```

## Logs & Diagnostics

While the `events` endpoint offers **mission event timelines** (such as waypoint reached, errors), the `logs` endpoint enables clients to **retrieve system logs**.

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

The `webhooks` endpoint, also referred to as event push, enables clients to create, list, and remove event subscriptions.
`mission.created`, `mission.status_changed`, `vessel.telemetry`, and `vessel.log` are examples of events.
*Note: HMAC signature required in header `X-Signature` for authenticity*
`POST /webhooks` — register event types and webhook URLs
Included in the webhook payload are `event_type`, `timestamp`, `resource`, and `data`.
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

### Types of Webhook Events

Clients can get real-time updates and event-driven notifications thanks to webhooks. Three categories are used to classify events: `mission`, `vessel`, and `system`/`fleet`.

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

- **Authentication & Authorization:** Only clients with valid access tokens are able to register for webhooks. Each webhook requires a secret key for HMAC verification in order to ensure security.
- **Verification of HMAC Signatures:** An HMAC hash of the payload using the secret key is contained in the `X-Signature` header of every webhook payload. This ensures that the payload is legitimate and comes from the trustworthy API server. 
- **TLS/HTTPS:** Webhooks must be delivered via HTTPS (Hypertext Transfer Protocol Secure) in order to encrypt data in transit and ensure payload integrity.
- **Protection for Replay:** Payloads avoid processing duplicate or out-of-date events by including timestamps and a unique `event_id`. Events are disqualified if they surpass a short threshold.
- **Rate Limiting:** Clients are limited to a maximum number of webhook events per minute (e.g., 120 requests/min) to prevent abuse and ensure reliability.

### Considerations for Reliability

- **Retry Mechanisms:** Even in the event of a brief failure, the webhook system guarantees that events consistently reach the client server. The API tries delivery again if the client's server times out or returns a 5xx error code (e.g., exponential backoff). 
- **Idempotency:** Because every webhook has a distinct `event_id`, clients can process retries safely and without duplication. The server can identify duplicates and process an event only once, even if it has been retired.
- **Event Ordering:** The API allows clients to register a webhook URL, which is then used to push events (such as mission updates and vessel telemetry) to this URL as they happen. Clients should use ordering logic if necessary because mission-critical events might arrive out of order.
- **Monitoring & Logging:** For the purposes of troubleshooting and auditing, the API records webhook delivery attempts and responses.

### Important vs. Optional Occasions

**Critical Events** — need to be addressed right away or cause subsequent actions:
- `mission.status_changed` — safety-critical actions and mission progress.
- `vessel.alert` — communications errors, low battery, or hardware issues.
- `mission.completed` — mission completed, initiates reporting, logging, or subsequent actions.

Events That Are Optional — not urgent, but helpful and/or instructive.
- `mission.created` — a new mission was registered - `mission.updated` — modifications to the mission's parameters
- `vessel.telemetry` — regular telemetry updates (battery, position, and speed)
- `vessel.sensor_data` — sensor-specific information for analysis
- `log.entry` — info messages, warnings, or general logging.


