# Esmer (ESMO) API Integration Specification: Calendar & Scheduling

> **Category:** 02 - Calendar & Scheduling
> **Version:** 1.0.0
> **Last Updated:** 2026-02-09
> **Status:** Draft

---

## Table of Contents

- [1. Google Calendar (Google Calendar API v3)](#1-google-calendar-google-calendar-api-v3)
  - [1.1 Service Overview](#11-service-overview)
  - [1.2 Authentication](#12-authentication)
  - [1.3 Base URL](#13-base-url)
  - [1.4 Rate Limits](#14-rate-limits)
  - [1.5 API Endpoints](#15-api-endpoints)
  - [1.6 Webhooks / Real-time](#16-webhooks--real-time)
  - [1.7 Error Handling](#17-error-handling)
  - [1.8 Mobile-Specific Notes](#18-mobile-specific-notes)
- [2. Zoom (Zoom API v2)](#2-zoom-zoom-api-v2)
  - [2.1 Service Overview](#21-service-overview)
  - [2.2 Authentication](#22-authentication)
  - [2.3 Base URL](#23-base-url)
  - [2.4 Rate Limits](#24-rate-limits)
  - [2.5 API Endpoints](#25-api-endpoints)
  - [2.6 Webhooks / Real-time](#26-webhooks--real-time)
  - [2.7 Error Handling](#27-error-handling)
  - [2.8 Mobile-Specific Notes](#28-mobile-specific-notes)
- [3. GoToWebinar (LogMeIn / GoTo API)](#3-gotowebinar-logmein--goto-api)
  - [3.1 Service Overview](#31-service-overview)
  - [3.2 Authentication](#32-authentication)
  - [3.3 Base URL](#33-base-url)
  - [3.4 Rate Limits](#34-rate-limits)
  - [3.5 API Endpoints](#35-api-endpoints)
  - [3.6 Webhooks / Real-time](#36-webhooks--real-time)
  - [3.7 Error Handling](#37-error-handling)
  - [3.8 Mobile-Specific Notes](#38-mobile-specific-notes)
- [Cross-Service Comparison Matrix](#cross-service-comparison-matrix)

---

## 1. Google Calendar (Google Calendar API v3)

### 1.1 Service Overview

Google Calendar API v3 provides programmatic access to Google Calendar data. It allows applications to create, read, update, and delete calendar events, query free/busy information, manage calendar lists, and configure access control.

**How Esmer uses Google Calendar:**

- Creating and scheduling events on behalf of users via voice or chat commands
- Querying free/busy availability to suggest optimal meeting times
- Managing attendees and sending invitations
- Listing upcoming events for daily briefings
- Updating or canceling events through natural language instructions
- Handling recurring event creation and modification
- Conference link generation (Google Meet) when creating events

### 1.2 Authentication

**Method:** OAuth 2.0

Google Calendar supports OAuth 2.0 for user-delegated access. Service Accounts are NOT supported for Google Calendar (per the n8n compatibility matrix). Esmer must use OAuth 2.0.

| Property | Value |
|---|---|
| Authorization URL | `https://accounts.google.com/o/oauth2/v2/auth` |
| Token URL | `https://oauth2.googleapis.com/token` |
| Revoke URL | `https://oauth2.googleapis.com/revoke` |
| Grant Type | `authorization_code` |
| PKCE Support | Yes (recommended for mobile) |

**Required Scopes:**

| Scope | Description | Esmer Usage |
|---|---|---|
| `https://www.googleapis.com/auth/calendar` | Full read/write access to Calendars | Event CRUD, attendee management |
| `https://www.googleapis.com/auth/calendar.events` | Read/write access to Events only | Event CRUD (narrower than full calendar) |
| `https://www.googleapis.com/auth/calendar.events.readonly` | Read-only access to Events | Listing events, daily briefings |
| `https://www.googleapis.com/auth/calendar.readonly` | Read-only access to Calendars | Free/busy queries, calendar listing |
| `https://www.googleapis.com/auth/calendar.settings.readonly` | Read-only access to Settings | Reading user timezone preferences |
| `https://www.googleapis.com/auth/calendar.calendarlist.readonly` | Read-only access to calendar list | Listing user's calendars |

**Recommendations for Esmer:**

- Request `https://www.googleapis.com/auth/calendar.events` and `https://www.googleapis.com/auth/calendar.readonly` as the minimum viable scopes. This allows event CRUD and free/busy queries without full calendar admin access.
- Use incremental authorization: request read-only scopes initially, then prompt for write scopes when the user first attempts to create/modify an event.
- Use PKCE (`code_verifier` / `code_challenge`) for the mobile OAuth flow since Esmer is a React Native app and cannot securely store a client secret.
- Store refresh tokens in the device's secure keychain (iOS Keychain / Android Keystore).
- Google OAuth tokens expire after 1 hour. Implement proactive token refresh 5 minutes before expiry.
- Set `access_type=offline` and `prompt=consent` on first authorization to ensure a refresh token is returned.

### 1.3 Base URL

```
https://www.googleapis.com/calendar/v3
```

Discovery document: `https://www.googleapis.com/discovery/v1/apis/calendar/v3/rest`

### 1.4 Rate Limits

| Limit Type | Value | Scope |
|---|---|---|
| Queries per day | 1,000,000 | Per project |
| Queries per 100 seconds per user | 500 | Per user per project |
| Queries per 100 seconds | 5,000 | Per project (all users) |
| Calendar creation per day | 60 | Per user |
| Event creation per day | ~10,000 | Per calendar |

**Notes:**

- Rate limits are enforced per Google Cloud project. All Esmer users share the per-project quota.
- Per-user limits use the authenticated user's identity.
- Batch requests count as a single HTTP request against rate limits but each sub-request counts against quota.
- When rate limited, Google returns HTTP 403 with `reason: "rateLimitExceeded"` or HTTP 429.
- Implement exponential backoff with jitter, starting at 1 second, doubling up to 32 seconds.

### 1.5 API Endpoints

#### 1.5.1 CalendarList Resource

The CalendarList represents the collection of calendars that appear on the user's calendar list.

##### GET /users/me/calendarList

**Description:** Returns the calendars on the user's calendar list.

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `maxResults` | integer | No | Maximum number of entries (default 100, max 250) |
| `minAccessRole` | string | No | Filter by minimum access role: `freeBusyReader`, `owner`, `reader`, `writer` |
| `pageToken` | string | No | Token for pagination |
| `showDeleted` | boolean | No | Include deleted entries (default false) |
| `showHidden` | boolean | No | Include hidden entries (default false) |
| `syncToken` | string | No | Token for incremental sync |

**Response Body (200 OK):**

```json
{
  "kind": "calendar#calendarList",
  "etag": "\"abc123\"",
  "nextPageToken": "token_string",
  "nextSyncToken": "sync_token_string",
  "items": [
    {
      "kind": "calendar#calendarListEntry",
      "etag": "\"xyz789\"",
      "id": "user@example.com",
      "summary": "My Calendar",
      "description": "Personal calendar",
      "location": "US",
      "timeZone": "America/Los_Angeles",
      "summaryOverride": "Work Calendar",
      "colorId": "9",
      "backgroundColor": "#7bd148",
      "foregroundColor": "#000000",
      "hidden": false,
      "selected": true,
      "accessRole": "owner",
      "defaultReminders": [
        { "method": "popup", "minutes": 10 }
      ],
      "primary": true
    }
  ]
}
```

**Error Codes:**

| Code | Description |
|---|---|
| 401 | Invalid or expired OAuth token |
| 403 | Rate limit exceeded or insufficient permissions |
| 404 | Calendar not found |

---

##### GET /users/me/calendarList/{calendarId}

**Description:** Returns a specific calendar from the user's calendar list.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `calendarId` | string | Yes | Calendar identifier. Use `primary` for the user's primary calendar. |

**Response Body (200 OK):**

```json
{
  "kind": "calendar#calendarListEntry",
  "etag": "\"abc123\"",
  "id": "user@example.com",
  "summary": "My Calendar",
  "timeZone": "America/Los_Angeles",
  "accessRole": "owner",
  "primary": true
}
```

---

#### 1.5.2 Calendars Resource

##### GET /calendars/{calendarId}

**Description:** Returns metadata for a calendar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `calendarId` | string | Yes | Calendar identifier |

**Response Body (200 OK):**

```json
{
  "kind": "calendar#calendar",
  "etag": "\"abc123\"",
  "id": "user@example.com",
  "summary": "My Calendar",
  "description": "Personal events",
  "location": "US",
  "timeZone": "America/Los_Angeles"
}
```

---

##### POST /calendars

**Description:** Creates a secondary calendar.

**Request Body:**

```json
{
  "summary": "Project Meetings",
  "description": "Calendar for project-related meetings",
  "location": "US",
  "timeZone": "America/New_York"
}
```

**Response Body (200 OK):** Same as GET /calendars/{calendarId}.

---

##### PUT /calendars/{calendarId}

**Description:** Updates metadata for a calendar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `calendarId` | string | Yes | Calendar identifier |

**Request Body:**

```json
{
  "summary": "Updated Calendar Name",
  "description": "Updated description",
  "timeZone": "America/Chicago"
}
```

---

##### PATCH /calendars/{calendarId}

**Description:** Partially updates metadata for a calendar. Only specified fields are modified.

**Request/Response:** Same schema as PUT, but only changed fields need to be sent.

---

##### DELETE /calendars/{calendarId}

**Description:** Deletes a secondary calendar. Cannot delete primary calendar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `calendarId` | string | Yes | Calendar identifier |

**Response:** 204 No Content on success.

---

#### 1.5.3 Events Resource

##### POST /calendars/{calendarId}/events

**Description:** Creates an event in the specified calendar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `calendarId` | string | Yes | Calendar identifier. Use `primary` for the user's primary calendar. |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `conferenceDataVersion` | integer | No | Set to `1` to enable conference data (Google Meet link) creation |
| `maxAttendees` | integer | No | Max attendees to include in response |
| `sendUpdates` | string | No | `all`, `externalOnly`, or `none` (default `none`) |
| `supportsAttachments` | boolean | No | Whether API client supports event attachments |

**Request Body:**

```json
{
  "summary": "Team Standup",
  "location": "Conference Room B",
  "description": "Daily standup meeting for the engineering team",
  "start": {
    "dateTime": "2026-02-10T09:00:00-05:00",
    "timeZone": "America/New_York"
  },
  "end": {
    "dateTime": "2026-02-10T09:30:00-05:00",
    "timeZone": "America/New_York"
  },
  "recurrence": [
    "RRULE:FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR;COUNT=52"
  ],
  "attendees": [
    { "email": "alice@example.com", "displayName": "Alice" },
    { "email": "bob@example.com", "displayName": "Bob", "optional": true }
  ],
  "reminders": {
    "useDefault": false,
    "overrides": [
      { "method": "email", "minutes": 1440 },
      { "method": "popup", "minutes": 10 }
    ]
  },
  "conferenceData": {
    "createRequest": {
      "requestId": "unique-request-id-123",
      "conferenceSolutionKey": {
        "type": "hangoutsMeet"
      }
    }
  },
  "guestsCanInviteOthers": true,
  "guestsCanModify": false,
  "guestsCanSeeOtherGuests": true,
  "visibility": "default",
  "transparency": "opaque",
  "colorId": "5",
  "source": {
    "title": "Created by Esmer",
    "url": "https://esmer.app"
  }
}
```

**Response Body (200 OK):**

```json
{
  "kind": "calendar#event",
  "etag": "\"3210987654321000\"",
  "id": "abc123def456",
  "status": "confirmed",
  "htmlLink": "https://www.google.com/calendar/event?eid=abc123",
  "created": "2026-02-09T12:00:00.000Z",
  "updated": "2026-02-09T12:00:00.000Z",
  "summary": "Team Standup",
  "description": "Daily standup meeting for the engineering team",
  "location": "Conference Room B",
  "creator": {
    "email": "user@example.com",
    "self": true
  },
  "organizer": {
    "email": "user@example.com",
    "self": true
  },
  "start": {
    "dateTime": "2026-02-10T09:00:00-05:00",
    "timeZone": "America/New_York"
  },
  "end": {
    "dateTime": "2026-02-10T09:30:00-05:00",
    "timeZone": "America/New_York"
  },
  "recurrence": [
    "RRULE:FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR;COUNT=52"
  ],
  "iCalUID": "abc123def456@google.com",
  "sequence": 0,
  "attendees": [
    {
      "email": "alice@example.com",
      "displayName": "Alice",
      "responseStatus": "needsAction"
    },
    {
      "email": "bob@example.com",
      "displayName": "Bob",
      "optional": true,
      "responseStatus": "needsAction"
    }
  ],
  "hangoutLink": "https://meet.google.com/abc-defg-hij",
  "conferenceData": {
    "entryPoints": [
      {
        "entryPointType": "video",
        "uri": "https://meet.google.com/abc-defg-hij",
        "label": "meet.google.com/abc-defg-hij"
      },
      {
        "entryPointType": "phone",
        "uri": "tel:+1-234-567-8901",
        "label": "+1 234-567-8901",
        "pin": "123456789"
      }
    ],
    "conferenceSolution": {
      "key": { "type": "hangoutsMeet" },
      "name": "Google Meet",
      "iconUri": "https://fonts.gstatic.com/s/i/productlogos/meet_2020q4/v6/web-512dp/logo_meet_2020q4_color_2x_web_512dp.png"
    },
    "conferenceId": "abc-defg-hij"
  },
  "reminders": {
    "useDefault": false,
    "overrides": [
      { "method": "email", "minutes": 1440 },
      { "method": "popup", "minutes": 10 }
    ]
  }
}
```

**Error Codes:**

| Code | Description |
|---|---|
| 400 | Invalid request body (bad date format, missing required fields) |
| 401 | Invalid or expired OAuth token |
| 403 | Insufficient permissions or rate limit exceeded |
| 404 | Calendar not found |
| 409 | Conflict (event ID already exists) |

---

##### GET /calendars/{calendarId}/events/{eventId}

**Description:** Returns a single event by ID.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `calendarId` | string | Yes | Calendar identifier |
| `eventId` | string | Yes | Event identifier |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `alwaysIncludeEmail` | boolean | No | Include email even if not available (uses generated noreply) |
| `maxAttendees` | integer | No | Max number of attendees to include |
| `timeZone` | string | No | Timezone for start/end times in response |

**Response Body (200 OK):** Same event schema as POST response above.

---

##### GET /calendars/{calendarId}/events

**Description:** Returns events on the specified calendar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `calendarId` | string | Yes | Calendar identifier |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `iCalUID` | string | No | Filter by iCal UID |
| `maxAttendees` | integer | No | Max attendees in response |
| `maxResults` | integer | No | Max events returned (default 250, max 2500) |
| `orderBy` | string | No | `startTime` (requires singleEvents=true) or `updated` |
| `pageToken` | string | No | Pagination token |
| `q` | string | No | Free text search across all fields |
| `showDeleted` | boolean | No | Include cancelled events (default false) |
| `showHiddenInvitations` | boolean | No | Include hidden invitations (default false) |
| `singleEvents` | boolean | No | Expand recurring events into instances (default false) |
| `syncToken` | string | No | Incremental sync token |
| `timeMax` | datetime | No | Upper bound (exclusive) for event start (RFC 3339) |
| `timeMin` | datetime | No | Lower bound (inclusive) for event start (RFC 3339) |
| `timeZone` | string | No | Timezone for response |
| `updatedMin` | datetime | No | Lower bound on lastModified (RFC 3339) |

**Response Body (200 OK):**

```json
{
  "kind": "calendar#events",
  "etag": "\"abc123\"",
  "summary": "My Calendar",
  "description": "",
  "updated": "2026-02-09T12:00:00.000Z",
  "timeZone": "America/New_York",
  "accessRole": "owner",
  "defaultReminders": [
    { "method": "popup", "minutes": 10 }
  ],
  "nextPageToken": "token_string",
  "nextSyncToken": "sync_token_string",
  "items": [
    {
      "kind": "calendar#event",
      "id": "event_id_1",
      "summary": "Team Standup",
      "start": {
        "dateTime": "2026-02-10T09:00:00-05:00"
      },
      "end": {
        "dateTime": "2026-02-10T09:30:00-05:00"
      },
      "status": "confirmed"
    }
  ]
}
```

---

##### PUT /calendars/{calendarId}/events/{eventId}

**Description:** Updates an event. This replaces the entire event resource (all fields must be provided).

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `calendarId` | string | Yes | Calendar identifier |
| `eventId` | string | Yes | Event identifier |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `conferenceDataVersion` | integer | No | Set to `1` for conference data support |
| `maxAttendees` | integer | No | Max attendees in response |
| `sendUpdates` | string | No | `all`, `externalOnly`, or `none` |
| `supportsAttachments` | boolean | No | Whether client supports attachments |

**Request Body:** Full event resource (same schema as POST).

**Response Body (200 OK):** Updated event resource.

---

##### PATCH /calendars/{calendarId}/events/{eventId}

**Description:** Partially updates an event. Only provided fields are modified; all other fields are preserved.

**Path/Query Parameters:** Same as PUT.

**Request Body:** Only the fields to be updated.

```json
{
  "summary": "Updated Meeting Title",
  "attendees": [
    { "email": "alice@example.com" },
    { "email": "charlie@example.com" }
  ]
}
```

**Response Body (200 OK):** Updated event resource.

---

##### DELETE /calendars/{calendarId}/events/{eventId}

**Description:** Deletes an event.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `calendarId` | string | Yes | Calendar identifier |
| `eventId` | string | Yes | Event identifier |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `sendUpdates` | string | No | `all`, `externalOnly`, or `none` |

**Response:** 204 No Content on success.

---

##### POST /calendars/{calendarId}/events/quickAdd

**Description:** Creates an event from a simple text string (natural language parsing).

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `calendarId` | string | Yes | Calendar identifier |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `text` | string | Yes | The text describing the event (e.g., "Lunch with Bob at noon tomorrow") |
| `sendUpdates` | string | No | `all`, `externalOnly`, or `none` |

**Response Body (200 OK):** Event resource.

---

##### POST /calendars/{calendarId}/events/{eventId}/move

**Description:** Moves an event to another calendar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `calendarId` | string | Yes | Source calendar identifier |
| `eventId` | string | Yes | Event identifier |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `destination` | string | Yes | Target calendar identifier |
| `sendUpdates` | string | No | `all`, `externalOnly`, or `none` |

**Response Body (200 OK):** Updated event resource with the new calendar.

---

##### GET /calendars/{calendarId}/events/{eventId}/instances

**Description:** Returns instances of a recurring event.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `calendarId` | string | Yes | Calendar identifier |
| `eventId` | string | Yes | Recurring event identifier |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `maxResults` | integer | No | Max instances to return |
| `pageToken` | string | No | Pagination token |
| `timeMax` | datetime | No | Upper bound for instance start time (RFC 3339) |
| `timeMin` | datetime | No | Lower bound for instance start time (RFC 3339) |
| `timeZone` | string | No | Timezone for response |

**Response Body (200 OK):** Same as events list response.

---

##### POST /calendars/{calendarId}/events/import

**Description:** Imports an event (uses iCalUID for deduplication instead of generating a new ID).

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `calendarId` | string | Yes | Calendar identifier |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `conferenceDataVersion` | integer | No | Set to `1` for conference data |
| `supportsAttachments` | boolean | No | Whether client supports attachments |

**Request Body:** Event resource with `iCalUID` field set.

**Response Body (200 OK):** Imported event resource.

---

#### 1.5.4 Freebusy Resource

##### POST /freeBusy

**Description:** Returns free/busy information for a set of calendars.

**Request Body:**

```json
{
  "timeMin": "2026-02-10T00:00:00Z",
  "timeMax": "2026-02-10T23:59:59Z",
  "timeZone": "America/New_York",
  "groupExpansionMax": 100,
  "calendarExpansionMax": 50,
  "items": [
    { "id": "primary" },
    { "id": "colleague@example.com" }
  ]
}
```

**Request Body Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `timeMin` | datetime | Yes | Start of the time range (RFC 3339) |
| `timeMax` | datetime | Yes | End of the time range (RFC 3339) |
| `timeZone` | string | No | Timezone (default UTC) |
| `groupExpansionMax` | integer | No | Max calendar group members to expand |
| `calendarExpansionMax` | integer | No | Max calendars to expand |
| `items` | array | Yes | Array of calendar identifiers to query |

**Response Body (200 OK):**

```json
{
  "kind": "calendar#freeBusy",
  "timeMin": "2026-02-10T00:00:00.000Z",
  "timeMax": "2026-02-10T23:59:59.000Z",
  "calendars": {
    "primary": {
      "busy": [
        {
          "start": "2026-02-10T09:00:00-05:00",
          "end": "2026-02-10T09:30:00-05:00"
        },
        {
          "start": "2026-02-10T14:00:00-05:00",
          "end": "2026-02-10T15:00:00-05:00"
        }
      ],
      "errors": []
    },
    "colleague@example.com": {
      "busy": [
        {
          "start": "2026-02-10T10:00:00-05:00",
          "end": "2026-02-10T11:00:00-05:00"
        }
      ],
      "errors": []
    }
  }
}
```

---

#### 1.5.5 ACL Resource (Access Control List)

##### GET /calendars/{calendarId}/acl

**Description:** Returns the ACL rules for a calendar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `calendarId` | string | Yes | Calendar identifier |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `maxResults` | integer | No | Max entries to return |
| `pageToken` | string | No | Pagination token |
| `showDeleted` | boolean | No | Include deleted entries |
| `syncToken` | string | No | Incremental sync token |

**Response Body (200 OK):**

```json
{
  "kind": "calendar#acl",
  "etag": "\"abc123\"",
  "items": [
    {
      "kind": "calendar#aclRule",
      "etag": "\"xyz789\"",
      "id": "user:alice@example.com",
      "scope": {
        "type": "user",
        "value": "alice@example.com"
      },
      "role": "writer"
    }
  ]
}
```

---

##### POST /calendars/{calendarId}/acl

**Description:** Creates an ACL rule (shares a calendar).

**Request Body:**

```json
{
  "scope": {
    "type": "user",
    "value": "alice@example.com"
  },
  "role": "reader"
}
```

**Role Values:** `none`, `freeBusyReader`, `reader`, `writer`, `owner`

---

##### DELETE /calendars/{calendarId}/acl/{ruleId}

**Description:** Deletes an ACL rule.

**Response:** 204 No Content.

---

#### 1.5.6 Settings Resource

##### GET /users/me/settings

**Description:** Returns all user settings.

**Response Body (200 OK):**

```json
{
  "kind": "calendar#settings",
  "etag": "\"abc123\"",
  "items": [
    {
      "kind": "calendar#setting",
      "etag": "\"xyz789\"",
      "id": "timezone",
      "value": "America/New_York"
    },
    {
      "kind": "calendar#setting",
      "id": "dateFieldOrder",
      "value": "MDY"
    },
    {
      "kind": "calendar#setting",
      "id": "format24HourTime",
      "value": "false"
    }
  ]
}
```

---

##### GET /users/me/settings/{settingId}

**Description:** Returns a single user setting.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `settingId` | string | Yes | Setting ID (e.g., `timezone`, `locale`, `format24HourTime`) |

---

#### 1.5.7 Colors Resource

##### GET /colors

**Description:** Returns the color definitions for calendars and events.

**Response Body (200 OK):**

```json
{
  "kind": "calendar#colors",
  "updated": "2026-01-01T00:00:00.000Z",
  "calendar": {
    "1": { "background": "#ac725e", "foreground": "#1d1d1d" },
    "2": { "background": "#d06b64", "foreground": "#1d1d1d" }
  },
  "event": {
    "1": { "background": "#a4bdfc", "foreground": "#1d1d1d" },
    "2": { "background": "#7ae7bf", "foreground": "#1d1d1d" }
  }
}
```

---

### 1.6 Webhooks / Real-time

Google Calendar supports push notifications via the Notifications API (Google Push Notifications, not Firebase).

#### Setting Up a Watch Channel

##### POST /calendars/{calendarId}/events/watch

**Description:** Subscribes to changes on a calendar's events.

**Request Body:**

```json
{
  "id": "unique-channel-id-01",
  "type": "web_hook",
  "address": "https://api.esmer.app/webhooks/google-calendar",
  "token": "verification-token-abc123",
  "expiration": 1739145600000,
  "params": {
    "ttl": "3600"
  }
}
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `id` | string | Yes | Unique channel identifier (UUID recommended) |
| `type` | string | Yes | Must be `web_hook` |
| `address` | string | Yes | HTTPS callback URL |
| `token` | string | No | Arbitrary string for verification |
| `expiration` | long | No | Channel expiration time (Unix ms). Max ~30 days. |

**Response Body (200 OK):**

```json
{
  "kind": "api#channel",
  "id": "unique-channel-id-01",
  "resourceId": "resource-id-abc",
  "resourceUri": "https://www.googleapis.com/calendar/v3/calendars/primary/events",
  "token": "verification-token-abc123",
  "expiration": 1739145600000
}
```

**Notification Payload (POST to callback URL):**

Headers received on notification:

| Header | Description |
|---|---|
| `X-Goog-Channel-ID` | The channel ID you provided |
| `X-Goog-Channel-Token` | The token you provided (if any) |
| `X-Goog-Channel-Expiration` | Channel expiration (human-readable) |
| `X-Goog-Resource-ID` | Opaque ID for the watched resource |
| `X-Goog-Resource-URI` | The resource URI being watched |
| `X-Goog-Resource-State` | `sync` (initial), `exists` (resource changed), `not_exists` (resource deleted) |
| `X-Goog-Message-Number` | Monotonically increasing message number |

**Important:** The notification body is empty. On receiving a notification, Esmer must call the events list endpoint (with `syncToken` for efficiency) to retrieve the actual changes.

#### Stopping a Watch Channel

##### POST /channels/stop

**Request Body:**

```json
{
  "id": "unique-channel-id-01",
  "resourceId": "resource-id-abc"
}
```

**Response:** 204 No Content.

**Esmer Recommendations:**

- Watch channels expire (max ~30 days). Implement a background job to renew channels before expiration.
- Use `syncToken` for incremental sync after receiving push notifications.
- The callback URL must be HTTPS with a valid SSL certificate.
- For mobile, route notifications through Esmer's backend server, then push to device via FCM/APNs.

### 1.7 Error Handling

| HTTP Code | Error Reason | Description | Esmer Action |
|---|---|---|---|
| 400 | `badRequest` | Malformed request | Fix request and retry |
| 401 | `authError` | Invalid credentials | Refresh OAuth token; if still failing, prompt re-auth |
| 403 | `rateLimitExceeded` | Per-user or per-project rate limit | Exponential backoff with jitter |
| 403 | `forbidden` | Insufficient permissions | Check scopes, prompt for additional permissions |
| 403 | `quotaExceeded` | Daily quota exhausted | Notify user; retry after midnight PT |
| 404 | `notFound` | Calendar or event not found | Inform user, refresh local cache |
| 409 | `conflict` | Version conflict (etag mismatch) | Re-fetch resource, merge changes, retry |
| 410 | `gone` | `syncToken` invalidated | Discard sync token, perform full sync |
| 429 | `rateLimitExceeded` | Too many requests | Exponential backoff |
| 500 | `backendError` | Google internal error | Retry with exponential backoff |
| 503 | `backendError` | Service temporarily unavailable | Retry with exponential backoff |

**Error Response Format:**

```json
{
  "error": {
    "errors": [
      {
        "domain": "calendar",
        "reason": "notFound",
        "message": "Not Found"
      }
    ],
    "code": 404,
    "message": "Not Found"
  }
}
```

### 1.8 Mobile-Specific Notes

- **PKCE Flow:** Esmer must use Authorization Code with PKCE for OAuth on mobile. Do not embed client secrets in the app binary. Use `S256` as the code challenge method.
- **Deep Linking:** Use `com.esmer.app://oauth/google/callback` as the custom URI scheme for the OAuth redirect. Register this in `AndroidManifest.xml` and `Info.plist`.
- **Token Storage:** Store access and refresh tokens in iOS Keychain (`SecItemAdd`) and Android Keystore. Never store tokens in `AsyncStorage` or `SharedPreferences` unencrypted.
- **Background Sync:** Use React Native Background Fetch to periodically sync calendar data. Respect iOS battery optimization constraints.
- **Offline Support:** Cache the last-fetched event list locally (encrypted SQLite). Display cached data when offline with a staleness indicator. Queue event creation/updates locally and sync when connectivity is restored.
- **Time Zones:** Always pass the user's device timezone (`Intl.DateTimeFormat().resolvedOptions().timeZone`) in requests. Never rely on the server default timezone.
- **Quick Add:** Use the `quickAdd` endpoint to support natural language event creation from Esmer's AI assistant, forwarding the user's spoken/typed command directly as the `text` parameter.
- **Google Meet Links:** Set `conferenceDataVersion=1` when creating events to auto-generate Google Meet links. This is critical for Esmer's meeting-creation flow.

---

## 2. Zoom (Zoom API v2)

### 2.1 Service Overview

The Zoom API v2 provides programmatic access to Zoom's video communication platform. It allows applications to create, manage, and delete meetings, retrieve meeting details and recordings, manage participants, and handle webinar operations.

**How Esmer uses Zoom:**

- Creating instant or scheduled Zoom meetings on behalf of users
- Retrieving meeting details and join URLs for sharing
- Updating meeting settings (time, duration, password, waiting room)
- Deleting/canceling meetings
- Listing a user's scheduled meetings
- Accessing cloud recording URLs after meetings
- Managing meeting registrants for registration-required meetings

### 2.2 Authentication

**Method:** OAuth 2.0 (JWT was deprecated by Zoom in June 2023 and must not be used)

| Property | Value |
|---|---|
| Authorization URL | `https://zoom.us/oauth/authorize` |
| Token URL | `https://zoom.us/oauth/token` |
| Revoke URL | `https://zoom.us/oauth/revoke` |
| Grant Type | `authorization_code` |
| PKCE Support | Yes |

**Required Scopes:**

| Scope | Description | Esmer Usage |
|---|---|---|
| `meeting:read` | View meeting information | Retrieve meeting details, list meetings |
| `meeting:write` | Create and manage meetings | Create, update, delete meetings |
| `recording:read` | View recording information | Access cloud recording URLs |
| `user:read` | View user information | Get user profile, timezone, settings |
| `webinar:read` | View webinar information | Read webinar data (if webinar features used) |
| `webinar:write` | Manage webinars | Create/update webinars (if used) |

**Note on Scope Granularity (Zoom Granular Scopes):**

Zoom has introduced granular scopes that replace the legacy scopes above. The granular equivalents are:

| Legacy Scope | Granular Scope |
|---|---|
| `meeting:read` | `meeting:read:list_meetings:admin`, `meeting:read:meeting:admin` |
| `meeting:write` | `meeting:write:meeting:admin`, `meeting:delete:meeting:admin` |
| `recording:read` | `cloud_recording:read:list_recording_files:admin` |
| `user:read` | `user:read:user:admin` |

**Recommendations for Esmer:**

- Create a **User-managed app** on the Zoom App Marketplace.
- Use OAuth 2.0 with PKCE for the mobile authorization flow.
- Store tokens in secure device storage (iOS Keychain / Android Keystore).
- Zoom access tokens expire in 1 hour. Refresh tokens expire after 90 days of inactivity.
- Implement proactive token refresh 5 minutes before access token expiry.
- When the refresh token itself is refreshed, Zoom returns a new refresh token. Always store the latest refresh token.

### 2.3 Base URL

```
https://api.zoom.us/v2
```

### 2.4 Rate Limits

Zoom implements tiered rate limiting:

| Limit Type | Value | Scope |
|---|---|---|
| Per-second (Light) | 10 requests/second | Per app, per user |
| Per-second (Medium) | 10 requests/second | Per app |
| Per-day (Heavy) | 100 requests/day | Per app, per user |
| Meeting creation | 100 requests/day | Per user |

**Rate Limit Categories:**

Zoom assigns each endpoint a rate limit category:

| Category | Limit | Examples |
|---|---|---|
| Light | 10 req/sec per user | GET meetings, GET meeting details |
| Medium | 10 req/sec total per app | List recordings |
| Heavy | 100 req/day per user | Create meeting, update meeting |
| Resource-intensive | 1 req/sec per user | Dashboard endpoints |

**Rate Limit Response Headers:**

| Header | Description |
|---|---|
| `X-RateLimit-Limit` | Max requests allowed in the window |
| `X-RateLimit-Remaining` | Requests remaining in the window |
| `X-RateLimit-Reset` | Unix timestamp when the window resets |
| `Retry-After` | Seconds to wait before retrying (on 429) |

### 2.5 API Endpoints

#### 2.5.1 Meetings Resource

##### POST /users/{userId}/meetings

**Description:** Creates a meeting for a user.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `userId` | string | Yes | The user's ID or email. Use `me` for the authenticated user. |

**Request Body:**

```json
{
  "topic": "Weekly Team Sync",
  "type": 2,
  "start_time": "2026-02-10T14:00:00Z",
  "duration": 60,
  "timezone": "America/New_York",
  "password": "abc123",
  "agenda": "Discuss Q1 roadmap and sprint priorities",
  "default_password": false,
  "pre_schedule": false,
  "settings": {
    "host_video": true,
    "participant_video": true,
    "join_before_host": false,
    "jbh_time": 0,
    "mute_upon_entry": true,
    "watermark": false,
    "use_pmi": false,
    "approval_type": 2,
    "registration_type": 1,
    "audio": "both",
    "auto_recording": "cloud",
    "enforce_login": false,
    "waiting_room": true,
    "meeting_authentication": false,
    "encryption_type": "enhanced_encryption",
    "breakout_room": {
      "enable": true,
      "rooms": [
        {
          "name": "Room 1",
          "participants": ["alice@example.com"]
        }
      ]
    },
    "alternative_hosts": "colleague@example.com",
    "allow_multiple_devices": false,
    "contact_name": "John Doe",
    "contact_email": "john@example.com"
  },
  "recurrence": {
    "type": 2,
    "repeat_interval": 1,
    "weekly_days": "2,4",
    "end_times": 12
  }
}
```

**Request Body Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `topic` | string | No | Meeting topic/title (max 200 chars) |
| `type` | integer | No | 1=Instant, 2=Scheduled, 3=Recurring (no fixed time), 8=Recurring (fixed time) |
| `start_time` | string | No | Start time (ISO 8601). Required for type 2 and 8. |
| `duration` | integer | No | Duration in minutes |
| `timezone` | string | No | Timezone (e.g., `America/New_York`) |
| `password` | string | No | Meeting password (max 10 chars) |
| `agenda` | string | No | Meeting description (max 2000 chars) |
| `settings` | object | No | Meeting settings (see below) |
| `recurrence` | object | No | Recurrence settings (for type 3 and 8) |

**Settings Object:**

| Parameter | Type | Description |
|---|---|---|
| `host_video` | boolean | Start with host video on |
| `participant_video` | boolean | Start with participant video on |
| `join_before_host` | boolean | Allow participants to join before host |
| `jbh_time` | integer | Minutes before host that participants can join (0, 5, 10) |
| `mute_upon_entry` | boolean | Mute participants upon entry |
| `waiting_room` | boolean | Enable waiting room |
| `auto_recording` | string | `local`, `cloud`, or `none` |
| `approval_type` | integer | 0=Auto approve, 1=Manually approve, 2=No registration required |
| `alternative_hosts` | string | Comma-separated email addresses |
| `audio` | string | `both`, `telephony`, `voip`, `thirdParty` |
| `encryption_type` | string | `enhanced_encryption` or `e2ee` |

**Recurrence Object:**

| Parameter | Type | Description |
|---|---|---|
| `type` | integer | 1=Daily, 2=Weekly, 3=Monthly |
| `repeat_interval` | integer | Interval (e.g., 1 = every week for weekly) |
| `weekly_days` | string | Days for weekly (1=Sun, 2=Mon, ..., 7=Sat) |
| `monthly_day` | integer | Day of month for monthly recurrence |
| `monthly_week` | integer | Week of month (-1=last, 1=first, etc.) |
| `monthly_week_day` | integer | Day of week for monthly-by-week |
| `end_times` | integer | Number of occurrences (max 365 daily, 52 weekly, 12 monthly) |
| `end_date_time` | string | End date (ISO 8601) |

**Response Body (201 Created):**

```json
{
  "uuid": "dGhpcyBpcyBhIHV1aWQ=",
  "id": 85746065234,
  "host_id": "abcdefghij1234567890",
  "host_email": "user@example.com",
  "topic": "Weekly Team Sync",
  "type": 2,
  "status": "waiting",
  "start_time": "2026-02-10T14:00:00Z",
  "duration": 60,
  "timezone": "America/New_York",
  "agenda": "Discuss Q1 roadmap and sprint priorities",
  "created_at": "2026-02-09T12:00:00Z",
  "start_url": "https://zoom.us/s/85746065234?zak=eyJ...",
  "join_url": "https://zoom.us/j/85746065234?pwd=abc123",
  "password": "abc123",
  "h323_password": "123456",
  "pstn_password": "123456",
  "encrypted_password": "encryptedString",
  "settings": {
    "host_video": true,
    "participant_video": true,
    "join_before_host": false,
    "mute_upon_entry": true,
    "waiting_room": true,
    "auto_recording": "cloud",
    "alternative_hosts": "colleague@example.com",
    "audio": "both"
  },
  "pre_schedule": false
}
```

**Error Codes:**

| Code | Description |
|---|---|
| 201 | Meeting created successfully |
| 300 | User {userId} has reached the maximum limit for creating and updating meetings |
| 400 | Bad request |
| 401 | Unauthorized (invalid token) |
| 404 | User not found or does not belong to this account |
| 429 | Rate limit exceeded |

---

##### GET /users/{userId}/meetings

**Description:** Lists all meetings for a user.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `userId` | string | Yes | User ID or email. Use `me` for authenticated user. |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | No | `scheduled`, `live`, `upcoming`, `upcoming_meetings`, `previous_meetings` (default `live`) |
| `page_size` | integer | No | Number of records per page (max 300, default 30) |
| `next_page_token` | string | No | Pagination token |
| `page_number` | integer | No | Page number (deprecated; use `next_page_token`) |
| `from` | date | No | Start date (yyyy-MM-dd) for `previous_meetings` |
| `to` | date | No | End date (yyyy-MM-dd) for `previous_meetings` |

**Response Body (200 OK):**

```json
{
  "page_count": 1,
  "page_number": 1,
  "page_size": 30,
  "total_records": 2,
  "next_page_token": "",
  "meetings": [
    {
      "uuid": "dGhpcyBpcyBhIHV1aWQ=",
      "id": 85746065234,
      "host_id": "abcdefghij1234567890",
      "topic": "Weekly Team Sync",
      "type": 2,
      "start_time": "2026-02-10T14:00:00Z",
      "duration": 60,
      "timezone": "America/New_York",
      "created_at": "2026-02-09T12:00:00Z",
      "join_url": "https://zoom.us/j/85746065234?pwd=abc123"
    }
  ]
}
```

---

##### GET /meetings/{meetingId}

**Description:** Retrieves a meeting's details.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `meetingId` | integer | Yes | The meeting ID |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `occurrence_id` | string | No | Occurrence ID for a specific recurring meeting instance |
| `show_previous_occurrences` | boolean | No | Include previous occurrences (default false) |

**Response Body (200 OK):**

```json
{
  "uuid": "dGhpcyBpcyBhIHV1aWQ=",
  "id": 85746065234,
  "host_id": "abcdefghij1234567890",
  "host_email": "user@example.com",
  "assistant_id": "",
  "topic": "Weekly Team Sync",
  "type": 2,
  "status": "waiting",
  "start_time": "2026-02-10T14:00:00Z",
  "duration": 60,
  "timezone": "America/New_York",
  "agenda": "Discuss Q1 roadmap and sprint priorities",
  "created_at": "2026-02-09T12:00:00Z",
  "start_url": "https://zoom.us/s/85746065234?zak=eyJ...",
  "join_url": "https://zoom.us/j/85746065234?pwd=abc123",
  "password": "abc123",
  "settings": {
    "host_video": true,
    "participant_video": true,
    "cn_meeting": false,
    "in_meeting": false,
    "join_before_host": false,
    "jbh_time": 0,
    "mute_upon_entry": true,
    "watermark": false,
    "use_pmi": false,
    "approval_type": 2,
    "audio": "both",
    "auto_recording": "cloud",
    "enforce_login": false,
    "waiting_room": true,
    "registrants_email_notification": true,
    "meeting_authentication": false
  },
  "occurrences": [
    {
      "occurrence_id": "1739188800000",
      "start_time": "2026-02-10T14:00:00Z",
      "duration": 60,
      "status": "available"
    }
  ]
}
```

---

##### PATCH /meetings/{meetingId}

**Description:** Updates a meeting's details.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `meetingId` | integer | Yes | The meeting ID |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `occurrence_id` | string | No | Occurrence ID to update a specific instance |

**Request Body:** Same schema as POST (only changed fields needed).

```json
{
  "topic": "Updated Meeting Title",
  "start_time": "2026-02-11T15:00:00Z",
  "duration": 45,
  "settings": {
    "waiting_room": false,
    "mute_upon_entry": false
  }
}
```

**Response:** 204 No Content on success.

**Error Codes:**

| Code | Description |
|---|---|
| 204 | Meeting updated successfully |
| 300 | Cannot update a meeting that has ended |
| 400 | Bad request |
| 404 | Meeting not found |
| 429 | Rate limit exceeded |

---

##### DELETE /meetings/{meetingId}

**Description:** Deletes a meeting.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `meetingId` | integer | Yes | The meeting ID |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `occurrence_id` | string | No | Delete a specific occurrence of a recurring meeting |
| `schedule_for_reminder` | boolean | No | Notify host and alternative host about cancellation (default true) |
| `cancel_meeting_reminder` | boolean | No | Send cancellation email to registrants (default false) |

**Response:** 204 No Content on success.

**Error Codes:**

| Code | Description |
|---|---|
| 204 | Meeting deleted successfully |
| 400 | Bad request |
| 404 | Meeting not found or has ended |
| 429 | Rate limit exceeded |

---

##### PUT /meetings/{meetingId}/status

**Description:** Updates the status of a meeting (end a meeting).

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `meetingId` | integer | Yes | The meeting ID |

**Request Body:**

```json
{
  "action": "end"
}
```

**Response:** 204 No Content.

---

#### 2.5.2 Meeting Registrants Resource

##### POST /meetings/{meetingId}/registrants

**Description:** Adds a registrant to a meeting (requires registration to be enabled).

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `meetingId` | integer | Yes | The meeting ID |

**Request Body:**

```json
{
  "email": "registrant@example.com",
  "first_name": "Jane",
  "last_name": "Doe",
  "address": "123 Main St",
  "city": "New York",
  "state": "NY",
  "zip": "10001",
  "country": "US",
  "phone": "+12125551234",
  "comments": "Looking forward to the meeting",
  "industry": "Technology",
  "job_title": "Engineer",
  "org": "Acme Corp",
  "no_of_employees": "51-100",
  "purchasing_time_frame": "Within a month",
  "role_in_purchase_process": "Decision Maker"
}
```

**Required Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `email` | string | Yes | Registrant's email |
| `first_name` | string | Yes | Registrant's first name |

**Response Body (201 Created):**

```json
{
  "id": 1234567890,
  "registrant_id": "abcdef123456",
  "start_time": "2026-02-10T14:00:00Z",
  "topic": "Weekly Team Sync",
  "join_url": "https://zoom.us/j/85746065234?pwd=abc123&tk=registrantToken"
}
```

---

##### GET /meetings/{meetingId}/registrants

**Description:** Lists registrants for a meeting.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `meetingId` | integer | Yes | The meeting ID |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `status` | string | No | `pending`, `approved`, `denied` (default `approved`) |
| `page_size` | integer | No | Max 300, default 30 |
| `page_number` | integer | No | Page number |
| `next_page_token` | string | No | Pagination token |

**Response Body (200 OK):**

```json
{
  "page_count": 1,
  "page_size": 30,
  "total_records": 1,
  "next_page_token": "",
  "registrants": [
    {
      "id": "abcdef123456",
      "email": "registrant@example.com",
      "first_name": "Jane",
      "last_name": "Doe",
      "create_time": "2026-02-09T12:00:00Z",
      "status": "approved",
      "join_url": "https://zoom.us/j/85746065234?pwd=abc123&tk=token"
    }
  ]
}
```

---

##### PUT /meetings/{meetingId}/registrants/status

**Description:** Approve, deny, or cancel registrants.

**Request Body:**

```json
{
  "action": "approve",
  "registrants": [
    { "id": "abcdef123456", "email": "registrant@example.com" }
  ]
}
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `action` | string | Yes | `approve`, `deny`, or `cancel` |
| `registrants` | array | Yes | Array of registrant objects with `id` and/or `email` |

**Response:** 204 No Content.

---

#### 2.5.3 Meeting Participants Resource

##### GET /past_meetings/{meetingUUID}/participants

**Description:** Retrieves participants from a past meeting.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `meetingUUID` | string | Yes | Meeting UUID (double-encode if it starts with `/` or contains `//`) |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `page_size` | integer | No | Max 300, default 30 |
| `next_page_token` | string | No | Pagination token |

**Response Body (200 OK):**

```json
{
  "page_count": 1,
  "page_size": 30,
  "total_records": 3,
  "next_page_token": "",
  "participants": [
    {
      "id": "participant-uuid",
      "name": "Alice Smith",
      "user_email": "alice@example.com",
      "join_time": "2026-02-10T14:00:30Z",
      "leave_time": "2026-02-10T14:58:00Z",
      "duration": 3450,
      "status": "in_meeting"
    }
  ]
}
```

---

#### 2.5.4 Cloud Recordings Resource

##### GET /users/{userId}/recordings

**Description:** Lists all cloud recordings for a user.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `userId` | string | Yes | User ID or email. Use `me` for authenticated user. |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `from` | date | No | Start date (yyyy-MM-dd). Default: last 30 days. |
| `to` | date | No | End date (yyyy-MM-dd) |
| `page_size` | integer | No | Max 300, default 30 |
| `next_page_token` | string | No | Pagination token |
| `trash` | boolean | No | List recordings in trash (default false) |
| `trash_type` | string | No | `meeting_recordings` or `recording_file` |

**Response Body (200 OK):**

```json
{
  "from": "2026-01-10",
  "to": "2026-02-09",
  "page_count": 1,
  "page_size": 30,
  "total_records": 1,
  "next_page_token": "",
  "meetings": [
    {
      "uuid": "dGhpcyBpcyBhIHV1aWQ=",
      "id": 85746065234,
      "host_id": "abcdefghij1234567890",
      "host_email": "user@example.com",
      "topic": "Weekly Team Sync",
      "type": 2,
      "start_time": "2026-02-10T14:00:00Z",
      "duration": 58,
      "total_size": 104857600,
      "recording_count": 3,
      "share_url": "https://zoom.us/rec/share/abc123",
      "recording_files": [
        {
          "id": "recording-file-uuid",
          "meeting_id": "dGhpcyBpcyBhIHV1aWQ=",
          "recording_start": "2026-02-10T14:00:30Z",
          "recording_end": "2026-02-10T14:58:00Z",
          "file_type": "MP4",
          "file_extension": "MP4",
          "file_size": 52428800,
          "play_url": "https://zoom.us/rec/play/abc123",
          "download_url": "https://zoom.us/rec/download/abc123",
          "status": "completed",
          "recording_type": "shared_screen_with_speaker_view"
        },
        {
          "id": "recording-file-uuid-2",
          "file_type": "CHAT",
          "file_extension": "TXT",
          "file_size": 1024,
          "download_url": "https://zoom.us/rec/download/chat123",
          "recording_type": "chat_file"
        },
        {
          "id": "recording-file-uuid-3",
          "file_type": "TRANSCRIPT",
          "file_extension": "VTT",
          "file_size": 4096,
          "download_url": "https://zoom.us/rec/download/transcript123",
          "recording_type": "audio_transcript"
        }
      ]
    }
  ]
}
```

---

##### GET /meetings/{meetingId}/recordings

**Description:** Returns all recordings for a specific meeting.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `meetingId` | string | Yes | Meeting ID or UUID |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `include_fields` | string | No | Set to `download_access_token` to get a download token |
| `ttl` | integer | No | Time-to-live for the download token (seconds) |

**Response Body (200 OK):** Same structure as individual meeting entry from the list endpoint.

---

##### DELETE /meetings/{meetingId}/recordings

**Description:** Deletes all recordings for a meeting.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `meetingId` | string | Yes | Meeting ID or UUID |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `action` | string | No | `trash` (move to trash, default) or `delete` (permanent) |

**Response:** 204 No Content on success.

---

#### 2.5.5 Users Resource

##### GET /users/me

**Description:** Returns the authenticated user's profile.

**Response Body (200 OK):**

```json
{
  "id": "abcdefghij1234567890",
  "first_name": "John",
  "last_name": "Doe",
  "email": "john@example.com",
  "type": 2,
  "role_name": "Owner",
  "pmi": 1234567890,
  "use_pmi": false,
  "timezone": "America/New_York",
  "verified": 1,
  "created_at": "2020-01-15T10:00:00Z",
  "last_login_time": "2026-02-09T08:00:00Z",
  "language": "en-US",
  "phone_number": "+12125551234",
  "status": "active",
  "pic_url": "https://zoom.us/p/abcdefghij1234567890",
  "personal_meeting_url": "https://zoom.us/j/1234567890",
  "account_id": "account-uuid-123",
  "account_number": 123456789,
  "plan_united_type": "2"
}
```

**User Type Values:**

| Value | Description |
|---|---|
| 1 | Basic |
| 2 | Licensed |
| 3 | On-Prem |
| 99 | None (pending invite) |

---

##### GET /users/{userId}/settings

**Description:** Returns a user's meeting settings.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `userId` | string | Yes | User ID or email. Use `me` for authenticated user. |

**Response Body (200 OK):**

```json
{
  "schedule_meeting": {
    "host_video": true,
    "participants_video": false,
    "audio_type": "both",
    "join_before_host": false,
    "jbh_time": 0,
    "force_pmi_jbh_password": true,
    "pstn_password_protected": true,
    "use_pmi_for_scheduled_meetings": false,
    "use_pmi_for_instant_meetings": false,
    "require_password_for_scheduling_new_meetings": true,
    "require_password_for_instant_meetings": true,
    "require_password_for_pmi_meetings": "all",
    "default_password_for_scheduled_meetings": "",
    "pmi_password": "123456",
    "mute_upon_entry": false,
    "upcoming_meeting_reminder": true
  },
  "in_meeting": {
    "e2e_encryption": false,
    "chat": true,
    "allow_participants_chat_with": 1,
    "allow_users_save_chats": 1,
    "private_chat": true,
    "auto_saving_chat": false,
    "entry_exit_chime": "all",
    "record_play_own_voice": false,
    "file_transfer": true,
    "feedback": true,
    "co_host": true,
    "polling": true,
    "attendee_on_hold": true,
    "annotation": true,
    "remote_control": true,
    "non_verbal_feedback": true,
    "breakout_room": true,
    "remote_support": true,
    "closed_caption": true,
    "virtual_background": true,
    "waiting_room": true
  },
  "recording": {
    "local_recording": true,
    "cloud_recording": true,
    "auto_recording": "local",
    "auto_delete": false,
    "auto_delete_cmr_days": 30
  }
}
```

---

### 2.6 Webhooks / Real-time

Zoom supports event-based webhooks through the Zoom App Marketplace.

#### Webhook Configuration

Webhooks are configured in the Zoom App Marketplace when creating or editing your app.

**Callback URL:** `https://api.esmer.app/webhooks/zoom`

**Verification:** Zoom sends a `challenge` request to verify the webhook endpoint. The endpoint must respond with:

```json
{
  "plainToken": "{received_plain_token}",
  "encryptedToken": "HMAC-SHA256({received_plain_token}, {your_secret_token})"
}
```

**Webhook Event Types (Meeting-related):**

| Event | Description |
|---|---|
| `meeting.created` | A meeting was created |
| `meeting.updated` | A meeting was updated |
| `meeting.deleted` | A meeting was deleted |
| `meeting.started` | A meeting started |
| `meeting.ended` | A meeting ended |
| `meeting.participant_joined` | A participant joined |
| `meeting.participant_left` | A participant left |
| `meeting.participant_joined_waiting_room` | Participant entered waiting room |
| `meeting.registration_created` | New registration submitted |
| `meeting.registration_approved` | Registration approved |
| `meeting.registration_cancelled` | Registration cancelled |
| `recording.completed` | Cloud recording finished processing |
| `recording.trashed` | Recording moved to trash |
| `recording.deleted` | Recording permanently deleted |
| `recording.recovered` | Recording recovered from trash |

**Webhook Payload Example (meeting.started):**

```json
{
  "event": "meeting.started",
  "event_ts": 1739188800000,
  "payload": {
    "account_id": "account-uuid-123",
    "operator": "user@example.com",
    "operator_id": "abcdefghij1234567890",
    "object": {
      "uuid": "dGhpcyBpcyBhIHV1aWQ=",
      "id": 85746065234,
      "host_id": "abcdefghij1234567890",
      "topic": "Weekly Team Sync",
      "type": 2,
      "start_time": "2026-02-10T14:00:00Z",
      "duration": 60,
      "timezone": "America/New_York"
    }
  }
}
```

**Webhook Headers:**

| Header | Description |
|---|---|
| `x-zm-request-timestamp` | Unix timestamp of the request |
| `x-zm-signature` | HMAC-SHA256 signature for verification |
| `authorization` | Verification token (deprecated in favor of signature) |

**Signature Verification:**

```
message = "v0:{x-zm-request-timestamp}:{request_body}"
signature = "v0=" + HMAC-SHA256(message, secret_token)
```

Compare the computed `signature` with the `x-zm-signature` header.

**Esmer Recommendations:**

- Verify all incoming webhooks using the `x-zm-signature` header.
- Use webhooks to update local meeting state in real-time (meeting started, ended, recording ready).
- Push notifications to mobile devices via FCM/APNs when meetings start or recordings become available.
- Implement a 3-second response time for webhook acknowledgment (return 200 OK quickly).

### 2.7 Error Handling

**Error Response Format:**

```json
{
  "code": 3001,
  "message": "Meeting does not exist: 85746065234."
}
```

**Common Error Codes:**

| HTTP Code | Zoom Code | Description | Esmer Action |
|---|---|---|---|
| 400 | 300 | Invalid request | Fix request parameters |
| 400 | 3000 | Cannot access meeting info | Check meeting permissions |
| 401 | 124 | Invalid access token | Refresh token; re-authenticate if needed |
| 401 | 401 | Access token is expired | Refresh using refresh token |
| 403 | 403 | Forbidden | Insufficient scopes or permissions |
| 404 | 1001 | User does not exist | Verify user ID |
| 404 | 3001 | Meeting does not exist | Meeting was deleted or invalid ID |
| 409 | 3003 | Cannot delete a meeting in progress | End meeting first, then delete |
| 429 | 429 | Rate limit exceeded | Respect `Retry-After` header; exponential backoff |

### 2.8 Mobile-Specific Notes

- **Join URL Handling:** When a user taps a meeting's join URL on mobile, Esmer should first attempt to open the native Zoom app via deep link (`zoomus://zoom.us/join?confno={meetingId}&pwd={password}`). If the Zoom app is not installed, fall back to the web join URL.
- **Start URL Security:** The `start_url` contains a ZAK (Zoom Access Key) token. Never cache or persist start URLs as the ZAK expires. Fetch a fresh meeting detail when the host needs to start.
- **PKCE OAuth:** Use Authorization Code with PKCE. Zoom supports PKCE for mobile apps.
- **Token Storage:** Same as Google Calendar -- store tokens in secure device keychain/keystore.
- **Meeting Creation UX:** Provide a simplified creation flow. Most users only need: topic, date/time, duration. Default the rest (waiting room on, mute on entry, auto cloud recording).
- **Recording Access:** When a `recording.completed` webhook fires, push a notification to the user's device with the play/download URL. Implement streaming playback in-app if possible (the `play_url` is a web URL).
- **Personal Meeting ID (PMI):** Allow users to quickly share their PMI link from Esmer. Fetch via `GET /users/me` and use the `personal_meeting_url` field.
- **Meeting UUID Encoding:** When using meeting UUIDs that start with `/` or contain `//`, double-encode the UUID for path parameters.

---

## 3. GoToWebinar (LogMeIn / GoTo API)

### 3.1 Service Overview

The GoToWebinar API (v2) provides programmatic access to GoTo's webinar platform. It supports creating and managing webinars, handling registrants, managing co-organizers and panelists, retrieving session data, and accessing attendee information.

**How Esmer uses GoToWebinar:**

- Creating and scheduling webinars on behalf of organizers
- Managing registrant sign-ups and approvals
- Listing upcoming webinars for organizer dashboards
- Retrieving attendee data and session analytics after webinars
- Managing co-organizers and panelists
- Updating webinar details (time, description, settings)

### 3.2 Authentication

**Method:** OAuth 2.0

| Property | Value |
|---|---|
| Authorization URL | `https://authentication.logmeininc.com/oauth/authorize` |
| Token URL | `https://authentication.logmeininc.com/oauth/token` |
| Revoke URL | N/A (tokens expire; no explicit revocation endpoint) |
| Grant Type | `authorization_code` |
| PKCE Support | Not documented; use standard authorization code flow with backend proxy |

**Required Scopes:**

GoToWebinar does not use granular OAuth scopes. Access is determined by the product entitlements of the authenticated user's account. When the user authorizes the app, they grant access to all GoToWebinar API operations their account supports.

| Scope | Description |
|---|---|
| (none required) | GoToWebinar does not require explicit scopes. The OAuth token grants access based on the user's product licenses. |

**Token Details:**

| Property | Value |
|---|---|
| Access token lifetime | 1 hour (3600 seconds) |
| Refresh token lifetime | 30 days |
| Token type | Bearer |

**Token Response:**

```json
{
  "access_token": "eyJhbGciOi...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "d1cp20yB3hrFAg...",
  "account_key": "1234567890",
  "organizer_key": "9876543210",
  "firstName": "John",
  "lastName": "Doe",
  "email": "john@example.com"
}
```

**Recommendations for Esmer:**

- Store the `organizer_key` from the token response. It is required as a path parameter for most API calls.
- Since GoTo does not document PKCE support, route the OAuth flow through Esmer's backend server to securely handle the client secret exchange.
- Refresh tokens proactively before the 1-hour expiration.
- The refresh token is single-use: each refresh returns a new refresh token. Always store the latest one.

### 3.3 Base URL

```
https://api.getgo.com/G2W/rest/v2
```

Legacy base URL (may still appear in older documentation):
```
https://api.getgo.com/G2W/rest
```

### 3.4 Rate Limits

| Limit Type | Value | Scope |
|---|---|---|
| Per-minute | 1,000 requests | Per organizer key |
| Per-day | 50,000 requests | Per account |
| Concurrent connections | 10 | Per organizer key |

**Notes:**

- Rate limit headers are returned in responses but are not formally documented.
- When rate limited, the API returns HTTP 429 with a `Retry-After` header.
- Implement exponential backoff starting at 1 second.

### 3.5 API Endpoints

#### 3.5.1 Webinars Resource

##### POST /organizers/{organizerKey}/webinars

**Description:** Creates a new webinar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key (from OAuth token response) |

**Request Body:**

```json
{
  "subject": "Introduction to AI in Healthcare",
  "description": "Learn how AI is transforming healthcare delivery and patient outcomes.",
  "times": [
    {
      "startTime": "2026-03-15T14:00:00Z",
      "endTime": "2026-03-15T15:30:00Z"
    }
  ],
  "timeZone": "America/New_York",
  "type": "single_session",
  "isPasswordProtected": false,
  "recordingAssetKey": "",
  "isOndemand": false,
  "experienceType": "CLASSIC"
}
```

**Request Body Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `subject` | string | Yes | Webinar title (max 128 chars) |
| `description` | string | No | Webinar description (max 2048 chars) |
| `times` | array | Yes | Array of time objects with `startTime` and `endTime` (ISO 8601 UTC) |
| `timeZone` | string | Yes | Timezone identifier |
| `type` | string | No | `single_session`, `series`, or `sequence` |
| `isPasswordProtected` | boolean | No | Require password to join (default false) |
| `isOndemand` | boolean | No | Make available on-demand after live session |
| `experienceType` | string | No | `CLASSIC`, `BROADCAST`, or `SIMULIVE` |

**Response Body (201 Created):**

```json
{
  "webinarKey": "1234567890123456"
}
```

**Error Codes:**

| Code | Description |
|---|---|
| 201 | Webinar created |
| 400 | Bad request (invalid parameters) |
| 401 | Unauthorized |
| 403 | Forbidden (insufficient license) |
| 409 | Conflict (scheduling conflict) |

---

##### GET /organizers/{organizerKey}/webinars

**Description:** Returns all webinars for an organizer.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `fromTime` | datetime | No | Start of time range (ISO 8601 UTC) |
| `toTime` | datetime | No | End of time range (ISO 8601 UTC) |
| `page` | integer | No | Page number (0-indexed) |
| `size` | integer | No | Page size (default 20, max 200) |

**Response Body (200 OK):**

```json
{
  "_embedded": {
    "webinars": [
      {
        "webinarKey": "1234567890123456",
        "webinarID": "987654321",
        "subject": "Introduction to AI in Healthcare",
        "description": "Learn how AI is transforming healthcare delivery.",
        "organizerKey": "9876543210",
        "organizerEmail": "john@example.com",
        "times": [
          {
            "startTime": "2026-03-15T14:00:00Z",
            "endTime": "2026-03-15T15:30:00Z"
          }
        ],
        "timeZone": "America/New_York",
        "type": "single_session",
        "registrationUrl": "https://attendee.gotowebinar.com/register/1234567890123456",
        "inSession": false,
        "numberOfRegistrants": 45,
        "isPasswordProtected": false,
        "experienceType": "CLASSIC",
        "recurrenceType": "single_session"
      }
    ]
  },
  "page": {
    "size": 20,
    "totalElements": 5,
    "totalPages": 1,
    "number": 0
  }
}
```

---

##### GET /organizers/{organizerKey}/webinars/{webinarKey}

**Description:** Returns details for a specific webinar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |

**Response Body (200 OK):**

```json
{
  "webinarKey": "1234567890123456",
  "webinarID": "987654321",
  "subject": "Introduction to AI in Healthcare",
  "description": "Learn how AI is transforming healthcare delivery and patient outcomes.",
  "organizerKey": "9876543210",
  "organizerEmail": "john@example.com",
  "times": [
    {
      "startTime": "2026-03-15T14:00:00Z",
      "endTime": "2026-03-15T15:30:00Z"
    }
  ],
  "timeZone": "America/New_York",
  "type": "single_session",
  "registrationUrl": "https://attendee.gotowebinar.com/register/1234567890123456",
  "inSession": false,
  "impromptu": false,
  "isPasswordProtected": false,
  "numberOfRegistrants": 45,
  "numberOfRegistrantsAbandoned": 2,
  "experienceType": "CLASSIC",
  "recurrenceType": "single_session"
}
```

---

##### PUT /organizers/{organizerKey}/webinars/{webinarKey}

**Description:** Updates an existing webinar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `notifyParticipants` | boolean | No | Send notification emails to registrants about changes (default false) |

**Request Body:**

```json
{
  "subject": "Updated: AI in Healthcare Webinar",
  "description": "Updated description with new agenda items.",
  "times": [
    {
      "startTime": "2026-03-15T15:00:00Z",
      "endTime": "2026-03-15T16:30:00Z"
    }
  ],
  "timeZone": "America/New_York",
  "isPasswordProtected": false
}
```

**Response:** 204 No Content on success.

---

##### DELETE /organizers/{organizerKey}/webinars/{webinarKey}

**Description:** Cancels and deletes a webinar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `sendCancellationEmails` | boolean | No | Send cancellation emails to registrants (default false) |

**Response:** 204 No Content on success.

---

#### 3.5.2 Registrants Resource

##### POST /organizers/{organizerKey}/webinars/{webinarKey}/registrants

**Description:** Registers a person for a webinar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `accept` | string | No | Set to `application/json` |
| `resendConfirmation` | boolean | No | Resend confirmation email if already registered (default false) |

**Request Body:**

```json
{
  "firstName": "Jane",
  "lastName": "Smith",
  "email": "jane@example.com",
  "source": "Esmer App",
  "address": "456 Oak Ave",
  "city": "San Francisco",
  "state": "CA",
  "zipCode": "94102",
  "country": "US",
  "phone": "+14155551234",
  "organization": "Acme Corp",
  "jobTitle": "VP Engineering",
  "questionsAndComments": "Looking forward to learning about AI applications.",
  "industry": "Technology",
  "numberOfEmployees": "101-500",
  "purchasingTimeFrame": "Within 3 months",
  "purchasingRole": "Influencer",
  "responses": [
    {
      "questionKey": 12345,
      "responseText": "Custom answer to registration question"
    }
  ]
}
```

**Required Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `firstName` | string | Yes | Registrant's first name |
| `lastName` | string | Yes | Registrant's last name |
| `email` | string | Yes | Registrant's email |

**Response Body (201 Created):**

```json
{
  "registrantKey": "5678901234567890",
  "joinUrl": "https://attendee.gotowebinar.com/register/1234567890123456",
  "asset": null,
  "status": "APPROVED"
}
```

---

##### GET /organizers/{organizerKey}/webinars/{webinarKey}/registrants

**Description:** Returns all registrants for a webinar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `page` | integer | No | Page number (0-indexed) |
| `size` | integer | No | Page size (default 20, max 200) |

**Response Body (200 OK):**

```json
{
  "_embedded": {
    "registrants": [
      {
        "registrantKey": "5678901234567890",
        "firstName": "Jane",
        "lastName": "Smith",
        "email": "jane@example.com",
        "status": "APPROVED",
        "registrationDate": "2026-02-09T12:00:00Z",
        "joinUrl": "https://attendee.gotowebinar.com/join/1234567890123456",
        "timeZone": "America/Los_Angeles",
        "source": "Esmer App",
        "organization": "Acme Corp",
        "jobTitle": "VP Engineering"
      }
    ]
  },
  "page": {
    "size": 20,
    "totalElements": 45,
    "totalPages": 3,
    "number": 0
  }
}
```

---

##### GET /organizers/{organizerKey}/webinars/{webinarKey}/registrants/{registrantKey}

**Description:** Returns a specific registrant's details.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |
| `registrantKey` | long | Yes | The registrant's key |

**Response Body (200 OK):**

```json
{
  "registrantKey": "5678901234567890",
  "firstName": "Jane",
  "lastName": "Smith",
  "email": "jane@example.com",
  "status": "APPROVED",
  "registrationDate": "2026-02-09T12:00:00Z",
  "joinUrl": "https://attendee.gotowebinar.com/join/1234567890123456",
  "timeZone": "America/Los_Angeles",
  "source": "Esmer App",
  "organization": "Acme Corp",
  "jobTitle": "VP Engineering",
  "phone": "+14155551234",
  "country": "US",
  "state": "CA",
  "city": "San Francisco",
  "zipCode": "94102",
  "address": "456 Oak Ave",
  "questionsAndComments": "Looking forward to learning about AI applications.",
  "responses": [
    {
      "questionKey": 12345,
      "question": "What is your primary interest?",
      "responseText": "Custom answer to registration question"
    }
  ]
}
```

---

##### DELETE /organizers/{organizerKey}/webinars/{webinarKey}/registrants/{registrantKey}

**Description:** Removes a registrant from a webinar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |
| `registrantKey` | long | Yes | The registrant's key |

**Response:** 204 No Content on success.

---

#### 3.5.3 Attendees Resource

##### GET /organizers/{organizerKey}/webinars/{webinarKey}/attendees

**Description:** Returns all attendees for a completed webinar (across all sessions).

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `page` | integer | No | Page number (0-indexed) |
| `size` | integer | No | Page size (default 20, max 200) |

**Response Body (200 OK):**

```json
{
  "_embedded": {
    "attendeeParticipantResponses": [
      {
        "registrantKey": "5678901234567890",
        "firstName": "Jane",
        "lastName": "Smith",
        "email": "jane@example.com",
        "attendanceTimeInSeconds": 5250,
        "sessionKey": "1111111111111111",
        "joinTime": "2026-03-15T14:02:00Z",
        "leaveTime": "2026-03-15T15:29:30Z"
      }
    ]
  },
  "page": {
    "size": 20,
    "totalElements": 38,
    "totalPages": 2,
    "number": 0
  }
}
```

---

##### GET /organizers/{organizerKey}/webinars/{webinarKey}/sessions/{sessionKey}/attendees

**Description:** Returns attendees for a specific session.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |
| `sessionKey` | long | Yes | The session's key |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `page` | integer | No | Page number |
| `size` | integer | No | Page size |

**Response Body (200 OK):** Same structure as webinar-level attendees.

---

##### GET /organizers/{organizerKey}/webinars/{webinarKey}/sessions/{sessionKey}/attendees/{registrantKey}

**Description:** Returns a specific attendee's details for a session.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |
| `sessionKey` | long | Yes | The session's key |
| `registrantKey` | long | Yes | The registrant/attendee's key |

**Response Body (200 OK):**

```json
{
  "registrantKey": "5678901234567890",
  "firstName": "Jane",
  "lastName": "Smith",
  "email": "jane@example.com",
  "attendanceTimeInSeconds": 5250,
  "joinTime": "2026-03-15T14:02:00Z",
  "leaveTime": "2026-03-15T15:29:30Z",
  "attendedSessionCount": 1
}
```

---

#### 3.5.4 Co-Organizers Resource

##### POST /organizers/{organizerKey}/webinars/{webinarKey}/coorganizers

**Description:** Adds co-organizers to a webinar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |

**Request Body:**

```json
[
  {
    "external": true,
    "givenName": "Alice",
    "email": "alice@example.com",
    "organizerKey": ""
  },
  {
    "external": false,
    "organizerKey": "1122334455667788"
  }
]
```

**Request Array Item Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `external` | boolean | Yes | Whether the co-organizer is external to the GoTo account |
| `organizerKey` | string | Conditional | Required if `external` is false |
| `givenName` | string | Conditional | Required if `external` is true |
| `email` | string | Conditional | Required if `external` is true |

**Response Body (201 Created):**

```json
[
  {
    "memberKey": "3344556677889900",
    "joinLink": "https://global.gotowebinar.com/join/1234567890123456/3344556677889900",
    "email": "alice@example.com",
    "givenName": "Alice",
    "external": true
  }
]
```

---

##### GET /organizers/{organizerKey}/webinars/{webinarKey}/coorganizers

**Description:** Returns all co-organizers for a webinar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |

**Response Body (200 OK):**

```json
[
  {
    "memberKey": "3344556677889900",
    "joinLink": "https://global.gotowebinar.com/join/1234567890123456/3344556677889900",
    "email": "alice@example.com",
    "givenName": "Alice",
    "surname": "Johnson",
    "external": true
  }
]
```

---

##### DELETE /organizers/{organizerKey}/webinars/{webinarKey}/coorganizers/{coorganizerKey}

**Description:** Removes a co-organizer from a webinar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |
| `coorganizerKey` | long | Yes | The co-organizer's member key |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `external` | boolean | Yes | Whether the co-organizer is external |

**Response:** 204 No Content on success.

---

##### POST /organizers/{organizerKey}/webinars/{webinarKey}/coorganizers/{coorganizerKey}/resendInvitation

**Description:** Resends the invitation email to a co-organizer.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |
| `coorganizerKey` | long | Yes | The co-organizer's member key |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `external` | boolean | Yes | Whether the co-organizer is external |

**Response:** 204 No Content on success.

---

#### 3.5.5 Panelists Resource

##### POST /organizers/{organizerKey}/webinars/{webinarKey}/panelists

**Description:** Adds panelists to a webinar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |

**Request Body:**

```json
[
  {
    "name": "Dr. Robert Chen",
    "email": "robert.chen@example.com"
  },
  {
    "name": "Prof. Sarah Lee",
    "email": "sarah.lee@example.com"
  }
]
```

**Request Array Item Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Panelist's display name |
| `email` | string | Yes | Panelist's email address |

**Response Body (201 Created):**

```json
[
  {
    "memberKey": "4455667788990011",
    "joinLink": "https://global.gotowebinar.com/join/1234567890123456/4455667788990011",
    "name": "Dr. Robert Chen",
    "email": "robert.chen@example.com"
  }
]
```

---

##### GET /organizers/{organizerKey}/webinars/{webinarKey}/panelists

**Description:** Returns all panelists for a webinar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |

**Response Body (200 OK):**

```json
[
  {
    "memberKey": "4455667788990011",
    "joinLink": "https://global.gotowebinar.com/join/...",
    "name": "Dr. Robert Chen",
    "email": "robert.chen@example.com"
  }
]
```

---

##### DELETE /organizers/{organizerKey}/webinars/{webinarKey}/panelists/{panelistKey}

**Description:** Removes a panelist from a webinar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |
| `panelistKey` | long | Yes | The panelist's member key |

**Response:** 204 No Content on success.

---

##### POST /organizers/{organizerKey}/webinars/{webinarKey}/panelists/{panelistKey}/resendInvitation

**Description:** Resends the invitation email to a panelist.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |
| `panelistKey` | long | Yes | The panelist's member key |

**Response:** 204 No Content on success.

---

#### 3.5.6 Sessions Resource

##### GET /organizers/{organizerKey}/webinars/{webinarKey}/sessions

**Description:** Returns all sessions for a webinar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `fromTime` | datetime | No | Start of time range (ISO 8601 UTC) |
| `toTime` | datetime | No | End of time range (ISO 8601 UTC) |
| `page` | integer | No | Page number (0-indexed) |
| `size` | integer | No | Page size (default 20, max 200) |

**Response Body (200 OK):**

```json
{
  "_embedded": {
    "sessionInfoResources": [
      {
        "sessionKey": "1111111111111111",
        "webinarKey": "1234567890123456",
        "webinarID": "987654321",
        "startTime": "2026-03-15T14:00:00Z",
        "endTime": "2026-03-15T15:30:00Z",
        "registrantsAttended": 38,
        "registrantsAbandoned": 7,
        "sessionPerformance": {
          "attendance": {
            "registrantCount": 45,
            "percentageAttendance": 84.4,
            "averageAttendanceTimeSeconds": 4950
          },
          "polls": {
            "pollCount": 3,
            "questionsAsked": 12
          },
          "interest": {
            "averageInterestRating": 72.5,
            "averageAttentivenessScore": 85.2
          }
        }
      }
    ]
  },
  "page": {
    "size": 20,
    "totalElements": 1,
    "totalPages": 1,
    "number": 0
  }
}
```

---

##### GET /organizers/{organizerKey}/webinars/{webinarKey}/sessions/{sessionKey}

**Description:** Returns details for a specific session.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |
| `sessionKey` | long | Yes | The session's key |

**Response Body (200 OK):** Same structure as individual session entry from the list endpoint.

---

##### GET /organizers/{organizerKey}/webinars/{webinarKey}/sessions/{sessionKey}/performance

**Description:** Returns session performance/analytics data.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |
| `sessionKey` | long | Yes | The session's key |

**Response Body (200 OK):**

```json
{
  "attendance": {
    "registrantCount": 45,
    "percentageAttendance": 84.4,
    "averageAttendanceTimeSeconds": 4950
  },
  "polls": {
    "pollCount": 3,
    "questionsAsked": 12
  },
  "interest": {
    "averageInterestRating": 72.5,
    "averageAttentivenessScore": 85.2
  }
}
```

---

#### 3.5.7 Webinar Audio Resource

##### GET /organizers/{organizerKey}/webinars/{webinarKey}/audio

**Description:** Returns audio/conferencing information for a webinar.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organizerKey` | long | Yes | The organizer's key |
| `webinarKey` | long | Yes | The webinar's key |

**Response Body (200 OK):**

```json
{
  "confCallNumbers": {
    "US": {
      "toll": "+1-888-123-4567",
      "tollFree": "+1-800-123-4567"
    },
    "GB": {
      "toll": "+44-20-1234-5678",
      "tollFree": "+44-800-123-4567"
    }
  },
  "type": "VOIP",
  "privateInfo": "Access code: 987-654-321"
}
```

---

##### POST /organizers/{organizerKey}/webinars/{webinarKey}/audio

**Description:** Updates audio information for a webinar.

**Request Body:**

```json
{
  "type": "PSTN",
  "pstnInfo": {
    "tollCountries": ["US", "GB"],
    "tollFreeCountries": ["US"]
  }
}
```

---

#### 3.5.8 Account Webinars (Cross-Organizer)

##### GET /accounts/{accountKey}/webinars

**Description:** Returns all webinars across all organizers in an account.

**Path Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `accountKey` | long | Yes | The account key |

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `fromTime` | datetime | No | Start of time range (ISO 8601 UTC) |
| `toTime` | datetime | No | End of time range (ISO 8601 UTC) |
| `page` | integer | No | Page number |
| `size` | integer | No | Page size |

**Response Body (200 OK):** Same structure as organizer-level webinar listing.

---

### 3.6 Webhooks / Real-time

GoToWebinar supports webhooks through the GoTo Developer Center.

#### Webhook Configuration

Webhooks are configured in the GoTo Developer Center when creating or editing your OAuth app.

**Callback URL:** `https://api.esmer.app/webhooks/gotowebinar`

**Supported Event Types:**

| Event | Description |
|---|---|
| `webinar.created` | A new webinar was created |
| `webinar.changed` | A webinar was updated |
| `webinar.deleted` | A webinar was cancelled/deleted |
| `registrant.added` | A new registrant signed up |
| `registrant.removed` | A registrant was removed |
| `attendee.joined` | An attendee joined a live session |
| `attendee.left` | An attendee left a live session |
| `session.started` | A webinar session started |
| `session.ended` | A webinar session ended |

**Webhook Payload Example (registrant.added):**

```json
{
  "eventName": "registrant.added",
  "eventTime": "2026-02-09T14:30:00Z",
  "productKey": "g2w",
  "webhookKey": "webhook-key-123",
  "data": {
    "registrantKey": "5678901234567890",
    "webinarKey": "1234567890123456",
    "organizerKey": "9876543210",
    "registrant": {
      "firstName": "Jane",
      "lastName": "Smith",
      "email": "jane@example.com",
      "registrationDate": "2026-02-09T14:30:00Z",
      "status": "APPROVED"
    }
  }
}
```

**Webhook Security:**

- GoTo sends a `X-Webhook-Signature` header containing an HMAC-SHA256 signature.
- Compute `HMAC-SHA256(request_body, webhook_secret)` and compare with the header value.
- The `webhook_secret` is provided when you configure the webhook in the Developer Center.

**Esmer Recommendations:**

- Use webhooks to track registrant sign-ups in real-time and update dashboards.
- Notify organizers on mobile when sessions start/end and when key attendance thresholds are reached.
- Route webhook events through Esmer's backend, then push to devices via FCM/APNs.

### 3.7 Error Handling

**Error Response Format:**

```json
{
  "errorCode": "InvalidRequest",
  "description": "The request could not be processed because it contained invalid parameters.",
  "incident": "12345678-abcd-efgh-ijkl-123456789012"
}
```

**Common Error Codes:**

| HTTP Code | Error Code | Description | Esmer Action |
|---|---|---|---|
| 400 | `InvalidRequest` | Malformed request or invalid parameters | Fix request body and retry |
| 400 | `ValidationError` | Request body validation failed | Check required fields and format |
| 401 | `InvalidToken` | Access token is invalid or expired | Refresh token; re-authenticate if needed |
| 403 | `Forbidden` | Insufficient permissions or license | Check user's GoToWebinar license status |
| 404 | `NotFound` | Resource not found | Verify webinar/registrant/session keys |
| 409 | `Conflict` | Scheduling conflict or duplicate registration | Inform user of conflict; suggest alternatives |
| 429 | `RateLimitExceeded` | Too many requests | Respect `Retry-After` header; exponential backoff |
| 500 | `InternalError` | GoTo server error | Retry with exponential backoff; log `incident` ID |
| 503 | `ServiceUnavailable` | Service temporarily unavailable | Retry with exponential backoff |

### 3.8 Mobile-Specific Notes

- **No PKCE Support:** GoTo's OAuth does not formally document PKCE support. Esmer should proxy the OAuth token exchange through its backend server to protect the client secret. The mobile app initiates the authorization URL in a system browser, receives the callback via deep link, and sends the authorization code to the Esmer backend, which exchanges it for tokens.
- **Deep Link Scheme:** Use `com.esmer.app://oauth/gotowebinar/callback` as the redirect URI registered in the GoTo Developer Center.
- **Organizer Key Storage:** The `organizer_key` is returned in the token response and is required for nearly every API call. Store it alongside the tokens in secure storage.
- **Webinar Join Links:** When sharing webinar registration or join links, use the native share sheet (`Share.share()` in React Native) for a native feel. Detect if the GoTo app is installed and offer to open directly.
- **Registration Flow:** Esmer can pre-fill registrant information from the user's profile to streamline the registration process. Only `firstName`, `lastName`, and `email` are required.
- **Session Analytics:** Session performance data is only available after a session ends. Display an "analytics pending" state until the session has completed.
- **Offline Consideration:** Cache upcoming webinar lists and registrant counts locally. Queue registration requests when offline and sync when connectivity returns.
- **Rate Limit Strategy:** GoToWebinar's rate limits (1,000/min per organizer) are generous. Standard retry logic should suffice.

---

## Cross-Service Comparison Matrix

### Authentication Comparison

| Feature | Google Calendar | Zoom | GoToWebinar |
|---|---|---|---|
| Auth Method | OAuth 2.0 | OAuth 2.0 | OAuth 2.0 |
| PKCE Support | Yes | Yes | Not documented |
| Access Token TTL | 1 hour | 1 hour | 1 hour |
| Refresh Token TTL | No expiration (revocable) | 90 days (if unused) | 30 days |
| Scopes Required | Yes (granular) | Yes (granular) | No (license-based) |
| Service Account | Not supported | Server-to-Server OAuth | Not supported |
| Token Refresh Returns New Refresh | No (same token) | Yes | Yes |

### Rate Limits Comparison

| Metric | Google Calendar | Zoom | GoToWebinar |
|---|---|---|---|
| Per-second limit | 5 req/sec per user | 10 req/sec per user (Light) | Not specified |
| Per-minute limit | 300 req/min per user | 600 req/min (Light) | 1,000 req/min per organizer |
| Per-day limit | 1,000,000 per project | 100/day (Heavy category) | 50,000 per account |
| Rate limit headers | No (uses retry) | Yes (`X-RateLimit-*`) | Limited |
| Retry-After header | No | Yes | Yes (on 429) |

### Capabilities Comparison

| Feature | Google Calendar | Zoom | GoToWebinar |
|---|---|---|---|
| Create Event/Meeting | Yes | Yes | Yes (webinar) |
| Update Event/Meeting | Yes (PUT/PATCH) | Yes (PATCH) | Yes (PUT) |
| Delete Event/Meeting | Yes | Yes | Yes |
| List Events/Meetings | Yes (with filtering) | Yes (by type) | Yes (by time range) |
| Recurring Events | Yes (RRULE) | Yes (recurrence object) | Yes (series/sequence) |
| Attendee Management | Yes (inline in event) | Yes (registrants) | Yes (registrants) |
| Free/Busy Query | Yes | No | No |
| Video Conferencing | Google Meet (auto-link) | Native (join_url) | Native (join link) |
| Recordings | No | Yes (cloud recordings) | No (via separate GoTo API) |
| Webhooks/Push | Yes (watch channels) | Yes (app-level webhooks) | Yes (app-level webhooks) |
| Natural Language Create | Yes (quickAdd) | No | No |
| Panelist Management | N/A | N/A | Yes |
| Co-Organizer Mgmt | N/A | Yes (alternative_hosts) | Yes (dedicated resource) |
| Session Analytics | No | Yes (dashboard APIs) | Yes (performance endpoint) |
| Batch Operations | Yes (batch endpoint) | No | No |
| Incremental Sync | Yes (syncToken) | No | No |

### Endpoint Count Summary

| Service | Resource Groups | Total Endpoints |
|---|---|---|
| Google Calendar | 6 (CalendarList, Calendars, Events, FreeBusy, ACL, Settings, Colors) | ~25 |
| Zoom | 5 (Meetings, Registrants, Participants, Recordings, Users) | ~15 |
| GoToWebinar | 8 (Webinars, Registrants, Attendees, Co-Organizers, Panelists, Sessions, Audio, Account) | ~22 |

### Mobile Implementation Priority

| Priority | Google Calendar | Zoom | GoToWebinar |
|---|---|---|---|
| P0 (Launch) | Event Create, Event List, Free/Busy | Meeting Create, Meeting List, Get Meeting | Webinar List, Webinar Get |
| P1 (Fast Follow) | Event Update, Event Delete, Attendees | Meeting Update, Meeting Delete, Recordings | Registrant Create, Registrant List |
| P2 (Enhancement) | Calendar List, Quick Add, Watch Channels | Registrants, Participants, User Settings | Co-Organizers, Panelists, Sessions, Analytics |

### Error Handling Strategy Summary

| Scenario | Strategy |
|---|---|
| 401 Unauthorized | Attempt token refresh. If refresh fails, prompt user to re-authenticate. |
| 403 Forbidden | Check if scopes are sufficient. Show permission request dialog if needed. |
| 404 Not Found | Invalidate local cache for the resource. Show "not found" message. |
| 429 Rate Limited | Exponential backoff (1s, 2s, 4s, 8s, 16s, 32s max). Respect `Retry-After` header if present. |
| 500/503 Server Error | Retry up to 3 times with exponential backoff. If persistent, show degraded state and queue for later. |
| Network Offline | Queue write operations locally. Serve cached data for reads. Sync on reconnect. |

---

*End of Category 02 - Calendar & Scheduling specification.*
