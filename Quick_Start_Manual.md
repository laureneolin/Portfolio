# Autonomous Fleet Coordination System API Quick Start Guide

The Autonomous Fleet Coordination System (AFCS) API Quick Start Guide explains how to send your first mission command and establish a connection to the AFCS API. You will have mastered the fundamentals of this system by the end, including how to authenticate, obtain an access token, and submit a simple request.

### Prerequisites:
- An AFCS developer account; the client ID and secret that you were given for the API
- Possession of a REST client (such as curl or Postman)
- A fundamental knowledge of JSON and REST APIs.

>[!NOTE]
>Token handling, authentication, and a test mission request are the main topics of this guide. Consult the *[System Overview and Integration Guide](https://github.com/laureneolin/portfolio/blob/Technical-Writing/System_Overview_Integration_Guide.md)* for a more thorough explanation of configuration and integration.

**Estimated Time: About 15-30 minutes**     

## Authenticate and Authorize Credentials

Everyone must authenticate their client and request an access token before they can access the AFCS API. You will be guided step-by-step through that process in this section.

### Step 1: Get Your API Login Information

Your system administrator or the Developer Portal will provide you with a Client ID and Client Secret when your AFCS account is created. 

- **Client ID — Identifies you to the AFCS**

- **Client Secret — Authenticates and authorizes access to the API**

>[!NOTE]
>Keep these credentials safe. They should never be kept in a format that only you can access, such as plain text, publicly shared, or version controlled.

### Step 2. Request an Access Token

The AFCS authenticates users using OAuth 2.0 client credentials. You will receieve a bearer token — not an API key — because AFCS implements token-based security for safer mission control. To request a token, send a `POST` request to the application's `/auth/token` endpoint with your Client ID and Secret.

**Example**

Request:
```http
POST https://api.afcs.com/auth/token
Content-Type: application/json

{
  "client_id": "your-client-id",
  "client_secret": "your-client-secret"
}
```

Response:
```json
{
  "access_token": "eor4nuy1309",
  "token_type": "Bearer",
  "expires_in": 3600
}
```
>[!NOTE]
>Tokens are temporary and will become invalid after a set period of time. The `expires_in` field indicates how long the token is valid before having to get a new one.

**Try It Out**
```bash
curl -X POST "https://api.afcs.com/auth/token" 
  -H "Content-Type: application/json" 
  -d '{
    "client_id": "your-client-id",
    "client_secret": "your-client-secret"
  }'
```

>[!TIP]
>Replace `"your-client-id"` with your Client ID & `"your-client-secret"` with your Client Secret that were provided to you by your System Administrator.

>[!NOTE]
>`-d` signals to include data to post and sends the JSON payload in the request body

### Step 3. Secure the Token Securely

Store the returned `access_token` in a secure location (e.g., environment variable, encrypted key vault). This will be required to be added in every request header.

**Example**

Header Format:
```http
Authorization: Bearer eor4nuy1309
```

**Try It Out**
```bash
curl -X GET "https://api.afcs.com/v1/missions/status?mission_id=MC-17269"
  -H "Authorization: Bearer eor4nuy1309"
```
>[!NOTE]
>- `-X GET` explicitly defines the HTTP method, but is optional, since `GET` is the default for curl.
>- Quotes around the URL prevent shell interpretation issues.
> `-H` submits your header.

### Step 4. Verify Authentication

You can send a simple request to the `/status` endpoint to verify that your token is valid.

**Example**
```http
GET https://api.afcs.com/status
Authorization: Bearer eor4nuy1309
```
##### **Possible Responses:**
- `200 OK` — Token is valid and API access is confirmed.
- `401 Unauthorized` — Token is missing, invalid, or expired

**Try It Out**
```bash
curl -X GET "https://api.afcs.com/status"
  -H "Authorization: Bearer eor4nuy1309"
```

>[!TIP]
>If your token expires, you must re-authenticate yourself using your Client ID and Secret to access a new token.

## Issue First Request

Once you have a valid `access_token`, you can start using the AFCS API. You will learn how to issue a basic command to a vessel in this section.

### Step 1. Choose Your Target Vessel

Decide which vessel you wish to command. Each vessel is assigned a unique `vessel_id`. To get a list of the vessels you can access, send a `GET` request to the `/vessels` endpoint.

**Example**

Request:
```http
GET https://api.afcs.com/v1/vessels
Authorization: Bearer eor4nuy1309
```

Response:
```json
[
  {
    "vessel_id": "UUV-013",
    "name": "Explorer-13",
    "status": "idle"
  },
  {
    "vessel_id": "UUV-014",
    "name": "Pathfinder-7",
    "status": "active"
  }
]
```

**Try It Out**
```bash
curl -X GET "https://api.afcs.com/v1/vessels"
  -H "Authorization: Bearer eor4nuy1309"
```
>[!TIP]
>Use the `vessel_id` from this response in all related command requests.

### Step 2. Create Your Command Request

Commands are sent via the `/missions/command` endpoint using a `POST` request. Include your access token in the header and the target vessel ID with the command in the request body.

**Example**

Request:
```http
POST https://api.afcs.com/v1/missions/command
Content-Type: application/json
Authorization: Bearer eor4nuy1309

{
  "vessel_id": "UUV-013",
  "command": "start_mission",
  "mission_id": "MC-17269"
}
```

### Step 3. Translate the Response

A successful request returns a `200 OK` response with details about the command status.

**Example**

Response:
```json
{
  "mission_id": "MC-17269",
  "vessel_id": "UUV-013",
  "status": "accepted",
  "timestamp": "2025-10-07T13:00:00Z"
}
```

##### **Common Errors:**
- `403 Forbidden` — Your token does not have permission to access this vessel.
- `429 Too Many Requests` — You are sending commands too quickly; respect the rate limits
- `500 Internal Server Error` — An unexpected issue occurred, retry the command or check the system status.

>[!NOTE]
>Rate limit means the amount of requests you are able to send in a period of time. Exceeding the limit temporarily blocks subsequent requests.

### Step 4. Confirm the Command

Verifying the vessel’s stats or mission progress is something you can do by sending a `GET` request to the `/missions/status` endpoint.

**Example**

Request:
```http
GET https://api.afcs.com/v1/missions/status?mission_id=MC-17269
Authorization: Bearer eor4nuy1309
```

**Try It Out**
```bash
curl -X GET "https://api.afcs.com/v1/missions/status?mission_id=MC-17269"
  -H "Authorization: Bearer eor4nuy1309"
```

>[!NOTE]
>This confirms that your command was received and processed and that the mission has entered an active or completed state.


## Monitoring Mission Data & Vessel Status

Once you start issuing requests to vessels, you must check mission progress and telemetry data to ensure accurate execution of commands.

### Step 1. Retrieve Mission Status

Requests are sent via `GET` to the `/missions/status` endpoint, inputting the `mission_id` to confirm command execution and track the mission’s state.

**Example**

Request:
```http
GET https://api.afcs.com/v1/missions/status?mission_id=MC-17269
Authorization: Bearer eor4nuy1309
```

Response:
```json
{
  "mission_id": "MC-17269",
  "status": "in_progress",
  "vessel_id": "UUV-013",
  "progress": 45,
  "last_update": "2025-10-07T13:15:00Z"
}
```

**Try It Out**
```bash
curl -X GET "https://api.afcs.com/v1/missions/status?mission_id=MC-17269"
  -H "Authorization: Bearer eor4nuy1309"
```

>[!NOTE]
>`progress: {%}` indicates the completeness percentage of the mission. It represents how far along the vessel is in executing the assigned mission.
>- `0` — Mission hasn’t begun or just started.
>- `50` — Mission is halfway complete.
>- `100` — Mission is complete.

### Step 2. Vessel Telemetry

To stream live or recent telemetry (e.g, location, speed, system health, status) you would use the `GET` command with the `/vessels/telemetry` endpoint.

**Example**

Request:
```http
GET https://api.afcs.com/v1/vessels/telemetry?vessel_id=UUV-013
Authorization: Bearer eor4nuy1309
```

Response:
```json
{
  "vessel_id": "UUV-013",
  "coordinates": {"lat": 27.9506, "lon": -82.4572},
  "speed": 5.2,
  "status": "active",
  "system_health": "nominal",
  "timestamp": "2025-10-07T13:15:00Z"
}
```

**Try It Out**
```bash
curl -X GET "https://api.afcs.com/v1/vessels/telemetry?vessel_id=UUV-013"
  -H "Authorization: Bearer eor4nuy1309"
```

### Step 3. Filtering & Query Parameters

To focus on the data you need, both the`mission_status` and `/vessels/telemetry` endpoints support filtering and query parameters (e.g, specific time windows, vessel IDs, telemetry types)

**Example — Mission Status with Time Filter**

Request:
```http
GET https://api.afcs.com/v1/missions/status?mission_id=MC-17269&from=2025-10-07T12:00:00Z&to=2025-10-07T13:00:00Z
Authorization: Bearer eor4nuy1309
```

**Try It Out**
```bash
curl -X GET "https://api.afcs.com/v1/missions/status?mission_id=MC-17269&from=2025-10-07T12:00:00Z&to=2025-10-07T13:00:00Z"
  -H "Authorization: Bearer eor4nuy1309"
```

**Example — Telemetry Filter by System Health**

Request:
```http
GET https://api.afcs.com/v1/vessels/telemetry?vessel_id=UUV-013&filter=system_health:nominal
Authorization: Bearer eor4nuy1309
```

**Try It Out**
```bash
curl -X GET "https://api.afcs.com/v1/vessels/telemetry?vessel_id=UUV-013&filter=system_health:nominal"
  -H "Authorization: Bearer eor4nuy1309"
```

>[!TIP]
>Filtering helps reduce unnecessary data transfer and makes analysis more efficient. You can also combine more than one parameter to narrow your scope.

### Step 4. Response Interpretation


Responses from the AFCS API provide both confirmation and status details, and understanding how to properly interpret them ensures accurate mission analysis and proper follow-up.

**Mission Status Responses:**
- `status` — The current state of a mission, followed by values such as:
	- `pending` — The command has been received but has not yet been carried out.
	- `in_progress` — The mission is actively running.
	- `completed` — The mission has been successfully completed.
	- `failed` — The mission could not be completed or was unsuccessful.
- `progress` — The percentage of completion, which can be used to calculate the mission's progress.
- `last_update` — The timestamp of the most recent status update, which can help identify potential delays.

### Step 5. Handling Delays & Errors

Delays and errors both can set mission requests back, due to their unforeseen obstacles. For example, a delay could be caused by network latency, vessel response time, or system load; while errors can be due to invalid tokens, vessel or mission ID not existing, exceeding the rate limit, and more.

**Strategies to Prevent Delays:**
- Implement polling intervals rather than constant requests. For example, check mission status every 15-30 seconds depending on the type.
- If `progress` has not been updated for multiple intervals, log a warning and consider either retrying the request or contacting system diagnostics.

**Key Error Codes:**
- `401 Unauthorized` — The token is missing, invalid, or expired. Re-authenticate and request a new access token.
- `403 Forbidden` — The token does not have permission for this vessel or mission. Verify permission tier or role.
- `404 Not Found` — The requested vessel, mission, or endpoint does not exist. Confirm `vessel_id` or `mission_id`.
- `429 Too Many Requests` — The rate limit has been exceeded. Wait and retry according to the `Retry-After` header.
- `500 Internal Server Error` — There is an unexpected system error. Retry after a short delay or report if persistent.

**Best Practices:**
- Always log error details (`error_code`, `error_type`, `message`) for troubleshooting and auditing.
- Use exponential backoff for retries on infrequent errors (e.g., `500`, `429`) to avoid overloading the system.
- Keep timeouts reasonable to avoid hanging requests.
- Correlate error responses with `mission_id` and `vessel_id` for faster root-cause analysis.

## Verify & Observe

You must verify that the system is reacting appropriately and that the mission and telemetry data match as you send commands and requests into it.

### Step 1. Confirm Mission Progress
Use the `/missions/status` endpoint to verify that the mission is proceeding successfully.

**Checklist:**
- [x] The `status` field changes from `pending` to `in_progress` to `completed`.
- [x] The `progress` percentage increases.
- [x] The `last_update` timestamp periodically refreshes.


**Example**

Request:
```http
GET https://api.afcs.com/v1/missions/status?mission_id=MC-17269
Authorization: Bearer eor4nuy1309
```

##### **Expected Response:**
- An updated `progress` field or `status` change

**Try It Out**
```bash
curl -X GET "https://api.afcs.com/v1/missions/status?mission_id=MC-17269"
  -H "Authorization: Bearer eor4nuy1309"
```

>[!TIP]
>If `progress` does not change for multiple refreshes, review the data or flag for system diagnostics.


### Step 2. Cross-Check Telemetry

Verify that the activity corresponds to the mission status using the `vessels/telemetry` endpoint.

**Checklist:**
- [x] The vessel's `status` should correspond with the mission's `status` (e.g., `active` when mission is `in_progress`).
- [x] When there is movement during missions, `coordinates` and `speed` should change.
- [x] Unless there is an operational problem, `system_health` should stay at `nominal`.

**Example**

Request:
```http
GET https://api.afcs.com/v1/vessels/telemetry?vessel_id=UUV-013
Authorization: Bearer eor4nuy1309
```
>[!TIP]
>If the mission shows `completed` but the data indicates any activity, wait a moment before confirming the final status; telemetry data can lag behind communication execution.

**Try It Out**
```bash
curl -X GET "https://api.afcs.com/v1/vessels/telemetry?vessel_id=UUV-013" \
  -H "Authorization: Bearer eor4nuy1309"
```

>[!TIP]
>Always include quotes around the URL when query parameters (`?`) are present to prevent shell interpretation issues.

### Step 3. Observe for Anomalies

Monitoring live feedback helps rely on system accuracy.


**Common Inconsistencies Include:**
- Telemetry data is unchanged during an active mission.
- Mission status becomes `failed` without warning.
- Timestamps are not updating and only showing past reporting.

**If Anomalies Occur:**
- Log both mission and data responses.
- Verify timestamps to ensure you are not reviewing outdated, cached, or delayed data.
- Retry requests after a 10-15 second interval to confirm if the issue is temporary or continuing.

>[!NOTE]
>Verification is about identifying the early warning signs of delays, faults, or other issues within communication, the system, or the vessel before they affect missions.


## Troubleshooting Tips

Even if you follow all the correct authentication and command steps, errors and unexpected responses may still occur. The tips below will help you to troubleshoot problems before they become more serious..

**Receiving Missing Updates or No Mission Progress:**
- Confirm that the `mission_id` and `vessel_id` are correct and active.
- Check timestamps with the `last_update` field; if unchanged, retry a status check after 15-30 seconds.
- Review telemetry for any vessel inactivity or system warnings

**Errors With Authentication or Authorization:**
- Re-authenticate after a `401 Unauthorized` or `403 Forbidden` response.
- Make sure your access token did not expire and that it carries the level of permission required..

**Telemetry Data is Inconsistent, Delayed, or Unchanged:**
- Compare timestamps between mission and telemetry endpoints.
- If the timestamp values don’t update, refresh the data and verify the request URL and parameters.
Avoid sending requests too often, spread them out by 15-30 seconds.

**Unexpected Mission Failure:**
- Check for logged error codes or messages in the mission response.
- Use the data to identify whether the issue was system-related or vessel-related.
- If the type of failure allows for safe, repeated requests, retry the command.

**Errors With Rate Limits or Timeouts**
- If you receive a `429 Too Many Requests` response, request after the allotted time given by `Retry-After` has passed.
- For `500 Internal Server Error` or a timeout, perform exponential backoff and retry after 15-30 seconds.

>[!TIP]
>Always keep your logs detailed. Record `mission_id`, `vessel_id`, `timestamp`, and `error_code` to help with faster troubleshooting if issues arise.

## Next Steps

By this point, you should have successfully authenticated, issued commands, and monitored missions. You also should know exactly what to do if issues arise. It is now time to build on these basics.
- **Automate workflows** — Use the mission and telemetry endpoints to schedule or script recurring missions.
- **Integrate systems** — Connect AFCS data to other dashboards and monitoring tools.
- **Log and analyze data** — Track and learn from performance over time to identify trends, strengths, and weaknesses to improve reliability.
- **Advance operations** — Explore more complicated endpoints for requests such as multi-vessel coordination.

>[!NOTE]
>For a more advanced walkthrough of AFCS and its capabilities, please refer to the *[System Overview and Integration Guide](https://github.com/laureneolin/portfolio/blob/Technical-Writing/System_Overview_Integration_Guide.md)*.


