# Autonomous Fleet Coordination System (AFCS) — System Overview & Integration Guide

## Overview

Unmanned surface vehicle (USV) and unmanned underwater vehicle (UUV) fleets are managed and observed by the cloud-based Autonomous Fleet Coordination System (AFCS). Technical integrators, system administrators, and developers can connect their own mission control or data systems with AFCS, which provides modularity, scalability, and security while facilitating coordination and unified automation across various interfaces.

The AFCS architecture will be described in this guide, along with how its services work together and how to integrate with APIs for real-time alerts, telemetry data access, and mission scheduling.

## Overview of the System

Control and monitoring of autonomous vehicles are distributed by the Autonomous Fleet Coordination System (AFCS), which uses a modular, service-oriented architecture. Mission management, vessel telemetry, event processing, and operator interfaces are all handled by the system's networked services.

```mermaid
graph TD
    subgraph Client Layer
        A1[Operator Console]
        A2[Mission Control UI]
        A3[External Integrations]
    end

    subgraph API Layer
        B1[AFCS API Gateway]
        B2[Authentication Service]
        B3[Rate Limiter]
    end

    subgraph Core System
        C1[Mission Management Service]
        C2[Vessel Command Service]
        C3[Telemetry Processor]
        C4[Logging & Monitoring Framework]
    end

    subgraph Data & Storage
        D1[Mission Database]
        D2[Telemetry Storage]
        D3[Audit Log Repository]
    end

    subgraph External Systems
        E1[ELK Stack / DataDog]
        E2[Secure Key Vault]
    end

    %% Connections
    A1 --> B1
    A2 --> B1
    A3 --> B1

    B1 --> B2
    B1 --> B3
    B1 --> C1
    B1 --> C2
    C2 --> C3
    C1 --> D1
    C3 --> D2
    C1 --> C4
    C3 --> C4
    C4 --> D3
    C4 --> E1
    B2 --> E2
```

### Core Services: 
- Mission Control Service — manages the scheduling, coordination, and lifecycle of missions.
- Telemetry Service — gathers, normalizes, and transmits real-time vessel telemetry data, such as position, battery level, and health status.
- Fleet Gateway — secures a communication link between ships and cloud infrastructure.
- Event Processing Service — controls the distribution of webhooks to external systems, logs, and alerts.
- User Access & Security Service — using roles and tokens to control access, authorization, and authentication.

All services communicate via message queues and HTTPS, ensuring dependable and asynchronous data exchange.

### Data Flow Summary:

```mermaid
sequenceDiagram
    participant Operator
    participant API as AFCS API **Gateway**
    participant Auth as Authentication Service
    participant Mission as Mission Management
    participant Vessel as Vessel Command
    participant Telemetry as Telemetry Processor
    participant Log as Logging Framework

    Operator->>API: Send mission command (HTTPS)
    API->>Auth: Validate token (HMAC / TLS 1.3)
    Auth-->>API: Token valid
    API->>Mission: Forward command for processing
    Mission->>Vessel: Dispatch mission command
    Vessel-->>Telemetry: Send telemetry updates
    Telemetry-->>Mission: Confirm execution status
    Mission-->>Operator: Return mission status (JSON response)
    API-->>Operator: Deliver response
    Mission->>Log: Write mission event log
    Telemetry->>Log: Record telemetry data
    Log-->>External: Stream metrics to ELK/DataDog
```

Operators or external systems issue mission commands through the API
Commands are routed through the Fleet **Gateway** to individual vessels
Vessels return telemetry and status data to the Telemetry Service
Event Processing Service distributes logs and alerts to subscribed clients
The Mission Control Service dashboard updates in real time as missions progress

## Core Components

To guarantee complete fleet management and integration capabilities, the AFCS is made up of a number of essential parts, each of which is in charge of carrying out particular sets of tasks while coordinating with the others to keep the system cohesive.

### Helix Core API

The creation of missions, telemetry retrieval, and endpoint authentication are the primary functions of the **Helix Core API's**. With support for REST interfaces and SDKs for various programming languages, it offers programmatic control and management of missions and vessels. In order to coordinate mission logic and guarantee that other services can access live vessel data, it works closely with the **Helix Orchestrator**.

**Inputs and Outputs:**

- Inputs — Mission creation requests (e.g., mission parameters, vessel IDs, priorities), authentication tokens, telemetry subscription requests.

- Outputs — Mission status updates, telemetry data (e.g., position, battery level, health status), authentication tokens, error responses.

### Example — Mission Creation

**Request:**

```http
POST /v1/missions
Content-Type: application/json
Authorization: Bearer <token>

{
  "mission_name": "Harbor Sweep Alpha",
  "priority": "routine",
  "vessel_id": "USV-001",
  "start_time": "2025-10-06T08:00:00Z",
  "end_time": "2025-10-06T12:00:00Z"
}
```

**Response:**

```json
{
  "mission_id": "MC-13023",
  "status": "queued",
  "created_at": "2025-10-05T19:12:00Z"
}
```

### Helix Orchestrator

Decision-making, mission logic, and fail-safe procedures are managed by the **Helix Orchestrator**. It decides how missions are carried out, controls priority handling (such as mission priority levels, conflict resolution, dynamic re-prioritization, and resource allocation), and makes sure that autonomous cars react correctly to errors or changing circumstances. Through internal APIs, the **Orchestrator** interacts with the other parts. It mainly uses the **Helix Core API** for mission management and the **Helix Stream** for real-time telemetry and event updates.

**Inputs and Outputs:**

- Inputs — Mission instructions (e.g., mission ID, vessel assignments, mission parameters, priority levels), real-time telemetry updates, external system triggers, fail-safe signals.

- Outputs — Execution status updates, mission completion or failure notifications, dynamic mission re-prioritization results, fail-safe trigger alerts.

### Example — Mission Prioritization Update

**Request:**

```http
PATCH /v1/missions/MC-13023/priority
Content-Type: application/json
Authorization: Bearer <token>

{
  "priority": "high",
  "reason": "Collision risk detected in planned path"
}
```

**Response:**

```json
{
  "mission_id": "MC-13023",
  "priority": "high",
  "status": "in-progress",
  "updated_at": "2025-10-06T09:15:00Z",
  "notes": "Priority escalated due to potential collision risk"
}
```

### Example — Telemetry-Based Fail-Safe Trigger

**Request:**

```http
POST /v1/orchestrator/failsafe
Content-Type: application/json
Authorization: Bearer <token>

{
  "mission_id": "MC-13023",
  "vessel_id": "USV-001",
  "event": "battery_low",
  "timestamp": "2025-10-06T10:15:00Z"
}
```

**Response:**

```json
{
  "mission_id": "MC-13023",
  "vessel_id": "USV-001",
  "action_taken": "return_to_base",
  "status": "in-progress",
  "timestamp": "2025-10-06T10:15:02Z"
}
```

### Helix Stream

The **Helix Stream** handles event subscriptions, supports message queues, and updates in real time. It is in charge of providing clients with streaming telemetry, events, and status updates while guaranteeing dependable, well-organized event delivery. The **Helix Stream** maintains a constant data flow between the **Helix Core API**, **Orchestrator**, and linked clients by interacting with the other components through WebSockets, message queues, and webhook subscriptions.

**Inputs and Outputs:**

- Inputs — Telemetry and event streams (e.g., position, battery level, mission updates), webhook registration requests, subscription filters, and authentication tokens.

- Outputs — Real-time event feeds, mission and telemetry updates, webhook event notifications, and delivery status acknowledgments.

### Example — Telemetry Stream Subscription

**Request:**

```http
POST /v1/stream/subscribe
Content-Type: application/json
Authorization: Bearer <token>

{
  "vessel_id": "USV-001",
  "event_types": ["telemetry", "mission_status"],
  "delivery_method": "websocket"
}
```

**Response:**

```json
{
  "subscription_id": "SUB-20482",
  "vessel_id": "USV-001",
  "status": "active",
  "connected_at": "2025-10-06T10:25:00Z"
}
```

### Example — Webhook Event Delivery

**Response:**

```json
{
  "event_type": "mission_status",
  "mission_id": "MC-13023",
  "status": "completed",
  "timestamp": "2025-10-06T12:45:30Z"
}
```

### Helix Secure Gateway

Token management, rate limiting, and encryption are handled by the **Helix Secure Gateway**. It acts as a secure connection between AFCS components and external clients, safeguarding system integrity and maintaining consistent access control across services. The **Gateway** uses HTTPS endpoints to connect to the **Orchestrator** and **Core API** in order to enable smooth component integration.

**Inputs and Outputs:**

- Inputs — Authentication requests (e.g., login credentials, API tokens), internal access verification requests, encryption key exchanges, and rate-limit checks.

- Outputs — Access tokens, session validation responses, encrypted communication acknowledgements, and rate-limit status/error responses (e.g., `429 Too Many Requests`).

### Example — Token Generation

**Request:**

```http
POST /v1/auth/token
Content-Type: application/json

{
  "client_id": "afcs_admin",
  "client_secret": "secure_key_84ds8f",
  "grant_type": "client_credentials"
}
```

**Response:**

```json
{
  "access_token": "gmrglwfwf06060245",
  "token_type": "Bearer",
  "expires_in": 3600,
  "issued_at": "2025-10-06T10:45:00Z"
}
```


### Helix SDKs

The **Helix SDKs** allow for client-facing tools for easy integration. It is responsible for providing libraries for supported programming languages (e.g., Python, Java, JavaScript), simplifying authentication, mission commands, and telemetry output, while ensuring consistent developer experience and API usage.

**Inputs and Outputs:**

- Inputs — SDK method calls (e.g., `createmission()`, `getTelemetry()`, `authenticate()`) configuration parameters, and credentials.

- Outputs — Mission creation results, telemetry data objects, authentication tokens, and standardized error responses.

### Example — Mission Creation (Python SDK)

**Request:**

```python
from helix_sdk import HelixClient

client = HelixClient(client_id="afcs_admin", client_secret="secure_key_84ds8f")

mission = client.create_mission(
    mission_name="Harbor Sweep Alpha",
    vessel_id="USV-001",
    priority="routine",
    start_time="2025-10-06T08:00:00Z",
    end_time="2025-10-06T12:00:00Z"
)

print(mission.status)
```

**Response:**

```json
{
  "mission_id": "MC-13023",
  "status": "queued",
  "created_at": "2025-10-05T19:12:00Z"
}
```

## System Integration and Communication Workflow

**Requirements:**

- Client information for the **Helix Secure Gateway** (e.g.,`client_id` and `client_secret`).
- HTTPS-based network access to the API base.
- Safekeeping of tokens and secrets.
- Client-server clock synchronization is crucial for replay protection.

### Step 1: Authentication

1. All API systems start with authentication, in which the client needs to obtain a token in order to proceed with submitting requests. `401 Unauthorized` or `403 Forbidden` errors will be returned for requests that do not contain a valid token.
2. The client uses `client_id` and `client_secret` to request a token from the **Gateway** (`POST /v1/auth/token`).
3. The Bearer token (`access_token`) with `expires_in` (120 seconds) is returned by the **Gateway**. The client appends the Bearer token (`Authorization: Bearer <token>`) to any further requests.
Clients must re-authenticate after the tokens' 120-second expiration.

### Example — Token Generation

**Request:**

```http
POST /v1/auth/token
Content-Type: application/json

{
  "client_id": "afcs_admin",
  "client_secret": "secure_key_84ds8f",
  "grant_type": "client_credentials"
}
```

**Response:**

```json
{
  "access_token": "gmrglwfwf06060245",
  "token_type": "Bearer",
  "expires_in": 120,
  "issued_at": "2025-10-06T10:45:00Z"
}
```

**Error Handling Notes:**

- `401 Unauthorized` — Invalid `client_id` or `client_secret`.
- `403 Forbidden` — Token scope insufficient for requested resource.
- `429 Too Many Requests` — Client exceeded rate limits.
- Tokens expire after 120 seconds — expired tokens must be refreshed with a new authentication request.

### Step 2. Mission Creation

Missions are created using the **Helix Core API**. The **Core API** is required to initiate fleet operations and is used to create and schedule autonomous vehicle missions.

1. The client sends mission parameters to the API using a valid Bearer token (`Authorization: Bearer <token>`).
2. The **Core API** communicates with the **Orchestrator**, schedules the mission, and checks the input.
3. A mission ID (`mission_id`) and status (`status`) are returned for tracking and updates.

### Example — Mission Creation

**Request:**

```http
POST /v1/missions
Content-Type: application/json
Authorization: Bearer gmrglwfwf06060245

{
"mission_name": "The Last Jedi",
"priority": "routine",
"vessel_id": "UUV-013",
"start_time": "2025-10-06T08:00:00Z",
"end_time": "2025-10-06T12:00:00Z"
}
```

**Response:**

```json
{
  "mission_id": "MC-17269",
  "status": "queued",
  "created_at": "2025-10-06T08:00:00Z"
}
```

**Error Handling Notes:**

- `401 Unauthorized` — The token is missing or invalid.
- `400 Bad Request` — Required fields are missing or incorrect.
- `404 Not Found` — The specific vessel does not exist.

### Example — `409 Conflict` Error

**Request:**

```http
POST /v1/missions
Content-Type: application/json
Authorization: Bearer gmrglwfwf06060245

{
  "mission_name": "Deep Sweep Beta",
  "priority": "routine",
  "vessel_id": "UUV-013",
  "start_time": "2025-10-06T08:00:00Z",
  "end_time": "2025-10-06T12:00:00Z"
}
```

**Response:**

```json
{
  "error": {
    "code": 409,
    "message": "Vessel UUV-013 is already assigned to an active mission (MC-17269).",
    "timestamp": "2025-10-06T08:01:45Z"
  }
}
```

### Step 3: Retrieving Telemetry

In order to provide the client with real-time operational awareness and mission safety, telemetry retrieval is a crucial procedure. Valid authentication, active vessel connections, and synchronized data channels are necessary for telemetry retrieval, which uses **Helix Stream** for live data and **Helix Core API** for on-demand pulls. This ensures accurate and timely updates during mission execution.

1. The client uses `/v1/stream/subscribe` to subscribe to telemetry or `/v1/telemetry/{vessel_id}` to request summary data.
2. The token is checked by the system to verify the request.
3. Structured JSON data (e.g., position, velocity, battery level) is streamed or returned.

### Example — Telemetry Stream Subscription

**Request:**

```http
POST /v1/stream/subscribe
Content-Type: application/json
Authorization: Bearer gmrglwfwf06060245

{
  "vessel_id": "UUV-013",
  "event_types": ["telemetry", "mission_status"],
  "delivery_method": "websocket"
}
```

**Response:**

```json

{
  "subscription_id": "SUB-30651",
  "vessel_id": "UUV-013",
  "status": "active",
  "connected_at": "2025-10-06T08:15:00Z"
}
```

**Error Handling Notes:**

- `401 Unauthorized` — Token invalid or expired.
- `404 Not Found` — Vessel not connected or does not exist.
- `409 Conflict` — Telemetry stream already active for vessels.
- `503 Service Unavailable` — Stream temporarily down or data channel unstable.

### Step 4. Event & Status Updates

By informing the client of changes to the mission state (e.g., launched, paused, completed, error), event and status updates guarantee operational transparency. These updates may originate from the **Helix Orchestrator** or the **Helix Stream**, depending on the setup.

1. The client subscribes to system or mission event channels (e.g., `/v1/events/subscribe`).
2. The system confirms the active mission context and authentication.
3. Depending on the delivery method (e.g., WebSocket, REST), events are either received in batches or streamed live.

Example event types: `mission_status`. `vessel_health`, `system_health`.

### Example — Mission Event Pull

**Request:**

```http
GET /v1/events/mission/MC-17269
Content-Type: application/json
Authorization: Bearer gmrglwfwf06060245
```

**Response:**

```json
{
  "mission_id": "MC-17269",
  "event_type": "mission_status",
  "status": "in_progress",
  "timestamp": "2025-10-06T09:45:00Z",
  "details": {
    "vessel_id": "UUV-013",
    "progress": 42,
    "battery_level": 83
  }
}
```

**Error Handling Notes:**

- `401 Unauthorized` — Token invalid or expired.
- `404 Not Found` — Mission or vessel not found.
- `409 Conflict` — Event stream already active or duplicate subscription request.
- `503 Service Unavailable` — **Orchestrator** or event stream temporarily offline.

### Step 5. Command Execution

Clients can send operational directives to the vessel through command execution, such as direct control or mission adjustment commands. This step, which guarantees operational safety and real-time responsiveness, can only take place following authentication and mission activation.

1. The client uses the **Helix Core API** to issue commands (e.g., `pause_mission`, `resume`, `abort`, `reroute`).
2. To guarantee safety and permission compliance, the **Orchestrator** verifies the command.
3. Using **Helix Stream**, the system broadcasts the action to the designated vessel.
4. The client receives a confirmation or error message back.

### Example — Mission Command

**Request:**

```http

POST /v1/missions/MC-17269/command
Content-Type: application/json
Authorization: Bearer gmrglwfwf06060245

{
  "action": "pause",
  "reason": "collision_risk"
}
```

**Response:**

```json
{
  "mission_id": "MC-17269",
  "status": "paused",
  "timestamp": "2025-10-06T09:22:00Z",
  "message": "Mission paused successfully due to collision risk."
}
```

**Error Handling Notes:**

- `401 Unauthorized` — Token invalid or expired.
- `403 Forbidden` — Command not permitted for this mission state.
- `404 Not Found` — Mission or vessel does not exist.
- `409 Conflict` — Another command is already in progress.
- `503 Service Unavailable` — **Orchestrator** temporarily unreachable.


### Step 6. Mission Completion & Data Archival

Mission completion and data archival signifies the end of a mission’s lifecycle. When the mission criteria are met (e.g., final waypoint reached, operator termination signal), the operations end. Mission data, telemetry logs, and event streams are closed and archived through the **Core API**, all data becoming read-only. Archival ensures long-time traceability, data integrity, and regulatory compliance.

1. The **Orchestrator** signals mission completion.
2. The **Core API** finalizes mission state (e.g., `status: completed).
3. **Helix Stream** terminates active telemetry and event feeds.
4. Mission data (e.g., telemetry, logs, and metadata) are archived and accessible for post-mission analysis.

### Example — Mission Completion Acknowledgment

**Response:**

```http
PATCH /v1/missions/MC-17269/status
Content-Type: application/json
Authorization: Bearer gmrglwfwf06060245

{
  "status": "completed",
  "completed_at": "2025-10-06T12:45:00Z"
}
```

**Response:**

```json
{
  "mission_id": "MC-17269",
  "status": "completed",
  "archived": true,
  "archived_at": "2025-10-07T12:50:00Z"
}
```

**Error Handling Notes:**

- `401 Unauthorized` — Token expired or invalid.
- `404 Not Found` — Mission not found or already archived.
- `409 Conflict` — Mission is still active; cannot archive.
- `500 Internal Server Error` — Archival service failed or data corruption detected.

### Workflow Summary

For autonomous vehicles, the system workflow—from authentication to archiving—shows a full operational cycle. The three core components—the **Helix Core API**, **Helix Stream**, and **Helix Orchestrator**—work together to maintain continuity, coordination, and safety throughout each mission. This structure supports scalable fleet operations and system reliability across multiple vessels. Once archived, mission data is securely stored and made read-only, ensuring it is traceable and meets compliance regulations. The archived data remains accessible for analysis, tracking, reporting, and playback via the Core API. For a complete list of available commands, reference the Glossary.


## Authentication & Security

To gain access to the AFCS and ensure system integrity, clients must receive an access token, which allows authentication into the fleet control platform. This process also guarantees encrypted communication and message validation while navigating with the core components.

### Authentication Overview

Authentication is necessary to prevent unauthorized access and protect mission and vessel data. While several authentication processes exist (e.g., OAuth2, API keys), the AFCS uses a custom token-based approach that expires after a set duration.

Through the **Helix Secure Gateway**, the client will request a token using their credentials. The **Gateway** will validate their identification and respond with a Bearer token and expiration time. Every request must contain this token to access the API and other system components.

#### Permission Tiers

- Administrator — Full access to all endpoints, mission management, user administration and overall system settings.
- Operator (Mission Controller) — can create, update, and manage missions for assigned vessels; can issue commands but cannot modify users or system settings
- Observer (Analyst) — read-only access to telemetry, mission logs, and archived data; cannot create or alter missions.

The AFCS uses permission tiers to control access. Each token carries role-based features, which reflect these tiers. Requests outside of the role’s scope are rejected with a `403 Forbidden` error, ensuring secure, role-appropriate access.

### Secure Communication & Token Usage

- HTTPS — To guarantee encryption while in transit, all requests to AFCS components must use HTTPS.
- HMAC (Message Validation) — To ensure message authenticity and guard against manipulation, some endpoints (like Helix Stream event delivery) need an HMAC signature.
- Rate Limits — To guard against misuse, the **Helix Secure Gateway** imposes request limits for each client. A `429 Too Many Requests` response is returned when the limits are exceeded.
- Token Expiration — To lessen the possibility of replay attacks, AFCS tokens are only valid for a period of 120 seconds. After a token expires, clients need to request a new one and re-authenticate.

### Example — Token Request

**Request:**

```http
POST /v1/auth/token
Content-Type: application/json

{
  "client_id": "afcs_admin",
  "client_secret": "secure_key_dsx360ps3",
  "grant_type": "client_credentials"
}
```

**Response:**

```json
{
  "access_token": "gmrglwfwf06060245",
  "token_type": "Bearer",
  "expires_in": 120,
  "issued_at": "2025-10-06T10:45:00Z"
}
```


**Error Handling Notes:**

`401 Unauthorized` — Invalid `client_id` or `client_secret`.
`403 Forbidden` — Token scope insufficient for requested resource.
`429 Too Many Requests` — Rate limit exceeded..


## Best Practices
These recommendations help developers integrate with the AFCS efficiently, securely, and reliably. Following these practices will reduce errors, enhance stability, and secure operation across missions and system components.

### Reliability & Retry Strategies
- Retry Logic — Use exponential backoff when retrying failed attempts to avoid overwhelming the API or network.
- Error Handling — Fix common HTTP errors (e.g., `401`, `403`, `429`, `500`) by implementing automated recovery paths to maintain system stability.
- Version Compatibility — Verify integrations against compatibility with the current AFCS API version, with endpoints and payloads validated before production.

### Request Management
- Batch Requests — Increase productivity and lower network overhead, combine several operations into a single API request.
- Rate-Limiting Awareness — Prevent throttling or service interruption, abide by API rate limits.
- Idempotency —  Block inadvertent duplication of actions, use distinct mission patterns with mission-critical commands.

### Security Hygiene
- Secure Key Storage — Store client credentials and API keys in encrypted vaults or environment variables, but never in source code or logs.
- Key Rotation — Rotate secrets and tokens periodically to lower the risk of compromise.
- Webhook & Message Verification — Validate message signatures (e.g., HMAC) to confirm authenticity and avoid tampering.

### Logging & Monitoring
- Structured Logging — Record key fields (e.g, `timestamp`, `mission_id`, `vessel_id`, `event`, `message`) for easy analysis.
- Centralized Monitoring — Forward logs to dashboards (e.g., ELK Stack, DataDog) for real-time data insight.
- Correlation & Filtering — Filter logs by vessel, mission, severity, and time to allow efficient troubleshooting.

## Appendix/Glossary

### A. Common API Endpoints

| Endpoint | Definition |
|--------|------|
| `/missions/start` | Launches a mission for a specified vessel |
| `missions/status` | Retrieves mission status updates |
| `vessels/{id}` | Responds with details for a specific vessel |


### B. Error Code Reference

| Error Code | Definition |
|--------|------|
| `400 Bad Request` | Invalid request format |
| `401 Unauthorized` | Missing or invalid authentication token |
| `403 Forbidden` | Insufficient permissions |
| `404 Not Found` | Resource unavailable |
| `429 Too Many Requests` | Rate limit exceeded |
| `500 Internal Server Error` | Unexpected server issue |


### C. Key Field Definitions

| Key Field | Definition |
|--------|------|
| `mission_id` | Unique mission identifier |
| `vessel_id` | Unique vessel identifier linked to mission operations |
| `timestamp` | UTC record of the event |
| `event` | System event type (e.g., `mission_started`, `command_rejected`) |
| `message` | Human-readable description of the log entry or error |


### D. Glossary

| Term | Definition |
|--------|------|
| AFCS | Autonomous Fleet Control System. The central platform that coordinates, monitors, and manages missions across connected autonomous vessels |
| API | Application Programming Interface. A defined set of endpoints and data structures used to communicate with the AFCS programmatically |
| Checksum | A calculated hash value used to verify data integrity and detect corruption or tampering |
| HMAC | Hash-based Message Authentication Code. A cryptographic signature method that verifies message authenticity and integrity between communicating systems |
| HTTPS | Hypertext Transfer Protocol Secure. The secure version of HTTP that uses encryption (typically TLS) to protect data in transit |
| Rate Limit | A defined threshold on the number of requests a client can make within a time window, preventing overload or abuse of the API |
| TLS 1.3 | Transport Layer Security version 1.3. The encryption protocol used by the AFCS to secure communications between components and clients |
| UTC | Coordinated Universal Time. The standardized global time reference used for all AFCS timestamps |
| Webhook | An automated HTTP callback that allows the AFCS to send event notifications to external systems in real time |
| Payload | The body content of a request or response, typically containing mission data, parameters, or configuration information |

