---
layout: default
title: Gonglu API — Guide for AI Agents
---

# Gonglu API — Guide for AI Agents

## Overview

Gonglu is an AI-powered alarm system. AI agents (ChatGPT, Claude, Gemini, etc.) set alarms by calling the Gonglu API. The user's mobile app polls the server, picks up the alarm, and schedules it locally.

## Base URL

```
https://<your-gonglu-server>
```

The user will provide the server URL. The interactive docs are at `/docs`.

## Prerequisites

Before an AI agent can set an alarm for a device, the device **must have polled the server at least once** to register itself. If the device has not polled, the `POST /alarm` endpoint will return a `404` error.

The user's Gonglu app registers automatically when it starts polling — no manual step is needed on the user's part.

## Setting an Alarm

### `POST /alarm`

Creates a new alarm for a registered device.

#### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id_token` | string | Yes | The user's Gonglu device ID (e.g., `gonglu-a1b2c3d4-...`) |
| `alarm_time` | string (ISO 8601) | Yes | When the alarm should fire. **Must be in the future.** Example: `2026-03-15T07:30:00` |
| `alarm_text` | string | No | Description/reminder text shown when the alarm rings |
| `alarm_data` | object | No | Additional data (e.g., `{"ringtone": "gong"}`) |

#### Example Request

```bash
curl -X POST https://<server>/alarm \
  -H "Content-Type: application/json" \
  -d '{
    "id_token": "gonglu-a1b2c3d4-5678-9abc-def0-1234567890ab",
    "alarm_time": "2026-03-15T07:30:00",
    "alarm_text": "Time for your morning meeting!"
  }'
```

#### Success Response (200)

```json
{
  "alarm_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "status": "created"
}
```

#### Error Responses

| Status | Meaning | When |
|--------|---------|------|
| 404 | Device not registered | The `id_token` has never polled the server. The user needs to open the Gonglu app first. |
| 422 | Validation error | `id_token` is empty, `alarm_time` is in the past, or `alarm_text` is an empty string. |

**404 Example:**
```json
{
  "detail": "Device not registered. The device must poll the server at least once before alarms can be set for it."
}
```

**422 Example:**
```json
{
  "detail": [
    {
      "loc": ["body", "alarm_time"],
      "msg": "Value error, alarm_time must be in the future",
      "type": "value_error"
    }
  ]
}
```

## Other Endpoints (for reference)

| Method | Path | Description | Used by |
|--------|------|-------------|---------|
| `GET` | `/alarms/{id_token}` | Get pending alarms for a device | Mobile app |
| `DELETE` | `/alarm/{alarm_id}` | Delete an alarm after local scheduling | Mobile app |
| `POST` | `/device/regenerate` | Regenerate a device ID | Mobile app |
| `GET` | `/health` | Health check | System |
| `GET` | `/` | API info | System |

## Device ID Format

Device IDs follow the pattern: `gonglu-{uuid-v4}`

Example: `gonglu-a1b2c3d4-5678-9abc-def0-1234567890ab`

## Timezone Handling

- `alarm_time` should be provided in ISO 8601 format.
- If no timezone offset is included, the time is treated as UTC.
- To specify a timezone, include an offset: `2026-03-15T07:30:00+05:30`

## Example User Prompt

A user might say to an AI agent:

> "Set an alarm for 7 AM tomorrow with the message 'Stand-up meeting in 30 minutes' using my Gonglu ID: gonglu-a1b2c3d4-5678-9abc-def0-1234567890ab"

The AI agent should:
1. Parse the time ("7 AM tomorrow") into an ISO 8601 datetime in the future.
2. Call `POST /alarm` with the user's `id_token`, the computed `alarm_time`, and the `alarm_text`.
3. Confirm to the user that the alarm was set, including the alarm time.

## Notes

- One device ID can have multiple pending alarms.
- Alarms are picked up and deleted from the server by the mobile app. The AI agent only needs to create them.
- If you receive a 404, tell the user to open the Gonglu app so it can register with the server.
