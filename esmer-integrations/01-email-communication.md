# Esmer (ESMO) API Integration Specification: Email & Communication

> **Category:** 01 - Email & Communication
> **Version:** 1.0.0
> **Last Updated:** 2026-02-09
> **Status:** Draft

---

## Table of Contents

- [1. Gmail (Google API)](#1-gmail-google-api)
- [2. Microsoft Outlook (Microsoft Graph API)](#2-microsoft-outlook-microsoft-graph-api)
- [3. Slack](#3-slack)
- [4. Microsoft Teams (Microsoft Graph API)](#4-microsoft-teams-microsoft-graph-api)
- [5. Telegram (Bot API)](#5-telegram-bot-api)
- [6. WhatsApp (Cloud API / Business Platform)](#6-whatsapp-cloud-api--business-platform)
- [7. Discord (Bot API)](#7-discord-bot-api)
- [8. Google Chat (Google API)](#8-google-chat-google-api)
- [9. Mattermost](#9-mattermost)
- [10. Matrix (Synapse API)](#10-matrix-synapse-api)
- [11. Zulip](#11-zulip)
- [12. Webex (Cisco Webex API)](#12-webex-cisco-webex-api)
- [Cross-Service Comparison Matrix](#cross-service-comparison-matrix)

---

## 1. Gmail (Google API)

### 1.1 Service Overview

Gmail is Google's email service, accessed programmatically via the Gmail REST API v1. Esmer delegates email triage, composition, reply, labeling, thread management, and draft creation on behalf of users. The API provides full access to a user's mailbox including messages, threads, labels, drafts, and settings.

### 1.2 Authentication

**Method:** OAuth 2.0 (recommended) or Service Account (limited, requires domain-wide delegation)

| Property | Value |
|---|---|
| Authorization URL | `https://accounts.google.com/o/oauth2/v2/auth` |
| Token URL | `https://oauth2.googleapis.com/token` |
| Revoke URL | `https://oauth2.googleapis.com/revoke` |
| Grant Type | `authorization_code` |
| PKCE Support | Yes (recommended for mobile) |

**Required Scopes:**

| Scope | Purpose |
|---|---|
| `https://www.googleapis.com/auth/gmail.readonly` | Read messages, threads, labels |
| `https://www.googleapis.com/auth/gmail.send` | Send and reply to messages |
| `https://www.googleapis.com/auth/gmail.compose` | Create, read, update, and delete drafts; send messages |
| `https://www.googleapis.com/auth/gmail.modify` | All read/write operations except permanent delete |
| `https://www.googleapis.com/auth/gmail.labels` | Create, read, update, and delete labels |
| `https://www.googleapis.com/auth/gmail.metadata` | Read message metadata (headers, IDs) only |

**Esmer Recommendation:** Use `gmail.modify` as the primary scope. This covers reading, sending, label management, trash/untrash, and marking read/unread without granting permanent delete. Add `gmail.compose` if drafts are needed. Avoid `https://mail.google.com/` (full access) unless absolutely necessary.

### 1.3 Base URL

```
https://gmail.googleapis.com/gmail/v1
```

### 1.4 Rate Limits

| Limit | Value |
|---|---|
| Per-user rate limit | 250 quota units per second per user |
| Daily sending limit (consumer) | 500 emails/day |
| Daily sending limit (Workspace) | 2,000 emails/day |
| messages.list | 5 quota units per call |
| messages.get | 5 quota units per call |
| messages.send | 100 quota units per call |
| messages.modify | 5 quota units per call |
| messages.delete | 10 quota units per call |
| threads.list | 10 quota units per call |
| threads.get | 10 quota units per call |
| labels.list | 1 quota unit per call |
| drafts.create | 10 quota units per call |
| Batch requests | Up to 100 calls per batch; each sub-request counted individually |
| Concurrent requests | No hard limit, but subject to per-user quota |

### 1.5 API Endpoints

#### Messages

**List Messages**

```
GET /users/{userId}/messages
```

Retrieves a list of message IDs in the user's mailbox.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `userId` | string | Yes | The user's email address or `me` for authenticated user |
| `maxResults` | integer | No | Maximum number of messages to return (default 100, max 500) |
| `pageToken` | string | No | Page token from previous response for pagination |
| `q` | string | No | Gmail search query (same syntax as Gmail search box) |
| `labelIds` | string[] | No | Only return messages with all specified label IDs |
| `includeSpamTrash` | boolean | No | Include messages from SPAM and TRASH (default false) |

Response:
```json
{
  "messages": [
    { "id": "18d1a2b3c4d5e6f7", "threadId": "18d1a2b3c4d5e6f0" }
  ],
  "nextPageToken": "token_string",
  "resultSizeEstimate": 150
}
```

Error codes: `400` Invalid query, `401` Unauthorized, `403` Insufficient permissions, `429` Rate limit exceeded, `500` Backend error.

---

**Get Message**

```
GET /users/{userId}/messages/{id}
```

Retrieves a specific message by ID.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `userId` | string | Yes | `me` or email address |
| `id` | string | Yes | The message ID |
| `format` | string | No | `full` (default), `metadata`, `minimal`, or `raw` |
| `metadataHeaders` | string[] | No | Headers to include when format is `metadata` |

Response (format=full):
```json
{
  "id": "18d1a2b3c4d5e6f7",
  "threadId": "18d1a2b3c4d5e6f0",
  "labelIds": ["INBOX", "UNREAD"],
  "snippet": "Preview text of the message...",
  "historyId": "12345",
  "internalDate": "1706000000000",
  "payload": {
    "mimeType": "multipart/alternative",
    "headers": [
      { "name": "From", "value": "sender@example.com" },
      { "name": "To", "value": "recipient@example.com" },
      { "name": "Subject", "value": "Hello" },
      { "name": "Date", "value": "Wed, 24 Jan 2026 10:00:00 -0800" }
    ],
    "body": { "size": 0 },
    "parts": [
      {
        "mimeType": "text/plain",
        "body": { "size": 100, "data": "base64url_encoded_body" }
      },
      {
        "mimeType": "text/html",
        "body": { "size": 250, "data": "base64url_encoded_html" }
      }
    ]
  },
  "sizeEstimate": 5000
}
```

Error codes: `400` Bad request, `401` Unauthorized, `404` Message not found, `429` Rate limit.

---

**Send Message**

```
POST /users/{userId}/messages/send
```

Sends an email message. The request body must be a base64url-encoded RFC 2822 formatted message.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `userId` | string | Yes | `me` or email address |

Request body (simple upload):
```json
{
  "raw": "base64url_encoded_rfc2822_message"
}
```

For constructing the raw message, the RFC 2822 content should include headers:
```
From: sender@example.com
To: recipient@example.com
Subject: Hello
Content-Type: text/plain; charset="UTF-8"

Message body here
```

Response: Returns the sent Message resource (same schema as Get Message).

Error codes: `400` Invalid message format, `401` Unauthorized, `403` Sending limit exceeded, `429` Rate limit.

---

**Modify Message (Add/Remove Labels, Mark Read/Unread)**

```
POST /users/{userId}/messages/{id}/modify
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `userId` | string | Yes | `me` or email address |
| `id` | string | Yes | The message ID |

Request body:
```json
{
  "addLabelIds": ["Label_1", "STARRED"],
  "removeLabelIds": ["UNREAD"]
}
```

To mark as read: remove `UNREAD` label. To mark as unread: add `UNREAD` label.

Response: Returns the modified Message resource.

Error codes: `400` Invalid label IDs, `401` Unauthorized, `404` Not found, `429` Rate limit.

---

**Delete Message (Permanent)**

```
DELETE /users/{userId}/messages/{id}
```

Immediately and permanently deletes the message. Cannot be undone.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `userId` | string | Yes | `me` or email address |
| `id` | string | Yes | The message ID |

Response: `204 No Content` on success.

Error codes: `401` Unauthorized, `404` Not found, `429` Rate limit.

---

**Trash Message**

```
POST /users/{userId}/messages/{id}/trash
```

Moves the message to the Trash folder. Can be undone with untrash.

Response: Returns the Message resource.

---

**Untrash Message**

```
POST /users/{userId}/messages/{id}/untrash
```

Removes the message from Trash.

Response: Returns the Message resource.

---

**Batch Modify Messages**

```
POST /users/{userId}/messages/batchModify
```

Modifies labels on multiple messages at once.

Request body:
```json
{
  "ids": ["msg_id_1", "msg_id_2", "msg_id_3"],
  "addLabelIds": ["Label_1"],
  "removeLabelIds": ["UNREAD"]
}
```

Response: `204 No Content` on success.

---

#### Threads

**List Threads**

```
GET /users/{userId}/threads
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `userId` | string | Yes | `me` or email address |
| `maxResults` | integer | No | Maximum threads to return (default 100, max 500) |
| `pageToken` | string | No | Pagination token |
| `q` | string | No | Gmail search query |
| `labelIds` | string[] | No | Only return threads with all specified labels |
| `includeSpamTrash` | boolean | No | Include SPAM and TRASH threads |

Response:
```json
{
  "threads": [
    { "id": "thread_id", "snippet": "preview...", "historyId": "12345" }
  ],
  "nextPageToken": "token",
  "resultSizeEstimate": 50
}
```

---

**Get Thread**

```
GET /users/{userId}/threads/{id}
```

Returns the full thread including all messages.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `userId` | string | Yes | `me` or email address |
| `id` | string | Yes | Thread ID |
| `format` | string | No | `full`, `metadata`, `minimal` |

Response:
```json
{
  "id": "thread_id",
  "historyId": "12345",
  "messages": [ /* array of Message resources */ ]
}
```

---

**Modify Thread**

```
POST /users/{userId}/threads/{id}/modify
```

Add/remove labels from all messages in a thread.

Request body:
```json
{
  "addLabelIds": ["Label_1"],
  "removeLabelIds": ["INBOX"]
}
```

---

**Delete Thread**

```
DELETE /users/{userId}/threads/{id}
```

Permanently deletes the thread and all its messages.

---

**Trash Thread**

```
POST /users/{userId}/threads/{id}/trash
```

**Untrash Thread**

```
POST /users/{userId}/threads/{id}/untrash
```

---

#### Labels

**List Labels**

```
GET /users/{userId}/labels
```

Response:
```json
{
  "labels": [
    {
      "id": "Label_1",
      "name": "My Label",
      "type": "user",
      "messageListVisibility": "show",
      "labelListVisibility": "labelShow",
      "messagesTotal": 42,
      "messagesUnread": 5,
      "threadsTotal": 30,
      "threadsUnread": 3
    }
  ]
}
```

---

**Create Label**

```
POST /users/{userId}/labels
```

Request body:
```json
{
  "name": "Esmer/Delegated",
  "labelListVisibility": "labelShow",
  "messageListVisibility": "show"
}
```

---

**Get Label**

```
GET /users/{userId}/labels/{id}
```

**Delete Label**

```
DELETE /users/{userId}/labels/{id}
```

**Update Label**

```
PUT /users/{userId}/labels/{id}
```

---

#### Drafts

**List Drafts**

```
GET /users/{userId}/drafts
```

**Get Draft**

```
GET /users/{userId}/drafts/{id}
```

Returns the draft with its enclosed message.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `format` | string | No | `full`, `metadata`, `minimal`, `raw` |

---

**Create Draft**

```
POST /users/{userId}/drafts
```

Request body:
```json
{
  "message": {
    "raw": "base64url_encoded_rfc2822_message",
    "threadId": "optional_thread_id"
  }
}
```

---

**Update Draft**

```
PUT /users/{userId}/drafts/{id}
```

**Delete Draft**

```
DELETE /users/{userId}/drafts/{id}
```

**Send Draft**

```
POST /users/{userId}/drafts/send
```

Request body:
```json
{
  "id": "draft_id"
}
```

---

#### History

**List History**

```
GET /users/{userId}/history
```

Returns change history since a given `historyId`. Useful for incremental sync.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `startHistoryId` | string | Yes | History ID to start from |
| `maxResults` | integer | No | Max history records to return |
| `pageToken` | string | No | Pagination token |
| `labelId` | string | No | Only return changes affecting this label |
| `historyTypes` | string[] | No | `messageAdded`, `messageDeleted`, `labelAdded`, `labelRemoved` |

### 1.6 Webhooks / Real-time

Gmail supports push notifications via Google Cloud Pub/Sub.

**Watch (Start Push Notifications)**

```
POST /users/{userId}/watch
```

Request body:
```json
{
  "topicName": "projects/my-project/topics/gmail-push",
  "labelIds": ["INBOX"],
  "labelFilterBehavior": "INCLUDE"
}
```

Response:
```json
{
  "historyId": "12345",
  "expiration": "1706000000000"
}
```

- Push notifications expire after 7 days; must be renewed.
- Notifications are delivered to a Cloud Pub/Sub topic.
- Each notification contains `historyId` -- use the History API to retrieve actual changes.
- Call `POST /users/{userId}/stop` to stop watching.

**Event Types:** `messageAdded`, `messageDeleted`, `labelAdded`, `labelRemoved`.

### 1.7 Error Handling

| Code | Meaning | Retry Strategy |
|---|---|---|
| `400` | Bad Request (invalid parameters) | Do not retry; fix request |
| `401` | Invalid/expired token | Refresh OAuth token, then retry once |
| `403` | Insufficient permissions or daily limit | Check scopes; do not retry for permission errors; for daily limits, wait until next day |
| `404` | Resource not found | Do not retry |
| `429` | Rate limit exceeded | Exponential backoff starting at 1s, max 32s, with jitter |
| `500` | Internal server error | Retry with exponential backoff, max 3 attempts |
| `503` | Service unavailable | Retry with exponential backoff, max 5 attempts |

### 1.8 Mobile-Specific Notes

- **Token Storage:** Store OAuth2 refresh tokens in the platform's secure keychain (iOS Keychain / Android Keystore). Never persist access tokens to disk.
- **PKCE:** Use PKCE (Proof Key for Code Exchange) for the OAuth flow on mobile to prevent authorization code interception.
- **Incremental Sync:** Use `history.list` with stored `historyId` for efficient background sync rather than re-fetching all messages.
- **Partial Responses:** Use the `fields` query parameter to request only needed fields, reducing bandwidth on mobile networks.
- **Batch Requests:** Combine multiple operations into batch requests to minimize network round-trips.
- **Offline Drafts:** Queue draft creation locally when offline and sync when connectivity is restored.

---

## 2. Microsoft Outlook (Microsoft Graph API)

### 2.1 Service Overview

Microsoft Outlook provides email, calendar, contacts, and tasks via the Microsoft Graph API. Esmer uses this integration for email triage, message composition, reply management, folder organization, calendar event management, contact lookup, and attachment handling for users with Microsoft 365 or Outlook.com accounts.

### 2.2 Authentication

**Method:** OAuth 2.0

| Property | Value |
|---|---|
| Authorization URL | `https://login.microsoftonline.com/common/oauth2/v2.0/authorize` |
| Token URL | `https://login.microsoftonline.com/common/oauth2/v2.0/token` |
| Revoke URL | N/A (use token endpoint with `revoke` or sign-out endpoint) |
| Grant Type | `authorization_code` |
| PKCE Support | Yes (required for public clients) |

**Government Cloud Endpoints:**

| Environment | Auth Base URL |
|---|---|
| Global | `https://login.microsoftonline.com` |
| US Government (GCC) | `https://login.microsoftonline.us` |
| US Government DOD | `https://login.microsoftonline.us` |
| China (21Vianet) | `https://login.chinacloudapi.cn` |

**Required Scopes:**

| Scope | Purpose |
|---|---|
| `Mail.Read` | Read user's mail |
| `Mail.ReadWrite` | Read and write (move, update) user's mail |
| `Mail.Send` | Send mail as the user |
| `MailboxSettings.Read` | Read mailbox settings (auto-replies, timezone) |
| `Calendars.ReadWrite` | Read and write calendar events |
| `Contacts.ReadWrite` | Read and write contacts |
| `User.Read` | Sign in and read user profile |
| `offline_access` | Obtain refresh tokens for background access |

**Esmer Recommendation:** Request `Mail.ReadWrite`, `Mail.Send`, `Calendars.ReadWrite`, `Contacts.Read`, `User.Read`, and `offline_access`. Request `MailboxSettings.Read` only if auto-reply management is needed.

### 2.3 Base URL

```
https://graph.microsoft.com/v1.0
```

Government cloud:
- US Gov: `https://graph.microsoft.us/v1.0`
- China: `https://microsoftgraph.chinacloudapi.cn/v1.0`

### 2.4 Rate Limits

| Limit | Value |
|---|---|
| Per-app per-tenant | 10,000 requests per 10 minutes |
| Per-app across all tenants | 150,000 requests per 20 seconds |
| Per-mailbox | 10,000 requests per 10 minutes |
| Concurrent requests per mailbox | 4 concurrent requests |
| Max message size | 150 MB (incl. base64 encoding overhead) |
| Max recipients per message | 500 |
| Batch request max | 20 individual requests per batch |
| Delta query | Pagination max 50 changes per page |

Throttled responses return `429 Too Many Requests` with a `Retry-After` header (in seconds).

### 2.5 API Endpoints

#### Messages

**List Messages**

```
GET /me/messages
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `$top` | integer | No | Number of results per page (max 1000, default 10) |
| `$skip` | integer | No | Number of results to skip |
| `$select` | string | No | Comma-separated fields to return |
| `$filter` | string | No | OData filter (e.g., `isRead eq false`) |
| `$orderby` | string | No | Sort field (e.g., `receivedDateTime desc`) |
| `$search` | string | No | Search query using KQL syntax |
| `$count` | boolean | No | Include count of matching resources |

Response:
```json
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#users('id')/messages",
  "@odata.nextLink": "https://graph.microsoft.com/v1.0/me/messages?$skip=10",
  "value": [
    {
      "id": "AAMkAGI2...",
      "subject": "Meeting Tomorrow",
      "bodyPreview": "Hi, let's meet...",
      "body": { "contentType": "html", "content": "<html>...</html>" },
      "from": {
        "emailAddress": { "name": "John", "address": "john@example.com" }
      },
      "toRecipients": [
        { "emailAddress": { "name": "Jane", "address": "jane@example.com" } }
      ],
      "ccRecipients": [],
      "bccRecipients": [],
      "receivedDateTime": "2026-02-09T10:00:00Z",
      "sentDateTime": "2026-02-09T09:59:50Z",
      "isRead": false,
      "isDraft": false,
      "hasAttachments": true,
      "importance": "normal",
      "conversationId": "AAQkAGI2...",
      "parentFolderId": "AAMkAGI2..."
    }
  ]
}
```

---

**Get Message**

```
GET /me/messages/{id}
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `id` | string | Yes | Message ID |
| `$select` | string | No | Fields to return |
| `$expand` | string | No | Expand navigation properties (e.g., `attachments`) |

---

**Send Message**

```
POST /me/sendMail
```

Request body:
```json
{
  "message": {
    "subject": "Hello from Esmer",
    "body": {
      "contentType": "HTML",
      "content": "<p>This is the email body.</p>"
    },
    "toRecipients": [
      { "emailAddress": { "address": "recipient@example.com", "name": "Recipient" } }
    ],
    "ccRecipients": [],
    "bccRecipients": [],
    "importance": "normal",
    "attachments": [
      {
        "@odata.type": "#microsoft.graph.fileAttachment",
        "name": "report.pdf",
        "contentType": "application/pdf",
        "contentBytes": "base64_encoded_content"
      }
    ]
  },
  "saveToSentItems": true
}
```

Response: `202 Accepted` (no body).

---

**Reply to Message**

```
POST /me/messages/{id}/reply
```

Request body:
```json
{
  "message": {
    "toRecipients": [
      { "emailAddress": { "address": "sender@example.com" } }
    ]
  },
  "comment": "Thanks for your email. I will review this."
}
```

**Reply All:**

```
POST /me/messages/{id}/replyAll
```

**Forward:**

```
POST /me/messages/{id}/forward
```

Request body:
```json
{
  "comment": "FYI - see below.",
  "toRecipients": [
    { "emailAddress": { "address": "colleague@example.com" } }
  ]
}
```

---

**Update Message**

```
PATCH /me/messages/{id}
```

Request body (mark as read example):
```json
{
  "isRead": true
}
```

Can also update: `categories`, `importance`, `inferenceClassification`, `isRead`.

---

**Move Message**

```
POST /me/messages/{id}/move
```

Request body:
```json
{
  "destinationId": "AAMkAGI2..."
}
```

Well-known folder names can be used: `inbox`, `drafts`, `sentitems`, `deleteditems`, `archive`, `junkemail`.

---

**Delete Message**

```
DELETE /me/messages/{id}
```

Response: `204 No Content`. Moves to Deleted Items (soft delete).

---

#### Drafts

**Create Draft**

```
POST /me/messages
```

Request body:
```json
{
  "subject": "Draft subject",
  "body": { "contentType": "Text", "content": "Draft body" },
  "toRecipients": [
    { "emailAddress": { "address": "recipient@example.com" } }
  ]
}
```

This creates a message in the Drafts folder.

**Update Draft**

```
PATCH /me/messages/{id}
```

**Send Draft**

```
POST /me/messages/{id}/send
```

**Delete Draft**

```
DELETE /me/messages/{id}
```

---

#### Folders

**List Folders**

```
GET /me/mailFolders
```

Response:
```json
{
  "value": [
    {
      "id": "AAMkAGI2...",
      "displayName": "Inbox",
      "parentFolderId": "AAMkAGI2...",
      "childFolderCount": 2,
      "unreadItemCount": 15,
      "totalItemCount": 150
    }
  ]
}
```

**Create Folder**

```
POST /me/mailFolders
```

Request body:
```json
{
  "displayName": "Esmer Triaged",
  "isHidden": false
}
```

**Get Folder**

```
GET /me/mailFolders/{id}
```

**Update Folder**

```
PATCH /me/mailFolders/{id}
```

**Delete Folder**

```
DELETE /me/mailFolders/{id}
```

**List Messages in Folder**

```
GET /me/mailFolders/{id}/messages
```

Supports the same query parameters as `GET /me/messages`.

---

#### Attachments

**List Attachments**

```
GET /me/messages/{message-id}/attachments
```

**Get Attachment**

```
GET /me/messages/{message-id}/attachments/{attachment-id}
```

For large attachments (>3MB), use:

```
GET /me/messages/{message-id}/attachments/{attachment-id}/$value
```

**Add Attachment**

```
POST /me/messages/{message-id}/attachments
```

Request body:
```json
{
  "@odata.type": "#microsoft.graph.fileAttachment",
  "name": "document.pdf",
  "contentType": "application/pdf",
  "contentBytes": "base64_encoded_content"
}
```

For large attachments (>3 MB), use upload sessions:

```
POST /me/messages/{message-id}/attachments/createUploadSession
```

---

#### Contacts

**List Contacts**

```
GET /me/contacts
```

**Create Contact**

```
POST /me/contacts
```

**Get, Update, Delete Contact**

```
GET /me/contacts/{id}
PATCH /me/contacts/{id}
DELETE /me/contacts/{id}
```

---

#### Calendar Events

**List Events**

```
GET /me/events
GET /me/calendarView?startDateTime={start}&endDateTime={end}
```

**Create Event**

```
POST /me/events
```

Request body:
```json
{
  "subject": "Team Standup",
  "body": { "contentType": "HTML", "content": "<p>Daily standup</p>" },
  "start": { "dateTime": "2026-02-10T09:00:00", "timeZone": "America/Los_Angeles" },
  "end": { "dateTime": "2026-02-10T09:30:00", "timeZone": "America/Los_Angeles" },
  "location": { "displayName": "Conference Room 1" },
  "attendees": [
    {
      "emailAddress": { "address": "attendee@example.com", "name": "Attendee" },
      "type": "required"
    }
  ],
  "isOnlineMeeting": true,
  "onlineMeetingProvider": "teamsForBusiness"
}
```

**Get, Update, Delete Event**

```
GET /me/events/{id}
PATCH /me/events/{id}
DELETE /me/events/{id}
```

### 2.6 Webhooks / Real-time

Microsoft Graph supports webhooks (subscriptions) for change notifications.

**Create Subscription**

```
POST /subscriptions
```

Request body:
```json
{
  "changeType": "created,updated,deleted",
  "notificationUrl": "https://esmer-api.example.com/webhooks/outlook",
  "resource": "me/messages",
  "expirationDateTime": "2026-02-12T10:00:00Z",
  "clientState": "esmer-secret-state-value"
}
```

- Max subscription lifetime: 3 days for mail resources (must renew).
- Change types: `created`, `updated`, `deleted`.
- Resources: `me/messages`, `me/events`, `me/contacts`, `me/mailFolders`.
- Microsoft sends a validation request to the `notificationUrl` before activating.

**Renew Subscription**

```
PATCH /subscriptions/{id}
```

**Delete Subscription**

```
DELETE /subscriptions/{id}
```

**Delta Queries (Alternative to webhooks):**

```
GET /me/messages/delta
```

Returns incremental changes since last sync. More reliable for mobile apps than webhooks.

### 2.7 Error Handling

| Code | Meaning | Retry Strategy |
|---|---|---|
| `400` | Bad request | Do not retry; fix request |
| `401` | Token expired or invalid | Refresh token, retry once |
| `403` | Forbidden / insufficient permissions | Do not retry; check scopes |
| `404` | Resource not found | Do not retry |
| `409` | Conflict (concurrent modification) | Retry after short delay |
| `429` | Throttled | Wait for `Retry-After` header seconds, then retry |
| `500` | Internal server error | Retry with exponential backoff |
| `502` | Bad gateway | Retry with exponential backoff |
| `503` | Service unavailable | Retry with exponential backoff, respect `Retry-After` |
| `504` | Gateway timeout | Retry with exponential backoff |

### 2.8 Mobile-Specific Notes

- **PKCE Required:** Microsoft requires PKCE for mobile/native app OAuth flows (public clients cannot use client secrets).
- **Token Storage:** Use platform secure storage. Refresh tokens can be long-lived; store them securely.
- **Delta Sync:** Prefer `delta` queries over webhooks for mobile. Store the `deltaLink` and poll periodically for changes.
- **Select Fields:** Always use `$select` to minimize response size. Request only `id,subject,from,receivedDateTime,bodyPreview,isRead` for list views.
- **Shared Mailbox:** Support shared mailbox access by using `/users/{user-id-or-upn}/messages` instead of `/me/messages`.
- **Government Cloud:** Dynamically switch the base URL based on the user's tenant type.

---

## 3. Slack

### 3.1 Service Overview

Slack is a channel-based messaging platform for teams. Esmer uses Slack to send messages on the user's behalf, manage channels, search conversations, handle reactions, upload files, and manage user groups. The Slack Web API uses HTTP methods with JSON payloads.

### 3.2 Authentication

**Method:** OAuth 2.0 (recommended) or Bot Token (API access token)

| Property | Value |
|---|---|
| Authorization URL | `https://slack.com/oauth/v2/authorize` |
| Token URL | `https://slack.com/api/oauth.v2.access` |
| Revoke URL | `https://slack.com/api/auth.revoke` |
| Grant Type | `authorization_code` |

**Required Scopes (Bot Token):**

| Scope | Purpose |
|---|---|
| `channels:read` | View basic channel info |
| `channels:history` | Read channel message history |
| `chat:write` | Send messages |
| `files:read` | View files |
| `files:write` | Upload and modify files |
| `groups:read` | View private channel info |
| `groups:history` | Read private channel history |
| `im:read` | View DM info |
| `im:history` | Read DM history |
| `mpim:read` | View multi-party DM info |
| `mpim:history` | Read multi-party DM history |
| `reactions:read` | View emoji reactions |
| `reactions:write` | Add and remove reactions |
| `users:read` | View user info |
| `users.profile:read` | View user profiles |
| `usergroups:read` | View user groups |
| `usergroups:write` | Manage user groups |
| `search:read` | Search messages and files |

**User Token Scopes (additional if acting as user):**

| Scope | Purpose |
|---|---|
| `channels:write` | Manage channels |
| `stars:read` | View starred items |
| `stars:write` | Star/unstar items |
| `users.profile:write` | Update user profile |

**Esmer Recommendation:** Use bot tokens for most operations. Use user tokens only when acting on behalf of a specific user (e.g., starring, profile updates). Request the minimum scopes needed.

### 3.3 Base URL

```
https://slack.com/api
```

### 3.4 Rate Limits

Slack uses a tiered rate limiting system:

| Tier | Rate | Methods |
|---|---|---|
| Tier 1 (special) | 1 request per minute | `admin.*`, `migration.*` |
| Tier 2 (post) | 1 request per second per channel | `chat.postMessage`, `chat.update` |
| Tier 3 (read) | 50+ requests per minute | `conversations.list`, `users.list`, most read methods |
| Tier 4 (generous) | 100+ requests per minute | `conversations.info`, `users.info` |
| Special | Varies | `search.messages` (20/min), `files.upload` (20/min) |

- Rate limits are per-token, not per-IP.
- Throttled responses return `429` with `Retry-After` header.
- Burst allowance exists for short spikes.
- Web API responses include `X-RateLimit-*` headers.

### 3.5 API Endpoints

#### Channels (Conversations)

**List Channels**

```
POST https://slack.com/api/conversations.list
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `token` | string | Yes | Bot or user token (typically sent as Bearer header) |
| `types` | string | No | Comma-separated: `public_channel`, `private_channel`, `mpim`, `im` |
| `exclude_archived` | boolean | No | Exclude archived channels (default true) |
| `limit` | integer | No | Max results per page (default 100, max 1000) |
| `cursor` | string | No | Pagination cursor |
| `team_id` | string | No | Team ID for Enterprise Grid |

Response:
```json
{
  "ok": true,
  "channels": [
    {
      "id": "C01ABCDEF",
      "name": "general",
      "is_channel": true,
      "is_private": false,
      "is_archived": false,
      "is_member": true,
      "num_members": 42,
      "topic": { "value": "General discussion", "creator": "U01ABC", "last_set": 1706000000 },
      "purpose": { "value": "Team chat", "creator": "U01ABC", "last_set": 1706000000 }
    }
  ],
  "response_metadata": { "next_cursor": "dGVhbTpDMDYx..." }
}
```

---

**Create Channel**

```
POST https://slack.com/api/conversations.create
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Channel name (lowercase, no spaces, max 80 chars) |
| `is_private` | boolean | No | Create as private channel (default false) |
| `team_id` | string | No | Team ID for Enterprise Grid |

---

**Get Channel Info**

```
POST https://slack.com/api/conversations.info
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `channel` | string | Yes | Channel ID |
| `include_num_members` | boolean | No | Include member count |

---

**Channel History**

```
POST https://slack.com/api/conversations.history
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `channel` | string | Yes | Channel ID |
| `cursor` | string | No | Pagination cursor |
| `inclusive` | boolean | No | Include messages with oldest/latest timestamps |
| `latest` | string | No | End of time range (Unix timestamp) |
| `oldest` | string | No | Start of time range (Unix timestamp) |
| `limit` | integer | No | Max results (default 100, max 1000) |
| `include_all_metadata` | boolean | No | Include message metadata |

---

**Thread Replies**

```
POST https://slack.com/api/conversations.replies
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `channel` | string | Yes | Channel ID |
| `ts` | string | Yes | Thread's parent message timestamp |
| `cursor` | string | No | Pagination cursor |
| `inclusive` | boolean | No | Include parent message |
| `limit` | integer | No | Max results (default 1000) |

---

**Invite User to Channel**

```
POST https://slack.com/api/conversations.invite
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `channel` | string | Yes | Channel ID |
| `users` | string | Yes | Comma-separated list of user IDs |

---

**Kick User from Channel**

```
POST https://slack.com/api/conversations.kick
```

**Join Channel**

```
POST https://slack.com/api/conversations.join
```

**Leave Channel**

```
POST https://slack.com/api/conversations.leave
```

**Archive Channel**

```
POST https://slack.com/api/conversations.archive
```

**Unarchive Channel**

```
POST https://slack.com/api/conversations.unarchive
```

**Rename Channel**

```
POST https://slack.com/api/conversations.rename
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `channel` | string | Yes | Channel ID |
| `name` | string | Yes | New channel name |

---

**Set Channel Topic**

```
POST https://slack.com/api/conversations.setTopic
```

**Set Channel Purpose**

```
POST https://slack.com/api/conversations.setPurpose
```

---

**List Channel Members**

```
POST https://slack.com/api/conversations.members
```

---

**Open/Close DM**

```
POST https://slack.com/api/conversations.open
POST https://slack.com/api/conversations.close
```

---

#### Messages

**Send Message**

```
POST https://slack.com/api/chat.postMessage
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `channel` | string | Yes | Channel ID, DM ID, or user ID |
| `text` | string | Conditional | Message text (required if no blocks/attachments) |
| `blocks` | array | No | Block Kit JSON array |
| `attachments` | array | No | Legacy attachment array |
| `thread_ts` | string | No | Timestamp of parent message (for threading) |
| `reply_broadcast` | boolean | No | Also post reply to channel |
| `unfurl_links` | boolean | No | Enable URL unfurling |
| `unfurl_media` | boolean | No | Enable media unfurling |
| `mrkdwn` | boolean | No | Enable markdown parsing (default true) |
| `metadata` | object | No | Message metadata |

Response:
```json
{
  "ok": true,
  "channel": "C01ABCDEF",
  "ts": "1706000000.000100",
  "message": {
    "text": "Hello from Esmer",
    "type": "message",
    "subtype": "bot_message",
    "ts": "1706000000.000100",
    "bot_id": "B01ABCDEF"
  }
}
```

---

**Update Message**

```
POST https://slack.com/api/chat.update
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `channel` | string | Yes | Channel ID |
| `ts` | string | Yes | Message timestamp to update |
| `text` | string | No | New text |
| `blocks` | array | No | New Block Kit blocks |

---

**Delete Message**

```
POST https://slack.com/api/chat.delete
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `channel` | string | Yes | Channel ID |
| `ts` | string | Yes | Message timestamp to delete |

---

**Get Permalink**

```
POST https://slack.com/api/chat.getPermalink
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `channel` | string | Yes | Channel ID |
| `message_ts` | string | Yes | Message timestamp |

---

**Search Messages**

```
POST https://slack.com/api/search.messages
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `query` | string | Yes | Search query (supports Slack search syntax) |
| `sort` | string | No | `score` or `timestamp` |
| `sort_dir` | string | No | `asc` or `desc` |
| `count` | integer | No | Results per page (default 20, max 100) |
| `page` | integer | No | Page number |

---

#### Reactions

**Add Reaction**

```
POST https://slack.com/api/reactions.add
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `channel` | string | Yes | Channel ID |
| `name` | string | Yes | Emoji name (without colons) |
| `timestamp` | string | Yes | Message timestamp |

**Get Reactions**

```
POST https://slack.com/api/reactions.get
```

**Remove Reaction**

```
POST https://slack.com/api/reactions.remove
```

---

#### Files

**Upload File**

```
POST https://slack.com/api/files.upload
```

Uses `multipart/form-data`.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `channels` | string | No | Comma-separated channel IDs to share to |
| `content` | string | Conditional | File content (if not using `file`) |
| `file` | file | Conditional | File data (multipart) |
| `filename` | string | No | Filename |
| `filetype` | string | No | File type identifier |
| `initial_comment` | string | No | Message to accompany the file |
| `thread_ts` | string | No | Thread timestamp |
| `title` | string | No | Title of file |

---

**Get File Info**

```
POST https://slack.com/api/files.info
```

**List Files**

```
POST https://slack.com/api/files.list
```

---

#### Users

**Get User Info**

```
POST https://slack.com/api/users.info
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `user` | string | Yes | User ID |

Response:
```json
{
  "ok": true,
  "user": {
    "id": "U01ABCDEF",
    "name": "johndoe",
    "real_name": "John Doe",
    "profile": {
      "display_name": "John",
      "email": "john@example.com",
      "image_72": "https://avatars.slack-edge.com/...",
      "status_text": "In a meeting",
      "status_emoji": ":calendar:"
    },
    "is_admin": false,
    "is_bot": false
  }
}
```

---

**List Users**

```
POST https://slack.com/api/users.list
```

**Get User Profile**

```
POST https://slack.com/api/users.profile.get
```

**Update User Profile**

```
POST https://slack.com/api/users.profile.set
```

**Get User Presence**

```
POST https://slack.com/api/users.getPresence
```

---

#### User Groups

**Create User Group**

```
POST https://slack.com/api/usergroups.create
```

**List User Groups**

```
POST https://slack.com/api/usergroups.list
```

**Update User Group**

```
POST https://slack.com/api/usergroups.update
```

**Enable/Disable User Group**

```
POST https://slack.com/api/usergroups.enable
POST https://slack.com/api/usergroups.disable
```

---

#### Stars

**Add Star**

```
POST https://slack.com/api/stars.add
```

**Remove Star**

```
POST https://slack.com/api/stars.remove
```

**List Stars**

```
POST https://slack.com/api/stars.list
```

### 3.6 Webhooks / Real-time

**Events API (recommended):**

Slack sends HTTP POST requests to a configured Request URL when subscribed events occur.

Setup:
1. Configure a Request URL endpoint in the Slack app settings under Event Subscriptions.
2. Slack sends a `url_verification` challenge that must be echoed back.
3. Subscribe to specific bot events.

Key event types:
- `message.channels` -- Message posted to a public channel
- `message.groups` -- Message posted to a private channel
- `message.im` -- Message posted in a DM
- `message.mpim` -- Message posted in a multi-party DM
- `reaction_added` / `reaction_removed`
- `channel_created` / `channel_deleted`
- `member_joined_channel` / `member_left_channel`
- `app_mention` -- Bot was mentioned

Payload format:
```json
{
  "token": "verification_token",
  "team_id": "T01ABCDEF",
  "event": {
    "type": "message",
    "channel": "C01ABCDEF",
    "user": "U01ABCDEF",
    "text": "Hello",
    "ts": "1706000000.000100"
  },
  "type": "event_callback",
  "event_id": "Ev01ABC",
  "event_time": 1706000000
}
```

**Socket Mode (alternative):**

Uses WebSocket connections instead of public HTTP endpoints. Better for development and when a public URL is unavailable. Esmer should use Events API for production.

### 3.7 Error Handling

Slack returns `200 OK` for most responses, even errors. Check the `ok` field in the response body.

```json
{
  "ok": false,
  "error": "channel_not_found"
}
```

| Error | Meaning | Retry |
|---|---|---|
| `not_authed` | No token or invalid token | Do not retry; re-authenticate |
| `invalid_auth` | Invalid token | Do not retry; re-authenticate |
| `token_revoked` | Token has been revoked | Do not retry; re-authenticate |
| `channel_not_found` | Channel does not exist or bot not in channel | Do not retry |
| `not_in_channel` | Bot not in the channel | Join channel first, then retry |
| `is_archived` | Channel is archived | Do not retry |
| `msg_too_long` | Message exceeds 40,000 characters | Truncate and retry |
| `no_text` | No text provided | Fix request |
| `ratelimited` | Rate limited (HTTP 429) | Wait `Retry-After` seconds |
| `request_timeout` | Request timed out | Retry once |
| `fatal_error` | Server error | Retry with exponential backoff |

### 3.8 Mobile-Specific Notes

- **Token Storage:** Store the bot/user OAuth tokens in secure keychain. Slack tokens do not expire by default but can be revoked.
- **Token Rotation:** Slack supports automatic token rotation for enhanced security. Enable this for mobile apps.
- **Block Kit:** Use Block Kit for rich message formatting. The JSON payloads can be large; consider constructing them server-side.
- **Rate Limits:** Cache channel lists and user info locally to avoid excessive API calls. Use cursor-based pagination.
- **Real-time:** For mobile, prefer polling with `conversations.history` using `oldest` parameter over maintaining WebSocket connections, which drain battery.
- **File Uploads:** For large files from mobile, use the newer `files.uploadV2` method which supports resumable uploads.

---

## 4. Microsoft Teams (Microsoft Graph API)

### 4.1 Service Overview

Microsoft Teams is a collaboration platform combining chat, video meetings, file storage, and application integration. Esmer uses Teams to send messages in channels and chats, manage channels, create tasks (via Planner integration), and monitor conversations. The API is accessed through Microsoft Graph.

### 4.2 Authentication

**Method:** OAuth 2.0 (same Microsoft identity platform as Outlook)

| Property | Value |
|---|---|
| Authorization URL | `https://login.microsoftonline.com/common/oauth2/v2.0/authorize` |
| Token URL | `https://login.microsoftonline.com/common/oauth2/v2.0/token` |
| Revoke URL | N/A |
| Grant Type | `authorization_code` |
| PKCE Support | Yes (required for public clients) |

**Required Scopes:**

| Scope | Purpose |
|---|---|
| `Team.ReadBasic.All` | Read teams the user is a member of |
| `Channel.ReadBasic.All` | Read channel names and descriptions |
| `ChannelMessage.Read.All` | Read channel messages |
| `ChannelMessage.Send` | Send channel messages |
| `Chat.ReadWrite` | Read and send chat messages |
| `ChatMessage.Read` | Read chat messages |
| `ChatMessage.Send` | Send chat messages |
| `Group.ReadWrite.All` | Manage teams and channels (create/update/delete) |
| `Tasks.ReadWrite` | Create and manage Planner tasks |
| `User.Read` | Read user profile |
| `offline_access` | Obtain refresh tokens |

**Esmer Recommendation:** Use delegated permissions. Request `ChannelMessage.Send`, `Chat.ReadWrite`, `Team.ReadBasic.All`, `Channel.ReadBasic.All`, `User.Read`, and `offline_access` as the minimum set.

### 4.3 Base URL

```
https://graph.microsoft.com/v1.0
```

### 4.4 Rate Limits

| Limit | Value |
|---|---|
| Per-app per-tenant | 10,000 requests per 10 minutes |
| Chat messages (create) | 2 requests per second per app per chat |
| Channel messages (create) | 2 requests per second per app per channel |
| Get channel messages | 5 requests per second per app per team |
| Listing teams/channels | Subject to general Graph throttling |
| Planner tasks | 500 requests per 10 seconds per app per tenant |

### 4.5 API Endpoints

#### Channels

**List Channels in a Team**

```
GET /teams/{team-id}/channels
```

Response:
```json
{
  "value": [
    {
      "id": "19:abc123@thread.tacv2",
      "displayName": "General",
      "description": "Main channel",
      "membershipType": "standard",
      "webUrl": "https://teams.microsoft.com/l/channel/..."
    }
  ]
}
```

---

**Create Channel**

```
POST /teams/{team-id}/channels
```

Request body:
```json
{
  "displayName": "Esmer Notifications",
  "description": "Channel managed by Esmer",
  "membershipType": "standard"
}
```

---

**Get Channel**

```
GET /teams/{team-id}/channels/{channel-id}
```

**Update Channel**

```
PATCH /teams/{team-id}/channels/{channel-id}
```

**Delete Channel**

```
DELETE /teams/{team-id}/channels/{channel-id}
```

---

#### Channel Messages

**Send Channel Message**

```
POST /teams/{team-id}/channels/{channel-id}/messages
```

Request body:
```json
{
  "body": {
    "contentType": "html",
    "content": "<p>Hello from Esmer!</p>"
  },
  "importance": "normal"
}
```

Response:
```json
{
  "id": "1706000000000",
  "createdDateTime": "2026-02-09T10:00:00Z",
  "from": {
    "user": {
      "id": "user-id",
      "displayName": "Esmer Bot"
    }
  },
  "body": {
    "contentType": "html",
    "content": "<p>Hello from Esmer!</p>"
  }
}
```

---

**List Channel Messages**

```
GET /teams/{team-id}/channels/{channel-id}/messages
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `$top` | integer | No | Number of messages to return (max 50) |

---

**Reply to Channel Message**

```
POST /teams/{team-id}/channels/{channel-id}/messages/{message-id}/replies
```

Request body:
```json
{
  "body": {
    "contentType": "text",
    "content": "This is a threaded reply."
  }
}
```

---

**List Message Replies**

```
GET /teams/{team-id}/channels/{channel-id}/messages/{message-id}/replies
```

---

#### Chat Messages

**List Chats**

```
GET /me/chats
```

Response:
```json
{
  "value": [
    {
      "id": "19:abc@thread.v2",
      "chatType": "oneOnOne",
      "topic": null,
      "createdDateTime": "2026-01-15T10:00:00Z",
      "lastUpdatedDateTime": "2026-02-09T08:00:00Z"
    }
  ]
}
```

---

**Send Chat Message**

```
POST /chats/{chat-id}/messages
```

Request body:
```json
{
  "body": {
    "contentType": "html",
    "content": "<p>Direct message from Esmer</p>"
  }
}
```

---

**Get Chat Message**

```
GET /chats/{chat-id}/messages/{message-id}
```

**List Chat Messages**

```
GET /chats/{chat-id}/messages
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `$top` | integer | No | Number of messages to return (max 50) |
| `$orderby` | string | No | Sort order |

---

#### Tasks (Planner)

**Create Task**

```
POST /planner/tasks
```

Request body:
```json
{
  "planId": "plan-id",
  "bucketId": "bucket-id",
  "title": "Review Q1 report",
  "assignments": {
    "user-id": {
      "@odata.type": "#microsoft.graph.plannerAssignment",
      "orderHint": " !"
    }
  },
  "dueDateTime": "2026-02-15T00:00:00Z"
}
```

---

**Get Task**

```
GET /planner/tasks/{task-id}
```

**Update Task**

```
PATCH /planner/tasks/{task-id}
```

Requires `If-Match` header with the task's `@odata.etag`.

**Delete Task**

```
DELETE /planner/tasks/{task-id}
```

**List Tasks in a Plan**

```
GET /planner/plans/{plan-id}/tasks
```

### 4.6 Webhooks / Real-time

Microsoft Graph supports change notifications (subscriptions) for Teams resources.

**Create Subscription**

```
POST /subscriptions
```

Request body:
```json
{
  "changeType": "created",
  "notificationUrl": "https://esmer-api.example.com/webhooks/teams",
  "resource": "/chats/{chat-id}/messages",
  "expirationDateTime": "2026-02-10T10:00:00Z",
  "clientState": "esmer-secret"
}
```

Supported resources:
- `/teams/{team-id}/channels/{channel-id}/messages` -- New channel messages
- `/chats/{chat-id}/messages` -- New chat messages
- `/chats/getAllMessages` -- All chat messages (application-only, requires license)
- `/teams/getAllMessages` -- All channel messages (application-only, requires license)

Max subscription lifetime: 60 minutes for chat/channel messages (must renew frequently).

### 4.7 Error Handling

Same error handling as Microsoft Outlook (Section 2.7). Additionally:

| Code | Meaning | Retry |
|---|---|---|
| `403` | Bot not installed in team/chat | Install bot, then retry |
| `404` | Team, channel, or chat not found | Verify resource IDs |
| `409` | Conflict (concurrent edit on tasks) | Re-fetch etag, then retry |
| `429` | Throttled | Wait `Retry-After` seconds |

### 4.8 Mobile-Specific Notes

- **PKCE Required:** Same as Outlook; use PKCE for mobile OAuth flows.
- **Deep Links:** Use Teams deep links (`https://teams.microsoft.com/l/...`) to open specific chats or channels from the Esmer app.
- **Message Limits:** Channel message responses are capped at 50 per page; implement cursor pagination.
- **Adaptive Cards:** Teams supports Adaptive Cards for rich interactive messages. Consider using these for task delegation UIs.
- **Subscription Renewal:** Chat message subscriptions expire in 60 minutes. Implement a background renewal timer.

---

## 5. Telegram (Bot API)

### 5.1 Service Overview

Telegram is a cloud-based messaging platform. Esmer interacts via the Telegram Bot API to send messages (text, photos, documents, audio, video, stickers, animations, locations), manage chats, handle inline keyboards and callbacks, pin/unpin messages, and retrieve files. All operations go through the bot; the bot must be added to a chat/channel to interact with it.

### 5.2 Authentication

**Method:** Bot Token (API Key)

| Property | Value |
|---|---|
| Auth Method | Bearer token in URL path |
| Token Format | `{bot_id}:{hash}` (e.g., `123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11`) |
| Token Source | Generated via BotFather (`/newbot` command) |
| Revoke | Use BotFather `/revoke` command |

No OAuth flow is needed. The bot token is generated once and used directly.

**Esmer Recommendation:** Create a dedicated bot for Esmer via BotFather. Store the token securely. The token grants full control of the bot. Rotate tokens periodically via `/revoke` and `/token` in BotFather.

### 5.3 Base URL

```
https://api.telegram.org/bot{token}
```

File downloads:
```
https://api.telegram.org/file/bot{token}/{file_path}
```

### 5.4 Rate Limits

| Limit | Value |
|---|---|
| Messages to a specific chat | 1 message per second per chat |
| Messages overall | 30 messages per second |
| Messages to a group | 20 messages per minute per group |
| Bulk notifications | Max 30 different chats per second |
| getUpdates long polling | 1 concurrent request per bot |
| Inline query answers | 50 results per query |
| File upload size | 50 MB max |
| File download size | 20 MB max |
| sendMessage text length | 4,096 characters max |
| Caption length | 1,024 characters max |

### 5.5 API Endpoints

All endpoints use POST (GET is also supported for most methods but POST is recommended).

#### Messages

**Send Message**

```
POST /bot{token}/sendMessage
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `chat_id` | integer/string | Yes | Chat ID or `@channelusername` |
| `text` | string | Yes | Message text (max 4096 chars) |
| `parse_mode` | string | No | `HTML`, `Markdown`, or `MarkdownV2` |
| `entities` | array | No | Special entities in the message |
| `link_preview_options` | object | No | Link preview settings |
| `disable_notification` | boolean | No | Send silently |
| `protect_content` | boolean | No | Protect from forwarding/saving |
| `reply_parameters` | object | No | Reply configuration |
| `reply_markup` | object | No | InlineKeyboardMarkup, ReplyKeyboardMarkup, etc. |
| `message_thread_id` | integer | No | Forum topic thread ID |

Request body example:
```json
{
  "chat_id": 123456789,
  "text": "Hello from Esmer!",
  "parse_mode": "HTML",
  "reply_markup": {
    "inline_keyboard": [
      [
        { "text": "Approve", "callback_data": "approve_123" },
        { "text": "Decline", "callback_data": "decline_123" }
      ]
    ]
  }
}
```

Response:
```json
{
  "ok": true,
  "result": {
    "message_id": 100,
    "from": { "id": 123456, "is_bot": true, "first_name": "Esmer" },
    "chat": { "id": 789, "type": "private" },
    "date": 1706000000,
    "text": "Hello from Esmer!"
  }
}
```

---

**Edit Message Text**

```
POST /bot{token}/editMessageText
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `chat_id` | integer/string | Conditional | Required if `inline_message_id` not set |
| `message_id` | integer | Conditional | Required if `inline_message_id` not set |
| `inline_message_id` | string | Conditional | Required if `chat_id`/`message_id` not set |
| `text` | string | Yes | New text |
| `parse_mode` | string | No | `HTML`, `Markdown`, `MarkdownV2` |
| `reply_markup` | object | No | Updated inline keyboard |

---

**Delete Message**

```
POST /bot{token}/deleteMessage
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `chat_id` | integer/string | Yes | Chat ID |
| `message_id` | integer | Yes | Message ID to delete |

Bot can delete its own messages at any time; other users' messages only within 48 hours in groups/supergroups.

---

**Forward Message**

```
POST /bot{token}/forwardMessage
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `chat_id` | integer/string | Yes | Target chat |
| `from_chat_id` | integer/string | Yes | Source chat |
| `message_id` | integer | Yes | Message to forward |
| `disable_notification` | boolean | No | Send silently |

---

**Copy Message**

```
POST /bot{token}/copyMessage
```

Same as forward but without the "Forwarded from" header.

---

**Pin Chat Message**

```
POST /bot{token}/pinChatMessage
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `chat_id` | integer/string | Yes | Chat ID |
| `message_id` | integer | Yes | Message to pin |
| `disable_notification` | boolean | No | Pin silently |

---

**Unpin Chat Message**

```
POST /bot{token}/unpinChatMessage
```

**Unpin All Chat Messages**

```
POST /bot{token}/unpinAllChatMessages
```

---

#### Media Messages

**Send Photo**

```
POST /bot{token}/sendPhoto
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `chat_id` | integer/string | Yes | Chat ID |
| `photo` | InputFile/string | Yes | Photo file or file_id or HTTP URL |
| `caption` | string | No | Caption (max 1024 chars) |
| `parse_mode` | string | No | Caption parse mode |
| `reply_markup` | object | No | Reply markup |

---

**Send Document**

```
POST /bot{token}/sendDocument
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `chat_id` | integer/string | Yes | Chat ID |
| `document` | InputFile/string | Yes | Document file, file_id, or URL |
| `caption` | string | No | Caption |
| `thumbnail` | InputFile | No | Thumbnail (JPEG, <200KB, 320x320 max) |

---

**Send Audio**

```
POST /bot{token}/sendAudio
```

**Send Video**

```
POST /bot{token}/sendVideo
```

**Send Animation**

```
POST /bot{token}/sendAnimation
```

**Send Sticker**

```
POST /bot{token}/sendSticker
```

**Send Location**

```
POST /bot{token}/sendLocation
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `chat_id` | integer/string | Yes | Chat ID |
| `latitude` | float | Yes | Latitude |
| `longitude` | float | Yes | Longitude |
| `horizontal_accuracy` | float | No | Accuracy in meters (0-1500) |
| `live_period` | integer | No | Period for live location (60-86400 seconds) |

---

**Send Media Group**

```
POST /bot{token}/sendMediaGroup
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `chat_id` | integer/string | Yes | Chat ID |
| `media` | array | Yes | Array of InputMediaPhoto/InputMediaVideo (2-10 items) |

---

**Send Chat Action**

```
POST /bot{token}/sendChatAction
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `chat_id` | integer/string | Yes | Chat ID |
| `action` | string | Yes | `typing`, `upload_photo`, `record_video`, `upload_video`, `record_voice`, `upload_voice`, `upload_document`, `find_location`, `record_video_note`, `upload_video_note` |

---

#### Chat Operations

**Get Chat**

```
POST /bot{token}/getChat
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `chat_id` | integer/string | Yes | Chat ID |

Response includes chat type, title, description, pinned message, permissions.

---

**Get Chat Administrators**

```
POST /bot{token}/getChatAdministrators
```

Returns an array of ChatMember objects.

---

**Get Chat Member**

```
POST /bot{token}/getChatMember
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `chat_id` | integer/string | Yes | Chat ID |
| `user_id` | integer | Yes | User ID |

---

**Get Chat Member Count**

```
POST /bot{token}/getChatMemberCount
```

---

**Leave Chat**

```
POST /bot{token}/leaveChat
```

---

**Set Chat Title**

```
POST /bot{token}/setChatTitle
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `chat_id` | integer/string | Yes | Chat ID |
| `title` | string | Yes | New title (1-128 chars) |

---

**Set Chat Description**

```
POST /bot{token}/setChatDescription
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `chat_id` | integer/string | Yes | Chat ID |
| `description` | string | No | New description (0-255 chars) |

---

#### Callbacks

**Answer Callback Query**

```
POST /bot{token}/answerCallbackQuery
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `callback_query_id` | string | Yes | Query ID from the callback |
| `text` | string | No | Notification text (max 200 chars) |
| `show_alert` | boolean | No | Show as alert vs notification |
| `url` | string | No | URL to open |
| `cache_time` | integer | No | Cache duration in seconds (default 0) |

---

**Answer Inline Query**

```
POST /bot{token}/answerInlineQuery
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `inline_query_id` | string | Yes | Query ID |
| `results` | array | Yes | Array of InlineQueryResult (max 50) |
| `cache_time` | integer | No | Cache seconds (default 300) |
| `is_personal` | boolean | No | Per-user caching |
| `next_offset` | string | No | Offset for pagination |

---

#### Files

**Get File**

```
POST /bot{token}/getFile
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `file_id` | string | Yes | File identifier |

Response:
```json
{
  "ok": true,
  "result": {
    "file_id": "BQACAgIAAxk...",
    "file_unique_id": "AgAD...",
    "file_size": 1024000,
    "file_path": "documents/file_0.pdf"
  }
}
```

Download via: `https://api.telegram.org/file/bot{token}/{file_path}`

---

#### Updates (Polling)

**Get Updates**

```
POST /bot{token}/getUpdates
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `offset` | integer | No | Identifier of first update to return |
| `limit` | integer | No | Max updates to receive (1-100, default 100) |
| `timeout` | integer | No | Long polling timeout in seconds (0-50) |
| `allowed_updates` | array | No | Update types to receive |

### 5.6 Webhooks / Real-time

**Set Webhook**

```
POST /bot{token}/setWebhook
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `url` | string | Yes | HTTPS URL for webhook |
| `certificate` | InputFile | No | Self-signed certificate |
| `ip_address` | string | No | Fixed IP for webhook |
| `max_connections` | integer | No | Max simultaneous connections (1-100, default 40) |
| `allowed_updates` | array | No | Update types to receive |
| `drop_pending_updates` | boolean | No | Drop pending updates |
| `secret_token` | string | No | Secret token for `X-Telegram-Bot-Api-Secret-Token` header |

**Delete Webhook**

```
POST /bot{token}/deleteWebhook
```

**Get Webhook Info**

```
POST /bot{token}/getWebhookInfo
```

Update types: `message`, `edited_message`, `channel_post`, `edited_channel_post`, `callback_query`, `inline_query`, `chosen_inline_result`, `chat_member`, `chat_join_request`.

Webhook payload is a JSON-serialized `Update` object sent as POST to the webhook URL.

### 5.7 Error Handling

Telegram returns JSON with `ok: false` and `description` field for errors.

```json
{
  "ok": false,
  "error_code": 429,
  "description": "Too Many Requests: retry after 35",
  "parameters": { "retry_after": 35 }
}
```

| Code | Meaning | Retry |
|---|---|---|
| `400` | Bad Request (missing/invalid parameters) | Do not retry; fix request |
| `401` | Unauthorized (invalid token) | Do not retry; check token |
| `403` | Forbidden (bot was blocked/kicked) | Do not retry for this chat |
| `404` | Not found | Do not retry |
| `409` | Conflict (webhook vs getUpdates conflict) | Resolve webhook/polling conflict |
| `429` | Too Many Requests | Wait `retry_after` seconds from response |
| `500` | Internal server error | Retry after 1-5 seconds |

### 5.8 Mobile-Specific Notes

- **Token Security:** The bot token grants full control of the bot. Store it server-side only. The mobile app should communicate with Esmer's backend, which holds the token.
- **Long Polling vs Webhook:** Use webhooks in production (server-side). For testing, use `getUpdates` with long polling.
- **Rate Limit Workaround:** When sending to many chats, use a queue with 30 msg/sec throttle. For groups, respect the 20 msg/min/group limit.
- **File Handling:** Files up to 20 MB can be downloaded. For sending, up to 50 MB. Use `file_id` for resending previously uploaded files to avoid re-uploading.
- **Inline Keyboards:** Use inline keyboards for interactive delegated task confirmations from mobile users.
- **Deep Links:** Use `https://t.me/{bot_username}?start={payload}` for deep linking from the Esmer mobile app into a Telegram bot conversation.

---

## 6. WhatsApp (Cloud API / Business Platform)

### 6.1 Service Overview

WhatsApp Business Cloud API enables businesses to send and receive messages programmatically. Esmer uses it to send text messages, template messages (for outbound notifications), and manage media (upload, download, delete). The API is hosted by Meta and accessed through the Graph API infrastructure.

### 6.2 Authentication

**Method:** API Key (Permanent Access Token) or OAuth 2.0 (for trigger/webhook)

| Property | Value |
|---|---|
| Auth Method (API) | Bearer token in Authorization header |
| Auth Method (Webhook) | OAuth 2.0 via Meta |
| Token Source | Meta for Developers Apps dashboard |
| Token Type | System User Access Token (permanent) or temporary token |

For OAuth 2.0 (webhook/trigger):

| Property | Value |
|---|---|
| Authorization URL | `https://www.facebook.com/v21.0/dialog/oauth` |
| Token URL | `https://graph.facebook.com/v21.0/oauth/access_token` |
| Grant Type | `authorization_code` |

**Required Permissions:**

| Permission | Purpose |
|---|---|
| `whatsapp_business_messaging` | Send and receive messages |
| `whatsapp_business_management` | Manage WhatsApp Business accounts |
| `business_management` | Access business portfolio info |

**Esmer Recommendation:** Use a System User Access Token (permanent, non-expiring) generated from the Meta Business Suite for production. Temporary tokens expire in 24 hours and are only suitable for testing.

### 6.3 Base URL

```
https://graph.facebook.com/v21.0
```

### 6.4 Rate Limits

| Limit | Value |
|---|---|
| Messages per second | 80 messages/second (default Business tier) |
| Messages per second (high) | Up to 1,000 msg/s with approved throughput |
| Template messages per phone | Based on messaging tier (250 / 1K / 10K / 100K per 24h) |
| Conversation-initiated messages | Unlimited within 24-hour customer service window |
| Media upload | 50 MB max per file |
| API calls (general) | 200 calls per hour per phone number |
| Webhook delivery | At-least-once delivery; handle deduplication |

**Messaging Tiers (outbound templates):**

| Tier | Limit |
|---|---|
| Tier 0 (unverified) | 250 unique contacts per 24h |
| Tier 1 | 1,000 unique contacts per 24h |
| Tier 2 | 10,000 unique contacts per 24h |
| Tier 3 | 100,000 unique contacts per 24h |
| Tier 4 (unlimited) | Unlimited |

### 6.5 API Endpoints

#### Messages

**Send Text Message**

```
POST /{phone-number-id}/messages
```

Request body:
```json
{
  "messaging_product": "whatsapp",
  "recipient_type": "individual",
  "to": "15551234567",
  "type": "text",
  "text": {
    "preview_url": false,
    "body": "Hello from Esmer! How can I help you today?"
  }
}
```

Response:
```json
{
  "messaging_product": "whatsapp",
  "contacts": [
    { "input": "15551234567", "wa_id": "15551234567" }
  ],
  "messages": [
    { "id": "wamid.HBgLMTU1NTEyMzQ1NjcVAgASGCA..." }
  ]
}
```

---

**Send Template Message**

```
POST /{phone-number-id}/messages
```

Request body:
```json
{
  "messaging_product": "whatsapp",
  "to": "15551234567",
  "type": "template",
  "template": {
    "name": "esmer_task_notification",
    "language": { "code": "en_US" },
    "components": [
      {
        "type": "body",
        "parameters": [
          { "type": "text", "text": "Your task 'Review Q1 report' is due tomorrow." }
        ]
      }
    ]
  }
}
```

Templates must be pre-approved by Meta before use.

---

**Send Image Message**

```
POST /{phone-number-id}/messages
```

Request body:
```json
{
  "messaging_product": "whatsapp",
  "to": "15551234567",
  "type": "image",
  "image": {
    "id": "media-id-from-upload"
  }
}
```

Alternatively, use `"link": "https://..."` instead of `"id"`.

---

**Send Document Message**

```
POST /{phone-number-id}/messages
```

Request body:
```json
{
  "messaging_product": "whatsapp",
  "to": "15551234567",
  "type": "document",
  "document": {
    "id": "media-id",
    "filename": "report.pdf",
    "caption": "Q1 Report"
  }
}
```

---

**Send Audio / Video / Sticker / Location Message**

Same pattern with `type` set to `audio`, `video`, `sticker`, or `location`.

Location example:
```json
{
  "messaging_product": "whatsapp",
  "to": "15551234567",
  "type": "location",
  "location": {
    "latitude": 37.7749,
    "longitude": -122.4194,
    "name": "San Francisco",
    "address": "1 Market St, San Francisco, CA"
  }
}
```

---

**Send Interactive Message**

```
POST /{phone-number-id}/messages
```

Request body (button example):
```json
{
  "messaging_product": "whatsapp",
  "to": "15551234567",
  "type": "interactive",
  "interactive": {
    "type": "button",
    "body": { "text": "Should I proceed with the task delegation?" },
    "action": {
      "buttons": [
        { "type": "reply", "reply": { "id": "approve", "title": "Approve" } },
        { "type": "reply", "reply": { "id": "decline", "title": "Decline" } }
      ]
    }
  }
}
```

---

**Mark Message as Read**

```
POST /{phone-number-id}/messages
```

Request body:
```json
{
  "messaging_product": "whatsapp",
  "status": "read",
  "message_id": "wamid.HBgLMTU1NTEyMzQ1NjcVAgASGCA..."
}
```

---

#### Media

**Upload Media**

```
POST /{phone-number-id}/media
```

Uses `multipart/form-data`.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `file` | file | Yes | The media file (max 50 MB) |
| `type` | string | Yes | MIME type (e.g., `image/jpeg`, `application/pdf`) |
| `messaging_product` | string | Yes | Must be `whatsapp` |

Response:
```json
{
  "id": "media-id-123"
}
```

---

**Download Media**

```
GET /{media-id}
```

Returns a URL to download the media. Then fetch the URL with the access token.

Response:
```json
{
  "url": "https://lookaside.fbsbx.com/whatsapp_business/attachments/...",
  "mime_type": "image/jpeg",
  "sha256": "hash",
  "file_size": 123456,
  "id": "media-id-123",
  "messaging_product": "whatsapp"
}
```

---

**Delete Media**

```
DELETE /{media-id}
```

Response:
```json
{
  "success": true
}
```

### 6.6 Webhooks / Real-time

WhatsApp uses Meta's webhook infrastructure for incoming message notifications.

**Webhook Configuration:**

1. Set up a webhook URL in the Meta App dashboard under WhatsApp > Configuration.
2. Verify the webhook with a GET challenge-response.
3. Subscribe to webhook fields: `messages`, `messaging_handovers`, `messaging_referrals`.

**Incoming Message Webhook Payload:**

```json
{
  "object": "whatsapp_business_account",
  "entry": [
    {
      "id": "WHATSAPP_BUSINESS_ACCOUNT_ID",
      "changes": [
        {
          "value": {
            "messaging_product": "whatsapp",
            "metadata": {
              "display_phone_number": "15551234567",
              "phone_number_id": "phone-number-id"
            },
            "contacts": [
              { "profile": { "name": "Customer" }, "wa_id": "15559876543" }
            ],
            "messages": [
              {
                "from": "15559876543",
                "id": "wamid.abc123",
                "timestamp": "1706000000",
                "type": "text",
                "text": { "body": "Hello, I need help" }
              }
            ]
          },
          "field": "messages"
        }
      ]
    }
  ]
}
```

**Status Updates (delivery receipts):**

```json
{
  "statuses": [
    {
      "id": "wamid.abc123",
      "status": "delivered",
      "timestamp": "1706000001",
      "recipient_id": "15551234567"
    }
  ]
}
```

Status values: `sent`, `delivered`, `read`, `failed`.

### 6.7 Error Handling

| Code | Meaning | Retry |
|---|---|---|
| `400` | Bad request (invalid parameters) | Do not retry; fix request |
| `401` | Unauthorized (invalid token) | Refresh token, retry |
| `403` | Permission denied | Check app permissions |
| `404` | Phone number or media not found | Do not retry |
| `429` | Rate limit exceeded | Wait and retry with backoff |
| `500` | Internal error | Retry with backoff |
| `131047` | Re-engagement message outside 24h window | Send template message instead |
| `131051` | Unsupported message type | Use supported type |
| `130472` | Recipient not on WhatsApp | Do not retry |

### 6.8 Mobile-Specific Notes

- **24-Hour Window:** After a user messages the business, you have 24 hours to send free-form messages. After that, only pre-approved template messages can be sent.
- **Template Approval:** Templates must be submitted and approved by Meta before use. Build a template management UI in Esmer's admin panel.
- **Media Handling:** Upload media first to get a media ID, then reference it in messages. This avoids timeouts on slow mobile connections.
- **Delivery Receipts:** Use webhook status updates to show delivery/read status in the Esmer UI.
- **Token Security:** The permanent access token should be stored server-side only. The mobile app should proxy all API calls through Esmer's backend.
- **Phone Number Registration:** Each WhatsApp Business account requires a verified phone number. This is a one-time setup, not a per-user flow.

---

## 7. Discord (Bot API)

### 7.1 Service Overview

Discord is a communication platform built around servers (guilds) with text and voice channels. Esmer uses the Discord API to send and manage messages, manage channels, handle member roles, and react to messages. Three authentication methods are available: Bot tokens, OAuth2, and Webhooks. For Esmer's delegation use cases, Bot authentication is primary.

### 7.2 Authentication

**Method:** Bot Token, OAuth2, or Webhook

**Bot Token:**

| Property | Value |
|---|---|
| Auth Method | `Authorization: Bot {token}` header |
| Token Source | Discord Developer Portal > Application > Bot > Reset Token |
| Revoke | Reset token in Developer Portal |

**OAuth2:**

| Property | Value |
|---|---|
| Authorization URL | `https://discord.com/api/oauth2/authorize` |
| Token URL | `https://discord.com/api/oauth2/token` |
| Revoke URL | `https://discord.com/api/oauth2/token/revoke` |
| Grant Type | `authorization_code` |

**Required Bot Permissions:**

| Permission | Bit | Purpose |
|---|---|---|
| Manage Roles | `0x10000000` | Assign roles to members |
| Manage Channels | `0x00000010` | Create/edit/delete channels |
| Read Messages/View Channels | `0x00000400` | View channel content |
| Send Messages | `0x00000800` | Send messages in channels |
| Send TTS Messages | `0x00001000` | Send text-to-speech messages |
| Manage Messages | `0x00002000` | Delete/pin messages |
| Embed Links | `0x00004000` | Send rich embeds |
| Attach Files | `0x00008000` | Upload files |
| Read Message History | `0x00010000` | View historical messages |
| Add Reactions | `0x00000040` | React to messages with emoji |
| Create Public Threads | `0x800000000` | Create public threads |
| Create Private Threads | `0x1000000000` | Create private threads |
| Send Messages in Threads | `0x4000000000` | Post in threads |
| Manage Threads | `0x400000000` | Manage thread settings |

**OAuth2 Scopes:**

| Scope | Purpose |
|---|---|
| `bot` | Add bot to server |
| `applications.commands` | Register slash commands |
| `identify` | Access user info |
| `guilds` | List user's guilds |
| `messages.read` | Read messages (DMs only) |

**Esmer Recommendation:** Use Bot authentication for server interactions. Use OAuth2 for user-installed apps. Use webhooks for simple one-way notifications to a channel.

### 7.3 Base URL

```
https://discord.com/api/v10
```

### 7.4 Rate Limits

| Limit | Value |
|---|---|
| Global rate limit | 50 requests per second |
| Per-route rate limit | Varies by endpoint (check response headers) |
| Create Message | 5 requests per 5 seconds per channel |
| Delete Message | 5 requests per 1 second per channel |
| Edit Message | 5 requests per 5 seconds per channel |
| React to Message | 1 request per 0.25 seconds per channel |
| Bulk delete messages | 1 request per 1 second |
| Modify guild member | 10 per 10 seconds |
| Gateway identify | 1 per 5 seconds |
| Invalid request limit | 10,000 invalid requests per 10 minutes (results in ban) |

Rate limit headers:
- `X-RateLimit-Limit` -- Total requests in the time window
- `X-RateLimit-Remaining` -- Remaining requests
- `X-RateLimit-Reset` -- Unix timestamp when the limit resets
- `X-RateLimit-Reset-After` -- Seconds until reset
- `X-RateLimit-Bucket` -- Rate limit bucket identifier

### 7.5 API Endpoints

#### Channels

**Get Channel**

```
GET /channels/{channel.id}
```

Response:
```json
{
  "id": "123456789",
  "type": 0,
  "guild_id": "987654321",
  "name": "general",
  "topic": "General discussion",
  "nsfw": false,
  "position": 0,
  "permission_overwrites": [],
  "rate_limit_per_user": 0,
  "parent_id": "category_id"
}
```

Channel types: `0` = GUILD_TEXT, `1` = DM, `2` = GUILD_VOICE, `3` = GROUP_DM, `4` = GUILD_CATEGORY, `5` = GUILD_ANNOUNCEMENT, `10` = ANNOUNCEMENT_THREAD, `11` = PUBLIC_THREAD, `12` = PRIVATE_THREAD, `13` = GUILD_STAGE_VOICE, `15` = GUILD_FORUM.

---

**Create Channel**

```
POST /guilds/{guild.id}/channels
```

Request body:
```json
{
  "name": "esmer-tasks",
  "type": 0,
  "topic": "Delegated tasks managed by Esmer",
  "parent_id": "category_id"
}
```

---

**Modify Channel**

```
PATCH /channels/{channel.id}
```

Request body:
```json
{
  "name": "new-channel-name",
  "topic": "Updated topic"
}
```

---

**Delete Channel**

```
DELETE /channels/{channel.id}
```

---

**List Guild Channels**

```
GET /guilds/{guild.id}/channels
```

---

#### Messages

**Get Channel Messages**

```
GET /channels/{channel.id}/messages
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `around` | snowflake | No | Get messages around this message ID |
| `before` | snowflake | No | Get messages before this ID |
| `after` | snowflake | No | Get messages after this ID |
| `limit` | integer | No | Max messages to return (1-100, default 50) |

Response:
```json
[
  {
    "id": "msg_id",
    "type": 0,
    "content": "Hello from Esmer",
    "channel_id": "channel_id",
    "author": {
      "id": "user_id",
      "username": "esmer_bot",
      "discriminator": "0",
      "bot": true
    },
    "timestamp": "2026-02-09T10:00:00.000000+00:00",
    "edited_timestamp": null,
    "tts": false,
    "mention_everyone": false,
    "mentions": [],
    "attachments": [],
    "embeds": [],
    "reactions": [],
    "pinned": false
  }
]
```

---

**Get Single Message**

```
GET /channels/{channel.id}/messages/{message.id}
```

---

**Create Message**

```
POST /channels/{channel.id}/messages
```

Request body:
```json
{
  "content": "Hello from Esmer!",
  "tts": false,
  "embeds": [
    {
      "title": "Task Delegation",
      "description": "A new task has been assigned.",
      "color": 3447003,
      "fields": [
        { "name": "Task", "value": "Review PR #42", "inline": true },
        { "name": "Due", "value": "2026-02-10", "inline": true }
      ],
      "footer": { "text": "Esmer AI Assistant" },
      "timestamp": "2026-02-09T10:00:00.000Z"
    }
  ],
  "message_reference": {
    "message_id": "parent_message_id"
  },
  "components": [
    {
      "type": 1,
      "components": [
        {
          "type": 2,
          "label": "Approve",
          "style": 3,
          "custom_id": "approve_task_42"
        },
        {
          "type": 2,
          "label": "Decline",
          "style": 4,
          "custom_id": "decline_task_42"
        }
      ]
    }
  ]
}
```

For file uploads, use `multipart/form-data` with the JSON payload in a `payload_json` field.

---

**Edit Message**

```
PATCH /channels/{channel.id}/messages/{message.id}
```

**Delete Message**

```
DELETE /channels/{channel.id}/messages/{message.id}
```

**Bulk Delete Messages**

```
POST /channels/{channel.id}/messages/bulk-delete
```

Request body:
```json
{
  "messages": ["msg_id_1", "msg_id_2"]
}
```

Messages must be less than 14 days old. Min 2 IDs, max 100 IDs.

---

**Create Reaction**

```
PUT /channels/{channel.id}/messages/{message.id}/reactions/{emoji}/@me
```

The `{emoji}` is URL-encoded (e.g., `%F0%9F%91%8D` for thumbs up, or `name:id` for custom emoji).

---

**Delete Reaction**

```
DELETE /channels/{channel.id}/messages/{message.id}/reactions/{emoji}/@me
```

**Get Reactions**

```
GET /channels/{channel.id}/messages/{message.id}/reactions/{emoji}
```

---

**Pin Message**

```
PUT /channels/{channel.id}/pins/{message.id}
```

**Unpin Message**

```
DELETE /channels/{channel.id}/pins/{message.id}
```

**Get Pinned Messages**

```
GET /channels/{channel.id}/pins
```

---

#### Members

**List Guild Members**

```
GET /guilds/{guild.id}/members
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `limit` | integer | No | Max members (1-1000, default 1) |
| `after` | snowflake | No | Get members after this user ID |

---

**Get Guild Member**

```
GET /guilds/{guild.id}/members/{user.id}
```

---

**Add Role to Member**

```
PUT /guilds/{guild.id}/members/{user.id}/roles/{role.id}
```

**Remove Role from Member**

```
DELETE /guilds/{guild.id}/members/{user.id}/roles/{role.id}
```

### 7.6 Webhooks / Real-time

**Discord Gateway (WebSocket):**

Discord uses a WebSocket-based Gateway for real-time events.

Gateway URL: `wss://gateway.discord.gg/?v=10&encoding=json`

Key events:
- `MESSAGE_CREATE` -- New message in a channel
- `MESSAGE_UPDATE` -- Message was edited
- `MESSAGE_DELETE` -- Message was deleted
- `MESSAGE_REACTION_ADD` / `MESSAGE_REACTION_REMOVE`
- `GUILD_MEMBER_ADD` / `GUILD_MEMBER_REMOVE`
- `CHANNEL_CREATE` / `CHANNEL_UPDATE` / `CHANNEL_DELETE`
- `INTERACTION_CREATE` -- Slash command or button click

Gateway requires heartbeating and session resuming. Use a library (e.g., discord.js) rather than implementing raw WebSocket handling.

**Channel Webhooks (simple):**

```
POST /channels/{channel.id}/webhooks
```

Create a webhook that can post messages to a specific channel without bot authentication.

Execute webhook:
```
POST /webhooks/{webhook.id}/{webhook.token}
```

### 7.7 Error Handling

Discord returns standard HTTP status codes with a JSON error body:

```json
{
  "code": 50035,
  "errors": {
    "content": {
      "_errors": [
        { "code": "BASE_TYPE_MAX_LENGTH", "message": "Must be 2000 or fewer in length." }
      ]
    }
  },
  "message": "Invalid Form Body"
}
```

| Code | Meaning | Retry |
|---|---|---|
| `400` | Bad request | Do not retry; fix request |
| `401` | Unauthorized | Check token |
| `403` | Missing permissions | Grant bot permissions |
| `404` | Resource not found | Do not retry |
| `405` | Method not allowed | Fix HTTP method |
| `429` | Rate limited | Wait `retry_after` from response body (in seconds) |
| `500` | Server error | Retry with backoff |
| `502` | Gateway unavailable | Retry after 1-5 seconds |

### 7.8 Mobile-Specific Notes

- **Bot Token Security:** Never embed the bot token in the mobile app. Route all API calls through Esmer's backend.
- **Gateway:** Do not connect to the Discord Gateway from a mobile app. Use the backend for real-time event handling and push results to the mobile app via push notifications.
- **Embeds:** Use rich embeds for visually formatted messages. Keep embed payloads compact (max 6,000 characters total across all embeds).
- **Message Components:** Use buttons and select menus for interactive task delegation. Handle `INTERACTION_CREATE` events server-side.
- **Content Length:** Messages are limited to 2,000 characters. Embeds allow more content (4,096 chars in description).
- **Snowflake IDs:** All Discord IDs are snowflakes (64-bit integers). Parse them as strings in JavaScript/React Native to avoid precision loss.

---

## 8. Google Chat (Google API)

### 8.1 Service Overview

Google Chat is Google Workspace's messaging platform for teams. Esmer uses it to send messages to spaces (rooms), manage memberships, and retrieve space information. The API supports both direct messages and space-based conversations. Google Chat is available only for Google Workspace accounts.

### 8.2 Authentication

**Method:** OAuth 2.0 or Service Account

| Property | Value |
|---|---|
| Authorization URL | `https://accounts.google.com/o/oauth2/v2/auth` |
| Token URL | `https://oauth2.googleapis.com/token` |
| Revoke URL | `https://oauth2.googleapis.com/revoke` |
| Grant Type | `authorization_code` |
| PKCE Support | Yes |

**Required Scopes:**

| Scope | Purpose |
|---|---|
| `https://www.googleapis.com/auth/chat.spaces` | View and manage spaces |
| `https://www.googleapis.com/auth/chat.spaces.readonly` | View spaces |
| `https://www.googleapis.com/auth/chat.messages` | Send, read, update, delete messages |
| `https://www.googleapis.com/auth/chat.messages.readonly` | Read messages |
| `https://www.googleapis.com/auth/chat.messages.create` | Send messages only |
| `https://www.googleapis.com/auth/chat.memberships` | View and manage memberships |
| `https://www.googleapis.com/auth/chat.memberships.readonly` | View memberships |

**Esmer Recommendation:** Use `chat.messages` and `chat.spaces.readonly` for core functionality. Add `chat.memberships.readonly` if membership listing is needed. Service Accounts require domain-wide delegation for impersonation.

### 8.3 Base URL

```
https://chat.googleapis.com/v1
```

### 8.4 Rate Limits

| Limit | Value |
|---|---|
| Per-project quota | 6,000 requests per minute |
| Messages.create | 300 per minute per space |
| Messages.list | 900 per minute per space |
| Spaces.list | 600 per minute |
| Memberships.list | 600 per minute |
| Max message size | 32,768 bytes |

### 8.5 API Endpoints

#### Spaces

**List Spaces**

```
GET /v1/spaces
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `pageSize` | integer | No | Max spaces per page (default 100, max 1000) |
| `pageToken` | string | No | Pagination token |
| `filter` | string | No | Filter expression |

Response:
```json
{
  "spaces": [
    {
      "name": "spaces/AAAA1234",
      "type": "ROOM",
      "displayName": "Engineering Team",
      "spaceType": "SPACE",
      "singleUserBotDm": false,
      "threaded": false,
      "spaceThreadingState": "THREADED_MESSAGES"
    }
  ],
  "nextPageToken": "token"
}
```

---

**Get Space**

```
GET /v1/{name}
```

Where `{name}` is `spaces/{spaceId}`.

---

#### Members

**List Members**

```
GET /v1/{parent}/members
```

Where `{parent}` is `spaces/{spaceId}`.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `pageSize` | integer | No | Max results per page |
| `pageToken` | string | No | Pagination token |

Response:
```json
{
  "memberships": [
    {
      "name": "spaces/AAAA1234/members/user123",
      "member": {
        "name": "users/user123",
        "displayName": "John Doe",
        "type": "HUMAN"
      },
      "state": "JOINED",
      "role": "ROLE_MEMBER"
    }
  ]
}
```

---

**Get Member**

```
GET /v1/{name}
```

Where `{name}` is `spaces/{spaceId}/members/{memberId}`.

---

#### Messages

**Create Message**

```
POST /v1/{parent}/messages
```

Where `{parent}` is `spaces/{spaceId}`.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `threadKey` | string | No | Thread key for threaded replies |
| `requestId` | string | No | Idempotency key |
| `messageReplyOption` | string | No | `REPLY_MESSAGE_FALLBACK_TO_NEW_THREAD` or `REPLY_MESSAGE_OR_FAIL` |

Request body:
```json
{
  "text": "Hello from Esmer! Here is your task update.",
  "thread": {
    "threadKey": "task-42-thread"
  },
  "cardsV2": [
    {
      "cardId": "taskCard",
      "card": {
        "header": {
          "title": "Task Delegation",
          "subtitle": "Review PR #42"
        },
        "sections": [
          {
            "widgets": [
              {
                "decoratedText": {
                  "topLabel": "Due Date",
                  "text": "February 10, 2026"
                }
              },
              {
                "buttonList": {
                  "buttons": [
                    {
                      "text": "Approve",
                      "onClick": {
                        "action": { "function": "approve_task", "parameters": [{ "key": "taskId", "value": "42" }] }
                      }
                    }
                  ]
                }
              }
            ]
          }
        ]
      }
    }
  ]
}
```

Response:
```json
{
  "name": "spaces/AAAA1234/messages/msg123",
  "sender": {
    "name": "users/bot123",
    "displayName": "Esmer",
    "type": "BOT"
  },
  "createTime": "2026-02-09T10:00:00Z",
  "text": "Hello from Esmer!",
  "thread": {
    "name": "spaces/AAAA1234/threads/thread123",
    "threadKey": "task-42-thread"
  }
}
```

---

**Get Message**

```
GET /v1/{name}
```

Where `{name}` is `spaces/{spaceId}/messages/{messageId}`.

---

**Update Message**

```
PUT /v1/{name}
```

Request body:
```json
{
  "text": "Updated message text",
  "cardsV2": []
}
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `updateMask` | string | No | Fields to update (e.g., `text,cardsV2`) |

---

**Delete Message**

```
DELETE /v1/{name}
```

### 8.6 Webhooks / Real-time

Google Chat uses the Google Chat API events framework:

**Incoming Webhooks (simple):**

Create a webhook URL in a Google Chat space, then POST to it:

```
POST https://chat.googleapis.com/v1/spaces/{space}/messages?key={key}&token={token}
```

Request body:
```json
{
  "text": "Notification from Esmer"
}
```

**App Events (for interactive bots):**

Google Chat sends HTTP POST requests to a configured endpoint when:
- Bot is added to or removed from a space (`ADDED_TO_SPACE` / `REMOVED_FROM_SPACE`)
- User sends a message mentioning the bot (`MESSAGE`)
- User clicks a card action (`CARD_CLICKED`)

Payload:
```json
{
  "type": "MESSAGE",
  "eventTime": "2026-02-09T10:00:00Z",
  "message": {
    "name": "spaces/AAAA1234/messages/msg456",
    "sender": { "name": "users/user789", "displayName": "Jane" },
    "text": "@Esmer delegate this task",
    "thread": { "name": "spaces/AAAA1234/threads/thread789" },
    "space": { "name": "spaces/AAAA1234", "type": "ROOM" }
  },
  "user": { "name": "users/user789", "displayName": "Jane" },
  "space": { "name": "spaces/AAAA1234", "displayName": "Engineering" }
}
```

### 8.7 Error Handling

| Code | Meaning | Retry |
|---|---|---|
| `400` | Invalid request | Do not retry; fix parameters |
| `401` | Unauthorized | Refresh token, retry |
| `403` | Permission denied or not in space | Do not retry; check access |
| `404` | Space or message not found | Do not retry |
| `429` | Quota exceeded | Exponential backoff |
| `500` | Internal error | Retry with backoff |
| `503` | Service unavailable | Retry with backoff |

### 8.8 Mobile-Specific Notes

- **Workspace Only:** Google Chat API is only available for Google Workspace accounts, not consumer Gmail.
- **Service Account:** For server-to-server communication (no user context), use Service Account with domain-wide delegation.
- **Cards V2:** Use Cards V2 for rich interactive messages. Keep card payloads small to render quickly on mobile.
- **Thread Keys:** Use `threadKey` to group related messages into threads, enabling conversation-like experiences.
- **Idempotency:** Use `requestId` parameter to prevent duplicate messages on retry from flaky mobile connections.

---

## 9. Mattermost

### 9.1 Service Overview

Mattermost is an open-source, self-hosted messaging platform designed for team collaboration. Esmer uses Mattermost to post messages, manage channels, handle reactions, and manage users. The API is RESTful and follows standard HTTP conventions.

### 9.2 Authentication

**Method:** Personal Access Token (recommended) or Session Token

| Property | Value |
|---|---|
| Auth Method | `Authorization: Bearer {token}` header |
| Token Source | Profile > Security > Personal Access Tokens |
| Revoke | Profile > Security > Delete token |

No OAuth flow; tokens are generated directly in the Mattermost UI.

**Esmer Recommendation:** Use a Personal Access Token generated from a dedicated service account. Ensure the account has `post:channel` permission. Enable personal access tokens in System Console if not already enabled.

### 9.3 Base URL

```
https://{your-mattermost-domain}/api/v4
```

The base URL is configurable since Mattermost is self-hosted.

### 9.4 Rate Limits

| Limit | Value |
|---|---|
| Default rate limit | 10 requests per second per connection |
| API rate limit | Configurable per-deployment |
| WebSocket events | No explicit limit; subject to server capacity |
| File upload | Default max 50 MB per file |
| Post length | Max 16,383 characters |

Rate limits are configurable by the system administrator in `config.json` under `RateLimitSettings`.

### 9.5 API Endpoints

#### Channels

**Create Channel**

```
POST /api/v4/channels
```

Request body:
```json
{
  "team_id": "team-id",
  "name": "esmer-tasks",
  "display_name": "Esmer Tasks",
  "type": "O",
  "purpose": "Task delegation via Esmer",
  "header": "Managed by Esmer AI"
}
```

Type: `O` = open (public), `P` = private, `D` = direct, `G` = group.

---

**Search Channels**

```
POST /api/v4/teams/{team_id}/channels/search
```

Request body:
```json
{
  "term": "esmer"
}
```

---

**Get Channel**

```
GET /api/v4/channels/{channel_id}
```

**Delete Channel (Soft)**

```
DELETE /api/v4/channels/{channel_id}
```

**Restore Channel**

```
POST /api/v4/channels/{channel_id}/restore
```

---

**Get Channel Statistics**

```
GET /api/v4/channels/{channel_id}/stats
```

Response:
```json
{
  "channel_id": "channel-id",
  "member_count": 42,
  "guest_count": 2,
  "pinnedpost_count": 5
}
```

---

**Add User to Channel**

```
POST /api/v4/channels/{channel_id}/members
```

Request body:
```json
{
  "user_id": "user-id"
}
```

---

**Get Channel Members**

```
GET /api/v4/channels/{channel_id}/members
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `page` | integer | No | Page number (default 0) |
| `per_page` | integer | No | Results per page (default 60, max 200) |

---

#### Messages (Posts)

**Create Post**

```
POST /api/v4/posts
```

Request body:
```json
{
  "channel_id": "channel-id",
  "message": "Hello from Esmer! Your task has been delegated.",
  "root_id": "",
  "props": {
    "attachments": [
      {
        "fallback": "Task: Review Q1 Report",
        "color": "#36a64f",
        "title": "Task Delegation",
        "text": "Review Q1 Report - Due: Feb 10",
        "fields": [
          { "short": true, "title": "Assignee", "value": "John Doe" },
          { "short": true, "title": "Priority", "value": "High" }
        ]
      }
    ]
  }
}
```

For threaded replies, set `root_id` to the parent post ID.

Response:
```json
{
  "id": "post-id",
  "create_at": 1706000000000,
  "update_at": 1706000000000,
  "delete_at": 0,
  "user_id": "user-id",
  "channel_id": "channel-id",
  "root_id": "",
  "message": "Hello from Esmer!",
  "type": "",
  "props": {}
}
```

---

**Create Ephemeral Post**

```
POST /api/v4/posts/ephemeral
```

Request body:
```json
{
  "user_id": "target-user-id",
  "post": {
    "channel_id": "channel-id",
    "message": "Only you can see this message."
  }
}
```

---

**Get Post**

```
GET /api/v4/posts/{post_id}
```

**Delete Post (Soft)**

```
DELETE /api/v4/posts/{post_id}
```

---

#### Reactions

**Add Reaction**

```
POST /api/v4/reactions
```

Request body:
```json
{
  "user_id": "user-id",
  "post_id": "post-id",
  "emoji_name": "thumbsup"
}
```

---

**Remove Reaction**

```
DELETE /api/v4/users/{user_id}/posts/{post_id}/reactions/{emoji_name}
```

---

**Get Post Reactions**

```
GET /api/v4/posts/{post_id}/reactions
```

---

#### Users

**Create User**

```
POST /api/v4/users
```

Request body:
```json
{
  "email": "user@example.com",
  "username": "newuser",
  "password": "SecureP@ss123"
}
```

---

**Get User by ID**

```
GET /api/v4/users/{user_id}
```

**Get User by Email**

```
GET /api/v4/users/email/{email}
```

**Get All Users**

```
GET /api/v4/users
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `page` | integer | No | Page number |
| `per_page` | integer | No | Results per page (max 200) |
| `in_team` | string | No | Filter by team ID |
| `in_channel` | string | No | Filter by channel ID |

---

**Deactivate User**

```
DELETE /api/v4/users/{user_id}
```

---

**Invite User to Team**

```
POST /api/v4/teams/{team_id}/invite/email
```

Request body:
```json
["user1@example.com", "user2@example.com"]
```

### 9.6 Webhooks / Real-time

**WebSocket API:**

```
wss://{your-mattermost-domain}/api/v4/websocket
```

Authenticate by sending:
```json
{
  "seq": 1,
  "action": "authentication_challenge",
  "data": { "token": "bearer-token" }
}
```

Key events:
- `posted` -- New message posted
- `post_edited` -- Message edited
- `post_deleted` -- Message deleted
- `channel_created` -- Channel created
- `user_added` -- User added to channel
- `user_removed` -- User removed from channel
- `reaction_added` / `reaction_removed`
- `typing` -- User is typing

**Incoming Webhooks:**

Create in Integrations > Incoming Webhooks. Send POST to the generated URL:

```json
{
  "text": "Message from Esmer",
  "channel": "esmer-tasks",
  "username": "esmer-bot",
  "icon_url": "https://esmer.app/icon.png"
}
```

**Outgoing Webhooks:**

Trigger on keywords or channels. Mattermost sends POST to a configured URL with the triggering message data.

### 9.7 Error Handling

| Code | Meaning | Retry |
|---|---|---|
| `400` | Bad request | Do not retry; fix parameters |
| `401` | Unauthorized | Check token validity |
| `403` | Forbidden (insufficient permissions) | Request permissions from admin |
| `404` | Resource not found | Do not retry |
| `429` | Rate limited | Retry after delay |
| `501` | Feature disabled | Contact admin to enable |

Mattermost error response:
```json
{
  "id": "api.post.create_post.channel_id.invalid",
  "message": "Invalid channel id",
  "detailed_error": "",
  "request_id": "abc123",
  "status_code": 400
}
```

### 9.8 Mobile-Specific Notes

- **Self-Hosted URL:** The base URL must be configured per-deployment. Esmer should support custom server URLs.
- **SSL Issues:** Some self-hosted instances use self-signed certificates. Provide an option to ignore SSL validation (with warning).
- **WebSocket:** Use WebSocket for real-time updates but implement reconnection logic with exponential backoff.
- **Personal Access Tokens:** Must be enabled by the system administrator. Document this requirement for Esmer users.
- **File Uploads:** Use multipart/form-data for file uploads. Respect the server's configured file size limit.

---

## 10. Matrix (Synapse API)

### 10.1 Service Overview

Matrix is an open standard for decentralized, real-time communication. Esmer uses the Matrix Client-Server API to send messages and media to rooms, manage room memberships, and retrieve account information. Matrix is federated, meaning users on different homeservers can communicate. The reference server implementation is Synapse.

### 10.2 Authentication

**Method:** Access Token (password-based login or SSO)

| Property | Value |
|---|---|
| Auth Method | `Authorization: Bearer {access_token}` header or `?access_token={token}` query param |
| Token Source | Login endpoint or client settings (Settings > Help & About > Advanced) |
| Session Management | Tokens are per-device; multiple tokens can exist |

**Login to Obtain Token:**

```
POST /_matrix/client/v3/login
```

Request body:
```json
{
  "type": "m.login.password",
  "identifier": {
    "type": "m.id.user",
    "user": "@esmer:matrix.org"
  },
  "password": "secret",
  "initial_device_display_name": "Esmer Mobile Client"
}
```

Response:
```json
{
  "user_id": "@esmer:matrix.org",
  "access_token": "syt_abc123...",
  "device_id": "ESMER_DEVICE_01",
  "home_server": "matrix.org"
}
```

**Esmer Recommendation:** Use the login endpoint to obtain an access token programmatically. Store the access token securely. Each device/session gets its own token. Use SSO (OAuth 2.0) when the homeserver supports it for better security.

### 10.3 Base URL

```
https://{homeserver}/_matrix/client/v3
```

Default public homeserver: `https://matrix.org`

The homeserver URL is user-configurable since Matrix is federated.

### 10.4 Rate Limits

| Limit | Value |
|---|---|
| Default rate limit | Varies by homeserver configuration |
| Synapse default | 3 requests per second per user (configurable) |
| Login attempts | 3 per 5 minutes per account |
| Room joins | Rate-limited by homeserver |
| Message sending | Typically 10 per second (configurable) |
| File upload max size | Configurable per server (default 50 MB on matrix.org) |

Rate limited responses return `429` with:
```json
{
  "errcode": "M_LIMIT_EXCEEDED",
  "error": "Too many requests",
  "retry_after_ms": 5000
}
```

### 10.5 API Endpoints

#### Account

**Get Account Info (Whoami)**

```
GET /_matrix/client/v3/account/whoami
```

Response:
```json
{
  "user_id": "@esmer:matrix.org",
  "device_id": "ESMER_DEVICE_01",
  "is_guest": false
}
```

---

**Get Display Name**

```
GET /_matrix/client/v3/profile/{userId}/displayname
```

**Set Display Name**

```
PUT /_matrix/client/v3/profile/{userId}/displayname
```

Request body:
```json
{
  "displayname": "Esmer AI"
}
```

---

**Get Avatar URL**

```
GET /_matrix/client/v3/profile/{userId}/avatar_url
```

---

#### Rooms

**Create Room**

```
POST /_matrix/client/v3/createRoom
```

Request body:
```json
{
  "name": "Esmer Task Room",
  "topic": "Delegated tasks managed by Esmer",
  "visibility": "private",
  "preset": "private_chat",
  "invite": ["@user1:matrix.org", "@user2:matrix.org"],
  "is_direct": false,
  "room_alias_name": "esmer-tasks"
}
```

Response:
```json
{
  "room_id": "!abcdef:matrix.org"
}
```

Presets: `private_chat`, `trusted_private_chat`, `public_chat`.

---

**Join Room**

```
POST /_matrix/client/v3/join/{roomIdOrAlias}
```

---

**Leave Room**

```
POST /_matrix/client/v3/rooms/{roomId}/leave
```

---

**Invite User**

```
POST /_matrix/client/v3/rooms/{roomId}/invite
```

Request body:
```json
{
  "user_id": "@user1:matrix.org"
}
```

---

**Kick User**

```
POST /_matrix/client/v3/rooms/{roomId}/kick
```

Request body:
```json
{
  "user_id": "@user1:matrix.org",
  "reason": "Removed from project"
}
```

---

**List Joined Rooms**

```
GET /_matrix/client/v3/joined_rooms
```

Response:
```json
{
  "joined_rooms": ["!abc:matrix.org", "!def:matrix.org"]
}
```

---

#### Room Members

**Get Room Members**

```
GET /_matrix/client/v3/rooms/{roomId}/members
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `at` | string | No | Point in time to return members for |
| `membership` | string | No | Filter by membership: `join`, `invite`, `leave`, `ban` |
| `not_membership` | string | No | Exclude this membership type |

Response:
```json
{
  "chunk": [
    {
      "type": "m.room.member",
      "state_key": "@user1:matrix.org",
      "content": {
        "membership": "join",
        "displayname": "User One",
        "avatar_url": "mxc://matrix.org/abc..."
      },
      "room_id": "!abc:matrix.org",
      "sender": "@user1:matrix.org"
    }
  ]
}
```

---

#### Messages

**Send Message**

```
PUT /_matrix/client/v3/rooms/{roomId}/send/{eventType}/{txnId}
```

Where `{eventType}` is `m.room.message` and `{txnId}` is a unique transaction ID (for idempotency).

Request body (text):
```json
{
  "msgtype": "m.text",
  "body": "Hello from Esmer! Your task has been delegated."
}
```

Request body (formatted):
```json
{
  "msgtype": "m.text",
  "body": "Hello from **Esmer**!",
  "format": "org.matrix.custom.html",
  "formatted_body": "Hello from <b>Esmer</b>!"
}
```

Request body (notice -- bot message):
```json
{
  "msgtype": "m.notice",
  "body": "Task update: Review PR #42 is due tomorrow."
}
```

Response:
```json
{
  "event_id": "$abc123:matrix.org"
}
```

---

**Get Room Messages**

```
GET /_matrix/client/v3/rooms/{roomId}/messages
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `dir` | string | Yes | Direction: `b` (backwards/older) or `f` (forwards/newer) |
| `from` | string | Yes | Pagination token (use `prev_batch` from sync) |
| `to` | string | No | End pagination token |
| `limit` | integer | No | Max events to return (default 10) |
| `filter` | string | No | JSON-encoded event filter |

Response:
```json
{
  "start": "t47-42_42_0_0_0",
  "end": "t47-43_43_0_0_0",
  "chunk": [
    {
      "type": "m.room.message",
      "content": {
        "msgtype": "m.text",
        "body": "Hello"
      },
      "event_id": "$abc123",
      "sender": "@user1:matrix.org",
      "origin_server_ts": 1706000000000,
      "room_id": "!abc:matrix.org"
    }
  ]
}
```

---

#### Events

**Get Event**

```
GET /_matrix/client/v3/rooms/{roomId}/event/{eventId}
```

---

#### Media

**Upload Media**

```
POST /_matrix/media/v3/upload
```

Uses binary body with `Content-Type` header set to the file MIME type.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `filename` | string | No | Filename for the uploaded file |

Response:
```json
{
  "content_uri": "mxc://matrix.org/abc123"
}
```

---

**Send Media Message (Image)**

```
PUT /_matrix/client/v3/rooms/{roomId}/send/m.room.message/{txnId}
```

Request body:
```json
{
  "msgtype": "m.image",
  "body": "screenshot.png",
  "url": "mxc://matrix.org/abc123",
  "info": {
    "mimetype": "image/png",
    "size": 123456,
    "w": 1920,
    "h": 1080
  }
}
```

Other media types: `m.file`, `m.audio`, `m.video`.

---

**Download Media**

```
GET /_matrix/media/v3/download/{serverName}/{mediaId}
```

### 10.6 Webhooks / Real-time

**Sync API (long polling):**

```
GET /_matrix/client/v3/sync
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `since` | string | No | Token from previous sync |
| `timeout` | integer | No | Long poll timeout in ms (default 0, max 30000) |
| `filter` | string | No | Filter ID or JSON filter |
| `full_state` | boolean | No | Return full state for all rooms |
| `set_presence` | string | No | `offline`, `online`, `unavailable` |

The sync endpoint is the primary way to receive real-time updates in Matrix. It returns all new events since the last `since` token.

Response includes:
- `rooms.join` -- Events in joined rooms
- `rooms.invite` -- New room invites
- `rooms.leave` -- Left rooms
- `presence` -- Presence updates
- `account_data` -- Account data changes
- `to_device` -- Device-to-device messages

**Application Services (for bots):**

Matrix supports Application Services that receive events via HTTP push. Configure in `homeserver.yaml`. This is the preferred approach for server-side bot implementations.

### 10.7 Error Handling

Matrix uses standard HTTP codes with a JSON error body:

```json
{
  "errcode": "M_FORBIDDEN",
  "error": "You are not invited to this room."
}
```

| Errcode | HTTP Code | Meaning | Retry |
|---|---|---|---|
| `M_FORBIDDEN` | 403 | Not allowed | Do not retry; check permissions |
| `M_UNKNOWN_TOKEN` | 401 | Invalid/expired token | Re-login, retry |
| `M_MISSING_TOKEN` | 401 | No token provided | Add token |
| `M_NOT_FOUND` | 404 | Resource not found | Do not retry |
| `M_LIMIT_EXCEEDED` | 429 | Rate limited | Wait `retry_after_ms`, then retry |
| `M_NOT_JSON` | 400 | Request body not valid JSON | Fix request |
| `M_UNKNOWN` | 500 | Unknown server error | Retry with backoff |
| `M_BAD_JSON` | 400 | Malformed JSON | Fix request |
| `M_USER_IN_USE` | 400 | Username already taken | Do not retry |
| `M_ROOM_IN_USE` | 400 | Room alias already taken | Do not retry |

### 10.8 Mobile-Specific Notes

- **Federated URL:** The homeserver URL varies per user. Esmer must support configurable homeserver URLs.
- **Sync:** Use the `/sync` endpoint with long polling for real-time updates. Store the `since` token locally for incremental sync.
- **Transaction IDs:** Use unique `txnId` values for message sending to ensure idempotency on flaky connections.
- **Media via MXC:** Media uses `mxc://` URIs. Convert to HTTP download URLs using the media download endpoint.
- **E2EE:** Matrix supports end-to-end encryption (E2EE) via Olm/Megolm. Implementing E2EE on mobile is complex; consider using the Matrix SDK (matrix-js-sdk or matrix-rust-sdk) rather than raw API calls.
- **Presence:** Set presence to `offline` during sync to avoid battery drain from presence pings.

---

## 11. Zulip

### 11.1 Service Overview

Zulip is an open-source team chat application with a unique topic-based threading model. Messages are organized into streams (channels) and topics (threads within channels). Esmer uses Zulip to send messages to streams and direct messages, manage streams, and manage users.

### 11.2 Authentication

**Method:** API Key (email + API key)

| Property | Value |
|---|---|
| Auth Method | HTTP Basic Auth (`email:api_key`) |
| Token Source | Settings > Account & Privacy > API Key |
| Encoding | Base64-encoded `email:api_key` in `Authorization: Basic {encoded}` header |

**Esmer Recommendation:** Use a dedicated bot account or the user's own API key. API keys do not expire but can be regenerated. Store the email and API key pair securely.

### 11.3 Base URL

```
https://{your-zulip-domain}/api/v1
```

Public Zulip Cloud: `https://{org}.zulipchat.com/api/v1`

### 11.4 Rate Limits

| Limit | Value |
|---|---|
| Default | 200 requests per minute per user |
| Sending messages | 12 per minute per user (configurable) |
| Upload files | Varies by deployment |
| Burst allowance | Short bursts above the limit are allowed |

Rate limited responses return `429` with:
```json
{
  "result": "error",
  "msg": "API usage exceeded rate limit",
  "code": "RATE_LIMIT_HIT",
  "retry-after": 1.5
}
```

### 11.5 API Endpoints

#### Messages

**Send Message to Stream**

```
POST /api/v1/messages
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | `stream` for stream messages |
| `to` | string/integer | Yes | Stream name or stream ID |
| `topic` | string | Yes | Topic name (max 60 chars) |
| `content` | string | Yes | Message content (Markdown supported) |

Request body (form-encoded):
```
type=stream&to=engineering&topic=Task+Delegation&content=Hello+from+Esmer!
```

Response:
```json
{
  "result": "success",
  "msg": "",
  "id": 42
}
```

---

**Send Direct Message**

```
POST /api/v1/messages
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | `direct` (or `private` for backward compat) |
| `to` | array/string | Yes | JSON-encoded list of user IDs or email addresses |
| `content` | string | Yes | Message content |

---

**Get Message**

```
GET /api/v1/messages/{message_id}
```

Response:
```json
{
  "result": "success",
  "msg": "",
  "message": {
    "id": 42,
    "sender_id": 123,
    "content": "Hello from Esmer!",
    "content_type": "text/html",
    "recipient_id": 456,
    "timestamp": 1706000000,
    "subject": "Task Delegation",
    "type": "stream",
    "display_recipient": "engineering",
    "stream_id": 789
  }
}
```

---

**Update Message**

```
PATCH /api/v1/messages/{message_id}
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `content` | string | No | New message content |
| `topic` | string | No | New topic (stream messages only) |
| `propagate_mode` | string | No | `change_one`, `change_later`, `change_all` |

---

**Delete Message**

```
DELETE /api/v1/messages/{message_id}
```

---

**Get Messages (History)**

```
GET /api/v1/messages
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `anchor` | string/integer | Yes | Message ID to anchor around, or `newest`/`oldest`/`first_unread` |
| `num_before` | integer | Yes | Number of messages before anchor |
| `num_after` | integer | Yes | Number of messages after anchor |
| `narrow` | string | No | JSON-encoded narrow filter (e.g., `[{"operator":"stream","operand":"engineering"}]`) |
| `apply_markdown` | boolean | No | Apply Markdown rendering (default true) |

---

**Upload File**

```
POST /api/v1/user_uploads
```

Uses `multipart/form-data` with a `file` field.

Response:
```json
{
  "result": "success",
  "msg": "",
  "uri": "/user_uploads/1/abc/document.pdf"
}
```

Reference the file in a message: `[document.pdf](/user_uploads/1/abc/document.pdf)`

---

#### Streams

**Get All Streams**

```
GET /api/v1/streams
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `include_public` | boolean | No | Include public streams (default true) |
| `include_subscribed` | boolean | No | Include subscribed streams (default true) |
| `include_all_active` | boolean | No | Include all active streams (admin only) |

Response:
```json
{
  "result": "success",
  "streams": [
    {
      "stream_id": 1,
      "name": "engineering",
      "description": "Engineering team discussions",
      "invite_only": false,
      "is_web_public": false,
      "date_created": 1706000000
    }
  ]
}
```

---

**Get Subscribed Streams**

```
GET /api/v1/users/me/subscriptions
```

---

**Create Stream (Subscribe)**

```
POST /api/v1/users/me/subscriptions
```

Request body (form-encoded):
```
subscriptions=[{"name":"new-stream","description":"Created by Esmer"}]
```

---

**Update Stream**

```
PATCH /api/v1/streams/{stream_id}
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `description` | string | No | New description |
| `new_name` | string | No | New stream name |
| `is_private` | boolean | No | Make stream private |

---

**Delete Stream**

```
DELETE /api/v1/streams/{stream_id}
```

---

#### Users

**Get All Users**

```
GET /api/v1/users
```

Response:
```json
{
  "result": "success",
  "members": [
    {
      "user_id": 123,
      "email": "user@example.com",
      "full_name": "John Doe",
      "is_bot": false,
      "is_active": true,
      "role": 400,
      "avatar_url": "https://..."
    }
  ]
}
```

Roles: `100` = Organization owner, `200` = Organization admin, `300` = Moderator, `400` = Member, `600` = Guest.

---

**Get User**

```
GET /api/v1/users/{user_id}
```

**Create User**

```
POST /api/v1/users
```

**Update User**

```
PATCH /api/v1/users/{user_id}
```

**Deactivate User**

```
DELETE /api/v1/users/{user_id}
```

---

**Get Own User**

```
GET /api/v1/users/me
```

### 11.6 Webhooks / Real-time

**Register Event Queue (long polling):**

```
POST /api/v1/register
```

Request body:
```
event_types=["message","update_message"]&narrow=[["stream","engineering"]]
```

Response:
```json
{
  "result": "success",
  "queue_id": "abc123:1",
  "last_event_id": -1
}
```

---

**Get Events from Queue:**

```
GET /api/v1/events
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `queue_id` | string | Yes | Queue ID from register |
| `last_event_id` | integer | Yes | Last received event ID (-1 for first call) |

This is a long-polling endpoint that blocks until new events are available.

---

**Delete Queue:**

```
DELETE /api/v1/events
```

Event types: `message`, `update_message`, `delete_message`, `reaction`, `stream`, `subscription`, `presence`, `typing`.

**Incoming Webhooks:**

Zulip supports incoming webhooks for various services. Configure in Settings > Integrations.

### 11.7 Error Handling

| Result | Code | Meaning | Retry |
|---|---|---|---|
| `error` | `BAD_REQUEST` | Invalid parameters | Do not retry |
| `error` | `UNAUTHORIZED` | Invalid credentials | Re-authenticate |
| `error` | `RATE_LIMIT_HIT` | Rate limited | Wait `retry-after` seconds |
| `error` | `BAD_EVENT_QUEUE_ID` | Queue expired | Re-register queue |
| `error` | `BAD_NARROW` | Invalid narrow filter | Fix filter |

Response format:
```json
{
  "result": "error",
  "msg": "Invalid stream name 'nonexistent'",
  "code": "STREAM_DOES_NOT_EXIST"
}
```

### 11.8 Mobile-Specific Notes

- **Self-Hosted URL:** Zulip can be self-hosted or used via Zulip Cloud. Esmer must support configurable base URLs.
- **Topic Threading:** Zulip's unique topic model means every stream message must have a topic. This is unlike Slack where threads are optional. Ensure the Esmer UI enforces topic selection.
- **Narrow Filters:** Use narrow filters to fetch only relevant messages, reducing bandwidth on mobile.
- **Event Queue:** Use the register/events long-polling pattern for real-time updates. Handle `BAD_EVENT_QUEUE_ID` errors by re-registering.
- **Markdown:** Zulip uses its own Markdown flavor. Test message formatting for compatibility.
- **API Key Storage:** Store the email and API key pair in the platform secure keychain.

---

## 12. Webex (Cisco Webex API)

### 12.1 Service Overview

Cisco Webex is a collaboration platform providing messaging, video meetings, and calling. Esmer uses the Webex REST API to send and manage messages in spaces (rooms), create and manage meetings, and handle memberships. The API supports both 1:1 conversations and group spaces.

### 12.2 Authentication

**Method:** OAuth 2.0

| Property | Value |
|---|---|
| Authorization URL | `https://webexapis.com/v1/authorize` |
| Token URL | `https://webexapis.com/v1/access_token` |
| Revoke URL | N/A (tokens expire; delete integration to revoke) |
| Grant Type | `authorization_code` |
| Token Lifetime | Access token: 14 days; Refresh token: 90 days |

**Required Scopes:**

| Scope | Purpose |
|---|---|
| `spark:rooms_read` | View rooms/spaces |
| `spark:rooms_write` | Create/manage rooms |
| `spark:messages_read` | Read messages |
| `spark:messages_write` | Send messages |
| `spark:memberships_read` | View room memberships |
| `spark:memberships_write` | Manage room memberships |
| `meeting:schedules_read` | View meetings |
| `meeting:schedules_write` | Create/manage meetings |
| `meeting:recordings_read` | View recordings |
| `meeting:recordings_write` | Manage recordings |
| `meeting:preferences_read` | View meeting preferences |
| `spark:people_read` | View user information |

**Esmer Recommendation:** Use `spark:messages_write`, `spark:messages_read`, `spark:rooms_read`, and `meeting:schedules_write` as the minimum scope set for core delegation features.

### 12.3 Base URL

```
https://webexapis.com/v1
```

### 12.4 Rate Limits

| Limit | Value |
|---|---|
| Overall API | Varies; Webex uses dynamic rate limiting |
| Messages (create) | Approximately 1 message per second per room |
| Rooms (list) | Approximately 5 requests per second |
| Webhooks (create) | Approximately 1 per second |
| Retry-After header | Included in 429 responses |

Rate limit headers: `Retry-After` (seconds to wait).

### 12.5 API Endpoints

#### Messages

**List Messages**

```
GET /v1/messages
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `roomId` | string | Yes | Room ID to list messages from |
| `parentId` | string | No | Parent message ID (for threads) |
| `mentionedPeople` | string | No | Filter by mentioned people (`me` for self) |
| `before` | string | No | Messages before this datetime (ISO 8601) |
| `beforeMessage` | string | No | Messages before this message ID |
| `max` | integer | No | Max messages to return (default 50, max 1000) |

Response:
```json
{
  "items": [
    {
      "id": "msg-id-123",
      "roomId": "room-id-456",
      "roomType": "group",
      "text": "Hello from Esmer!",
      "personId": "person-id-789",
      "personEmail": "esmer@example.com",
      "created": "2026-02-09T10:00:00.000Z",
      "updated": "2026-02-09T10:00:00.000Z",
      "html": "<p>Hello from Esmer!</p>"
    }
  ]
}
```

---

**Create Message**

```
POST /v1/messages
```

Request body:
```json
{
  "roomId": "room-id-456",
  "text": "Hello from Esmer! Your task has been delegated.",
  "markdown": "**Hello from Esmer!** Your task has been delegated.\n\n- Task: Review Q1 Report\n- Due: Feb 10, 2026",
  "parentId": "parent-msg-id"
}
```

For direct messages (without a room), use `toPersonId` or `toPersonEmail` instead of `roomId`:

```json
{
  "toPersonEmail": "user@example.com",
  "text": "Direct message from Esmer"
}
```

For file attachments:
```json
{
  "roomId": "room-id",
  "text": "Here is the report",
  "files": ["https://example.com/report.pdf"]
}
```

Or use `multipart/form-data` to upload files directly.

Response:
```json
{
  "id": "msg-id-new",
  "roomId": "room-id-456",
  "text": "Hello from Esmer!",
  "personId": "person-id",
  "personEmail": "esmer@example.com",
  "created": "2026-02-09T10:00:00.000Z"
}
```

---

**Get Message**

```
GET /v1/messages/{messageId}
```

---

**Update Message**

```
PUT /v1/messages/{messageId}
```

Request body:
```json
{
  "roomId": "room-id-456",
  "text": "Updated message text",
  "markdown": "**Updated** message text"
}
```

---

**Delete Message**

```
DELETE /v1/messages/{messageId}
```

---

#### Rooms (Spaces)

**List Rooms**

```
GET /v1/rooms
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `teamId` | string | No | Filter by team |
| `type` | string | No | `direct` or `group` |
| `sortBy` | string | No | `id`, `lastactivity`, `created` |
| `max` | integer | No | Max results (default 100, max 1000) |

Response:
```json
{
  "items": [
    {
      "id": "room-id",
      "title": "Engineering Team",
      "type": "group",
      "isLocked": false,
      "lastActivity": "2026-02-09T10:00:00.000Z",
      "created": "2025-01-15T10:00:00.000Z",
      "creatorId": "person-id"
    }
  ]
}
```

---

**Create Room**

```
POST /v1/rooms
```

Request body:
```json
{
  "title": "Esmer Task Room",
  "teamId": "team-id"
}
```

---

**Get Room**

```
GET /v1/rooms/{roomId}
```

**Update Room**

```
PUT /v1/rooms/{roomId}
```

**Delete Room**

```
DELETE /v1/rooms/{roomId}
```

---

#### Memberships

**List Memberships**

```
GET /v1/memberships
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `roomId` | string | No | Filter by room |
| `personId` | string | No | Filter by person |
| `personEmail` | string | No | Filter by email |
| `max` | integer | No | Max results |

---

**Create Membership**

```
POST /v1/memberships
```

Request body:
```json
{
  "roomId": "room-id",
  "personEmail": "user@example.com",
  "isModerator": false
}
```

---

**Get, Update, Delete Membership**

```
GET /v1/memberships/{membershipId}
PUT /v1/memberships/{membershipId}
DELETE /v1/memberships/{membershipId}
```

---

#### Meetings

**Create Meeting**

```
POST /v1/meetings
```

Request body:
```json
{
  "title": "Task Review Meeting",
  "start": "2026-02-10T09:00:00Z",
  "end": "2026-02-10T09:30:00Z",
  "timezone": "America/Los_Angeles",
  "invitees": [
    { "email": "user@example.com", "displayName": "John Doe" }
  ],
  "enabledAutoRecordMeeting": false,
  "allowAnyUserToBeCoHost": false
}
```

Response:
```json
{
  "id": "meeting-id",
  "title": "Task Review Meeting",
  "meetingNumber": "123456789",
  "password": "abc123",
  "webLink": "https://cisco.webex.com/meet/abc123",
  "sipAddress": "123456789@cisco.webex.com",
  "start": "2026-02-10T09:00:00Z",
  "end": "2026-02-10T09:30:00Z",
  "state": "scheduled"
}
```

---

**List Meetings**

```
GET /v1/meetings
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `meetingType` | string | No | `scheduledMeeting`, `meeting` |
| `state` | string | No | `scheduled`, `ready`, `lobby`, `inProgress`, `ended` |
| `from` | string | No | Start time (ISO 8601) |
| `to` | string | No | End time (ISO 8601) |
| `max` | integer | No | Max results |

---

**Get Meeting**

```
GET /v1/meetings/{meetingId}
```

**Update Meeting**

```
PUT /v1/meetings/{meetingId}
```

**Delete Meeting**

```
DELETE /v1/meetings/{meetingId}
```

---

#### People

**List People**

```
GET /v1/people
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `email` | string | No | Filter by email |
| `displayName` | string | No | Filter by name |
| `max` | integer | No | Max results |

---

**Get Person (Self)**

```
GET /v1/people/me
```

**Get Person**

```
GET /v1/people/{personId}
```

### 12.6 Webhooks / Real-time

**Create Webhook**

```
POST /v1/webhooks
```

Request body:
```json
{
  "name": "Esmer Message Webhook",
  "targetUrl": "https://esmer-api.example.com/webhooks/webex",
  "resource": "messages",
  "event": "created",
  "filter": "roomId=room-id-456",
  "secret": "webhook-secret-for-hmac"
}
```

Supported resources and events:

| Resource | Events |
|---|---|
| `messages` | `created`, `deleted`, `updated` |
| `rooms` | `created`, `updated` |
| `memberships` | `created`, `updated`, `deleted` |
| `meetings` | `created`, `started`, `ended`, `cancelled` |
| `recordings` | `created`, `updated`, `deleted` |

---

**List Webhooks**

```
GET /v1/webhooks
```

**Get Webhook**

```
GET /v1/webhooks/{webhookId}
```

**Update Webhook**

```
PUT /v1/webhooks/{webhookId}
```

**Delete Webhook**

```
DELETE /v1/webhooks/{webhookId}
```

Webhook payload:
```json
{
  "id": "webhook-event-id",
  "name": "Esmer Message Webhook",
  "resource": "messages",
  "event": "created",
  "filter": "roomId=room-id-456",
  "orgId": "org-id",
  "createdBy": "person-id",
  "appId": "app-id",
  "ownedBy": "creator",
  "status": "active",
  "actorId": "person-who-triggered",
  "data": {
    "id": "msg-id",
    "roomId": "room-id",
    "personId": "person-id",
    "personEmail": "user@example.com",
    "created": "2026-02-09T10:00:00.000Z"
  }
}
```

Note: The webhook payload contains only metadata. To get the full message content, make a separate `GET /v1/messages/{id}` call.

### 12.7 Error Handling

| Code | Meaning | Retry |
|---|---|---|
| `400` | Bad request | Do not retry; fix parameters |
| `401` | Unauthorized | Refresh token, retry |
| `403` | Forbidden | Check scopes/permissions |
| `404` | Not found | Do not retry |
| `405` | Method not allowed | Fix HTTP method |
| `409` | Conflict | Retry after resolving conflict |
| `429` | Rate limited | Wait `Retry-After` seconds |
| `500` | Server error | Retry with backoff |
| `502` | Bad gateway | Retry after 1-5 seconds |
| `503` | Service unavailable | Retry with backoff |

### 12.8 Mobile-Specific Notes

- **Token Refresh:** Access tokens expire in 14 days, refresh tokens in 90 days. Implement proactive token refresh before expiry.
- **Webhook Payload:** Webhook payloads are intentionally minimal. Always make a follow-up API call to get full message/meeting details.
- **Markdown:** Webex supports Markdown in messages. Use the `markdown` field for formatted content.
- **Adaptive Cards:** Webex supports Adaptive Cards (same format as Microsoft Teams). Use for rich interactive messages.
- **File Sharing:** Files can be shared via URLs or direct upload. For mobile, use URL-based sharing when possible to avoid large uploads.
- **Meeting Integration:** Use the `webLink` from meeting creation to deep-link users into Webex meetings from the Esmer app.

---

## Cross-Service Comparison Matrix

### Authentication Methods

| Service | OAuth 2.0 | API Key / Token | Bot Token | Webhook | Service Account |
|---|:---:|:---:|:---:|:---:|:---:|
| Gmail | Yes | -- | -- | -- | Yes |
| Outlook | Yes | -- | -- | -- | -- |
| Slack | Yes | Yes | Yes | -- | -- |
| Teams | Yes | -- | -- | -- | -- |
| Telegram | -- | -- | Yes | -- | -- |
| WhatsApp | Yes | Yes | -- | -- | -- |
| Discord | Yes | -- | Yes | Yes | -- |
| Google Chat | Yes | -- | -- | Yes | Yes |
| Mattermost | -- | Yes | -- | Yes | -- |
| Matrix | -- | Yes | -- | -- | -- |
| Zulip | -- | Yes | -- | Yes | -- |
| Webex | Yes | -- | -- | -- | -- |

### Rate Limits Summary

| Service | Primary Limit | Message Send Limit | Daily Quota |
|---|---|---|---|
| Gmail | 250 units/sec/user | 100 units/send | 500-2000 emails/day |
| Outlook | 10K req/10min/tenant | Per-mailbox: 10K/10min | None explicit |
| Slack | Tiered (1-100+ rpm) | 1 msg/sec/channel | None explicit |
| Teams | 10K req/10min/tenant | 2 msg/sec/channel | None explicit |
| Telegram | 30 msg/sec global | 1 msg/sec/chat | None explicit |
| WhatsApp | 80 msg/sec default | 80 msg/sec | Tiered (250-unlimited contacts/24h) |
| Discord | 50 req/sec global | 5 msg/5sec/channel | None explicit |
| Google Chat | 6K req/min/project | 300 msg/min/space | None explicit |
| Mattermost | 10 req/sec (configurable) | Configurable | None (self-hosted) |
| Matrix | ~3 req/sec (configurable) | ~10 msg/sec (configurable) | None (self-hosted) |
| Zulip | 200 req/min/user | 12 msg/min/user | None explicit |
| Webex | Dynamic | ~1 msg/sec/room | None explicit |

### Supported Operations Matrix

| Operation | Gmail | Outlook | Slack | Teams | Telegram | WhatsApp | Discord | G.Chat | Mattermost | Matrix | Zulip | Webex |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Send message | Y | Y | Y | Y | Y | Y | Y | Y | Y | Y | Y | Y |
| Read messages | Y | Y | Y | Y | -- | -- | Y | Y | -- | Y | Y | Y |
| Reply/thread | Y | Y | Y | Y | Y | -- | Y | Y | Y | -- | Y | Y |
| Delete message | Y | Y | Y | -- | Y | -- | Y | Y | Y | -- | Y | Y |
| Edit message | -- | Y | Y | -- | Y | -- | Y | Y | -- | -- | Y | Y |
| Search | Y | Y | Y | -- | -- | -- | -- | -- | Y | -- | Y | -- |
| File upload | Y | Y | Y | -- | Y | Y | Y | -- | -- | Y | Y | Y |
| Reactions | -- | -- | Y | -- | -- | -- | Y | -- | Y | -- | -- | -- |
| User management | -- | -- | Y | -- | Y | -- | Y | -- | Y | Y | Y | Y |
| Channel/room mgmt | -- | Y | Y | Y | Y | -- | Y | Y | Y | Y | Y | Y |
| Webhooks/Push | Y | Y | Y | Y | Y | Y | Y | Y | Y | Y | Y | Y |
| Meetings | -- | Y | -- | -- | -- | -- | -- | -- | -- | -- | -- | Y |
| Contacts | -- | Y | -- | -- | -- | -- | -- | -- | -- | -- | -- | Y |
| Calendar | -- | Y | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- |

Legend: Y = Supported, -- = Not supported or not applicable

### Real-time / Push Mechanism

| Service | Mechanism | Renewal Required | Max Lifetime |
|---|---|---|---|
| Gmail | Pub/Sub push | Yes | 7 days |
| Outlook | Graph subscriptions | Yes | 3 days (mail) |
| Slack | Events API / Socket Mode | No (persistent) | N/A |
| Teams | Graph subscriptions | Yes | 60 minutes (chat) |
| Telegram | Webhook or long-poll | No (persistent) | N/A |
| WhatsApp | Meta webhooks | No (persistent) | N/A |
| Discord | Gateway WebSocket | Yes (heartbeat) | Session-based |
| Google Chat | HTTP push to bot URL | No (persistent) | N/A |
| Mattermost | WebSocket | Yes (reconnect) | Session-based |
| Matrix | Sync long-poll | Yes (continuous) | N/A |
| Zulip | Event queue long-poll | Yes (re-register on expiry) | Queue-based (10 min idle) |
| Webex | Webhooks | No (persistent) | N/A |

### Mobile Integration Complexity

| Service | Auth Complexity | SDK Available | Offline Support | Recommended Approach |
|---|---|---|---|---|
| Gmail | Medium (OAuth+PKCE) | Yes (Google) | Delta sync via history API | Server-side proxy with push notifications |
| Outlook | Medium (OAuth+PKCE) | Yes (MSAL) | Delta queries | Server-side proxy with push notifications |
| Slack | Low (OAuth) | Yes (Bolt) | Cache + polling | Bot token server-side; push to mobile |
| Teams | Medium (OAuth+PKCE) | Yes (MSAL) | Limited | Server-side proxy |
| Telegram | Low (token) | Yes (many) | Queue messages locally | Backend bot; push to mobile |
| WhatsApp | Low (API key) | No official SDK | Limited (24h window) | Backend proxy only |
| Discord | Low (bot token) | Yes (discord.js) | Cache messages | Backend bot; push to mobile |
| Google Chat | Medium (OAuth) | Yes (Google) | Limited | Service account server-side |
| Mattermost | Low (token) | Yes | WebSocket + cache | Direct API or backend proxy |
| Matrix | Low (login token) | Yes (matrix-sdk) | Sync API + local store | Client SDK recommended for E2EE |
| Zulip | Low (API key) | Yes (zulip-js) | Event queue + cache | Backend proxy or direct API |
| Webex | Medium (OAuth) | Yes (Webex SDK) | Limited | Server-side proxy |

---

*End of Category 01: Email & Communication Specification*
